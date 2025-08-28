
# Snowflake — One‑Stop Cheat Sheet

> **Goal:** a compact but comprehensive reference covering core concepts, architecture, commands, patterns, performance, security, cost control, and common examples. Use this as a quick daily reference or learning path.

---

## Quick Facts
- **What:** Cloud-native data warehouse (multi-cloud: AWS, Azure, GCP).
- **Core idea:** **Separation of storage and compute** — storage is shared, compute lives in virtual warehouses.
- **Data types:** Structured and semi-structured (`VARIANT`, `OBJECT`, `ARRAY`).
- **Special features:** Time Travel, Zero-copy Cloning, Secure Data Sharing, Snowpipe (continuous ingest), Streams & Tasks (CDC & scheduled jobs).

---

## Architecture (short)
- **Account** → top-level container (organizations may have multiple accounts).
- **Region/Cloud** → where the account resides (AWS/Azure/GCP region).
- **Virtual Warehouse** → compute resources; size options XS → 6XL and multi-cluster for concurrency.
- **Storage** → managed by Snowflake (S3/GCS/Azure Blob behind the scenes).
- **Database → Schema → Objects** → tables, views, stages, file formats, pipes, tasks, streams.

---

## Common Objects
- **Databases, Schemas**
- **Tables** — permanent, transient, temporary
- **Views** — standard & secure views
- **Materialized views**
- **Stages** — internal/external (S3/GCS/Azure)
- **File formats**
- **Pipes & Snowpipe**
- **Streams** — change tracking
- **Tasks** — scheduled SQL execution
- **Functions** — SQL UDFs, JavaScript stored procedures, external functions
- **Shares** — data sharing to other Snowflake accounts

---

## Account & Role Basics
```sql
-- switch context
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE compute_wh;
USE DATABASE analytics_db;
USE SCHEMA public;

-- create role and grant
CREATE ROLE data_eng;
GRANT ROLE data_eng TO USER alice;
GRANT USAGE ON DATABASE analytics_db TO ROLE data_eng;
GRANT USAGE ON SCHEMA analytics_db.public TO ROLE data_eng;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics_db.public TO ROLE data_eng;
```

### Essential SHOW / INFORMATION_SCHEMA queries
```sql
SHOW WAREHOUSES;
SHOW USERS;
SHOW ROLES;
SHOW DATABASES;
SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='PUBLIC';
```

---

## Warehouses (compute)
- Create and tune warehouses:
```sql
CREATE WAREHOUSE wh_small
  WITH WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  MIN_CLUSTER = 1
  MAX_CLUSTER = 1;
```
- **Auto-suspend** to save credits; **auto-resume** for convenience.
- Use **multi-cluster warehouses** for high concurrency: `MIN_CLUSTER`, `MAX_CLUSTER`.
- **Suspend** when not in use: `ALTER WAREHOUSE wh_small SUSPEND;`
- **Cache layers:** result cache, warehouse cache, remote disk cache — use small warehouses when queries are cached.

---

## Tables & Loading

### Create table
```sql
CREATE OR REPLACE TABLE customers (
  customer_id NUMBER,
  email STRING,
  created_at TIMESTAMP_LTZ,
  profile VARIANT -- semi-structured JSON
);
```

### File formats & stages
```sql
CREATE FILE FORMAT csv_fmt TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1;

CREATE STAGE my_stage
  FILE_FORMAT = csv_fmt
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID='...' AWS_SECRET_KEY='...');
-- or create INTERNAL stage:
CREATE STAGE my_int_stage FILE_FORMAT = csv_fmt;
```

### Load data (COPY INTO)
```sql
COPY INTO customers
FROM @my_int_stage/customers.csv
FILE_FORMAT = (FORMAT_NAME = 'csv_fmt')
ON_ERROR = 'CONTINUE';
```

### Unload data
```sql
COPY INTO @my_int_stage/export/customers_ FROM (SELECT * FROM customers)
FILE_FORMAT = (TYPE = CSV);
```

---

## Semi‑structured Data (JSON / VARIANT)
```sql
-- insert JSON into VARIANT
INSERT INTO users (profile) VALUES (PARSE_JSON('{"name":"Alice","age":30,"tags":["a","b"]}'));

-- query nested fields
SELECT profile:name::string as name, profile:age::int as age FROM users;

-- flatten arrays
SELECT f.value
FROM users, LATERAL FLATTEN(input => users.profile:tags) f;
```

Useful functions: `PARSE_JSON`, `TO_VARIANT`, `OBJECT_INSERT`, `OBJECT_DELETE`, `ARRAY_SIZE`, `FLATTEN`.

---

