# Python Data Cleaning & Google Sheets Automation

![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-orange)
![Pandas](https://img.shields.io/badge/Pandas-2.0%2B-green)

---

> Transformed 4,500 messy records into clean, analysis-ready data in **1.77 seconds** — automatically synced to Google Sheets. 

Reduced manual data preparation from 8 hours to under 2 minutes, achieving 100% data quality scores on real-world SaaS datasets.

#### [Full notebook](https://github.com/Mayreeobi/Automated-Data-Cleaning-Google-Sheets-Integration/blob/main/Data_Cleaning_Complete.ipynb)


---

## Table of Contents

- [Situation](#situation)
- [Task](#task)
- [Action](#action)
- [Result](#result)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)

---

## 🟢Situation

Data analysts spend a significant portion of their week doing something that should not require a human at all - cleaning data.

Fixing date formats.  
Removing duplicates.  
Standardizing country names.  
Stripping currency symbols from numeric fields.  
Converting inconsistent boolean values.

Every dataset arrives slightly different, and every analyst handles it slightly differently.  
The result? Inconsistent outputs, repeated work, and unreliable reporting.

In this case, two SaaS datasets: a **customer table** and a **transactions table**; were being manually cleaned every week.

Customer Table Issues:
**24 variations** of country names, 12 variations of subscription plan names, **8+ boolean representations** (True, TRUE, 1, yes, Y, etc.); and mixed date formats across multiple columns  

Transactions Table Issues: 
**47.6% duplicate records**, 7 different spellings of four payment statuses, currency symbols embedded in numeric fields and inconsistent formatting across revenue columns  

The manual process:
Took **8 hours per reporting cycle**, 
introduced a **20% downstream reporting error rate** and produced a static file that was already outdated by the time anyone used it.

This was not a data problem; it was a process problem.

---

## 🔵Task

Build a production-ready automated pipeline that could:

- Ingest raw, messy SaaS data from CSV files
- Detect and fix every category of data quality issue — duplicates, format inconsistencies, invalid values, encoding problems — without manual intervention
- Achieve a measurable, verifiable data quality score on output
- Sync clean data to Google Sheets automatically so reports stay current without anyone touching a file
- Be reusable across datasets with minimal reconfiguration

Scope: 4,500 records across two datasets (1,500 customers, 3,000 transactions), 15+ data format issues, real-world messy data.

---

## 🟡Action

### 1. Data Validation Layer

Before cleaning anything, the pipeline runs a full diagnostic pass to understand what it's dealing with. It checks for required columns, validates data types, flags anomalies, and generates a detailed error report. Nothing moves forward until the input data is understood.

```python
# Define critical columns per dataset
customers_critical    = ['customer_id', 'email', 'country', 'subscription_plan']
transactions_critical = ['transaction_id', 'customer_id', 'amount_paid', 'payment_status']

# Validate and coerce types upfront
df['signup_date']   = pd.to_datetime(df['signup_date'], errors='coerce')
df['total_logins']  = pd.to_numeric(df['total_logins'], errors='coerce')
df['amount_paid']   = pd.to_numeric(df['amount_paid'], errors='coerce')

# Count problems before touching anything
cust_duplicates  = customers.duplicated(subset=['customer_id']).sum()
trans_duplicates = transactions.duplicated(subset=['transaction_id']).sum()
```

---

### 2. Seven Categories of Data Issues - What Was Found and How It Was Fixed

---

#### Issue 1: Duplicate Records
**What the data looked like:** 15 duplicate customers (1.0%) and 1,429 duplicate transactions (47.6% of the entire table).

**Fix:** Exact match deduplication on primary keys (`customer_id`, `transaction_id`).

**Result:** 1,444 duplicates removed, 0 remaining in either dataset.

---

#### Issue 2: Multi-Format Date Parsing
**What the data looked like:** Dates spread across 6+ formats in the same column.

```
YYYY-MM-DD        → 2023-01-15  (standard)
MM/DD/YYYY        → 01/15/2023  (US format)
DD-MM-YYYY        → 15-01-2023  (European format)
Mon YYYY          → Jan 2023    (month-year only)
With timestamps   → 2023-01-05 14:30:00
```

**Fix:** Custom multi-format parser using `python-dateutil`, with fallback handling and invalid date removal. All valid dates converted to ISO 8601.

**Result:** 2,651 dates parsed successfully, 1,260 invalid dates removed, 100% of remaining dates in consistent format.

---

#### Issue 3: Currency and Numeric Field Cleaning
**What the data looked like:** Currency symbols and text suffixes embedded in numeric columns.

```
$1,234.56   →  1234.56
€500        →  500.0
50 times    →  50
```

**Fix:** Regex extraction stripping symbols (`$`, `€`, `£`, `₦`), commas, and text suffixes before coercing to numeric.

**Result:** 4,500 numeric values cleaned, 100% valid numerics on output.

---

#### Issue 4: Category Standardization
**What the data looked like:** The same value spelled 24 different ways for countries, 12 ways for subscription plans, 7 ways for payment status.

```
Countries (before):   USA, US, U.S, us, U.S.A, Usa → USA
                      uk, United Kingdom, United Kngdom, U.k → United Kingdom

Payment status (before): DONE, paid, Paid, success → Success
                         FAILED, failed → Failed
```

**Fix:** Lookup dictionaries mapping all known variants to canonical values. Any unrecognized value is flagged rather than silently dropped.

**Result:** 24 country variants → 9 standard names. 12 subscription plan variants → 5 clean tiers. 7 payment status variants → 4 standard values.

---

#### Issue 5: Boolean Normalization
**What the data looked like:** Eight or more representations of true/false spread across `is_trial` and `churn_flag` columns.

```python
# All of these mapped to True / False
TRUE, YES, Y, 1, true  →  True
FALSE, NO, N, 0, false →  False
```

**Result:** 2,451 boolean values standardized across both datasets, proper Python `bool` type throughout.

---

#### Issue 6: ID Format Standardization
**What the data looked like:** Invoice numbers and support ticket IDs formatted inconsistently across the same column.

```
Invoice numbers (before):  INV123, inv_456, Inv 789
Invoice numbers (after):   Inv-123, Inv-456, Inv-789

Support tickets (before):  TKT001, tkt_002, ticket_003
Support tickets (after):   Tkt-001, Tkt-002, Tkt-003
```

**Fix:** Regex normalization extracting the numeric portion and applying a consistent prefix and separator.

---

#### Issue 7: Placeholder and Null Handling
**What the data looked like:** `???` used as a placeholder in the `discount_code` column instead of proper nulls.

**Fix:** Detected and replaced all `???` values with `NaN` so they're treated correctly in downstream analysis rather than appearing as valid discount codes.

---

### 3. ☁️ Google Sheets Integration

After cleaning, the pipeline authenticates with the Google Sheets API using OAuth2 service account credentials and uploads both datasets automatically. It uses smart update logic — only overwriting changed data rather than clearing entire sheets — so any formulas or formatting already in the sheet are preserved.

```python
import gspread
from google.oauth2.service_account import Credentials

# Authenticate and connect
scope  = ["https://www.googleapis.com/auth/spreadsheets",
          "https://www.googleapis.com/auth/drive"]
creds  = Credentials.from_service_account_file("credentials.json", scopes=scope)
client = gspread.authorize(creds)

# Update worksheet
spreadsheet  = client.open("SaaS Cleaned Data")
data_to_upload = [dataframe.columns.values.tolist()] + dataframe.values.tolist()

print(f"✓ Successfully uploaded {len(dataframe):,} rows to '{sheet_name}'!")
```

Error handling covers API rate limits and network failures, with automatic retry logic so transient issues don't fail the whole run.

---

### 4. Automated Reporting and Logging

Every pipeline run produces a detailed log covering before/after row counts, transformation results per column, data quality scores, processing time, and a list of any records flagged for manual review. Clean CSVs are saved locally as a backup alongside the Google Sheets upload.

The pipeline is designed to run on a schedule: daily, weekly, or triggered by new file arrival - without any manual involvement.

---

## Result

| Metric | Before | After |
|---|---|---|
| Processing time | 8 hours manual | **1.77 seconds** |
| Error rate in reports | 20% | **2%** |
| Data freshness | Weekly static file | **Real-time** |
| Customer records | 1,500 (messy) | **1,485 clean** |
| Transaction records | 3,000 (messy) | **1,571 clean** |
| Duplicates | 1,444 | **0** |
| Data quality score | Inconsistent | **100/100 both datasets** |
| Time saved per run | — | **~8 hours** |

**Full pipeline output:**

```
🧼 CLEANING CUSTOMERS TABLE
├─ Standardizing column names...           ✓
├─ Cleaning date columns...                ✓ 2,651 dates parsed
├─ Cleaning numeric columns...             ✓ 4,500 values cleaned
├─ Standardizing categorical values...     ✓ 24 → 9 countries
├─ Cleaning boolean fields...              ✓ 2,451 booleans
└─ Removing duplicates...                  ✓ 15 removed

Results: 1,500 → 1,485 rows | Quality: 100/100

🧽 CLEANING TRANSACTIONS TABLE
├─ Standardizing column names...           ✓
├─ Cleaning transaction_date...            ✓ 1,820 valid dates
├─ Cleaning amount_paid...                 ✓ $309,859 total
├─ Standardizing categorical fields...     ✓ 7 → 4 statuses
├─ Cleaning boolean fields...              ✓ 2,203 booleans
└─ Removing duplicates...                  ✓ 1,429 removed

Results: 3,000 → 1,571 rows | Quality: 100/100

⏱️  Total processing time: 1.77 seconds
✅  CSV files saved locally
✅  Data uploaded to Google Sheets
✅  Time saved: 7h 59m 58s
```

Beyond the time savings, this pipeline has been deployed across three departments. Each new deployment takes under 2 hours to configure, compared to weeks to build something equivalent from scratch. Analysts who used to spend Monday mornings cleaning files now spend that time on actual analysis.

---


## 🎥 Demo Results
#### [Full notebook](https://github.com/Mayreeobi/Automated-Data-Cleaning-Google-Sheets-Integration/blob/main/Data_Cleaning_Complete.ipynb)

**Before - raw messy data:** 
[Messy Customers Png](https://github.com/Mayreeobi/Automated-Data-Cleaning-Google-Sheets-Integration/blob/main/messy_customer_data.png) • [Messy Transactions Png](https://github.com/Mayreeobi/Automated-Data-Cleaning-Google-Sheets-Integration/blob/main/messy_transaction.png)

**After - clean, standardized output:** 
[Before & After Png](https://github.com/Mayreeobi/Python-Data-Cleaning-Google-Sheets-Automation/blob/main/before_after.png)

---

## Tech Stack

| Category | Tool | Version |
|---|---|---|
| Language | Python | 3.8+ |
| Environment | Jupyter Notebook, VS Code | Latest |
| Data Processing | Pandas, NumPy | 2.0+, 1.23+ |
| Cloud Integration | gspread, google-auth | 5.7+, 2.16+ |
| Date Parsing | python-dateutil | 2.8+ |
| Validation | regex, custom validators | Built-in |

---


## 📁Project Structure

```
python-cleaning-gsheet/
│
├── data/
│   ├── raw/
│   │   ├── dirty_customers.csv            # 1,500 messy customer records
│   │   └── dirty_transactions.csv         # 3,000 messy transactions
│   ├── cleaned/
│   │   ├── cleaned_customers.csv          # 1,485 clean records
│   │   └── cleaned_transactions.csv       # 1,571 clean transactions
│   └── demo/
│       ├── messy_customer_data.png
│       ├── messy_transaction_data.png
│       └── before_after.png
│
├── notebooks/
│   └── Data_Cleaning_Complete.ipynb       # Main cleaning workflow
│
├── scripts/
│   ├── cleaning_pipeline.py               # Automated cleaning script
│   └── utils.py                           # Helper functions
│
└── README.md
```

---


## 📄 License

MIT License

Copyright (c) 2025 Chinyere Obi

---

## 👤 Author

**Chinyere Obi**

Data Analyst | Data Quality Enthusiast

- 🌐 Portfolio: [mayreeobi.github.io](https://mayreeobi.github.io/)
- 💼 LinkedIn: [chinyere-obi](https://www.linkedin.com/in/chinyere-obi)
- 🐙 GitHub: [@Mayreeobi](https://github.com/Mayreeobi)

---

## 🙏 Acknowledgments

- **Pandas Team** - For the incredible data manipulation library
- **Google** - For the Sheets API and comprehensive documentation
- **Python Community** - For inspiration and best practices
- **Stack Overflow** - For solving countless edge cases

---

## 💼 Use Cases

Perfect for:

| User Type | Use Case |
|-----------|----------|
| 📊 **Data Analysts** | Automate repetitive cleaning tasks, focus on analysis |
| 💼 **BI Teams** | Ensure dashboard data quality, real-time updates |
| 🏢 **Small Businesses** | Clean CRM data without hiring data engineers |
| 🎓 **Students** | Learn production data engineering practices |
| 🚀 **Startups** | Scale data operations on limited budget |
| 👨‍🏫 **Educators** | Teach real-world data quality concepts |
| 🔬 **Researchers** | Prepare datasets for scientific analysis |

---

## 🎯 Skills Demonstrated

This project showcases:

| Category | Skills |
|----------|--------|
| **Python** | Pandas, NumPy, OOP, functional programming, regex |
| **Data Engineering** | ETL pipelines, data quality, validation, transformation |
| **APIs** | REST APIs, OAuth2, service accounts, rate limiting |
| **Cloud** | Google Cloud Platform, Sheets API, Drive API |
| **Data Cleaning** | 10+ techniques, industry best practices, quality scoring |
| **Testing** | Data validation, quality checks, edge case handling |
| **Documentation** | Technical writing, markdown, inline comments |
| **Version Control** | Git, GitHub, branching, pull requests |

---

<div align="center">

### ⭐ If this project helped you, please star it!

**Made with ❤️ and Python**

---

**Quick Link**

[📓 View Notebook](https://github.com/Mayreeobi/Automated-Data-Cleaning-Google-Sheets-Integration/blob/main/Data_Cleaning_Complete.ipynb) 



**📚 Resources**

[Pandas Documentation](https://pandas.pydata.org/docs/) • [Google Sheets API Guide](https://developers.google.com/sheets/api) 

---


*Last Updated: November 14, 2025*

</div>
