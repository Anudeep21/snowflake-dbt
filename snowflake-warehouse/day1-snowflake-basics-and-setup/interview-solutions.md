# Day 1 — Interview Solutions: Snowflake Basics and Setup

> Complete answers to all 30 questions. Read these after attempting the questions yourself.

---

## Concept 1: What is Snowflake?

**Q1 — What type of system is Snowflake, and what cloud providers does it run on?**

Snowflake is a **cloud-native data warehouse** (also called a cloud data platform). It is not a database you install — it is a fully managed SaaS product. It runs natively on three cloud providers:
- **AWS** (Amazon Web Services)
- **Azure** (Microsoft)
- **GCP** (Google Cloud Platform)

You choose your cloud and region at account creation. Your data stays within that cloud and region unless you explicitly configure replication.

---

**Q2 — What does "cloud-native" mean? How does it differ from "cloud-hosted"?**

- **Cloud-hosted:** An existing product (e.g., SQL Server or Oracle) installed on a cloud VM. The software was originally designed for on-premise hardware. The cloud is just the infrastructure.

- **Cloud-native:** Software designed from scratch to exploit cloud primitives — elastic storage (S3/Blob/GCS), elastic compute, pay-per-use billing, managed networking. Snowflake was built this way from day one (2012). It has no on-premise version.

**Why it matters:** Cloud-native systems get features like near-infinite scale, zero infrastructure management, and per-second billing "for free" because the architecture was designed around them. Cloud-hosted systems require significant engineering work to achieve the same.

---

**Q3 — Describe Snowflake's three-layer architecture.**

| Layer | Name | What it Does | Managed By |
|---|---|---|---|
| Bottom | **Storage** | All data stored as compressed, columnar micro-partitions in cloud blob storage (S3/Blob/GCS). Data is always there, regardless of compute. | Snowflake |
| Middle | **Compute (Virtual Warehouses)** | Clusters of VMs that execute SQL queries. Multiple independent warehouses can read the same storage simultaneously. Billing only when running. | You (size, count, suspend policy) |
| Top | **Cloud Services** | Authentication, metadata, query optimizer, transaction management, security. This layer coordinates everything and runs continuously. | Snowflake |

---

**Q4 — Explain how Snowflake separates compute from storage. Why is this significant?**

In traditional databases (Oracle, SQL Server, PostgreSQL), compute and storage live on the same server. If you need more CPU for faster queries, you buy a bigger server — which also gives you more storage whether you need it or not. Scaling is all-or-nothing.

In Snowflake:
- **Storage** scales independently (you pay per TB of compressed data)
- **Compute** scales independently (you pay per second of warehouse use)

**Significance:**
1. You can run a large query on a big warehouse, then suspend it — paying nothing until the next query
2. Multiple teams can query the same data with separate warehouses — no contention
3. You never pay for idle compute sitting next to your data

---

**Q5 — Analytics team runs 9am–11am, ETL team runs midnight–4am. How does Snowflake solve the contention problem?**

Create **two separate virtual warehouses** pointing at the same underlying storage:

```sql
CREATE WAREHOUSE ANALYTICS_WH WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 120;
CREATE WAREHOUSE ETL_WH       WAREHOUSE_SIZE = 'LARGE'  AUTO_SUSPEND = 60;
```

The analytics team uses `ANALYTICS_WH`; the ETL team uses `ETL_WH`. Both warehouses read from the same tables in storage simultaneously without blocking each other.

**Result:** No contention. Each team gets dedicated compute. The ETL warehouse auto-suspends at 4am after the job finishes; the analytics warehouse auto-suspends at 11am. You pay for each only when it's running.

---

**Q6 — What is the Cloud Services Layer? List three responsibilities.**

The Cloud Services Layer is the "brain" of Snowflake — it runs continuously and coordinates all activity above the storage layer. It is included in your Snowflake pricing (no separate billing unless cloud services usage exceeds 10% of your daily compute spend).