## Time Travel & Fail-safe
- **Time Travel**: query historical data up to the account's retention period.
```sql
SELECT * FROM customers AT (TIMESTAMP => '2025-08-01 00:00:00');
SELECT * FROM customers BEFORE (STATEMENT => '12345'); -- statement id
```
- **Clone** (zero-copy):
```sql
CREATE TABLE customers_clone CLONE customers AT (OFFSET => -3600); -- clone from 1 hour ago
```
- **Fail-safe**: 7-day additional recovery period managed by Snowflake (not user-controlled).

---

## Zero-Copy Clone & Time Travel Use Cases
- Create dev copies without extra storage.
- Fast testing of schema changes.
- Recover accidentally dropped objects.

```sql
CREATE DATABASE db_clone CLONE db_prod;
```

---

## Streams & Tasks (CDC / Scheduling)
- **Stream**: track DML changes.
```sql
CREATE STREAM customers_stream ON TABLE customers;
-- read changes
SELECT * FROM customers_stream;
```
- **Task**: schedule SQL to run (can be chained).
```sql
CREATE OR REPLACE TASK daily_agg
  WAREHOUSE = wh_small
  SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
  INSERT INTO daily_customer_agg SELECT ... FROM customers;
ALTER TASK daily_agg RESUME;
```
- Combine stream+task for near-real-time ETL.

---

## Snowpipe (continuous ingestion)
- Auto-load files from cloud storage using notifications.
- Create pipe:
```sql
CREATE PIPE my_pipe
  AS COPY INTO customers FROM @my_stage
  FILE_FORMAT = (FORMAT_NAME = 'csv_fmt');
-- then configure cloud events for S3/Azure to notify Snowpipe
```

---

## Query Performance Tuning
- Right-size warehouses; use multi-cluster for concurrency.
- Use clustering keys for very large tables (not tiny tables).
```sql
ALTER TABLE events CLUSTER BY (event_date, user_id);
```
- Use `RESULT_CACHE`, `WAREHOUSE_CACHE` — repeated queries are fast and cheap.
- Use `MATERIALIZED VIEW` for expensive aggregations with high read frequency:
```sql
CREATE MATERIALIZED VIEW mv_user_counts AS
SELECT user_id, COUNT(*) as cnt FROM events GROUP BY user_id;
```
- Use `EXPLAIN` and `QUERY_HISTORY` to troubleshoot.
```sql
EXPLAIN SELECT ...;
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION());
```
- Avoid `SELECT *` in production queries.

---

## Security & Access Control
- Use least-privilege roles and RBAC.
- Role hierarchy example:
  - `ACCOUNTADMIN` (top)
  - `SECURITYADMIN` (manage users/roles)
  - `SYSADMIN` (create DB/Warehouse)
  - `PUBLIC` (default)
- Masking policies (dynamic data masking):
```sql
CREATE MASKING POLICY mask_email AS (val string) RETURNS string ->
  CASE WHEN CURRENT_ROLE() IN ('DATA_SCIENCE') THEN val
       ELSE '***@***' END;

ALTER TABLE customers MODIFY COLUMN email SET MASKING POLICY mask_email;
```
- Row Access Policies for row-level security.
- Use **Client-side** and **Server-side encryption** is automatic; manage keys with Tri-Secret (customer-managed keys) if needed.
- Enable multi-factor auth (MFA) and SSO (SAML).

---

## Data Sharing & Marketplace
- **Secure Shares** share data without copying:
```sql
CREATE SHARE marketing_share;
GRANT USAGE ON DATABASE marketing_db TO SHARE marketing_share;
GRANT SELECT ON ALL TABLES IN SCHEMA marketing_db.public TO SHARE marketing_share;
ALTER SHARE marketing_share ADD ACCOUNTS = ('ACCOUNT1');
```
- **Data Marketplace**: acquire shared datasets.

---

## Stored Procedures & UDFs
- JavaScript stored procedure example:
```sql
CREATE OR REPLACE PROCEDURE add_user(u VARIANT)
  RETURNS STRING
  LANGUAGE JAVASCRIPT
AS
$$
  var sql = "INSERT INTO users(profile) VALUES(?)";
  var stmt = snowflake.createStatement({sqlText: sql, binds:[u]});
  stmt.execute();
  return 'ok';
$$;
```
- SQL UDF example:
```sql
CREATE FUNCTION safe_divide(a FLOAT, b FLOAT)
RETURNS FLOAT
AS (CASE WHEN b=0 THEN NULL ELSE a/b END);
```
- External functions: call external services (AWS Lambda, etc.) securely.

---

## Useful ADMIN Queries
```sql
-- Query credits used last 7 days
SELECT start_time, warehouse_name, credits_used
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP());

-- Check long-running queries
SELECT query_id, user_name, start_time, total_elapsed_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE total_elapsed_time > 600000;

-- Table storage size
SELECT table_catalog, table_schema, table_name, bytes
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY bytes DESC LIMIT 20;
```

