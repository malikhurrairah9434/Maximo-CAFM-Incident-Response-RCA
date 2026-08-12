# SQL Diagnostics Pack — IBM Maximo

A reference set of queries against core Maximo tables, organized by the scenario an L2/L3
engineer would use them for. Written for Oracle syntax; SQL Server equivalents noted where
syntax differs materially.

## 1. Work Order health checks

```sql
-- Work orders stuck in a status longer than expected (SLA breach risk)
SELECT wonum, status, statusdate, worktype, siteid,
       SYSDATE - statusdate AS days_in_status
FROM workorder
WHERE status IN ('WAPPR','WSCH','INPRG')
  AND SYSDATE - statusdate > 2
ORDER BY days_in_status DESC;

-- Work orders with no active workflow assignment (orphaned)
SELECT w.wonum, w.status
FROM workorder w
LEFT JOIN wfassignment wf ON w.wfinstanceid = wf.processinstanceid
WHERE w.status = 'WAPPR' AND wf.assignmentid IS NULL;
```

## 2. Preventive Maintenance integrity

```sql
-- PM records pointing to an inactive/revised job plan
SELECT p.pmnum, p.assetnum, p.jpnum, j.status AS jobplan_status
FROM pm p
JOIN jobplan j ON p.jpnum = j.jpnum
WHERE p.status = 'ACTIVE' AND j.status <> 'ACTIVE';

-- PMs overdue for generation
SELECT pmnum, assetnum, nextdate, frequency
FROM pm
WHERE status = 'ACTIVE' AND nextdate < SYSDATE;
```

## 3. Integration / MIF monitoring

```sql
-- Failed interface transactions in the last 24 hours, by interface
SELECT intfacename, COUNT(*) AS error_count
FROM maxintfile
WHERE status = 'ERROR' AND transdate > SYSDATE - 1
GROUP BY intfacename
ORDER BY error_count DESC;

-- Full error detail for a specific failing interface
SELECT transid, intfacename, extsysname, transdate, message
FROM maxintfile
WHERE intfacename = :interface_name AND status = 'ERROR'
ORDER BY transdate DESC;
```

**SQL Server note:** replace `SYSDATE` with `GETDATE()`.

## 4. Cron task health

```sql
-- Any cron task instance that's inactive but shouldn't be
SELECT cronjobname, instancename, active, laststartdate, lastrunstatus
FROM crontaskinstance
WHERE active = 0
ORDER BY cronjobname;

-- Cron tasks that haven't run recently (possible silent failure)
SELECT cronjobname, instancename, laststartdate
FROM crontaskinstance
WHERE active = 1
  AND laststartdate < SYSDATE - 1;
```

## 5. Inventory data-quality checks

```sql
-- Items with a max level set but no reorder point configured (likely migration gap)
SELECT itemnum, location, minlevel, maxlevel, curbal
FROM invbalances
WHERE maxlevel > 0 AND minlevel = 0;

-- Items currently below reorder point with no open PR/PO
SELECT i.itemnum, i.location, i.curbal, i.minlevel
FROM invbalances i
WHERE i.curbal < i.minlevel
  AND NOT EXISTS (
    SELECT 1 FROM poline pl WHERE pl.itemnum = i.itemnum AND pl.status IN ('WAPPR','APPR')
  );
```

## 6. General diagnostic habits worth having on hand

- Always check `status` + `statusdate` together — status alone doesn't tell you how long
  something's been stuck.
- For any "nothing happened" ticket, check the relevant cron task before assuming
  application/workflow logic is broken — a disabled or silently-failed cron task is a very
  common root cause and is often overlooked in favor of debugging workflow XML first.
- For integration issues, `MAXINTFILE` (or `MAXINTFILE`-equivalent error tables depending on
  version) is almost always the fastest place to get the actual error message rather than
  digging through application server logs first.