Three responsibilities:
1. **Authentication & access control** — validates user logins, enforces role-based permissions on every query
2. **Query optimization** — the optimizer parses SQL, builds execution plans, and decides which micro-partitions to scan (pruning)
3. **Metadata management** — tracks the location of every micro-partition, table statistics, schema definitions, transaction log — without this, Snowflake couldn't find your data

---

**Q7 — Can you read raw S3 files directly to avoid Snowflake query costs?**

No, this is not feasible:

1. Snowflake stores data in its **own proprietary compressed, columnar format** — not in readable CSV or Parquet files. The S3 paths are internal Snowflake paths that your AWS account has no access to.

2. Even if you could access the files, the format is not documented and subject to change.

3. Snowflake's storage access is managed through its own IAM roles and encryption keys — your AWS credentials have no permission to read them.

**The correct way** to export data out of Snowflake is to use `COPY INTO <stage>` to write to an external stage (an S3 bucket you own), then read from that bucket.

---

## Concept 2: Account Setup and Navigation

**Q8 — Free trial period and credit amount?**

Snowflake's free trial is **30 days** and includes **$400 in Snowflake credits**. You get Enterprise edition features during the trial. No credit card required to sign up. After 30 days (or when credits are exhausted, whichever comes first), you must enter billing information to continue.

---

**Q9 — What is Snowsight? Name three navigation sections.**

Snowsight is Snowflake's modern web-based UI (replaced the older Classic UI in 2022). It is accessible at `app.snowflake.com` after login.

Three navigation sections:

| Section | Purpose |
|---|---|
| **Worksheets** | Write and run SQL queries. Supports multiple tabs, sharing worksheets with teammates, version history. |
| **Data** | Browse databases, schemas, tables, views. Preview table data, see column definitions, row counts. |
| **Monitor** | View query history, warehouse usage over time, credit consumption. Used for debugging slow queries and tracking costs. |

---

**Q10 — Two features Enterprise has that Standard does not?**

| Feature | Standard | Enterprise |
|---|---|---|
| **Multi-cluster warehouses** | No | Yes — automatically scale out to N clusters during peak load |
| **Time Travel up to 90 days** | 1 day max | Up to 90 days — recover accidentally dropped or modified data |
| Column-level security | No | Yes |
| Row access policies | No | Yes |

For production use, Enterprise is the minimum recommended edition. Standard is sufficient only for very small teams or non-sensitive workloads.

---

**Q11 — What is an account identifier? Give both format examples.**

An account identifier uniquely identifies your Snowflake account for external connections (Python, dbt, JDBC, ODBC, etc.).

**Modern format (recommended):**
```
<organization_name>-<account_name>
Example: mycompany-prod_account
```

**Legacy locator format (still works):**
```
<locator>.<region>.<cloud>
Example: xy12345.us-east-1.aws
```

Find yours by running:
```sql
SELECT CURRENT_ORGANIZATION_NAME(), CURRENT_ACCOUNT();
```

Or navigate to **Admin → Accounts** in Snowsight.

---

**Q12 — Client is US company, primary users in Europe. What region to pick?**

Consider these factors:

1. **User latency** — Queries from Europe to a US region will add ~100–150ms of network latency for each result set. For interactive analytics, this matters.
2. **Data residency / compliance** — European data may need to stay in EU under GDPR. Even if the company is US-based, if they store EU personal data, they may need EU hosting.
3. **Cost** — EU regions are sometimes slightly more expensive per credit.
4. **Cloud provider alignment** — If their existing infrastructure is on Azure Europe, choose Azure West Europe to keep data transfer costs low.

**Recommendation:** If users are primarily in Europe and data residency is a concern, choose **AWS EU (Frankfurt)** or **Azure West Europe**. The latency savings and compliance alignment outweigh the slight cost difference.

---

**Q13 — SQL to verify account is active and return account details?**

```sql
-- Verify Snowflake is responding (returns version number)
SELECT CURRENT_VERSION();

-- Return account context
SELECT
    CURRENT_ACCOUNT()           AS account_name,
    CURRENT_REGION()            AS region,
    CURRENT_ORGANIZATION_NAME() AS org_name;
```

