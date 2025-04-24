### Tools and Reproduction Steps

Follow these steps to replicate and verify the results presented in this analysis.

#### Prerequisites

- **Docker** installed and running
- **Git** installed
- **Python** (v3.11 used)

#### Step-by-Step Instructions

1. **Clone the Repository:**

```bash
git clone https://github.com/miri1337/imago-data-challenge.git
cd imago-data-challenge
```

2. **Start SQL Server in Docker:**

```bash
docker-compose up -d
```

Check if SQL Server is running:

```bash
docker ps
```

3. **Setup Python Environment:**

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

pip install -r requirements.txt
```

4. **Data Import to SQL Server:**

Run the Jupyter Notebook responsible for importing CSV data into the database:

```bash
jupyter notebook notebooks/analysis.ipynb
```

### Data Analysis & Understanding

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

#### Deep Dive into Issues
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

#### Key Observations & Next Steps
- **Data Quality Concerns:** It's clear that there are significant issues that will skew revenue and financial reports. The volume of unpaid invoices and improperly classified revenue is alarming.
- **Operational Implications:** These issues will cause unreliable financial reporting, directly impacting business decision-making.
- **Recommended Immediate Actions:**
  - Introduce stricter validation steps in ETL processes.
  - Coordinate closely with finance and operational teams to address and rectify these issues at their sources.
  - Consider implementing automated data-quality checks and alerts to catch these issues earlier.

These insights will help us address the data pipeline issues, improve overall reporting accuracy and ensure we can do reliable business decisions more reliably.



