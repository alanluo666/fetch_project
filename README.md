# fetch_project

# Data Exploration Project

## **Overview**

This project focuses on exploring three datasets (`PRODUCTS_TAKEHOME.csv`, `TRANSACTION_TAKEHOME.csv`, and `USER_TAKEHOME.csv`) to identify data quality issues and fields that may be challenging to understand. The analysis is performed using Python with data visualization support.

## **Objective**

- Identify data quality issues (missing values, duplicates, inconsistencies).
- Highlight fields that are challenging to interpret.
- Document findings, assumptions, and conclusions clearly.

## **Project Structure**

```
├── data_exploration.py   # Python script for data analysis
├── PRODUCTS_TAKEHOME.csv
├── TRANSACTION_TAKEHOME.csv
├── USER_TAKEHOME.csv
└── README.md             # Project documentation
```

## **Approach**

1. **Data Loading:** Imported all datasets using pandas.
2. **Data Profiling:** Checked data types, null values, and sample records.
3. **Missing Value Analysis:** Identified patterns of missing data using visual heatmaps.
4. **Inconsistency Detection:** Checked for unusual entries in categorical fields (e.g., gender, product categories).
5. **Duplicate Check:** Identified duplicate records that may impact analysis.
6. **Visualization:** Used Seaborn for missing data heatmaps.

## **Assumptions**

- All CSV files are in UTF-8 encoding and follow standard CSV formatting.
- Missing values are represented as `NaN` or blank spaces.
- Placeholder values like "NONE" or "PLACEHOLDER MANUFACTURER" are treated as incomplete data.
- Date fields are expected to be in ISO format unless specified otherwise.

## **Data Quality Issues and Challenges**

### **Users Dataset**

- **Data Quality Issues:**
  - 100,000 rows with unique `ID`s.
  - **Missing Data:**
    - `BIRTH_DATE`: 3,675 (4%) missing values.
    - `STATE`: 4,812 (5%) missing values.
    - `LANGUAGE`: 30,508 (31%) missing values.
    - `GENDER`: 5,892 (6%) missing values.
  - No duplicate `ID`s or rows.
- **Inconsistencies:**
  - `GENDER`: Label inconsistencies between 2014 and 2022. Since 2023, the format is uniform.
  - `LANGUAGE`: Uses codes like `en`, `es-419`, which might be better represented as `English`, `Spanish`.
- **Challenging Fields:**
  - `LANGUAGE`: Consider replacing codes with full names for better readability.
  - `GENDER`: Should older inconsistent labels be standardized?

### **Transactions Dataset**

- **Data Quality Issues:**
  - 50,000 rows, but **no primary key** for unique transactions.
  - `RECEIPT_ID` can have multiple transactions, making it hard to identify true duplicates.
  - **Missing Data:**
    - `BARCODE`: 5,762 missing values (affects product matching).
    - `FINAL_SALE`: 12,500 whitespace values instead of NULL.
  - **Duplicates:**
    - 171 duplicate rows, but without a unique key, it’s unclear if they are real duplicates.
- **Inconsistencies:**
  - `FINAL_QUANTITY`: Non-numeric `"zero"` instead of `0`.
  - Outliers like `276` in `FINAL_QUANTITY`, indicating potential data entry errors.
- **Challenging Fields:**
  - `FINAL_QUANTITY`: What does `"zero"` mean? Transaction canceled or a data issue?
  - `FINAL_SALE`: Missing values, unclear if `0` means free or unrecorded. Should we include a `Price per item` column?

### **Products Dataset**

- **Data Quality Issues:**
  - 845,552 rows, but **no unique product ID**.
  - `BARCODE` should be a unique identifier, but missing and duplicated values exist.
  - **Missing Data:**
    - `CATEGORY_1`: 111 missing values.
    - `CATEGORY_2`: 1,424 missing values.
    - `CATEGORY_3`: 60,566 missing values.
    - `CATEGORY_4`: 778,093 missing values (\~90%).
    - `MANUFACTURER`: 226,474 missing values.
    - `BRAND`: 226,472 missing values.
    - `BARCODE`: 4,025 missing values (affects product mapping).
  - **Duplicates:**
    - 215 duplicate rows.
    - 185 duplicate `BARCODE` entries, sometimes linked to different brands or manufacturers.
- **Inconsistencies:**
  - `MANUFACTURER`: 4,973 entries labeled `"NONE"`, 86,902 labeled `"PLACEHOLDER"`.
  - `BARCODE`: Stored as `float`, which may cause rounding errors.
- **Challenging Fields:**
  - `BARCODE`: Not unique and stored as `float`, should be a primary identifier.
  - `CATEGORY_4`: 90% missing values—should it be removed?
  - `MANUFACTURER`: Contains `"NONE"` and `"PLACEHOLDER"`—should they be treated as `NULL`?

## **Next Steps**

- Clean and standardize inconsistent fields (`FINAL_QUANTITY`, `GENDER`, `LANGUAGE`).
- Address missing values based on business context (`CATEGORY_4`, `FINAL_SALE`, `BARCODE`).
- Determine if a unique product ID should be created.
- Perform deeper exploratory data analysis (EDA) if needed.