The `CURRENT_REGION()` output includes the cloud provider, e.g., `AWS_US_EAST_1`.

---

**Q14 — Error "No active warehouse selected." Root cause and fix?**

**Root cause:** Every query in Snowflake requires a virtual warehouse (compute cluster) to execute. If no warehouse has been set for the session, Snowflake doesn't know what compute to use and throws this error. This commonly happens when:
- You just created a new worksheet and haven't set context yet
- Your default role has no default warehouse configured
- The warehouse you were using was dropped

**Fix (three approaches):**

```sql
-- Option 1: Set it via SQL in the worksheet
USE WAREHOUSE LEARNING_WH;

-- Option 2: Set it in the worksheet dropdown
-- (top right of Snowsight worksheet — select warehouse from dropdown)

-- Option 3: Set a default warehouse on your user profile
ALTER USER my_username SET DEFAULT_WAREHOUSE = 'LEARNING_WH';
```

---

## Concept 3: Databases, Schemas, and Warehouses

**Q15 — Snowflake object hierarchy from top to bottom (5+ levels)?**

```
Organization
  └── Account
        ├── Database
        │     └── Schema
        │           ├── Table
        │           ├── View
        │           ├── Stage
        │           ├── File Format
        │           ├── Stored Procedure
        │           └── UDF (User-Defined Function)
        ├── Virtual Warehouse
        ├── Role
        └── User
```

---

**Q16 — Two default schemas in every Snowflake database?**

| Schema | Purpose |
|---|---|
| `PUBLIC` | Default schema created for convenience. Snowflake puts objects here if no schema is specified. Can be used freely or ignored. |
| `INFORMATION_SCHEMA` | Read-only system schema. Contains views like `TABLES`, `COLUMNS`, `VIEWS`, `USAGE_PRIVILEGES` that describe all objects in the database. Used for metadata queries. |

Example:
```sql
-- List all tables in your database using INFORMATION_SCHEMA
SELECT table_name, row_count, bytes
FROM SNOWFLAKE_LEARNING.INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'RAW_DATA';
```

---

**Q17 — What is a Virtual Warehouse? How does it differ from a "database" in a traditional RDBMS?**

A **Virtual Warehouse** is a named, dynamically-sized cluster of compute resources (CPUs + memory) that executes SQL queries. It has nothing to do with data storage.

In a traditional RDBMS (PostgreSQL, MySQL):
- The "database server" is one machine (or fixed cluster) that handles both compute AND storage
- All queries compete for the same CPU/memory on that server
- Scaling requires changing the server

In Snowflake:
- A virtual warehouse is **pure compute** — no data lives in it
- Multiple warehouses can read the same data simultaneously
- Each warehouse is billed independently and can be suspended/resumed/resized independently
- Creating a warehouse does not reserve any storage

Think of warehouses as **rented compute power** that you turn on when needed and turn off when done.

---

**Q18 — Explain AUTO_SUSPEND and AUTO_RESUME. Short vs. long interval trade-off.**

**AUTO_SUSPEND = N:** After N seconds of no query activity, the warehouse automatically shuts down. Billing stops immediately.

**AUTO_RESUME = TRUE:** When a query arrives at a suspended warehouse, it automatically restarts. The first query waits ~5–10 seconds for startup.

**Trade-off:**

| Setting | Advantage | Disadvantage |
|---|---|---|
| Short interval (e.g., 60s) | Saves credits quickly during idle periods | First query after idle has startup delay |
| Long interval (e.g., 600s) | Avoids startup delay between queries | Pays for idle compute during the wait period |

**Recommendation by use case:**
- Learning/development: `60` seconds — save every credit
- Nightly batch ETL: `300` seconds — the job runs continuously so suspensions rarely happen
- Interactive BI dashboard: `120` seconds — balance between cost and user experience

---

