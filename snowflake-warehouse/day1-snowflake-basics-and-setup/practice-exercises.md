# Day 1 — Practice Exercises: Snowflake Basics and Setup

> All exercises are done in the Snowflake trial account (Snowsight UI + SQL worksheets).
> Complete them in order — each builds on the previous one.
> Estimated total time: 60–90 minutes.

---

## Exercise 1 — Create Your Snowflake Trial Account

**Concept:** Snowflake account setup, cloud provider selection, Snowsight navigation

**Tasks:**

**1a. Sign up for a free trial**
- Go to `app.snowflake.com` → "Start for free"
- Use your work or personal email
- Choose **AWS** as the cloud provider and **US East (N. Virginia)** as the region

**1b. Activate your account**
- Check your email for the activation link
- Set a secure password
- Note your account URL — it will look like: `https://abc12345.snowflakecomputing.com`

**1c. Log in and explore Snowsight**
- Navigate to each section: Home, Worksheets, Data, Monitor, Admin
- Under **Admin → Warehouses**, observe the `COMPUTE_WH` that was created automatically

**1d. Find your account identifier**
- Open a new worksheet
- Run:
  ```sql
  SELECT
      CURRENT_ACCOUNT()           AS account,
      CURRENT_REGION()            AS region,
      CURRENT_ORGANIZATION_NAME() AS org,
      CURRENT_VERSION()           AS snowflake_version;
  ```
- Record these values — you will need them when connecting Python, dbt, or Airflow later

**Expected outcome:** You can log in, run a query, and see version information returned.

**Discussion:** Why does Snowflake ask you to pick a cloud provider and region at account creation? What would happen if you wanted to change regions later?

---

## Exercise 2 — Explore System Databases and Sample Data

**Concept:** Object hierarchy, built-in databases, Snowsight Data browser

**Tasks:**

**2a. Browse databases in Snowsight**
- Click **Data** in the left nav
- Expand the **Databases** section
- Identify which databases exist by default (`SNOWFLAKE`, `SNOWFLAKE_SAMPLE_DATA`)

**2b. Explore SNOWFLAKE_SAMPLE_DATA**
- In a worksheet, run:
  ```sql
  SHOW SCHEMAS IN DATABASE SNOWFLAKE_SAMPLE_DATA;
  ```
- How many schemas are available?

**2c. Query the TPC-H sample data**
- Run each query and observe the results:
  ```sql
  USE DATABASE SNOWFLAKE_SAMPLE_DATA;
  USE SCHEMA TPCH_SF1;

  -- How many orders are in the dataset?
  SELECT COUNT(*) FROM ORDERS;

  -- Who are the top 5 customers by total spend?
  SELECT
      C.C_NAME          AS customer_name,
      SUM(O.O_TOTALPRICE) AS total_spend
  FROM CUSTOMER C
  JOIN ORDERS O ON C.C_CUSTKEY = O.O_CUSTKEY
  GROUP BY C.C_NAME
  ORDER BY total_spend DESC
  LIMIT 5;
  ```

**2d. Check the query profile**
- After running the JOIN query, click the **Query ID** in the results panel
- Observe the Query Profile — notice the scan, aggregate, and sort steps
- How many rows were scanned? How long did the query take?

**Expected outcome:** You can explore sample data, run a multi-table query, and read the query profile.

**Discussion:** The TPC-H dataset has 1.5 million orders at scale factor 1. If you used `TPCH_SF100` (scale factor 100), how many rows would you expect? Why is using sample data useful for learning without loading your own?

---

## Exercise 3 — Create Your Own Database, Schema, and First Table

**Concept:** Object hierarchy, CREATE statements, session context

**Tasks:**

**3a. Create a learning database**
```sql
CREATE DATABASE SNOWFLAKE_LEARNING
  COMMENT = 'Day 1 practice database';
```

**3b. Create schemas for different data layers**
```sql
USE DATABASE SNOWFLAKE_LEARNING;

CREATE SCHEMA RAW_DATA   COMMENT = 'Raw ingested data — no transformations';
CREATE SCHEMA STAGING    COMMENT = 'Cleaned and validated data';
CREATE SCHEMA ANALYTICS  COMMENT = 'Business-ready tables and views';
```

