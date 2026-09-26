# HDB Resale Flat Prices — Technical Assessment

## Overview

This project performs data quality assessment and transformation on historical Singapore HDB resale flat price datasets.

The workflow combines five source datasets into a single master dataset, profiles and cleans the data, calculates remaining lease as of **1 January 2027**, validates key categorical fields against the **January 2012 dataset as the strict authoritative reference**, identifies potentially anomalous resale prices using an IQR-based heuristic, and generates a unique **Resale Identifier** for the resulting records.

---

## Objectives

The notebook addresses the following data quality and transformation requirements:

1. Combine the five HDB resale price datasets into a single master dataset.
2. Perform data profiling on the combined dataset.
3. Remove duplicate and completely empty records.
4. Calculate remaining lease as of 1 January 2027, assuming a 99-year lease.
5. Validate `town`, `flat_type`, `flat_model`, and `storey_range` using January 2012 as the authoritative reference set.
6. Quarantine records that fail the categorical validation rules.
7. Identify potentially anomalous resale prices using an IQR-based heuristic.
8. Generate a Resale Identifier for the cleaned dataset.

---

## Source Datasets

The following five CSV files are combined:

| # | Source Dataset |
|---|---|
| 1 | `Resale Flat Prices (Based on Approval Date), 1990 - 1999.csv` |
| 2 | `Resale Flat Prices (Based on Approval Date), 2000 - Feb 2012.csv` |
| 3 | `Resale Flat Prices (Based on Registration Date), From Mar 2012 to Dec 2014.csv` |
| 4 | `Resale Flat Prices (Based on Registration Date), From Jan 2015 to Dec 2016.csv` |
| 5 | `Resale flat prices based on registration date from Jan-2017 onwards.csv` |

The expected standardised columns are:

```text
month
town
flat_type
block
street_name
storey_range
floor_area_sqm
flat_model
lease_commence_date
remaining_lease
resale_price
```

For older datasets where `remaining_lease` is not available, the column is added before the datasets are combined.

---

## Data Processing Workflow

```text
Source CSV files
       │
       ▼
Combine datasets
       │
       ▼
Master dataset
       │
       ▼
Data profiling
       │
       ▼
Remove duplicate / empty records
       │
       ▼
Calculate remaining lease
       │
       ▼
Jan 2012 authoritative validation
       │
       ├──────────────► Quarantined records
       │
       ▼
Validated records
       │
       ▼
IQR resale-price anomaly detection
       │
       ▼
Retain records classified as Normal
       │
       ▼
Generate Resale Identifier
```

---

# 1. Dataset Combination

The five source datasets are read using Pandas and standardised to the same column structure.

The datasets are concatenated into a single master dataset and sorted by `month`.

The resulting dataset is saved as:

```text
HDB_Resale_Prices_Master_Dataset.csv
```

### Cleaning applied during combination

- Ensures all source datasets contain the expected standard columns.
- Adds `remaining_lease` where it is absent from older source datasets.
- Combines all records into one DataFrame.
- Sorts the resulting data chronologically by `month`.

---

# 2. Data Profiling

A custom Pandas-based profiling function is used to assess the quality and characteristics of the combined dataset.

The profiling covers:

### Structural profiling

- Number of rows
- Number of columns
- Number of duplicate rows
- Memory usage
- Column names

### Completeness

For each column:

- Total records
- Non-null records
- Null records
- Null percentage
- Blank records
- Blank percentage

### Uniqueness

- Number of distinct values
- Distinct-value percentage
- Duplicate count
- Whether the non-null values are unique

### Numeric profiling

For numeric columns:

- Minimum
- Maximum
- Mean
- Median
- Standard deviation
- 25th percentile
- 50th percentile
- 75th percentile
- IQR-based outlier count
- IQR-based outlier percentage

### Text / categorical profiling

For text-based columns:

- Minimum string length
- Maximum string length
- Average string length
- Most frequent values
- Pattern checks

This provides mechanisms to identify completeness, uniqueness, distribution, and potential anomalous-value issues without relying on an external profiling package.

---

# 3. Basic Data Cleaning

