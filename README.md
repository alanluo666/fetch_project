# Fetch Project

## **Overview**

This project explores 3 datasets (`PRODUCTS_TAKEHOME.csv`, `TRANSACTION_TAKEHOME.csv`, and `USER_TAKEHOME.csv`) to answer 3 questions:

1. **Data Quality Assessment** - assessing data quality issues and identifying **challenging fields**.

2. **SQL Querying** - Writing SQL queries to answer 3 closed-ended and open-ended questions.

3. **Stakeholder Communication** - Constructing a professional email summarizing key findings for stakeholders.

## **Project Structure**

```
├── data_quality_assessment.ipynb   # Python script for data quality assessment
├── PRODUCTS_TAKEHOME.csv
├── TRANSACTION_TAKEHOME.csv
├── USER_TAKEHOME.csv
└── README.md             # Project documentation
```

## **Data Quality Assessment**

In summary:
- `Users` dataset has missing values, inconsistent `GENDER` labels, and `LANGUAGE` codes that could be improved.
- `Transactions` dataset lacks a Primary Key, has missing and inconsistent `FINAL_SALE` and `FINAL_QUANTITY` values, and contains duplicates.
- `Products dataset` has significant missing values, duplicate `BARCODEs`, and placeholder `MANUFACTURER` data.

***Please continue reading for more detailed information**.*

*Python script `data_quality_assessment.ipynb` is also attached.*

For each dataset, I followed this procedure:
- **Data Profiling** – Checking data types, null values, and sample records.
- **Missing Value Analysis** – Identifying fields with missing values.
- **Duplicate Check** – Detecting duplicate records that might affect analysis.
- **Inconsistency Check** – Finding unusual or inconsistent entries.
- **Visualization** – Generating heatmaps for missing data.

1. ### **Users Dataset**
- Total Records: 100,000
- Unique `IDs`: 100,000 (No duplicate `IDs`)

**Data Quality Issues**
- **Missing Values:**
  - `BIRTH_DATE`: 3,675 missing values (4%).
  - `STATE`: 4,812 missing values (5%).
  - `LANGUAGE`: 30,508 missing values (31%).
  - `GENDER`: 5,892 missing values (6%).
- **Duplicates:**
  - No duplicate `IDs` or rows.
- **Inconsistencies:**
  - `GENDER`: Label inconsistencies between `2014` and `2022`. Since `2023`, the format is uniform.
  - `LANGUAGE`: Uses codes like `en`, `es-419`, which might be better represented as `English`, `Spanish`.

**Challenging Fields**
- `LANGUAGE`: Consider replacing codes with full names for better readability.
- `GENDER`: Should older inconsistent labels be standardized?

2. ### **Transactions Dataset**
- Total Records: 50,000
- No Primary Key: No unique `TRANSACTION_ID`, making it difficult to track individual transactions.
- Receipts & Scanned Items: A `RECEIPT_ID` can have multiple transactions, and `BARCODEs` may be scanned multiple times.

**Data Quality Issues**
- **Missing Values:**
  - `BARCODE`: 5,762 missing values (12%), affecting product mapping.
  - `FINAL_SALE`: 12,500 missing values (25%), recorded as `whitespace` instead of `NULL`.
- **Duplicates:**
  - 171 duplicate rows. Without a Primary Key, it is difficult to determine whether these are true duplicates or cases where users scanned the same `BARCODE` multiple times.
- **Inconsistencies:**
  - `FINAL_QUANTITY`: Some entries contain non-numeric values like `"zero"` instead of `0`.
  - `FINAL_QUANTITY` extreme values: Some values, like `276`, seem unusually high, possibly due to data entry errors.

**Challenging Fields**
- `FINAL_QUANTITY`: `"zero"` is stored as a string. Should be numeric. Also, does `"zero"` mean the transaction was canceled?
- `FINAL_SALE`: Some missing values `NULL`. Does `0` mean free, or is the value missing? Should we add a column for `PRICE_PER_ITEM`?

