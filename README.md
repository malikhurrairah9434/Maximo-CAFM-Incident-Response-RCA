# CAFM / IBM Maximo Application Support

**Note on scope:** These are simulated incidents, written the way real Maximo support tickets
and RCAs are written, based on the actual Maximo data model, MIF architecture, and standard
troubleshooting workflow. They are not pulled from a live production system (no NDA'd or
employer data is used) — they're built to demonstrate the diagnostic reasoning, SQL fluency,
and documentation discipline the role requires.

## Contents

| File | What it shows |
|---|---|
| `incident-log.md` | 6 simulated production tickets across Work Orders, PM, Inventory, MIF integration, and cron tasks — each with symptom, diagnosis steps, SQL used, root cause, and fix |
| `sql-diagnostics.md` | A reference pack of SQL queries against core Maximo tables (WORKORDER, ASSET, MAXINTFILE, CRONTASKINSTANCE, etc.) for common diagnostic scenarios |
| `rca-report.pdf` | One incident taken to a full formal Root Cause Analysis document, in the format you'd actually submit to a bank's change/incident management process |
| `mif-troubleshooting-note.md` | A focused writeup on diagnosing a failed MIF/REST integration transaction, including a sample error payload and fix |


