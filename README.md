# 🛫 AWS DMS CDC Pipeline — MySQL to Amazon S3

<p align="center">
  <img src="https://img.shields.io/badge/AWS-DMS-orange?style=for-the-badge&logo=amazonaws&logoColor=white"/>
  <img src="https://img.shields.io/badge/MySQL-RDS-blue?style=for-the-badge&logo=mysql&logoColor=white"/>
  <img src="https://img.shields.io/badge/Amazon-S3-green?style=for-the-badge&logo=amazons3&logoColor=white"/>
  <img src="https://img.shields.io/badge/CDC-Enabled-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Status-Complete-brightgreen?style=for-the-badge"/>
</p>

---

## 📌 Overview

A production-style, end-to-end **real-time data pipeline** built on AWS, migrating data from **Amazon RDS MySQL** to **Amazon S3** using **AWS Database Migration Service (DMS)** with **Change Data Capture (CDC)**.

The pipeline handles both:
- **Full Load** — initial bulk migration of existing data
- **Incremental Replication (CDC)** — continuous capture of INSERT, UPDATE, and DELETE operations in near real-time

> Every change made in MySQL automatically propagates to S3 as new CDC files — no manual intervention required.

---

## 🏗️ Architecture

```
┌──────────────────────────┐
│   Amazon RDS MySQL       │  ← Source Database (airport-db)
│   (airport database)     │
└───────────┬──────────────┘
            │  Binary Logs (binlog_format=ROW)
            ▼
┌──────────────────────────┐
│  AWS DMS Replication     │  ← airport-dms (Engine 3.6.1)
│  Instance                │
└───────────┬──────────────┘
            │  Full Load + CDC Task
            ▼
┌──────────────────────────┐
│  Amazon S3 Bucket        │  ← Target Storage
│  airport-ved             │
│  └── /dms-incremental-   │
│       output/            │
└──────────────────────────┘
```

---

## 🧰 Technologies Used

| Service | Role |
|---|---|
| **Amazon RDS MySQL** | Source database hosting the `airport` schema |
| **AWS DMS** | Orchestrates full load and CDC replication |
| **Amazon S3** | Target for migrated and incremental data files |
| **IAM Roles & Policies** | Secure access between DMS and S3 |
| **MySQL Workbench** | SQL execution and database management |

---

## 🚀 Implementation Steps

### Step 1 — Create Amazon RDS MySQL Instance

Provisioned an RDS MySQL instance named `airport-db`:

| Parameter | Value |
|---|---|
| Engine | MySQL |
| Instance Name | `airport-db` |
| Port | `3306` |
| Public Access | Enabled |
| Region | `ap-south-1` |

> Public access was enabled to allow AWS DMS (running in the same VPC) to reach the RDS endpoint.

---

### Step 2 — Configure Security Group

Added an inbound rule to allow DMS to connect:

| Type | Protocol | Port | Source |
|---|---|---|---|
| MYSQL/Aurora | TCP | 3306 | `0.0.0.0/0` (Anywhere) |

> **Note:** For production use, restrict the source to the DMS replication instance's IP or security group.

---

### Step 3 — Create DMS Replication Instance

Provisioned a DMS Replication Instance:

| Parameter | Value |
|---|---|
| Instance Name | `airport-dms` |
| Engine Version | `3.6.1` |
| Publicly Accessible | Yes |
| VPC | Same as RDS instance |

---

### Step 4 — Create Source Endpoint (MySQL)

| Property | Value |
|---|---|
| Endpoint Name | `airport-db` |
| Endpoint Type | Source |
| Engine | MySQL |
| Server | `airport-db.c94e6mam893j.ap-south-1.rds.amazonaws.com` |
| Port | `3306` |
| Username | `admin` |

---

### Step 5 — Create Target Endpoint (Amazon S3)

