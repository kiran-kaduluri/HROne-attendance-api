# Employee Attendance & Analytics API

FastAPI and MongoDB backend service for employee attendance tracking, regularization audit trails, and reporting analytics built to satisfy OpenAPI contract v2.0.

---

## Tech Stack
- **Framework:** FastAPI
- **Database Driver:** PyMongo
- **Database Engine:** MongoDB 6.0+ (Tested on MongoDB 7.0)
- **Runtime:** Python 3.11+

---

## Prerequisites
- Python 3.11 or newer
- MongoDB running locally on port `27017` (e.g., via Docker) or an active MongoDB Atlas cluster

---

## Local Setup & Run

1. **Clone the repository and navigate into the directory:**
   ```bash
   git clone <your-repo-link>
   cd hrone-attendance-api
   ```

2. **Create and activate a virtual environment:**
   ```bash
   python -m venv .venv
   source .venv/bin/activate       # On Windows: .venv\Scripts\activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Database Configuration:**
   The service reads `MONGO_URI` and `MONGO_DB` from environment variables, defaulting to:
   - `MONGO_URI`: `mongodb://localhost:27017`
   - `MONGO_DB`: `attendance_db`

   *(Optional: If running MongoDB via Docker)*
   ```bash
   docker run -d -p 27017:27017 --name hrone-mongo mongo:7
   ```

5. **Seed Sample Data (Optional):**
   ```bash
   python sample_seed.py
   ```

6. **Start the API:**
   ```bash
   uvicorn app.main:app --port 8000 --reload
   ```

---

## API Documentation

Once the server is running, interactive documentation is available at:
- **Swagger UI:** [http://localhost:8000/docs](http://localhost:8000/docs)
- **OpenAPI Spec:** [http://localhost:8000/openapi.json](http://localhost:8000/openapi.json)

---

## Architecture & Implementation Details

### 1. Startup Indexing (`lifespan`)
- **Integrity Constraints:**
  - Unique index on `employees.emp_code`
  - Compound unique index on `attendance_logs.(emp_code, date)` to enforce data integrity and prevent check-then-insert race conditions.
- **Query Performance:**
  - Sorting and range indexes on `(date, emp_code)`, `(emp_code, punch_in)`, and `(date, status)` to guarantee `IXSCAN` operations on high-volume datasets (~100k+ records).

### 2. Business Rules Enforcement (R1–R10)
- **R1 (IST & Overnight):** Dates are resolved in Indian Standard Time (UTC+05:30). Night shifts spanning midnight evaluate early-morning punches against the preceding calendar shift date.
- **R2 (Late Arrival):** 10-minute grace window evaluated in whole seconds; arrivals exceeding grace compute elapsed floor minutes from scheduled shift start.
- **R3 (Overtime):** Floor minutes between shift end and punch-out, counted only if 30 minutes or greater.
- **R4 & R8 (Work Hours & Precision):** Decimal arithmetic using round half-up (`ROUND_HALF_UP`) to two decimal places (rates to four decimal places).
- **R5 & R6 (Presence & Half-Day):** Rounded work hours strictly under 4.50 hours automatically mark `half_day = True`.
- **R7 & R9 (Calendar & Headcount):** Monday through Friday working calendar incorporating mid-month hire dates (`joined_on`). Aggregations preserve employees with zero log entries in departmental totals.

### 3. Atomic State Updates & Audit Log
- **Punch-Out:** `POST /attendance/punch-out` performs atomic document updates via `find_one_and_update` with `punch_out: None` guarding against parallel close events.
- **Regularization:** `PATCH /attendance/{emp_code}/{date}` captures field-level change deltas into an immutable history array, guarding against lost updates via array length pre-conditions.

### 4. Database Aggregation Pipelines
- **Monthly Summary:** Aggregates present days, leave records, and tardiness totals natively per employee.
- **Department Summary:** Cross-collection `$lookup` rooted on employees preserving complete headcount regardless of log presence.
- **Late Leaderboard:** Implements standard competition ranking (`1, 2, 2, 4`) using MongoDB `$setWindowFields` and `$rank`.
- **Daily Trend:** Continuous time-series generation with gap-filling using MongoDB `$densify` and rolling 7-day window moving averages.

### 5. Admin Query Introspection
- `/admin/explain/{endpoint}` exposes query execution plans and verifies index utilization (`IXSCAN`).