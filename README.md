# HIMS Data Warehouse - Analytical Database

A comprehensive relational data warehouse engineered to centralize hospital operations, financial transactions, and pharmacy inventory.

---

## Overview

The HIMS (Hospital Information and Management System) Data Warehouse is designed to facilitate evidence-based decision-making for healthcare administrators. By normalizing operational data into a unified Star Schema, this project solves immediate reporting bottlenecks and enables advanced SQL analytics.

---

## Problem Statement

In standard healthcare environments, operational data — ranging from appointments to billing — is frequently siloed, making it difficult for stakeholders to answer critical questions like "Which departments have the highest cancellation rates?" or "What is our 30-day patient revisit ratio?"

---

## Proposed Solution

This project architecst a centralized Data Warehouse that resolves data fragmentation by:

- Normalizing hospital entities into distinct Dimension (`dim_`) and Fact (`fact_`) tables.
- Enforcing strict data integrity via `UNIQUE` constraints (e.g., 1:1 mapping between consultations and invoices).
- Automating financial calculations using `GENERATED ALWAYS AS` columns.
- Implementing autonomous pharmacy stock tracking via SQLite `BEFORE/AFTER` Triggers.

---

## System Architecture

The database is built on a Star Schema architecture, prioritizing read performance and analytical efficiency.

```
dim_department ──(1:N)──> dim_doctor
dim_patient    ──(1:N)──> fact_appointment ──(1:1)──> fact_consultation
                                                            ├──(1:1)──> fact_invoice
dim_medication ─────────────────────────────────────────────└──(1:N)──> fact_prescription_detail
      │
      └──(1:N)──> ops_stock_alert
```

---

## Technology Stack

- **Database Engine:** SQLite (`PRAGMA foreign_keys = ON`)
- **Architecture:** Star Schema (Dimensional Modeling)
- **SQL Features:** CTEs, Window Functions (`LEAD`), Conditional Aggregation (`CASE WHEN`), Triggers, Generated Columns

---

## Core Features

- **Patient Revisit Tracking:** Calculates 7-day and 30-day patient return rates using Window Functions.
- **Department Occupancy Analytics:** Compares appointment completion vs. cancellation ratios across departments.
- **Financial Dashboards:** Monthly grouping of gross revenue, discounts, and net collections.
- **Autonomous Stock Alerts:** Automatically flags medication inventory when it drops below critical thresholds.

---

## Key Query Implementations

### 1. Patient Revisit Rate Analysis (CTE + Window Function)

```sql
WITH ordered AS (
    SELECT patient_key, datetime(appointment_datetime) AS appointment_dt,
           LEAD(datetime(appointment_datetime)) OVER (
               PARTITION BY patient_key ORDER BY datetime(appointment_datetime)
           ) AS next_appointment_dt
    FROM fact_appointment
    WHERE appointment_status IN ('COMPLETED', 'SCHEDULED')
)
SELECT COUNT(*) AS patient_count,
       ROUND(100.0 * SUM(CASE WHEN julianday(next_appointment_dt) - julianday(appointment_dt) <= 7
                         THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0), 2) AS revisit_7d_rate_pct
FROM ordered;
```

### 2. Department Occupancy (JOIN + Conditional Aggregation)

```sql
SELECT d.department_name,
       COUNT(a.appointment_key) AS total_appointments_90d,
       ROUND(100.0 * SUM(CASE WHEN a.appointment_status = 'COMPLETED' THEN 1 ELSE 0 END)
             / NULLIF(COUNT(a.appointment_key), 0), 2) AS completion_rate_pct
FROM dim_department d
LEFT JOIN fact_appointment a ON a.department_key = d.department_key
  AND datetime(a.appointment_datetime) >= datetime('now', '-90 day')
GROUP BY d.department_name;
```

### 3. Invoice Trend (strftime + Grouped Metrics)

```sql
SELECT CAST(strftime('%Y', invoice_date) AS INTEGER) AS invoice_year,
       CAST(strftime('%m', invoice_date) AS INTEGER) AS invoice_month,
       COUNT(*) AS invoice_count,
       ROUND(SUM(gross_amount), 2) AS gross_total,
       ROUND(100.0 * SUM(CASE WHEN payment_status = 'PAID' THEN 1 ELSE 0 END)
             / NULLIF(COUNT(*), 0), 2) AS paid_ratio_pct
FROM fact_invoice
GROUP BY CAST(strftime('%Y', invoice_date) AS INTEGER), CAST(strftime('%m', invoice_date) AS INTEGER);
```

---

## How to Run

1. Clone the repository
   git clone https://github.com/mertbayram1/HIMS-DATA-WAREHOUSE.git

2. Navigate to the project folder
   cd HIMS-DATA-WAREHOUSE

3. Run the schema file to create all tables, triggers, and views
   sqlite3 hbys_dwh.sqlite ".read 01_Setup_DDL/00_schema_hospital.sql"

4. (Optional) Load sample data
   sqlite3 hbys_dwh.sqlite ".read 02_Data_Load_DML/01_master_data.sql"
   sqlite3 hbys_dwh.sqlite ".read 02_Data_Load_DML/02_dim_patient.sql"
   sqlite3 hbys_dwh.sqlite ".read 02_Data_Load_DML/03_fact_appointment.sql"

5. Open the database
   sqlite3 hbys_dwh.sqlite