| Property | Value |
|---|---|
| Endpoint Name | `airport-db-end` |
| Endpoint Type | Target |
| Target Engine | Amazon S3 |
| S3 Bucket | `airport-ved` |
| CDC Output Folder | `s3://airport-ved/dms-incremental-output` |

> A dedicated folder (`dms-incremental-output`) was created for CDC output to keep it isolated from Full Load files and avoid contamination from sample JSON files.

---

### Step 6 — Create IAM Role for S3 Access

Created IAM Role `dms-vpc-role` with the following policies attached:

- `AmazonDMSVPCManagementRole`
- `AmazonS3FullAccess`

This allowed the DMS service to write replicated data files directly into the S3 bucket.

---

### Step 7 — Create MySQL Database and Tables

Connected via MySQL Workbench and executed the following schema setup:

```sql
CREATE DATABASE airport;
USE airport;

-- Source table for CDC replication
CREATE TABLE source_airport (
    id          INT,
    airport_name VARCHAR(100),
    city        VARCHAR(100),
    updated_at  TIMESTAMP
);

-- Seed data
INSERT INTO source_airport VALUES
    (1, 'IGI Airport',     'Delhi',   NOW()),
    (2, 'Mumbai Airport',  'Mumbai',  NOW()),
    (3, 'Chennai Airport', 'Chennai', NOW());

-- Supporting tables
CREATE TABLE airport_full_load (
    id          INT,
    airport_name VARCHAR(100),
    city        VARCHAR(100),
    updated_at  TIMESTAMP
);

CREATE TABLE airport_incremental (
    id          INT,
    airport_name VARCHAR(100),
    city        VARCHAR(100),
    updated_at  TIMESTAMP
);
```

---

### Step 8 — Run Full Load DMS Task

Created a DMS task named `mysql-to-s3-task`:

| Setting | Value |
|---|---|
| Migration Type | Migrate existing data |
| Schema | `airport` |
| Table | `%` (all tables) |
| Action | Include |

**Result:** `Load completed` — all tables successfully migrated to S3.

---

### Step 9 — Run Full Load + CDC Incremental Task

Created a second DMS task named `incremental-flow`:

| Setting | Value |
|---|---|
| Migration Type | Migrate existing data and replicate ongoing changes |
| Target Preparation Mode | Do Nothing |
| Stop Task After Full Load | Do Not Stop |
| CDC Stop Mode | Disable Custom CDC Stop Mode |
| Replication Duration | Indefinitely |
| Schema | `airport` |
| Table | `%` (all tables) |

**Task Status:** `Load completed, replicating` ✅

This confirmed that CDC replication was active and listening for changes.

---

### Step 10 — Enable CDC in MySQL (Binary Logging)

To allow AWS DMS to read MySQL binary logs for CDC, the following RDS parameters were configured:

```
binlog_format     = ROW
binlog_row_image  = FULL
backup_retention  > 0
```

These settings instruct MySQL to write complete row-level change events to the binary log, which DMS consumes to replicate changes to S3.

---

### Step 11 — Simulate Incremental Changes

After the CDC task was running, the following changes were made to test replication:

```sql
-- Disable safe update mode
SET sql_safe_updates = 0;

-- UPDATE: Modify an existing record
UPDATE source_airport
SET city = 'Agra'
WHERE id = 2;

-- INSERT: Add a new record
INSERT INTO source_airport VALUES
    (9, 'Jaipur Airport', 'Jaipur', NOW());
```

---

### Step 12 — Verify CDC Output in Amazon S3

After executing the changes, AWS DMS automatically generated new CDC files in:

```
s3://airport-ved/dms-incremental-output/
```

✅ New files appeared within seconds, confirming:
- CDC was capturing MySQL binary log events
- Changes were being replicated in near real-time
- The pipeline was fully operational end-to-end

---

## ⚠️ Errors Encountered & Solutions

### ❌ Error 1 — DMS Connection Timeout

```
Test endpoint connection failed:
Can't connect to MySQL server on 'airport-db...'
```

