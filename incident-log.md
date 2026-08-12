# Incident Log — Simulated Maximo CAFM Production Support Tickets

Symptom → Diagnostic Steps → SQL/Evidence → Root Cause → Fix → Preventive Measure.

---

## INC-001 — Work Orders stuck in "WAPPR" status, not escalating to supervisor

**Module:** Work Orders / Workflow
**Reported by:** Facilities helpdesk (multiple WOs, same building)
**Priority:** High (SLA breach risk)

**Symptom:** ~40 work orders created via Service Request self-service were stuck in WAPPR
(Waiting for Approval) for over 48 hours despite the workflow's escalation SLA being 4 hours.

**Diagnostic steps:**
1. Confirmed the affected WOs all belonged to one Location hierarchy (Building C).
2. Checked whether the WORKORDER workflow instance was active or orphaned.
3. Checked the ESCALATION and ESCCONDITION setup tied to the workflow node.
4. Checked whether the escalation cron task was actually running.

**SQL used:**
```sql
-- Confirm WOs stuck and how long
SELECT wonum, status, statusdate, location, siteid,
       SYSDATE - statusdate AS days_stuck
FROM workorder
WHERE status = 'WAPPR'
  AND location LIKE 'BLDG-C%'
ORDER BY statusdate;

-- Check the escalation config tied to this workflow
SELECT escalationname, active, schedule, appliesto
FROM escalation
WHERE appliesto = 'WORKORDER'
  AND active = 1;

-- Confirm the escalation cron task instance is actually enabled/running
SELECT cronjobname, instancename, active, lastrunstatus, laststartdate
FROM crontaskinstance
WHERE cronjobname = 'ESCALATION';
```

**Root cause:** The `ESCALATION` cron task instance for that org had `active = 0` — it had
been disabled during a prior maintenance window and never re-enabled. No escalations were
firing for any workflow instance org-wide, not just Building C; it just hadn't been noticed
elsewhere yet because other queues were being cleared manually.

**Fix:** Re-enabled the cron task instance, forced an immediate run, confirmed the backlog of
WOs escalated correctly on the next cycle.

**Preventive measure:** Added the escalation cron task to the daily automated health-check
script (see `sql-diagnostics.md`) so a disabled critical cron task triggers an alert instead
of relying on someone noticing a backlog.

---

## INC-002 — Preventive Maintenance work orders not generating for a set of assets

**Module:** Preventive Maintenance / Assets
**Priority:** Medium

**Symptom:** A batch of HVAC assets stopped generating scheduled PM work orders after a
Preventive Maintenance record update.

**Diagnostic steps:**
1. Checked whether the PM records were still active and correctly linked to the asset/location.
2. Checked the `frequency`, `nextdate`, and `seqnum` fields on the PM record.
3. Checked whether the PM's job plan (JPNUM) reference was still valid.
4. Reviewed the PM generation cron task run history for errors.

**SQL used:**
```sql
SELECT pmnum, assetnum, status, frequency, nextdate, jpnum
FROM pm
WHERE assetnum IN (SELECT assetnum FROM asset WHERE classstructureid LIKE 'HVAC%')
  AND status <> 'ACTIVE';

-- Confirm the linked job plan wasn't deactivated
SELECT jpnum, status FROM jobplan WHERE jpnum IN
  (SELECT jpnum FROM pm WHERE assetnum IN
     (SELECT assetnum FROM asset WHERE classstructureid LIKE 'HVAC%'));
```

**Root cause:** A bulk job plan revision (new JPNUM created, old one set to status `PENDREV`)
had not been re-linked to the affected PM records — they were still pointing to the deprecated
job plan, which silently blocked generation since it was no longer in ACTIVE status.

**Fix:** Bulk-updated affected PM records to point to the revised, active job plan; manually
triggered PM generation cron to catch up the missed cycle.

**Preventive measure:** Recommended job plan revisions go through a checklist step to confirm
all PM records referencing the old JPNUM are re-pointed before the old one is deactivated.

---

## INC-003 — MIF interface failing to push completed Work Orders to the finance system

**Module:** Integration (MIF) / Work Orders
**Priority:** Critical (financial reconciliation impact)

**Symptom:** Completed work order cost data was not appearing in the downstream finance
system. No error visible to end users — discovered during month-end reconciliation.

**Diagnostic steps:**
1. Checked the outbound interface queue table for failed/pending transactions.
2. Pulled the actual error message from the interface error table.
3. Cross-checked against the external endpoint's expected schema.