---

## Cost Control
- Set `AUTO_SUSPEND` and low `AUTO_RESUME` limits.
- Use `RESOURCE MONITORS` to cap credits:
```sql
CREATE RESOURCE MONITOR rm_quota WITH CREDIT_QUOTA = 100;
ALTER WAREHOUSE wh_small SET RESOURCE_MONITOR = rm_quota;
```
- Monitor query history and warehouse metering.
- Use transient tables for temporary data (no Fail-safe).
- Use result caching to reduce compute.

---

## Backup / Recovery Patterns
- Use Time Travel + Clone to recover or branch data.
- Example: recover dropped table
```sql
UNDROP TABLE customers;
-- or recover from time travel
CREATE TABLE customers_recovered CLONE customers BEFORE (TIMESTAMP => '2025-08-01 10:00:00');
```

---

## Common Gotchas & Troubleshooting
- **Large COPY failures**: inspect `VALIDATION_MODE` and `LOAD_HISTORY`.
- **Concurrency spikes**: enable multi-cluster or queue jobs.
- **Unexpected cost**: check warehouses left running; check auto-resume behavior when many tasks run.
- **Schema drift** with semi-structured data: define explicit VARIANT checks or views.

---

## Best Practices (short list)
1. Use small warehouses + auto-suspend; right-size for workloads.
2. Use roles & least privilege.
3. Use Time Travel & cloning for safe development/testing.
4. Keep file loads idempotent (use COPY options to avoid duplicates).
5. Prefer `COPY INTO` from staged files, use Snowpipe for streaming loads.
6. Use Streams + Tasks for incremental ETL.
7. Monitor with ACCOUNT_USAGE and INFORMATION_SCHEMA views.
8. Document schema changes and version control DDL (Terraform, Liquibase).
9. Use clustering keys thoughtfully and only on large tables.
10. Use search optimization service if needed for low-latency point queries.

---

## Quick Reference (cheat commands)
```sql
-- Create essentials
CREATE DATABASE db; CREATE SCHEMA db.public; CREATE WAREHOUSE wh;

-- User & role
CREATE USER bob PASSWORD='...';
CREATE ROLE analyst; GRANT ROLE analyst TO USER bob;

-- Table operations
CREATE TABLE t (id INT);
ALTER TABLE t ADD COLUMN c STRING;
DROP TABLE t;

-- Load/unload
PUT file://local.csv @%t;
COPY INTO t FROM @%t FILE_FORMAT=(TYPE='CSV');

-- Time travel & clone
SELECT * FROM t AT (OFFSET => -3600);
CREATE TABLE t_clone CLONE t;

-- Streams/Tasks
CREATE STREAM s ON TABLE t;
CREATE TASK tk WAREHOUSE=wh SCHEDULE='USING CRON 0 3 * * * UTC' AS INSERT...
ALTER TASK tk RESUME;
```

---

## Learning Path (if you're new)
1. Understand accounts, warehouses, databases/schemas.
2. Load and query sample data (CSV/JSON) using stages and COPY.
3. Practice VARIANT & FLATTEN with JSON datasets.
4. Build simple ETL with COPY and stored procedures.
5. Add Streams + Tasks for incremental loads.
6. Learn Time Travel + Cloning for safe testing.
7. Study RBAC, masking policies, and row access for security.
8. Automate deployments with Terraform / Snowflake Terraform provider.

---

## FAQ (short)
- **How do I limit cost?** Auto-suspend, resource monitors, small warehouses, use result cache.
- **Is Snowflake multi-cloud?** Yes — available on AWS, Azure, GCP (account is deployed into a cloud/region).
- **Can I share data without copying?** Yes — secure shares and data marketplace.
- **How to handle JSON?** Store in VARIANT, use FLATTEN + path expressions.

---

## Appendix: Useful Functions (selected)
- `PARSE_JSON`, `TO_VARIANT`, `FLATTEN`, `OBJECT_INSERT`, `GET_PATH`, `ARRAY_SIZE`
- `DATE_TRUNC`, `DATEDIFF`, `TRY_TO_TIMESTAMP`, `TO_TIMESTAMP_LTZ`
- `IFF`, `NVL`, `COALESCE`, `NULLIF`
- `HASH`, `MD5`, `RANDOM`, `SEQ8()` (sequence)

---

## References / Next Steps
- Official docs (search "Snowflake documentation") — use for latest features and limitations.
- Try small projects: ingest sample JSON logs, create a stream/task ETL, try zero-copy cloning workflow.

---

_End of cheat sheet._