After profiling, duplicate records and completely empty rows are removed.

```python
raw_cleaned = raw.drop_duplicates()
raw_cleaned = raw_cleaned.dropna(how='all')
```

This step removes:

- Exact duplicate rows
- Rows where every field is missing

Individual missing values are not automatically removed at this stage because they may require field-specific treatment.

---

# 4. Remaining Lease Calculation

The assessment requires the remaining lease to be calculated as of:

**1 January 2027**

The notebook assumes:

- Every HDB flat has a **99-year lease**.
- The `lease_commence_date` represents the commencement year.
- The lease commencement date is treated as **1 January of the commencement year**.
- Remaining lease is calculated using calendar years and months.
- The result is represented as:

```text
X years Y months
```

For example:

```text
84 years 0 months
```

The calculation uses `relativedelta` so that the result is expressed in years and months rather than simply as a number of days.

---

# 5. Data Validation Using January 2012

## Authoritative Reference Set

The **January 2012 dataset** is used as the strict authoritative reference set.

The following four fields are validated:

```text
town
flat_type
flat_model
storey_range
```

For each field, the unique values appearing in January 2012 are extracted.

For example:

```text
January 2012 towns
January 2012 flat types
January 2012 flat models
January 2012 storey ranges
```

Each record in the cleaned dataset is then checked against these reference sets.

### Validation rule

A record is considered valid only when **all four fields** contain values that exist in the corresponding January 2012 authoritative set.

Conceptually:

```text
Valid Record =
    Valid Town
AND Valid Flat Type
AND Valid Flat Model
AND Valid Storey Range
```

Records failing one or more rules are separated into a quarantine dataset.

### Why quarantine?

Invalid records are not immediately deleted because quarantine preserves the records for further investigation and provides visibility into potential data quality issues.

The notebook returns:

```text
Validated records
Quarantined records
```

---

# 6. Resale Price Anomaly Detection

Potentially anomalous resale prices are identified using the **Interquartile Range (IQR)** method.

Rather than applying one threshold to all HDB transactions, the calculation is performed within:

```text
town + flat_type
```

This provides a more context-specific comparison of resale prices.

For each `town` and `flat_type` group:

```text
Q1 = 25th percentile
Q3 = 75th percentile
IQR = Q3 - Q1
```

The anomaly boundaries are:

```text
Lower Bound = Q1 - 1.5 × IQR

Upper Bound = Q3 + 1.5 × IQR
```

A record is flagged when:

```text
resale_price < Lower Bound
```

or

```text
resale_price > Upper Bound
```

The notebook classifies the records as:

| Classification | Definition |
|---|---|
| `Normal` | Resale price falls within the IQR bounds |
| `Unusually High` | Resale price is above the upper IQR bound |
| `Unusually Low` | Resale price is below the lower IQR bound |

The anomaly detection is a **heuristic for identifying records that warrant further investigation**. An IQR outlier does not by itself prove that a transaction is incorrect.

The current notebook subsequently retains records classified as:

```text
Normal
```

for the Resale Identifier transformation.

---

# 7. Resale Identifier

A Resale Identifier is generated for the cleaned records.

The identifier is constructed from several transaction attributes:

```text
S
+ Block
+ Average Resale Price
+ Year-Month
+ First Character of Town
```

The resulting structure is:

```text
S[3-digit block][3-digit average price][YYMM][town initial]
```

### Block component

The block value is converted to text, digits are extracted, and the result is padded to three digits.

### Average resale price component

The average resale price is calculated within:

```text
Year-Month + Town + Flat Type
```

The rounded average price is converted to a three-digit prefix.

### Date component

The transaction month is converted into:

```text
YYMM
```

For example:

```text
2024-08 → 2408
```

### Town component

The first character of the town is converted to uppercase.

### Prefix

Every identifier begins with:

```text
S
```

---

## Duplicate Identifier Handling

The notebook checks whether the generated Resale Identifier is duplicated.

If duplicate identifiers exist, an MD5 hash derived from the record is appended as a four-character suffix to distinguish the records.

This provides an additional mechanism to ensure that the generated identifiers can differentiate otherwise-colliding records.