**3c. Set your session context**
```sql
USE DATABASE SNOWFLAKE_LEARNING;
USE SCHEMA RAW_DATA;
```

**3d. Create a sample table**
```sql
CREATE TABLE CUSTOMERS (
    customer_id     INTEGER,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(200),
    country         VARCHAR(50),
    signup_date     DATE
);
```

**3e. Insert a few rows and query them**
```sql
INSERT INTO CUSTOMERS VALUES
    (1, 'Alice',   'Johnson', 'alice@example.com',   'USA',    '2024-01-15'),
    (2, 'Bob',     'Smith',   'bob@example.com',     'Canada', '2024-02-20'),
    (3, 'Priya',   'Sharma',  'priya@example.com',   'India',  '2024-03-05'),
    (4, 'Carlos',  'Mendez',  'carlos@example.com',  'Mexico', '2024-01-30'),
    (5, 'Yuki',    'Tanaka',  'yuki@example.com',    'Japan',  '2024-04-01');

SELECT * FROM CUSTOMERS;
```

**3f. Verify what you built using INFORMATION_SCHEMA**
```sql
-- List all tables in your database
SELECT table_name, table_schema, row_count, bytes
FROM SNOWFLAKE_LEARNING.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA';
```

**Expected outcome:** You have a database with 3 schemas and a table with 5 rows.

**Discussion:** You used `USE SCHEMA RAW_DATA` before creating the table. What would have happened if you'd skipped that step and just run the CREATE TABLE directly after creating the schemas?

---

## Exercise 4 — Create and Manage a Virtual Warehouse

**Concept:** Virtual warehouses, auto-suspend, auto-resume, session context

**Tasks:**

**4a. Create your own warehouse**
```sql
CREATE WAREHOUSE LEARNING_WH
  WAREHOUSE_SIZE    = 'X-SMALL'
  AUTO_SUSPEND      = 60
  AUTO_RESUME       = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Day 1 learning warehouse — X-Small, auto-suspends in 60s';
```

**4b. Check warehouse status**
```sql
SHOW WAREHOUSES;
```
- What is the state of `LEARNING_WH`? (Should be `SUSPENDED` since we set `INITIALLY_SUSPENDED = TRUE`)
- What is the state of `COMPUTE_WH` (the default one)?

**4c. Use your new warehouse**
```sql
USE WAREHOUSE LEARNING_WH;

-- This query will auto-resume the warehouse
SELECT COUNT(*) FROM SNOWFLAKE_LEARNING.RAW_DATA.CUSTOMERS;
```
- Run `SHOW WAREHOUSES` again. Is `LEARNING_WH` now `STARTED`?

**4d. Wait for auto-suspend**
- Wait 2 minutes without running any queries
- Run `SHOW WAREHOUSES` again — `LEARNING_WH` should now show `SUSPENDED`

**4e. Resize the warehouse**
```sql
-- Resize (can be done whether suspended or running)
ALTER WAREHOUSE LEARNING_WH SET WAREHOUSE_SIZE = 'SMALL';

-- Verify
SHOW WAREHOUSES;
-- SIZE column should now say SMALL
```

**4f. Check session context**
```sql
SELECT
    CURRENT_ROLE()      AS current_role,
    CURRENT_WAREHOUSE() AS current_wh,
    CURRENT_DATABASE()  AS current_db,
    CURRENT_SCHEMA()    AS current_schema;
```

**Expected outcome:** You have created, suspended, resumed, and resized a virtual warehouse.

**Discussion:** You set `AUTO_SUSPEND = 60`. In a real production ETL pipeline that runs for 4 hours, would this be a good or bad setting? What would happen if the pipeline paused for 90 seconds mid-run?

---

## Exercise 5 — Understand the Three-Layer Architecture Hands-On

**Concept:** Storage vs. compute separation, concurrent warehouses, query history

**Tasks:**

**5a. Open two worksheet tabs in Snowsight**
- Create a second worksheet tab
- In worksheet 1: `USE WAREHOUSE LEARNING_WH;`
- In worksheet 2: `USE WAREHOUSE COMPUTE_WH;`

