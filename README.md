# 🏠 Airbnb End-to-End Data Engineering Pipeline

> **Stack:** Snowflake · dbt (Data Build Tool) · AWS S3 · Python 3.12

A complete, production-style data engineering pipeline built on Airbnb listings, bookings, and hosts data. The project implements a **Medallion Architecture** (Bronze → Silver → Gold) on Snowflake, with dbt handling all transformations — including incremental models, SCD Type 2 snapshots, custom Jinja macros, and a fully tested data lineage graph.

---

## 📐 Architecture

```
Raw CSVs  →  AWS S3  →  Snowflake Staging  →  Bronze  →  Silver  →  Gold
                                                  ↓           ↓          ↓
                                             Raw Tables  Cleaned     Analytics
```

| Layer      | Purpose                                       | Materialization     |
|------------|-----------------------------------------------|---------------------|
| **Bronze** | Raw ingest from staging, minimal transforms   | Table               |
| **Silver** | Cleaned, validated, standardised data          | Table (Incremental) |
| **Gold**   | Business-ready: OBT + Fact table              | Table + Ephemeral   |

---

## 🗂️ Project Structure

```
airbnb-snowflake-dbt-pipeline/
├── README.md
├── pyproject.toml                        # Python deps (dbt-snowflake, sqlfmt)
├── main.py
│
├── SourceData/                           # Raw CSV files for staging load
│   ├── bookings.csv
│   ├── hosts.csv
│   └── listings.csv
│
├── DDL/                                  # Snowflake table creation scripts
│   ├── ddl.sql
│   └── resources.sql
│
└── aws_dbt_snowflake_project/            # dbt project root
    ├── dbt_project.yml
    ├── ExampleProfiles.yml               # Template for ~/.dbt/profiles.yml
    │
    ├── models/
    │   ├── sources/
    │   │   └── sources.yml               # Snowflake source definitions
    │   ├── bronze/
    │   │   ├── bronze_bookings.sql
    │   │   ├── bronze_hosts.sql
    │   │   └── bronze_listings.sql
    │   ├── silver/
    │   │   ├── silver_bookings.sql
    │   │   ├── silver_hosts.sql
    │   │   └── silver_listings.sql
    │   └── gold/
    │       ├── fact.sql
    │       ├── obt.sql                   # One Big Table (Jinja loop joins)
    │       └── ephemeral/
    │           ├── bookings.sql
    │           ├── hosts.sql
    │           └── listings.sql
    │
    ├── macros/
    │   ├── generate_schema_name.sql      # Custom schema routing (bronze/silver/gold)
    │   ├── multiply.sql
    │   ├── tag.sql                       # Price categorisation (low/medium/high)
    │   └── trimmer.sql
    │
    ├── snapshots/                        # SCD Type 2 — historical tracking
    │   ├── dim_bookings.yml
    │   ├── dim_hosts.yml
    │   └── dim_listings.yml
    │
    ├── tests/
    │   └── source_tests.sql
    │
    └── analyses/                         # Ad-hoc Jinja/SQL exploration
        ├── explore.sql
        ├── if_else.sql
        └── loop.sql
```

---

## ✨ Key Features

### 1. Incremental Loading
Bronze and Silver models process only new or changed records on each run:
```sql
{{ config(materialized='incremental') }}
{% if is_incremental() %}
  WHERE CREATED_AT > (SELECT COALESCE(MAX(CREATED_AT), '1900-01-01') FROM {{ this }})
{% endif %}
```

### 2. Slowly Changing Dimensions (SCD Type 2)
Snapshots track the full history of hosts, listings, and bookings — valid-from/to dates are maintained automatically by dbt.

### 3. Custom Macros
- **`tag()`** — classifies price per night as `low`, `medium`, or `high`
- **`trimmer()`** — strips whitespace from string fields
- **`generate_schema_name()`** — routes each layer to its own Snowflake schema (`AIRBNB.BRONZE`, `AIRBNB.SILVER`, `AIRBNB.GOLD`)

### 4. One Big Table (OBT) with Jinja Loops
The Gold OBT model uses a dynamic Jinja `for` loop to join bookings, listings, and hosts — keeping SQL concise and maintainable as column lists grow.

### 5. dbt Tests & Data Quality
- Uniqueness and not-null checks on key columns
- Custom source validation tests
- Full lineage graph available via `dbt docs serve`

---