3. ### **Products Dataset**
- Total Records: 845,552
- No Unique `PRODUCT_ID`: There is no `PRODUCT_ID`. `BARCODE` should be a unique identifier, but there are missing values and duplicates.

**Data Quality Issues**
- **Missing Values:**
  - `CATEGORY_1`: 111 missing values (0.01%).
  - `CATEGORY_2`: 1,424 missing values (0.2%).
  - `CATEGORY_3`: 60,566 missing values (7%).
  - `CATEGORY_4`: 778,093 missing values (92%).
  - `MANUFACTURER`: 226,474 missing values (27%).
  - `BRAND`: 226,472 missing values (27%).
  - `BARCODE`: 4,025 missing values (0.5%), which affects product mapping.
- **Duplicates:**
  - 215 duplicate rows.
  - 185 duplicate `BARCODEs`. Some duplicate `BARCODEs` are associated with different `BRANDs` under the same `MANUFACTURER`, while others appear multiple times for the same `BRAND`.
  - To ensure data integrity, we should clean duplicates and create a unique `PRODUCT_ID` as the Primary Key or use `BARCODE` as the Primary Key, ensuring uniqueness.
- **Inconsistencies:**
  - `MANUFACTURER`: 4,973 entries labeled as `"NONE"` and 86,902 entries labeled as `"PLACEHOLDER"`.
  - `BARCODE`: stored as a float, which may cause rounding issues.

**Challenging Fields**
- `CATEGORY_4`: Over 90% missing values. Is this field necessary for analysis? Should we remove it?
- `MANUFACTURER`: Contains placeholder values `"NONE"`, `"PLACEHOLDER"`. Should we replace these with `NULL`?
- `BARCODE`: Not unique and stored as a float. Should be converted to an integer and used as a unique identifier.



## **SQL Querying**

Closed-ended question 1:

What are the top 5 brands by receipts scanned among users 21 and over?

```sql
-- get unique user_id with age, from question1 there is no duplicate user_id
WITH users AS (
 SELECT DISTINCT id, FLOOR((JULIANDAY(CURRENT_DATE) - JULIANDAY(birth_date)) / 365) AS age
 FROM USER_TAKEHOME)

-- get unique barcode and its brand, removing null and whitespace
, products_0 AS (
 SELECT DISTINCT barcode, brand
 FROM PRODUCTS_TAKEHOME
 WHERE barcode IS NOT NULL AND TRIM(barcode) != '')
 -- keep the null brand so that we can investigate whether it is popular just in case
 --AND brand IS NOT NULL AND TRIM(brand) != '')

-- some barcodes have multiple brands, so concat them for 1-1 relationship
, products AS (
 SELECT barcode, GROUP_CONCAT(brand) AS brands
 FROM products_0
 GROUP BY 1)

-- use dense_rank in case there is a tie
, ranking AS (
 SELECT p.brands,
        COUNT(*) AS scanned_count,
        DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) AS rank
 -- 1. lack of primary key like transaction_id make it hard to examine duplicates
 --    because the same barcode could be scanned multiple times
 -- 2. zero in final_quantity might mean cancel, or data pipeline issues that generate non-numeric number
 -- 3. null in final_sale might happen in the process of scanning, or data pipeline issues
 -- both 2 and 3 need further investigation to determine whether to remove these records
 FROM TRANSACTION_TAKEHOME t
 JOIN users u ON t.user_id = u.id
 JOIN products p ON t.barcode = p.barcode
 WHERE u.age >= 21
 -- need further investigation to determine whether apply these filters
 --WHERE t.final_quantity != 'zero'
 --AND t.final_sale IS NOT NULL AND TRIM(t.final_sale) != ''
 GROUP BY p.brands)

SELECT *
FROM ranking
-- show the top 5 brands including tie
WHERE rank <= 5
ORDER BY rank;
```

## **Next Steps**

- Clean and standardize inconsistent fields (`FINAL_QUANTITY`, `GENDER`, `LANGUAGE`).
- Address missing values based on business context (`CATEGORY_4`, `FINAL_SALE`, `BARCODE`).
- Determine if a unique product ID should be created.
- Perform deeper exploratory data analysis (EDA) if needed.
