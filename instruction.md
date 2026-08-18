Build a deterministic ETL pipeline that produces three artifacts under `/app/output/`.

Input files (already present):
- `/app/data/users_raw.csv` – initial user snapshot + change events
- `/app/data/events_raw.csv` – fact events (may arrive out-of-order and late)

1. Create `/app/output/users_dim.db` containing a single table `users_dim` that implements SCD Type 2 for users:
   - Columns (exact order and types): user_id INTEGER, name TEXT, plan TEXT, effective_from TEXT NOT NULL, effective_to TEXT, is_current INTEGER NOT NULL
   - Primary key is the combination (user_id, effective_from)
   - For every change a new version row is inserted; the previous version’s effective_to is set to the change timestamp and is_current becomes 0
   - The latest version of each user has effective_to = NULL and is_current = 1
   - effective_from / effective_to are ISO-8601 strings (YYYY-MM-DDTHH:MM:SS)
   - Initial load uses the earliest timestamp present for that user as effective_from

2. Create `/app/output/events_fact.db` containing a single table `events_fact`:
   - Columns: event_id INTEGER PRIMARY KEY, user_id INTEGER NOT NULL, event_ts TEXT NOT NULL, amount REAL NOT NULL, plan_at_event TEXT, is_late INTEGER NOT NULL
   - Join each event to the correct SCD version of the user that was active at event_ts (effective_from <= event_ts AND (effective_to IS NULL OR event_ts < effective_to))
   - A watermark of exactly 2 hours is applied relative to the maximum event_ts present in the whole file. Any event whose event_ts is more than 2 hours earlier than that global maximum is marked is_late = 1; all others is_late = 0
   - Late events are still stored but must be flagged

3. Create `/app/output/quality_report.json` with exactly these keys and types:
   {
     "total_users": <int>,
     "total_scd_versions": <int>,
     "current_users": <int>,
     "total_events": <int>,
     "late_events": <int>,
     "events_with_plan": <int>,
     "sum_amount_current_plan": <float>
   }
   - sum_amount_current_plan is the sum of amount for events whose plan_at_event equals the user’s current plan (is_current = 1)

All timestamps are compared as strings in ISO-8601 order. The three output files must exist and satisfy the schema and numeric values exactly.
