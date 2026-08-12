# MIF / REST Integration Troubleshooting Note

## Scenario
An external CAFM vendor system pushes Service Request creation requests into Maximo via a
REST endpoint backed by the Maximo Integration Framework (MIF). A subset of requests began
failing with malformed payload errors after the vendor updated their system.

## Sample failing request (simplified)

```json
POST /maximo/oslc/os/mxsr
Content-Type: application/json

{
  "description": "AC unit not cooling - Floor 3",
  "reportedby": "VENDOR_SYS",
  "location": "BLDG-A-F3",
  "priority": "2",
  "affecteddate": "2026-08-10"
}
```

## Error returned

```json
{
  "Error": {
    "reasonCode": "BMXAA4021E",
    "message": "BMXAA4021E - Value 2 is not valid for field PRIORITY.
                Value is not in the domain WOPRIORITY."
  }
}
```

## Diagnosis

The `priority` field was sent as a **string** `"2"`, but more importantly the value itself
had never been part of the `WOPRIORITY` domain used by this org — this org's Service Request
priority domain only had values 1, 3, 5. The vendor's system had its own internal 1-5 scale
that didn't map 1:1 onto Maximo's configured domain.

This is a very common integration failure mode: the payload is syntactically valid JSON, and
the field names are correct, but the *domain values* don't match what's configured on the
Maximo side. It shows up as a MIF/OSLC validation error rather than a connectivity or auth
error, which is often the first clue it's a domain/config mismatch rather than a network or
credentials issue.

## Fix

1. Pulled the actual configured domain values:
   ```sql
   SELECT domainid, value, description
   FROM synonymdomain
   WHERE domainid = 'WOPRIORITY';
   ```
2. Confirmed with the vendor which of their internal priority levels should map to which
   Maximo value.
3. Added a translation step in the integration's inbound processing (a crossover/mapping
   table) so vendor priority `2` maps to Maximo priority `3`, rather than asking the vendor
   to change their internal scale.

## Preventive measure

Recommended that any new inbound integration go through a domain-value mapping review as a
required step before go-live — not just a field-name mapping review — since field-name
mismatches throw obvious errors immediately, but domain-value mismatches can pass initial
testing if the test data happens to use compatible values, then fail later on edge cases.
