### Global Energy Price Data Engineering Platform Project Explanation Guide

> **Audience:** Data Engineering graduates building a portfolio-ready, production-style project.  
> **Important:** This guide explains *what to build, why it matters, and how each part should work* so you can implement it yourselves. It is intentionally instructional and not a done-for-you codebase.

---

#### 1) Project Overview

The **Global Energy Price Data Platform** is an end-to-end data engineering project that simulates a real analytics backend for an energy intelligence startup.

Your platform should:
- ingest global energy price data (fuel, electricity, natural gas) from the **Global Petrol Prices API**,
- process and validate the data,
- store curated datasets in **MongoDB Atlas**,
- orchestrate workflows with **Apache Airflow**,
- and expose reporting via a **Flask dashboard/API**.

This project demonstrates the complete lifecycle expected from a data engineer:
**source integration → pipeline orchestration → quality controls → storage modeling → analytics delivery**.

---

#### 2) Business Problem (Why this exists)

Energy prices are highly dynamic and influence:
- household cost of living,
- transportation and logistics spending,
- national inflation trends,
- and business forecasting.

A global platform needs to answer questions like:
- Which countries saw the biggest fuel price increase in the last 24 hours?
- How do electricity prices compare by region?
- Are there anomalies (e.g., negative values, stale records, duplicate snapshots)?

Without a proper data platform, teams end up with manual CSVs, inconsistent updates, and no trustworthy reporting.

Your project solves this by creating a **repeatable, observable, and scalable pipeline**.

---

#### 3) Data Source Explanation

##### Primary source
- **Global Petrol Prices API**: https://www.globalpetrolprices.com/data_access.php

##### What you should expect
Depending on subscription and endpoint access, the provider may expose data in:
- XML,
- Excel files,
- or other structured payloads.

##### Expected fields (conceptually)
Your ingestion layer should normalize fields such as:
- `country`
- `product_type` (fuel / electricity / natural_gas)
- `price`
- `currency`
- `unit`
- `reporting_date`
- `source`
- metadata (request timestamp, batch id)

##### Historical vs Incremental
- **Historical load**: bulk backfill for older dates.
- **Incremental load**: only new/updated records since last successful run.

A robust design supports both.

---

#### 4) High-Level Architecture (Text Walkthrough)

You should structure the system in this logical sequence:

1. **Ingestion service** calls external API and captures raw payload.
2. **Raw zone loader** stores untouched records in MongoDB (`raw_api_data`).
3. **Validation layer** checks schema, nulls, duplicates, negative values.
4. **Transformation layer** standardizes fields, dates, currency metadata, product mapping.
5. **Curated loader** writes clean records to product-specific collections.
6. **Orchestrator (Airflow)** schedules and monitors each task with retries.
7. **Reporting service** computes 24-hour summaries.
8. **Flask dashboard/API** serves latest prices and report endpoints.

Think in zones:
- **Raw** (immutable, auditable)
- **Curated** (analytics-ready)
- **Operational metadata** (pipeline runs + quality checks)

---

#### 5) Technology Stack (and why)

- **Python**: ecosystem maturity for data APIs, ETL, orchestration.
- **Requests / lxml / openpyxl / pandas**: robust parsing + transformation stack.
- **MongoDB Atlas**: flexible schema for semi-structured source variations.
- **Apache Airflow**: production-grade scheduling, retries, task dependencies, observability.
- **Flask**: lightweight service layer for reporting endpoints and simple UI.
- **python-dotenv**: secure config via environment variables.

This combination mirrors common startup-scale and mid-size production environments.

---

#### 6) Recommended Project Structure (What each part should do)

Use this as your target scaffold:

```text
project_root/
│
├── airflow/
│   └── dags/
│       └── energy_pipeline_dag.py
│
├── app/
│   ├── ingestion/
│   ├── transformation/
│   ├── validation/
│   ├── loaders/
│   ├── services/
│   └── utils/
│
├── flask_app/
│   ├── templates/
│   ├── static/
│   ├── routes/
│   └── app.py
│
├── config/
├── tests/
├── requirements.txt
├── .env.example
├── docker-compose.yml
└── README.md
```

##### Folder intent
- `app/ingestion`: API client + response parsing + ingestion metadata.
- `app/transformation`: field normalization, date formatting, schema mapping.
- `app/validation`: data quality rules and structured results.
- `app/loaders`: MongoDB writers for raw + curated collections.
- `app/services`: reporting logic (24-hour summaries, trend metrics).
- `app/utils`: logging config, common helpers, constants.
- `airflow/dags`: orchestration DAG(s).
- `flask_app/routes`: HTTP endpoints and business-response formatting.
- `flask_app/templates` + `static`: dashboard HTML/CSS.
- `tests`: unit + integration tests (mock API + test DB).

---

#### 7) MongoDB Data Modeling Strategy

Create these collections:

1. `raw_api_data`
2. `fuel_prices`
3. `electricity_prices`
4. `natural_gas_prices`
5. `pipeline_runs`
6. `data_quality_checks`

### Core document contract (minimum)
Every price record should carry:
- `country`
- `product_type`
- `price`
- `currency`
- `unit`
- `source`
- `reporting_date`
- `ingestion_timestamp`
- `batch_id`

##### Why split by product collections?
- Faster product-specific queries.
- Cleaner index strategy.
- Easier operational ownership for downstream reporting.

##### Suggested indexes
- On curated collections:
  - `(country, reporting_date, product_type)` compound index
  - `batch_id` index for traceability