## 🚀 Getting Started

### Prerequisites
- Python 3.12+
- A Snowflake account (free trial works)
- An AWS account (for S3 staging)

### 1. Clone the repo
```bash
git clone https://github.com/Mansha0805/airbnb-snowflake-dbt-pipeline.git
cd airbnb-snowflake-dbt-pipeline
```

### 2. Set up Python environment
```bash
python -m venv .venv
source .venv/bin/activate        # Mac/Linux
.venv\Scripts\Activate.ps1       # Windows PowerShell

pip install -e .
```

### 3. Configure Snowflake connection
Copy the example profile and fill in your credentials:
```bash
cp aws_dbt_snowflake_project/ExampleProfiles.yml ~/.dbt/profiles.yml
# Edit ~/.dbt/profiles.yml with your Snowflake account, user, and password
```

> ⚠️ Never commit your real `profiles.yml` — it is already in `.gitignore`.

### 4. Set up Snowflake objects
Run `DDL/ddl.sql` in your Snowflake worksheet to create the database, schemas, and staging tables.

### 5. Load source data
Upload the CSVs from `SourceData/` to Snowflake staging:

| File           | Target                    |
|----------------|---------------------------|
| `bookings.csv` | `AIRBNB.STAGING.BOOKINGS` |
| `hosts.csv`    | `AIRBNB.STAGING.HOSTS`    |
| `listings.csv` | `AIRBNB.STAGING.LISTINGS` |

---

## 🔧 Running dbt

```bash
cd aws_dbt_snowflake_project

dbt debug           # Verify Snowflake connection
dbt deps            # Install dbt packages
dbt run             # Run all models
dbt test            # Run data quality tests
dbt snapshot        # Run SCD Type 2 snapshots
dbt docs generate   # Build lineage documentation
dbt docs serve      # Open docs in browser
```

Run a specific layer only:
```bash
dbt run --select bronze.*
dbt run --select silver.*
dbt run --select gold.*
```

Full refresh (rebuilds tables from scratch):
```bash
dbt run --full-refresh
```

---

## 📊 Data Model

### Bronze — Raw ingest
| Model             | Description                          |
|-------------------|--------------------------------------|
| `bronze_bookings` | Raw booking transactions from staging |
| `bronze_hosts`    | Raw host records                     |
| `bronze_listings` | Raw property listings                |

### Silver — Cleaned & validated
| Model            | Key Transforms                        |
|------------------|---------------------------------------|
| `silver_bookings`| Validated dates, type casting         |
| `silver_hosts`   | Quality flags, trimmed strings        |
| `silver_listings`| Price categorisation via `tag()` macro|

### Gold — Analytics-ready
| Model       | Description                                             |
|-------------|----------------------------------------------------------|
| `obt`       | One Big Table — denormalised join of all three entities  |
| `fact`      | Fact table for dimensional reporting                    |
| `ephemeral/*` | Intermediate CTE-style models (not materialised)      |

### Snapshots — Historical dimensions
`dim_bookings`, `dim_hosts`, `dim_listings` — SCD Type 2 tables with `dbt_valid_from` / `dbt_valid_to` columns.

---

## 🔐 Security Notes

- Credentials are never stored in the repo — use `~/.dbt/profiles.yml` locally or environment variables in CI/CD
- Role-based access control (RBAC) configured in Snowflake (`ACCOUNTADMIN` for setup; scoped roles recommended for production)

---

## 🛠️ Tech Stack

| Tool          | Version   | Purpose                  |
|---------------|-----------|--------------------------|
| Snowflake     | —         | Cloud data warehouse     |
| dbt-snowflake | ≥ 1.11.0  | Transformation layer     |
| dbt-core      | ≥ 1.11.2  | dbt engine               |
| AWS S3        | —         | Raw file staging         |
| Python        | 3.12+     | Environment & tooling    |
| sqlfmt        | ≥ 0.0.3   | SQL formatting           |

---

## 📚 Resources

- [dbt Documentation](https://docs.getdbt.com/)
- [Snowflake Documentation](https://docs.snowflake.com/)
- [dbt Best Practices Guide](https://docs.getdbt.com/guides/best-practices)

---

## 👤 Author

**Mansha** · [@Mansha0805](https://github.com/Mansha0805)

---

## 📝 License

This project is shared as part of a data engineering portfolio. Feel free to fork and adapt it for your own learning.
