# Tools and Reproduction Steps

Follow these steps to replicate and verify the results presented in this analysis.

#### Prerequisites

- **Docker** installed and running
- **Git** installed
- **Python** (v3.11 used)

#### Instruction

1. **Clone the Repository:**

```bash
git clone https://github.com/miri1337/imago-data-challenge.git
cd imago-data-challenge
```

2. **Put the CSVs with data to imago-data-challenge/data**

3. **Set SQL Server password in docker-compose.yml**

Replace placeholder value with your password.

4. **Start SQL Server in Docker:**

```bash
docker-compose up -d
```

Check if SQL Server is running:

```bash
docker ps
```

5. **Setup Python Environment:**

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

pip install -r requirements.txt
```

6. **Data Import to SQL Server:**

Run the Jupyter Notebook responsible for importing CSV data into the database:

```bash
jupyter notebook notebooks/analysis.ipynb
```

# Data Analysis & Understanding

#### Purpose
The goal here was to dive into the data provided, check its quality, and pinpoint any critical issues that might be impacting our revenue reporting.

#### Data Quality Check
First, I did some basic checks for missing values and duplicates to get a feel for data quality:

- **Positions (`Abrechnung_Positionen`)**:
  - Missing values:
    - Customer Number (`KdNr`): 1 missing
    - Net Amount (`Nettobetrag`): 1 missing
    - Image Number (`Bildnummer`): 1 missing
    - Invoice Date (`VerDatum`): 4 missing
  - No duplicates found

- **Invoices (`Abrechnung_Rechnungen`)**:
  - Missing values:
    - Gross Payment Amount (`ZahlungsbetragBrutto`): 1 missing
    - Additional Costs (`Summenebenkosten`): 2 missing
    - Payment Date (`Zahlungsdatum`): 399 missing (very significant)
  - No duplicate invoice numbers

- **Customers (`Abrechnung_Kunden`)**:
  - Missing values:
    - Region: 321 missing (potentially problematic if regional analysis is required)
  - No duplicate customer IDs

#### Current Issues
After initial checks, I ran deeper SQL analyses (see Jupyter notebook for SQL queries) to uncover significant data problems:

1. **Positions Linked to Invoices Without Payments**
   - **Total Count:** 18,011 positions
   - **Analysis:** This is a big concern. Almost half of positions are attached to invoices that haven’t recorded payment dates. This situation significantly affects cash-flow insights and revenue accuracy.

2. **Revenue Under Placeholder Media ID (`Bildnummer` = 100000000)**
   - **Total Revenue:** €1,319,897.91
   - **Analysis:** Ovver a million euros categorized under a placeholder ID means we have either data-entry mistakes or major process issues at play. This directly impacts revenue accuracy and reporting reliability.

3. **Invoices Without Linked Positions**
   - **Total Count:** Only 2 invoices
   - **Analysis:** Although the count is minimal, this points to possible process lapses, as invoices typically should always be linked to one or more positions. This needs to be addressed to ensure data consistency.

#### Observations
- **Data Quality Concerns:** It's clear that there are significant issues that will skew revenue and financial reports. The volume of unpaid invoices and improperly classified revenue is alarming.
- **Operational Implications:** These issues will cause unreliable financial reporting, directly impacting business decision-making.
- **Recommended Immediate Actions:**
  - Introduce stricter validation steps in ETL processes.
  - Coordinate closely with finance and operational teams to address these issues at their sources.
  - Consider implementing automated data-quality checks and alerts to catch these issues earlier.

# Pipeline Proposal

#### Understanding the Problem
The analysis clearly highlights that issues with data quality, particularly regarding missing payments, placeholder media identifiers, and inconsistent customer information, are causing inaccuracies in our revenue reporting. After examining the data closely, it’s clear that these issues originate from how data is entered and processed through our current ETL pipeline.

#### Recommended ETL Improvements

To enhance the reliability, transparency, and accuracy of the data pipeline, the following improvements are proposed:

1. **Handling of Missing or Delayed Payments:**

   Currently, invoices without recorded payment dates (`Zahlungsdatum`) or gross payment amounts (`ZahlungsbetragBrutto`) significantly skew revenue reporting. To tackle this:

   - Introduce a dedicated flag (`IsPaymentMissing`) within the `Abrechnung_Rechnungen` table:

   ```sql
   -- Adding flag for tracking payment issues
   ALTER TABLE Abrechnung_Rechnungen ADD IsPaymentMissing BIT DEFAULT 0;

   UPDATE Abrechnung_Rechnungen
   SET IsPaymentMissing = 1
   WHERE Zahlungsdatum IS NULL OR ZahlungsbetragBrutto IS NULL;
   ```

   This explicit flag allows for easy identification and targeted follow-ups on unpaid or improperly documented invoices, thereby reducing ambiguity.

2. **Handling Placeholder Media IDs Clearly:**

   We have identified substantial revenue linked to a placeholder media ID (`Bildnummer = 100000000`). To ensure these are properly addressed:

   - Introduce a flag (`IsPlaceholderMedia`) directly in the `Abrechnung_Positionen` table:

   ```sql
   -- Flagging placeholder media entries
   ALTER TABLE Abrechnung_Positionen ADD IsPlaceholderMedia BIT DEFAULT 0;

   UPDATE Abrechnung_Positionen
   SET IsPlaceholderMedia = 1
   WHERE Bildnummer = 100000000;
   ```

   With these clearly flagged, placeholder entries can be separately analyzed and corrected, maintaining accurate reporting of revenue and media utilization.

3. **Strengthening Upstream Data Validation:**

   Ensuring data quality at the earliest stage of data ingestion reduces errors downstream:

   - Validate the integrity of the foreign keys (invoice references `ReId` and customer IDs `KdNr`) as part of the ETL:

   ```sql
   -- Identifying positions with invalid or missing references
   SELECT *
   FROM Abrechnung_Positionen
   WHERE ReId IS NULL OR ReId NOT IN (SELECT DISTINCT ReNummer FROM Abrechnung_Rechnungen)
      OR KdNr IS NULL OR KdNr NOT IN (SELECT DISTINCT KdNr FROM Abrechnung_Kunden);
   ```

   - Automate these checks and create a dedicated logging mechanism to handle problematic records. Any records failing these checks should trigger alerts to the responsible teams.

#### Recommended Data Model Adjustments

The refined data model with the new flags integrated will look like this:

```
Abrechnung_Rechnungen
├── ReNummer (Primary Key)
├── ZahlungsbetragBrutto
├── Zahlungsdatum
├── IsPaymentMissing (NEW)