**Q19 — Billing calculation for 10-second query. Two 5-second queries with 30-second pause.**

**Rule:** Minimum billing per warehouse startup is **60 seconds**. After that, billing is per-second.

**Single 10-second query:**
- Warehouse resumes → runs query (10 sec) → no more queries → auto-suspends
- Billed: **60 seconds** (minimum applies — query was only 10s but minimum is 60s)

**Two 5-second queries with 30-second pause (AUTO_SUSPEND = 60):**
- 9:00:00 → warehouse resumes, query 1 runs (5 sec)
- 9:00:05 → query done, idle timer starts
- 9:00:35 → second query arrives (30 sec after first) — warehouse still running (hasn't hit 60s idle)
- 9:00:40 → query 2 done, idle timer resets
- 9:01:40 → warehouse suspends after 60s idle
- Total runtime: 100 seconds (from 9:00:00 to 9:01:40)
- Billed: **100 seconds** (more than 60s minimum, so pay actual runtime)

---

**Q20 — Three workloads: nightly ETL, ad hoc queries, dashboard. One or multiple warehouses?**

Use **three separate warehouses**:

```sql
-- Nightly batch ETL — Large size, long suspend (job runs continuously)
CREATE WAREHOUSE ETL_WH
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE;

-- Ad hoc analyst queries — Medium size, short suspend
CREATE WAREHOUSE ANALYST_WH
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;

-- Dashboard — Small or Medium, multi-cluster enabled (handles concurrent users)
CREATE WAREHOUSE DASHBOARD_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3;  -- Enterprise: auto-scale if many users hit simultaneously
```

**Why separate:**
1. The ETL job running overnight won't slow down analyst queries the next morning
2. Dashboard users don't compete with the ETL job
3. Each warehouse can be sized independently for its workload
4. Each warehouse's credits are tracked separately — easier cost attribution

---

**Q21 — What do USE WAREHOUSE, USE DATABASE, USE SCHEMA do? What happens without them?**

These commands set the **session context** — the defaults Snowflake uses for subsequent queries.

| Command | Sets Default | Without It |
|---|---|---|
| `USE WAREHOUSE wh_name` | Compute cluster for queries | Error: "No active warehouse selected" |
| `USE DATABASE db_name` | Database to look in | Must fully-qualify every object: `DB.SCHEMA.TABLE` |
| `USE SCHEMA schema_name` | Schema within the database | Must prefix all objects with schema name |

```sql
-- Without context — verbose
SELECT * FROM PROD_DW.RAW_DATA.ORDERS;

-- With context set — clean
USE DATABASE PROD_DW;
USE SCHEMA RAW_DATA;
SELECT * FROM ORDERS;  -- same query, much cleaner
```

---

**Q22 — Can you resize a warehouse while it's running? What happens to in-flight queries?**

Yes, you can resize a warehouse while it is running:

```sql
ALTER WAREHOUSE LEARNING_WH SET WAREHOUSE_SIZE = 'LARGE';
```

**What happens:**
- **In-flight queries continue running** on the current cluster size — they are not interrupted
- The new size applies to queries that start **after** the resize completes
- Resize typically takes a few seconds

**Key gotcha:** If you resize from Large to XSmall while a heavy query is running, that query finishes at Large (and is billed at Large rate). The XSmall size applies only to the next query.

---

## Mixed / Senior-Level Questions

**Q23 — Snowflake vs BigQuery vs Redshift for an AWS-native startup that might go multi-cloud?**

**Recommendation: Snowflake**

Reasoning:

| Factor | Snowflake | BigQuery | Redshift |
|---|---|---|---|
| AWS-native | Runs on AWS natively | GCP only | AWS only |
| Multi-cloud readiness | Native AWS + Azure + GCP | No | No |
| Team size (2 engineers) | Minimal admin — fully managed | Minimal admin | Requires tuning (VACUUM, ANALYZE, distribution keys) |
| SQL analytics | Excellent | Excellent | Good |
| Future Azure expansion | Supported | Not possible | Not possible |

**Trade-off:** Snowflake costs slightly more than BigQuery per query at large scale, but the multi-cloud flexibility and lower operational overhead justify it for a small team with expansion plans.

---

**Q24 — Snowflake billing vs traditional database server billing?**

**Traditional (e.g., RDS):**
- You pay per hour for the instance, whether queries are running or not
- 2h/day workload: still pays for 24h of server availability (unless you stop the instance, which has startup overhead)

**Snowflake:**
- You pay per second of compute (virtual warehouse) usage
- 2h/day workload: pay for ~2 hours of credits
- 24h/day workload: pay for ~24 hours of credits

**Cost implication:**
- Workload at 2h/day → Snowflake is **12× cheaper** than a fixed server on compute
- Workload at 24h/day → Similar cost to a fixed server, but Snowflake is still simpler to manage

This is why Snowflake is especially cost-effective for data teams with batch workloads (runs at night, idle during the day).

---

**Q25 — SNOWFLAKE_SAMPLE_DATA database. Two schemas and their use cases.**

`SNOWFLAKE_SAMPLE_DATA` is a shared read-only database in every Snowflake account. It contains industry-standard benchmark datasets.

| Schema | Dataset | Use Case |
|---|---|---|
| `TPCH_SF1` | TPC-H at scale factor 1 (~1GB) | Benchmarking SQL query performance, learning JOIN patterns, testing warehouse sizing |
| `TPCDS_SF10TCL` | TPC-DS at scale factor 10 | More complex star-schema queries, testing query optimizer, simulating retail analytics |

```sql
-- Count rows in TPC-H orders table
USE DATABASE SNOWFLAKE_SAMPLE_DATA;
USE SCHEMA TPCH_SF1;
SELECT COUNT(*) FROM ORDERS;  -- 1,500,000 rows at SF1
```

Useful for: testing a new warehouse size, verifying query optimizer behavior, learning SQL without loading your own data.

---

**Q26 — First 5 things to set up on a new Snowflake account before the team loads data?**

1. **Create a resource monitor** — Prevents runaway costs by setting credit quotas and alerting or suspending warehouses if limits are hit
   ```sql
   CREATE RESOURCE MONITOR monthly_limit
     CREDIT_QUOTA = 500  -- alert when 500 credits consumed
     TRIGGERS ON 80 PERCENT DO NOTIFY
              ON 100 PERCENT DO SUSPEND;
   ```

2. **Create an admin role hierarchy** — Separate SYSADMIN (objects), SECURITYADMIN (users/roles), ACCOUNTADMIN (billing/account level)

3. **Create warehouses per team** — Separate ETL, analyst, and reporting warehouses with appropriate sizes and auto-suspend settings

4. **Create the database and schema structure** — Establish conventions (RAW, STAGING, MARTS or similar) before anyone starts loading

5. **Create user accounts with appropriate roles** — Never let team members use ACCOUNTADMIN for daily work; create purpose-specific roles and assign minimum necessary privileges

---

**Q27 — AUTO_SUSPEND = 60. Query finishes at 9:00:45. Second query arrives at 9:01:30. Suspended?**

Timeline:
- 9:00:00 → warehouse resumes, query starts
- 9:00:45 → query finishes, idle timer starts
- 9:01:30 → second query arrives (45 seconds after query finished)

Since 45 seconds < 60 seconds (AUTO_SUSPEND), the idle timer has NOT expired. The warehouse is **still running**.

**Credits billed for the first query session:**
- Warehouse was running from 9:00:00
- Second query arrives at 9:01:30 — warehouse continues
- (Credit billing continues until the warehouse eventually suspends after the second query + 60s)

**For the first query alone:** The warehouse ran 45 seconds of idle after the query. Since the second query arrived before the 60s idle timer expired, there's no suspension event yet — it's one continuous billing window.

---

**Q28 — Difference between a Snowflake database and schema? Real-world example.**

**Database:** A top-level namespace container. Usually represents an environment or major product area. No compute or storage is allocated — it's pure logical organization.

**Schema:** A sub-namespace inside a database. Holds the actual objects (tables, views, etc.). Usually represents a data layer or functional area.

**Real-world example — e-commerce company:**

```
ECOMM_PROD (database)          ECOMM_DEV (database)
├── RAW_DATA (schema)          ├── RAW_DATA (schema)
│   ├── ORDERS (table)         │   └── ORDERS (table)
│   └── CUSTOMERS (table)      └── ANALYTICS (schema)
├── STAGING (schema)               └── SALES_SUMMARY (view)
│   └── STG_ORDERS (table)
└── MARTS (schema)
    ├── SALES_MART (table)
    └── CUSTOMER_MART (table)
```

The database boundary separates environments (prod vs dev). The schema boundary separates data layers (raw, staging, marts). This prevents dev data from accidentally appearing in prod queries.

---

**Q29 — Warehouse running continuously for 7 days. What questions and changes?**

**Questions to ask:**
1. Is this warehouse running actual queries, or just sitting idle? (Check Query History in Monitor tab)
2. What is the AUTO_SUSPEND setting? (`SHOW WAREHOUSES`)
3. Who owns this warehouse and what process uses it?
4. Is there a long-running query stuck? (Check active queries)

**Investigation:**
```sql
-- Check warehouse configuration
SHOW WAREHOUSES LIKE 'WAREHOUSE_NAME';

-- Check if queries are actually running
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE WAREHOUSE_NAME = 'WAREHOUSE_NAME'
  AND START_TIME > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY START_TIME DESC;
```

**Changes to make:**
```sql
-- Set AUTO_SUSPEND if it's currently 0 (never suspends) or too high
ALTER WAREHOUSE WAREHOUSE_NAME SET AUTO_SUSPEND = 300;

-- If it's idle right now, suspend immediately
ALTER WAREHOUSE WAREHOUSE_NAME SUSPEND;
```

**Root cause:** `AUTO_SUSPEND = 0` means "never automatically suspend" — this is the most common cause of runaway warehouse costs. Always set AUTO_SUSPEND for every warehouse.

---

**Q30 (System design) — Snowflake object structure for e-commerce: 3 environments, 2 teams, 3 layers.**

**Databases (one per environment):**

```sql
CREATE DATABASE ECOMM_DEV;
CREATE DATABASE ECOMM_STAGING;
CREATE DATABASE ECOMM_PROD;
```

**Schemas (three layers in each database):**

```sql
-- In each database:
CREATE SCHEMA RAW_DATA;    -- raw ingested data, no transformation
CREATE SCHEMA STAGING;     -- cleaned, deduplicated intermediate data
CREATE SCHEMA MARTS;       -- business-ready aggregated tables and views
```

**Warehouses (by team and environment):**

```sql
-- Data Engineering team
CREATE WAREHOUSE DE_DEV_WH   WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60;
CREATE WAREHOUSE DE_PROD_WH  WAREHOUSE_SIZE = 'LARGE'   AUTO_SUSPEND = 300;

-- Analytics team
CREATE WAREHOUSE ANALYST_WH  WAREHOUSE_SIZE = 'MEDIUM'  AUTO_SUSPEND = 60;
CREATE WAREHOUSE REPORT_WH   WAREHOUSE_SIZE = 'SMALL'   AUTO_SUSPEND = 120;
```

**Roles (access control):**

```sql
CREATE ROLE DATA_ENGINEER;    -- full access to all schemas in all databases
CREATE ROLE ANALYST;          -- read-only on MARTS schema in PROD only
CREATE ROLE STAGING_ACCESSOR; -- read/write on STAGING schema (for QA)
```

**Reasoning:**
- Separate databases per environment prevent accidental cross-environment queries
- Three schema layers enforce data quality progression (raw → clean → aggregated)
- Separate warehouses per team mean DE's heavy ETL jobs don't slow down analyst dashboards
- Role-based access ensures analysts can't accidentally modify raw data
