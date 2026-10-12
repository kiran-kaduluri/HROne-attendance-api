# DECISIONS.md

1. **Indexes.**
   Created `employees`: `{emp_code: 1}` (unique, guarantees primary lookup and race safety), `{department: 1, emp_code: 1}` (covers filtered pagination), and `{department: 1, joined_on: 1}` (optimizes headcount filtering).
   Created `attendance_logs`: `{(emp_code, date): 1}` (unique, enforces single record per day and prevents punch-in races), `{(date: -1, emp_code: 1)}` (powers index scan for `/attendance`), `{(emp_code: 1, punch_in: -1)}` (optimizes latest punch-in lookup during punch-out), and `{(date: 1, status: 1)}` (accelerates monthly summary aggregations).
   Rejected `{emp_code: 1, date: 1, status: 1, work_hours: 1}`: adding large covering indexes on mutable fields introduced excessive index write overhead without query performance gains.

2. **Punch-in race.**
   Both requests validate the employee and calculate the IST attendance date concurrently. Both attempt an `insert_one` with identical `(emp_code, date)`. The unique compound index on `attendance_logs` ensures MongoDB's internal locking allows exactly one write to succeed. The winner gets HTTP 201. The second write triggers `pymongo.errors.DuplicateKeyError`, caught by our exception handler and returned as HTTP 409.

3. **Ties.**
   The aggregation pipeline uses MongoDB's `$rank` window function within `$setWindowFields` partitioned by month, implementing standard competition ranking (e.g., ranks 1, 2, 2, 4). Filtering is applied post-ranking with `rank <= limit`. If a tie occurs at the threshold rank, all tied records are retained and returned, sorted by `total_late_minutes` descending and `emp_code` ascending.

4. **Headcount.**
   The aggregation roots on `employees` filtered by `joined_on <= month_end`, ensuring all valid employees are matched regardless of attendance. A `$lookup` brings in matching monthly attendance logs, followed by an `$unwind` with `preserveNullAndEmptyArrays: True`. Grouping by department counts unique employee IDs for headcount, including employees with zero attendance documents.

5. **One thing you would change** if this had to serve 100x the data.
   I would shard `attendance_logs` using a hashed compound key on `{emp_code: "hashed", date: 1}` to evenly distribute write traffic, and maintain pre-aggregated daily/monthly department counters updated atomically via change streams rather than executing full aggregations on read.