Abrechnung_Positionen
├── id (Primary Key)
├── ReId (Foreign Key)
├── Bildnummer
├── Nettobetrag
├── IsPlaceholderMedia (NEW)

Abrechnung_Kunden
├── id (Primary Key)
├── KdNr
├── Region
```

#### Assumptions Made
- Entries with `Bildnummer` as `100000000` represent temporary placeholders rather than genuine media IDs.
- Missing or null payment details (`Zahlungsdatum`, `ZahlungsbetragBrutto`) indicate delayed, disputed, or otherwise problematic invoices rather than simply missing data entries.

#### Impact on Downstream Reporting

By implementing these improvements:

- **Accuracy in Reporting:** Reports generated from dashboards like Tableau become reliable, directly improving trust and decision-making capabilities across the company.
- **Operational Efficiency:** Automated flags and validations streamline the detection and correction of anomalies, significantly reducing manual review efforts.
- **Improved Financial Oversight:** Explicit flags enhance financial transparency, allowing finance teams to prioritize actions on outstanding payments and misplaced revenues.

#### Essential Business Conversations

To successfully implement these changes, recommended discussions:

- **Finance Team:**
  - Clarify the root cause behind missing payments. Are payments truly delayed or lost, or is the issue simply poor documentation? Adjust processes accordingly.
  - Develop policies or guidelines for handling unresolved or delayed payment entries.

- **Backoffice and Data Entry Teams:**
  - Investigate the frequent use of placeholder media IDs. Are they temporary or indicative of process misunderstandings?

#### Next Steps
- Start with phased implementation to minimize operational disruption.
- Continuously monitor, review, and adjust the process based on real-time feedback and results.

# Modern Tooling Strategy

### Overview
To modernize the existing SSIS-based ETL pipeline and build a future-proof data infrastructure, I propose a phased migration to a modern data stack based on dbt (Data Build Tool), Apache Airflow, and optionally Snowflake. The aim is to improve maintainability, scalability, transparency, and developer experience, while reducing risk of errors in revenue reporting.

## Migration Strategy

### 1. dbt (Data Build Tool)
**Purpose:** Transform raw data into structured, validated models using SQL. dbt brings version control, modular logic, and automated data testing.

**Implementation Plan:**
- Replace existing SSIS transformation logic with modular dbt models.
- Define staging, intermediate, and mart layers for greater clarity and traceability.
- Introduce `dbt tests` for nulls, unique constraints, and referential integrity.

**Detailed Example of dbt Models:**

#### `models/staging/stg_invoices.sql`
```sql
SELECT
    ReNummer AS invoice_id,
    KdNr AS customer_id,
    Zahlungsdatum,
    ZahlungsbetragBrutto,
    SummeNetto,
    MwStSatz,
    Summenebenkosten