**Root Cause:** Security group was blocking port 3306.

**Fix:**
- Added inbound rule for port `3306` in the RDS security group
- Enabled Public Access on the RDS instance
- Ensured DMS and RDS shared the same VPC

---

### ❌ Error 2 — Safe Update Mode Rejection

```
Error Code: 1175
You are using safe update mode and tried to update a table without WHERE using a KEY column.
```

**Fix:**
```sql
SET sql_safe_updates = 0;
```

---

### ❌ Error 3 — Invalid S3 Folder Path (Trailing Slash)

```
ResultLocationFolder should not end with '/'
```

| | Path |
|---|---|
| ❌ Wrong | `s3://airport-ved/dms-incremental-output/` |
| ✅ Correct | `s3://airport-ved/dms-incremental-output` |

---

### ❌ Error 4 — Duplicate Table Mapping Rules

**Problem:** Multiple duplicate `include` rules were created in the table mapping, causing task configuration errors.

**Fix:** Retain only a single selection rule:

| Field | Value |
|---|---|
| Schema | `airport` |
| Table | `%` |
| Action | Include |

---

## ✅ Final Outcome

Successfully built and validated a complete **MySQL → AWS DMS → S3 CDC Pipeline**:

| Feature | Status |
|---|---|
| Full Load Migration | ✅ Completed |
| Incremental CDC Replication | ✅ Active |
| Real-time UPDATE capture | ✅ Verified |
| Real-time INSERT capture | ✅ Verified |
| Automatic S3 file generation | ✅ Working |
| End-to-end AWS integration | ✅ Complete |

---

## 📚 Concepts Covered

- **AWS DMS Architecture** — Replication instances, source/target endpoints, and task types
- **Change Data Capture (CDC)** — How DMS reads MySQL binary logs to detect row-level changes
- **MySQL Binary Logging** — `binlog_format=ROW` and `binlog_row_image=FULL` configuration
- **Full Load vs Incremental Load** — When and why to use each migration type
- **IAM Roles & Policies** — Granting DMS permission to write to S3
- **VPC & Security Groups** — Network configuration for RDS and DMS connectivity
- **Amazon S3 as a Data Lake** — Using S3 as a raw CDC data store

---

## 🔭 Future Enhancements

```
MySQL → DMS → S3
                └── AWS Glue Crawler → Glue Data Catalog
                                             └── Amazon Athena (SQL queries on S3)
                                             └── Amazon Redshift (data warehouse)
                                             └── Power BI / QuickSight (dashboards)
                └── Apache Spark (large-scale transformations)
                └── Snowflake (cloud data warehouse)
```

Planned improvements:
- Integrate **AWS Glue** crawlers to auto-catalog CDC files in S3
- Query S3 data directly using **Amazon Athena**
- Load processed data into **Snowflake** or **Amazon Redshift**
- Build real-time dashboards using **Amazon QuickSight** or **Power BI**
- Process CDC files at scale with **Apache Spark on EMR**

---

## 🧹 Cleanup (Avoid Unexpected AWS Charges)

After testing, delete resources in this order to avoid ongoing charges:

```
1. Stop / Delete DMS Tasks         (incremental-flow, mysql-to-s3-task)
2. Delete DMS Endpoints            (airport-db, airport-db-end)
3. Delete DMS Replication Instance (airport-dms)
4. Stop / Delete RDS Instance      (airport-db)
5. Empty and delete S3 Bucket      (airport-ved) — if no longer needed
6. Delete IAM Role                 (dms-vpc-role) — if not reused
```

> ⚠️ DMS Replication Instances and RDS instances incur hourly charges even when idle. Always stop or delete them after testing.

---

## 👤 Author

**Vedang Sharma**
Data Engineering Project — AWS DMS CDC Pipeline

---

<p align="center">
  Built with ☁️ on AWS &nbsp;|&nbsp; Real-time Data Engineering &nbsp;|&nbsp; CDC Pipeline
</p>