**5b. Run concurrent queries on the same table from different warehouses**
- Worksheet 1 (LEARNING_WH):
  ```sql
  SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;
  ```
- Worksheet 2 (COMPUTE_WH — run this immediately after, don't wait):
  ```sql
  SELECT AVG(O_TOTALPRICE) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;
  ```

**5c. Observe query history**
- Navigate to **Monitor → Query History** in Snowsight
- Find both queries — notice they ran on different warehouses but accessed the same data
- This proves storage is shared; compute is separate

**5d. Explore the query plan**
- Click on one of the query IDs to see its detail
- Look at the **Query Profile** tab
- Identify: TableScan, Aggregate nodes

**Expected outcome:** You observe two warehouses querying the same underlying data simultaneously without conflict.

**Discussion:** If two warehouses can read the same table simultaneously, what happens when two warehouses try to INSERT into the same table at the same time? Does Snowflake handle this safely?

---

## Exercise 6 — Apply Naming Conventions and Organize Your Objects

**Concept:** Best practices, object organization, real-world patterns

**Tasks:**

**6a. Add a second schema layer with a different naming convention**
```sql
-- Create a staging version of your CUSTOMERS table
USE DATABASE SNOWFLAKE_LEARNING;
USE SCHEMA STAGING;

CREATE TABLE STG_CUSTOMERS AS
SELECT
    customer_id,
    UPPER(first_name)   AS first_name,
    UPPER(last_name)    AS last_name,
    LOWER(email)        AS email,
    country,
    signup_date
FROM SNOWFLAKE_LEARNING.RAW_DATA.CUSTOMERS;

SELECT * FROM STG_CUSTOMERS;
```

**6b. Create a view in ANALYTICS schema**
```sql
USE SCHEMA ANALYTICS;

CREATE VIEW CUSTOMERS_BY_COUNTRY AS
SELECT
    country,
    COUNT(*)        AS customer_count,
    MIN(signup_date) AS earliest_signup,
    MAX(signup_date) AS latest_signup
FROM SNOWFLAKE_LEARNING.STAGING.STG_CUSTOMERS
GROUP BY country
ORDER BY customer_count DESC;

SELECT * FROM CUSTOMERS_BY_COUNTRY;
```

**6c. Full inventory of what you built today**
```sql
-- See all objects across all schemas
SELECT
    table_schema,
    table_name,
    table_type,
    row_count
FROM SNOWFLAKE_LEARNING.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;
```

**6d. Clean up (optional — keep the objects if you want to use them in Day 2)**
```sql
-- Only run this if you want to remove everything
-- DROP DATABASE SNOWFLAKE_LEARNING;
-- DROP WAREHOUSE LEARNING_WH;
```

**Expected outcome:** You have a 3-layer database structure (RAW_DATA → STAGING → ANALYTICS) with a table, a derived table, and a view — mirroring a real data warehouse pattern.

**Discussion:** In a production system, would you grant an analyst role access to `RAW_DATA`, `STAGING`, or `ANALYTICS`? Why? What about a data engineer role?

---

## Bonus Challenge — Map Your Account to the Architecture

**Concept:** Synthesis — connect theory to what you observed

Using only SQL queries (no Snowsight UI clicks), answer these questions:

```sql
-- 1. What Snowflake version are you on?
SELECT CURRENT_VERSION();

-- 2. How many databases exist in your account?
SHOW DATABASES;

-- 3. How many warehouses exist?
SHOW WAREHOUSES;

-- 4. What is your current default role?
SELECT CURRENT_ROLE();

-- 5. How many rows did your CUSTOMERS table end up with?
SELECT COUNT(*) FROM SNOWFLAKE_LEARNING.RAW_DATA.CUSTOMERS;
```

**Design question:** Draw (on paper or in a text file) the three-layer Snowflake architecture as it applies to what YOU built today:
- What is in your Storage layer? (Which tables, in which database/schema?)
- What is your Compute layer? (Which warehouses did you use?)
- What does the Cloud Services layer manage that you never had to touch?

This mental model — separating storage from compute from services — is the most important foundation for the rest of this course.