FROM {{ source('raw', 'invoices') }}
```

#### `models/staging/stg_positions.sql`
```sql
SELECT
    id AS position_id,
    ReId AS invoice_id,
    KdNr AS customer_id,
    Nettobetrag,
    Bildnummer,
    VerDatum
FROM {{ source('raw', 'positions') }}
```

#### `models/intermediate/int_flagged_positions.sql`
```sql
SELECT *,
    CASE WHEN Bildnummer = 100000000 THEN TRUE ELSE FALSE END AS is_placeholder_media
FROM {{ ref('stg_positions') }}
```

#### `models/intermediate/int_missing_payments.sql`
```sql
SELECT *,
    CASE WHEN Zahlungsdatum IS NULL OR ZahlungsbetragBrutto IS NULL THEN TRUE ELSE FALSE END AS is_payment_missing
FROM {{ ref('stg_invoices') }}
```

#### `models/marts/revenue_metrics.sql`
```sql
SELECT
    c.Verlagsname,
    c.Region,
    i.invoice_id,
    p.Nettobetrag,
    p.is_placeholder_media,
    i.is_payment_missing
FROM {{ ref('int_missing_payments') }} i
JOIN {{ ref('int_flagged_positions') }} p ON i.invoice_id = p.invoice_id
JOIN {{ ref('stg_customers') }} c ON i.customer_id = c.Kdnr
```

**Folder Structure:**
```
dbt/
├── models/
│   ├── staging/
│   │   └── stg_*.sql
│   ├── intermediate/
│   │   └── int_*.sql
│   └── marts/
│       └── revenue_metrics.sql
├── seeds/
├── snapshots/
├── dbt_project.yml
├── tests/
│   └── schema.yml  # constraints, referential tests
```

### 2. Apache Airflow
**Purpose:** Schedule and orchestrate tasks and dependencies with observability.

**How to Use It:**
- Use DAGs to schedule file ingestion, dbt model runs, and validations.
- Implement alerting for task failures or validation anomalies.
- Enable retries, conditional logic, and SLA monitoring.

**Proposed DAG Flow:**
```
start → ingest_raw_csv → run_dbt_staging → run_dbt_marts → run_tests → notify_teams
```

### 3. Snowflake (Optional Future Move)
**Purpose:** Replace SQL Server with a scalable cloud-native data warehouse.

**Migration Plan:**
- Mirror core tables from SQL Server into Snowflake using CDC or batch replication.
- Compare query performance, run cost benchmarks.
- Gradually switch dbt's target to Snowflake for transformation models.

**Why Consider Snowflake:**
- Decouples compute and storage.
- Integrates well with dbt and Airflow.
- Reduces on-prem maintenance.


## What Should Stay for Now
- **Data ingestion from flat files (CSV/Excel):** Still necessary until upstream systems are upgraded.
- **SQL Server as storage:** Can remain during transition. Migration to Snowflake should only occur after validation.
- **Manual overrides:** Business rules for handling missing or incorrect data will still need some manual input and oversight during early phases.


### Risks and Mitigations

**1. Disrupting Existing Reporting**
- Risk: Changing logic could break existing dashboards.
- Mitigation: Run new pipeline in parallel for validation before switching production.

**2. Cloud Costs (Snowflake)**
- Risk: Unmanaged compute use can lead to high costs.
- Mitigation: Use role-based access, auto-suspend warehouses, and strict resource monitoring.

**3. Migrating Bad Data**
- Risk: Garbage data may get migrated.
- Mitigation: Use dbt tests and validation layers to prevent contaminated data from reaching reporting systems.

---

### Summary
Switching to dbt and Airflow improves modularity, tracking, and data quality right away. Adding Snowflake helps scale and cuts down on maintenance. With a solid plan for models, DAGs, and validation, the move can happen gradually without disrupting key reports.





