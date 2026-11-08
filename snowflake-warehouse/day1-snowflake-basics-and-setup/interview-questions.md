# Day 1 — Interview Questions: Snowflake Basics and Setup

> 30 questions across all 3 concepts. Attempt your own answer first, then check the solutions file.
> Difficulty tags: (Warm-up) → (Conceptual) → (Scenario) → (Tricky) → (Deep dive) → (System design)

---

## Concept 1: What is Snowflake?

**Q1 (Warm-up)** — What type of system is Snowflake, and what cloud providers does it run on?

**Q2 (Warm-up)** — What does "cloud-native" mean in the context of Snowflake? How does it differ from "cloud-hosted"?

**Q3 (Conceptual)** — Describe Snowflake's three-layer architecture. What does each layer do, and who is responsible for managing it?

**Q4 (Conceptual)** — Explain how Snowflake separates compute from storage. Why is this architecturally significant?

**Q5 (Scenario)** — Your analytics team runs complex reports from 9am–11am daily. Your ETL team runs data loads from midnight–4am. With a traditional shared server, peak usage from one team slows the other. How does Snowflake solve this problem?

**Q6 (Conceptual)** — What is the Cloud Services Layer in Snowflake? List three responsibilities it handles.

**Q7 (Tricky)** — A colleague says "Snowflake stores data in S3, so I can just read the raw S3 files directly to avoid Snowflake query costs." Is this correct? What would actually happen?

---

## Concept 2: Account Setup and Navigation

**Q8 (Warm-up)** — What is the Snowflake free trial period, and how much in credits does it include?

**Q9 (Warm-up)** — What is Snowsight? Name three sections of its navigation panel and what each is used for.

**Q10 (Conceptual)** — Snowflake has four editions: Standard, Enterprise, Business Critical, and Virtual Private Snowflake. What are two features that Enterprise has but Standard does not?

**Q11 (Conceptual)** — What is an account identifier in Snowflake? Give an example of both the modern format and the legacy locator format.

**Q12 (Scenario)** — You are setting up Snowflake for a client. They are a US company but their primary users are in Europe. They ask which cloud region to pick. What factors would you consider?

**Q13 (Conceptual)** — What SQL function would you run to verify your Snowflake account is active, and what SQL returns your current account name, region, and cloud provider?

**Q14 (Tricky)** — A new employee tries to run a SELECT query in a Snowsight worksheet but gets the error: "No active warehouse selected for this session." What is the root cause and how do you fix it?

---

## Concept 3: Databases, Schemas, and Warehouses

**Q15 (Warm-up)** — What is the Snowflake object hierarchy from top to bottom? List at least five levels.

**Q16 (Warm-up)** — What are the two schemas that exist by default in every Snowflake database, and what is each used for?

**Q17 (Conceptual)** — What is a Virtual Warehouse in Snowflake? Explain how it differs from a "database" in a traditional RDBMS sense.

**Q18 (Conceptual)** — Explain AUTO_SUSPEND and AUTO_RESUME for virtual warehouses. What is the trade-off between a short vs. long AUTO_SUSPEND interval?

**Q19 (Conceptual)** — Snowflake charges for warehouses by the second with a minimum of 60 seconds. If you run a query that takes 10 seconds, how many seconds are you billed for? What if you run two back-to-back queries that take 5 seconds each with a 30-second pause between them?

**Q20 (Scenario)** — Your team has three distinct workloads: (a) a nightly ETL job that runs for 3 hours, (b) ad hoc analyst queries throughout the day, (c) a dashboard that many users hit simultaneously. Should you use one warehouse or multiple warehouses? Design the solution.

**Q21 (Conceptual)** — What does `USE WAREHOUSE`, `USE DATABASE`, and `USE SCHEMA` do in a Snowflake session? What happens if you don't set them?

**Q22 (Tricky)** — Can you resize a virtual warehouse while it is running queries? What happens to the in-flight queries during the resize?

---

## Mixed / Senior-Level Questions

**Q23 (Scenario)** — A startup is choosing between Snowflake, BigQuery, and Redshift. They are AWS-native, have a small team (2 data engineers), need SQL analytics, and might expand to Azure in 2 years. What do you recommend and why?

**Q24 (Conceptual)** — How does Snowflake's billing model differ from a traditional database server model (e.g., RDS)? What are the cost implications for a workload that runs for 2 hours per day vs. 24 hours per day?

**Q25 (Deep dive)** — What is the SNOWFLAKE_SAMPLE_DATA database? Name two schemas inside it and their use cases. How is this useful for learning or benchmarking?

**Q26 (Scenario)** — You are onboarding a new Snowflake account for a data team. List the first 5 things you would set up before allowing the team to start loading data. Explain why each is important.

**Q27 (Tricky)** — A warehouse is set to AUTO_SUSPEND = 60. A developer runs a query at 9:00:00 AM that finishes at 9:00:45 AM. At 9:01:30 AM another query arrives. Is the warehouse still running or has it suspended? How much credit is billed for the first query?

**Q28 (Deep dive)** — What is the difference between a Snowflake "database" and a "schema"? Give a real-world example of how a company might organize multiple databases and schemas.

**Q29 (Scenario)** — Your manager asks you to reduce Snowflake costs. You notice a warehouse has been running continuously for 7 days. What questions would you ask and what changes would you make?

**Q30 (System design)** — Design a Snowflake object structure for an e-commerce company with three environments (dev, staging, prod), two teams (data engineering and analytics), and three data layers (raw, staging, marts). Name the databases, schemas, and warehouses you would create and explain your reasoning.
