# REVIEW.md

List every defect you found in the starter `app/main.py` (helpers and endpoints). For each one:

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | `lifespan` / Startup | No indexes were created at startup; only MongoDB's default `_id` index existed. | Explaining queries on ~100k records showed `COLLSCAN` and caused timeouts. | Created compound and unique indexes at startup inside an async lifespan context manager. |
| 2 | `compute_late_minutes` | Checked `minutes > 10` after integer floor division, masking seconds-level tardiness. | A punch at `09:40:01` for a `09:30` shift resulted in `0` late minutes instead of `10`. | Evaluated elapsed seconds strictly exceeding 600 (`delta_seconds > 600`), then floored to whole minutes. |
| 3 | `compute_work_hours` | Used Python's built-in `round()`, which applies banker's rounding (round-to-even) instead of half-up. | `2.125` rounded to `2.12` instead of `2.13`, failing business rule R4/R8. | Replaced with `decimal.Decimal` using `ROUND_HALF_UP`. |
| 4 | `create_employee` | Performed check-then-insert (`find_one` then `insert_one`), causing race conditions under parallel execution. | Concurrent requests with identical `emp_code` created duplicate documents. | Added a unique database index on `emp_code` and caught `DuplicateKeyError` to return HTTP 409. |
| 5 | `punch_in` | Computed `date` using naive local system time instead of IST calendar date with overnight shifts. | Overnight shifts (e.g. 22:00-06:00) punching at 04:00 IST received today's date instead of previous day's shift date. | Implemented `get_attendance_date` enforcing IST conversion and previous-day mapping per R1. |
| 6 | `punch_in` | Check-then-insert race condition on `(emp_code, date)`. | Concurrent punch-in calls created duplicate documents for the same day. | Created a unique compound index on `(emp_code, date)` and caught `DuplicateKeyError` returning HTTP 409. |
| 7 | `punch_in` / `list_attendance` | Leaked internal MongoDB `_id` as string `id` in JSON responses, violating OpenAPI contract. | Automated contract validation failed with unexpected `id` property. | Stripped `_id` and mapped records strictly to schema fields. |
| 8 | `list_employees` | Skip calculation was `page * page_size`, skipping the entire first page. | Calling `page=1` returned items 21–40 instead of items 1–20. | Corrected formula to `(page - 1) * page_size`. |
| 9 | `list_attendance` | Loaded entire dataset into server memory with `list(db.attendance_logs.find(q))` and sorted in Python. | Server memory exhaustion and latency timeouts on collections of ~100k records. | Delegated sort, skip, and limit directly to MongoDB index scans. |
| 10 | `punch_in` / Models | Allowed seconds-looking timestamps (< 100,000,000,000), violating instant format specs. | Supplying standard epoch seconds returned 201 instead of being rejected with 422. | Added integer boundaries `ge=100000000000, le=4102444800000` on epoch fields. |

### Examined and Verified as Non-Defects
* **String-based dates (`YYYY-MM-DD`) and times (`HH:MM`):** The data model deliberately avoids native BSON date conversions for calendar dates and shift times to prevent timezone drift. Keeping them as plain strings matches the contract and allows clean lexicographical range queries.
* **Storing derived values:** Derived fields (`work_hours`, `late_minutes`, `overtime_minutes`, `half_day`) must be materialized and stored during write operations rather than recomputed dynamically in analytics, preserving historic audit validity.