---

# Outputs

The notebook creates the following primary dataset during the workflow:

| Output | Description |
|---|---|
| `HDB_Resale_Prices_Master_Dataset.csv` | Combined master dataset containing records from all five source datasets |

The notebook also produces in-memory DataFrames representing the different processing stages:

| DataFrame | Description |
|---|---|
| `raw` | Combined master dataset |
| `raw_cleaned` | Dataset after duplicate/empty-row removal and categorical validation |
| `raw_quarantined` | Records that fail the January 2012 validation rules |
| `cleaned` | Validated records after resale-price anomaly filtering |
| `transformed` | Dataset with intermediate Resale Identifier components |
| `hashed` | Final dataset after duplicate identifier handling |

---

# Data Quality Rules Summary

| Area | Rule |
|---|---|
| Dataset combination | All five source datasets are combined using a standard column structure |
| Missing schema | `remaining_lease` is added to older datasets where unavailable |
| Duplicate rows | Exact duplicate rows are removed |
| Empty rows | Completely empty rows are removed |
| Lease | Remaining lease calculated as of 1 Jan 2027 using a 99-year lease assumption |
| Town validation | Must exist in January 2012 authoritative set |
| Flat type validation | Must exist in January 2012 authoritative set |
| Flat model validation | Must exist in January 2012 authoritative set |
| Storey range validation | Must exist in January 2012 authoritative set |
| Failed validation | Records are quarantined |
| Price anomaly | IQR method applied within `town + flat_type` |
| Price filtering | Records classified as non-normal are excluded from the subsequent transformation |
| Identifier | Resale Identifier generated from block, average price, month and town |

---

# Technologies

- **Python**
- **Pandas**
- **NumPy**
- **Regular Expressions**
- **Python `hashlib`**
- **Python `dateutil.relativedelta`**
- **Jupyter Notebook**

---

# Repository Structure

A suggested project structure is:

```text
ResaleFlatPrices/
│
├── 1_combine_datasets.ipynb
│
├── Resale Flat Prices (Based on Approval Date), 1990 - 1999.csv
├── Resale Flat Prices (Based on Approval Date), 2000 - Feb 2012.csv
├── Resale Flat Prices (Based on Registration Date), From Mar 2012 to Dec 2014.csv
├── Resale Flat Prices (Based on Registration Date), From Jan 2015 to Dec 2016.csv
├── Resale flat prices based on registration date from Jan-2017 onwards.csv
│
└── HDB_Resale_Prices_Master_Dataset.csv
```

---

# Assumptions and Limitations

### January 2012 as authoritative reference

The validation approach assumes that the unique values observed in January 2012 represent the authoritative permitted values for:

- `town`
- `flat_type`
- `flat_model`
- `storey_range`

Therefore, a value that appears in another period but not in January 2012 will be treated as invalid and quarantined.

### Remaining lease

The assessment provides only the lease commencement year. The calculation therefore assumes that the lease starts on 1 January of that year.

### Price anomalies

IQR-based detection identifies statistical outliers rather than confirmed erroneous transactions. A high or low resale price may be legitimate and should be investigated before being considered a data error.

### Identifier uniqueness

The Resale Identifier is constructed from selected transaction attributes. Where collisions occur, an MD5-based suffix is added to distinguish the records.

---

# How to Run

1. Place the five source CSV files in the configured `ResaleFlatPrices` directory.
2. Open:

```text
1_combine_datasets.ipynb
```

3. Update `base_dir` in the notebook if the project is stored in another location.

4. Run the notebook cells from top to bottom.

The notebook will combine the datasets, perform data quality processing, calculate remaining lease, validate records, identify price anomalies, and generate the Resale Identifier.

---

# Assessment Coverage

The notebook demonstrates the following data engineering and data quality capabilities:

- Multi-source dataset integration
- Schema standardisation
- Data profiling
- Data cleaning
- Completeness assessment
- Duplicate detection
- Reference-data validation
- Data quarantine
- Date-based transformation
- Statistical anomaly detection
- Group-level aggregation
- Identifier generation
- Duplicate identifier handling