- On raw collection:
  - `ingestion_timestamp`
- On metadata collections:
  - `pipeline_run_id`, `dag_run_id`, `created_at`

##### Deduplication key idea
Use a deterministic business key, e.g.:
`country + product_type + reporting_date + unit + source`

Perform upsert/merge behavior so reruns are idempotent.

---

#### 8) Airflow Orchestration Design

Your DAG should represent a strict data lifecycle:

1. `ingest_data`
2. `validate_data`
3. `transform_data`
4. `load_to_mongodb`
5. `generate_report`

##### Production-ready expectations
- retries + retry delay,
- task-level logging,
- failure visibility,
- run metadata written to `pipeline_runs`,
- quality outcomes stored in `data_quality_checks`.

#### Scheduling idea
- daily runs (or hourly for near-real-time scenarios),
- parameterized backfill for historical loads.

---

#### 9) Data Quality Strategy

At minimum, enforce these checks:
- **Null checks** on required fields,
- **Type checks** (price numeric, dates parseable),
- **Duplicate checks** based on business key,
- **Negative price checks**.

Your validator should return a structured object, for example:
- total records,
- valid count,
- invalid count,
- check-level error breakdown,
- sample failing records.

Quality results must be stored for auditability and future observability dashboards.

---

#### 10) Reporting Layer Design (24-hour summary)

Build a reporting service that computes:
- latest available snapshot per product,
- countries with max/min price movements in last 24 hours,
- count of records processed in latest successful run,
- data quality status summary.

This can be served by Flask endpoints and displayed in HTML tables.

---

#### 11) Flask Dashboard Overview

##### Expected routes
- `/` → dashboard page (HTML/CSS)
- `/api/prices` → latest curated prices (JSON)
- `/api/report` → 24-hour summary (JSON)

##### UI should include
- **Latest Prices table**,
- **24-hour Summary section**,
- **Pipeline status highlights** (optional but recommended).

Keep the UI simple and readable; your value is data correctness and clarity.

---

#### 12) Setup Guide (What students should implement)

> This section explains the setup flow you should include in your final implementation docs.

### A) Prerequisites
- Python 3.10+
- MongoDB Atlas account
- Airflow-compatible environment
- Access credentials for Global Petrol Prices API

### B) Environment variables

Create `.env` from `.env.example` and define:
- `MONGODB_URI`
- `API_USERNAME`
- `API_PASSWORD`
- `DATABASE_NAME`

##### C) MongoDB Atlas setup

1. Create cluster.
2. Create DB user with least-privilege access.
3. Add IP access (or temporary open for dev only).
4. Copy connection string.
5. Set connection string in `MONGODB_URI`.

##### D) Airflow local run (conceptual)
- initialize metadata DB,
- create admin user,
- start scheduler + webserver,
- enable your DAG and trigger.

##### E) Flask local run (conceptual)
- install dependencies,
- set env vars,
- run app,
- verify `/`, `/api/prices`, `/api/report`.

---

#### 13) Sample API / Query Expectations

##### Example `/api/prices` response shape
```json
{
  "as_of": "2026-04-20T00:00:00Z",
  "records": [
    {
      "country": "Germany",
      "product_type": "fuel",
      "price": 1.85,
      "currency": "EUR",
      "unit": "liter",
      "reporting_date": "2026-04-19"
    }
  ]
}
```

##### Example `/api/report` response shape
```json
{
  "run_id": "airflow_2026_04_20_01",
  "processed_records": 1240,
  "invalid_records": 17,
  "top_increase": [{"country": "CountryA", "delta": 0.12}],
  "top_decrease": [{"country": "CountryB", "delta": -0.09}]
}
```

##### Example MongoDB queries students should support
- latest price by country + product,
- 24-hour delta by country,
- pipeline run audit by date,
- validation failures by check type.

---

#### 14) Testing Strategy (What to prove)

Students should implement:
- **Unit tests** for transformation and validation functions.
- **Integration tests** for ingestion → Mongo load (with test DB).
- **DAG-level sanity checks** (import and dependency graph).
- **API tests** for Flask JSON endpoints.

Focus on correctness + idempotency + traceability.

---

#### 15) Limitations to Discuss in Final Submission

Common realistic limitations:
- API access quota or latency,
- currency conversion availability and refresh timing,
- historical backfill cost/time,
- schema drift from source provider,
- single-region deployment constraints.

Showing limitations demonstrates engineering maturity.

---

#### 16) Future Improvements (Roadmap thinking)

Strong next steps:
- add Great Expectations / Soda for richer data quality,
- move from batch to near-real-time streaming,
- containerize full stack and deploy to cloud,
- add CI/CD with automated tests and linting,
- implement role-based access + auth on dashboard,
- add anomaly detection models on price changes.

---

#### 17) What Recruiters / Reviewers Look For

When graduates present this project, reviewers care less about UI polish and more about:
- clear data contracts,
- reliability (retries/idempotency),
- quality checks and observability,
- thoughtful storage and indexing,
- maintainable module boundaries,
- meaningful documentation.

If you can explain trade-offs and operational behavior, you will stand out.

---

#### 18) Final Guidance for Students

Build this project in iterations:
1. working ingestion,
2. clean transformations,
3. dependable validation,
4. robust loading,
5. orchestrated scheduling,
6. clear reporting/API output.

Do not rush to “finish features.” Prioritize **data reliability** and **explainability**.