**SQL used:**
```sql
SELECT intfacename, status, transdate, extsysname, message
FROM maxintfile
WHERE status = 'ERROR'
  AND intfacename = 'WOCOST_OUT'
ORDER BY transdate DESC;
```

**Root cause:** The external finance system had changed a field's data type (a cost center
code moved from numeric to alphanumeric) without notifying the integration team. The MIF
outbound processing rule was still validating against the old numeric format, so every
transaction after the change silently failed and queued as ERROR status instead of
completing.

**Fix:** Updated the MIF processing rule's field validation, reprocessed the queued failed
transactions, verified totals reconciled in finance system after replay.

**Preventive measure:** Proposed a lightweight interface health dashboard querying
`MAXINTFILE` status counts by interface name, refreshed hourly, with alerting on any ERROR
count above zero — rather than discovering failures at month-end.

---

## INC-004 — Inventory reorder point not triggering purchase requisitions

**Module:** Inventory
**Priority:** Medium

**Symptom:** Critical spare parts (HVAC filters) ran out at a site despite reorder-point
automation supposedly being configured.

**Diagnostic steps:**
1. Checked the item's reorder point, min/max, and current balance at that storeroom.
2. Checked whether the reorder-point cron task ran and what it processed.

**SQL used:**
```sql
SELECT itemnum, location, curbal, minlevel, maxlevel, orderunit
FROM invbalances
WHERE itemnum = 'HVAC-FILT-24X24'
  AND location = 'STR-RIY-01';

SELECT cronjobname, laststartdate, lastrunstatus
FROM crontaskinstance
WHERE cronjobname = 'REORDER';
```

**Root cause:** The `minlevel` field for that item at that storeroom was set to 0 — it had
never actually been configured, only the maxlevel was set during initial data load. The
reorder cron ran fine; it just had nothing to trigger against.

**Fix:** Set correct min/max levels based on historical usage; manually raised an emergency
PR for the immediate shortage.

**Preventive measure:** Flagged this as a data-quality gap from original go-live data
migration — recommended a validation query (included in `sql-diagnostics.md`) to catch any
inventory item with `maxlevel > 0` but `minlevel = 0`, which is very likely a missed config,
not an intentional setting.

---

## INC-005 — REST API integration returning HTTP 401 intermittently

**Module:** Integration (REST) / Authentication
**Priority:** High

**Symptom:** A third-party vendor's REST calls into Maximo to create Service Requests were
failing intermittently with 401 Unauthorized, roughly once every few hours.

**Diagnostic steps:**
1. Reviewed the app server / MIF REST log around failure timestamps.
2. Checked the service account's password/token expiry policy.
3. Correlated failure times against the org's security/session timeout settings.

**Root cause:** The integration used a maxauth service account subject to the same
password-expiry and session-timeout policy as interactive users. Under load, concurrent
sessions from the same service account were hitting the max concurrent session limit,
causing the OSLC/REST layer to reject the extra session with a 401 rather than queuing it.

**Fix:** Worked with the security team to exempt the service account from session-count
limits (standard practice for integration accounts) and confirmed with the vendor that their
client was reusing a single persistent token rather than opening a new session per call.

**Preventive measure:** Documented the requirement that all new integration/service accounts
be provisioned with non-expiring, non-interactive session policies from day one — added to
the integration onboarding checklist.

---

## INC-006 — Scheduled cron task silently stopped running after a Maximo patch

**Module:** System Administration / Cron Tasks
**Priority:** Medium

**Symptom:** No new automatic Service Request notifications were being sent after a routine
patch deployment over the weekend.

**Diagnostic steps:**
1. Checked `crontaskinstance` for the relevant notification cron task's last run status.
2. Reviewed the application server logs around the patch deployment window for startup errors.

**SQL used:**
```sql
SELECT cronjobname, instancename, active, laststartdate, lastrunstatus
FROM crontaskinstance
WHERE cronjobname = 'SRNOTIFY';
```

**Root cause:** The patch redeployment reset the cron task's `active` flag to its default
(inactive) because the deployment scripts did a fresh redeploy of the cron task XML config
rather than an in-place update, overwriting the customized "active" state.

**Fix:** Re-enabled the cron task instance and manually triggered a catch-up run for the
missed notification window.

**Preventive measure:** Added a post-deployment checklist step: verify all customized cron
task instances retain their pre-deployment active/inactive state before sign-off, since
patch deployments are a recurring source of this exact issue.
