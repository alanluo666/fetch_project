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

---

***Please continue reading for more detailed information**.*

*Python script `data_quality_assessment.ipynb` is also attached.*

---

For each dataset, I followed this procedure:
- **Data Profiling** – Checking data types, null values, and sample records.
- **Missing Value Analysis** – Identifying fields with missing values.
- **Duplicate Check** – Detecting duplicate records that might affect analysis.
- **Inconsistency Check** – Finding unusual or inconsistent entries.
- **Visualization** – Generating heatmaps for missing data.

---

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

---

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

---

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

---

<img width="858" alt="visualizations" src="https://github.com/user-attachments/assets/f8ff27e5-d367-4b39-8ab8-9e9e58e65934" />

## **SQL Querying**

Closed-ended question 1:

**What are the top 5 brands by receipts scanned among users 21 and over?**

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

---

Closed-ended question 2:

**What are the top 5 brands by sales among users that have had their account for at least six months?**

```sql
-- get unique user_id with account_age, from question1 there is no duplicate user_id
WITH users AS (
 SELECT DISTINCT id, (JULIANDAY(CURRENT_DATE) - JULIANDAY(created_date)) / 30 AS account_age
 FROM USER_TAKEHOME)

-- get unique barcode and its brand, removing null and blank/whitespace
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
 -- upon investigation, final_sale is the total_sale, not item_price
        SUM(t.final_sale) AS total_sales,
        DENSE_RANK() OVER (ORDER BY SUM(t.final_sale) DESC) AS rank
 -- 1. lack of primary key like transaction_id make it hard to examine duplicates
 --    because the same barcode could be scanned multiple times
 -- 2. zero in final_quantity might mean cancel, or data pipeline issues that generate non-numeric number
 -- 3. null in final_sale might happen in the process of scanning, or data pipeline issues
 -- both 2 and 3 need further investigation to determine whether to remove these records
 FROM TRANSACTION_TAKEHOME t
 JOIN users u ON t.user_id = u.id
 JOIN products p ON t.barcode = p.barcode
 WHERE u.account_age >= 6
 -- need further investigation to confirm whether apply these filters
 -- remove zero quantity and null sale for now
 AND t.final_quantity != 'zero'
 AND t.final_sale IS NOT NULL AND TRIM(t.final_sale) != ''
 GROUP BY p.brands)

SELECT *
FROM ranking
-- show the top 5 brands including tie
WHERE rank <= 5
ORDER BY rank
```

---

Open-ended question 3:

**Who are Fetch’s power users?**

---

Assumptions:
1. **Definition of Power Users**
- Fetch offers multiple products (e.g., brand purchases, special offers, receipt uploads), but our dataset only contains **receipt-related** data.
- Therefore, power users in this analysis refer specifically to **users who actively scan receipts** rather than being representative of all Fetch users.

2. **Data Availability & Limitations**
- The **transactions** table consists of scanned receipts from **June to September 2024**, which is the entire dataset available.
- This means that power users are identified **only within this time period**, without insights into longer-term behaviors.
- The **users** table contains accounts created between **April 2014 and September 2024**, but it’s unclear if this dataset includes all users or a subset.

3. **Data Interpretation Assumptions**
- The dataset is **unstructured**, and we infer the following based on a deep dive:
  - `final_quantity = 0` likely indicates **canceled** transactions.
  - `final_sale` represents **total sales** per transaction, rather than item-level prices.
- However, it would be beneficial to confirm these assumptions with business stakeholders.

4. **RFM Model to Identify Power Users**
- We score users based on **Recency**, **Frequency**, and **Monetary** value (RFM) of receipt scanning.
- Each factor is scored from **0 to 5**, with a maximum score of **15**.
- The **top 20%** of users (by RFM score) are defined as power users, meaning they:
  - Scan receipts the most recently
  - Scan receipts the most frequently
  - Have the highest total spend
- We then analyze their demographic and behavioral traits to uncover insights and potential strategies to increase the number of power users.

---

Insights:
1. **Small Sample Size**
- The transactions table has 17,694 unique users, and the users table has 100,000 unique users.
- However, after joining these two tables, **only 91 unique users** remain.
- This indicates that a significant portion of users is missing from the analysis, limiting our ability to generate broader insights on power user behavior.
- Since we define power users as the top 20%, this results in **fewer than 20 users**, making it difficult to generalize trends.

2. **Key Differentiator: Age**
- Most demographic factors (account age, state, language, gender) **do not differ significantly** between power and non-power users.
- However, **age** does show a distinction:
  - Power Users: Average age 46.7 years, mostly between 30-40 years old.
  - Non-Power Users: Average age 55.1 years, mostly between 50-65 years old.
- This suggests that younger users may be more engaged with receipt scanning.
<img src="https://github.com/user-attachments/assets/ea70236e-f9a4-43a2-95a7-4da98dc52734" width="500" />

---

Additional Data that Helps:

To better understand who Fetch’s **power users** are and how to **grow this segment**, additional data points would be highly valuable:
1. **More Comprehensive User Data**
- Ensure we have a complete user dataset, rather than a small subset, to allow for broader analysis.
2. **User Behavior & Engagement Data**
- How often do users open the app?
- What activities do they have after opening the app?
- Do power users use Fetch for other products (e.g., cashback, product rewards)?
3. **Retention & Churn Rates**
- What percentage of power users remain engaged over time?
- Are they long-term Fetch users, or do they churn quickly?
- How does their behavior compare to non-power users in terms of drop-off rates?
4. **Acquisition Channel**
- Where did power users come from (e.g., ads, referrals, organic app downloads)?
- How can we optimize marketing efforts to acquire similar users?
5. **Receipt & Spending Patterns**
- How does their spending behavior evolve over time?
6. **Personalization & Loyalty Factors**
- Do power users engage more with Fetch rewards, promotions, or personalized offers?
- Would exclusive power user incentives increase retention and engagement?

More data availability would increase our ability to better define power users, understand what drives their engagement, and develop strategies to attract and retain more of them.

---

```sql
-- get the RFM data for each user
WITH rfm_data AS (
 SELECT user_id,
 -- how recent did the user use the service
 -- which is the duration (in days) between the last scan_date and current_date
        JULIANDAY(CURRENT_DATE) - JULIANDAY(MAX(scan_date)) AS recency,
 -- how many receipt did the user scan during the period
        COUNT(DISTINCT receipt_id) AS frequency,
 -- how much did the user spend during the period
        SUM(final_sale) AS monetary
 FROM TRANSACTION_TAKEHOME
 -- remove zero quantity and null sale for now; might need further investigation
 WHERE final_quantity != 'zero'
 AND final_sale IS NOT NULL AND TRIM(final_sale) != ''
 GROUP BY user_id)

-- assign scores for RFM accordingly
, rfm_rank AS (
 SELECT user_id,
        recency,
        frequency,
        monetary,
 -- rank the recency from 1 to 5; the most recent ones get the highest 5 points
        NTILE(5) OVER (ORDER BY recency DESC) AS recency_score,
 -- since most of the frequency values are concentrated around 1 and 2, we manually divide them
        CASE WHEN frequency = 1 THEN 1
             WHEN frequency = 2 THEN 2
             WHEN frequency = 3 THEN 3
             WHEN frequency = 4 THEN 4
             ELSE 5 END AS frequency_score,
 -- monetary values are also concentrated around small numbers, use 100 for more granular division
        NTILE(100) OVER (ORDER BY monetary) AS monetary_score
 FROM rfm_data)

-- join user table to supplement the dataset, order by the rfm_score
SELECT user_id,
-- reset the monetary_score to a maximum of 5 points
       recency_score, frequency_score, monetary_score/20.0,
-- add the 3 scores together for rfm_score
       recency_score + frequency_score + monetary_score/20.0 AS rfm_score,
-- if user is in the user table then 'yes', otherwise 'no'
       CASE WHEN id IS NOT NULL THEN 'Yes' ELSE 'No' END AS user_table_existance,
       created_date, birth_date, state, language, gender
FROM rfm_rank r
LEFT JOIN USER_TAKEHOME u ON r.user_id = u.id
-- whether show users not in the user table
--WHERE id IS NOT NULL
ORDER BY rfm_score DESC
```

## **Stakeholder Communication**

Question:

Construct an email that is understandable to a leader who is not familiar with your day-to-day work. Summarize the results of your investigation. Include:
- Key data quality issues and outstanding questions about the data
- One interesting trend in the data
  - Use a finding from part 2 or come up with a new insight
- Request for action: explain what additional help, info, etc. you need to make sense of the data and resolve any outstanding issues

---

Subject: **Data Quality, Power User Insights, & Next Steps**

Hi [Leader's Name],

This is Yu from the data analytics team. I wanted to share key findings from my recent analysis of our `USERS`, `TRANSACTIONS`, and `PRODUCTS` datasets. This email includes (1) data quality concerns, (2) one interesting trend, and (3) a request for additional information to improve insights.

1. **Data Quality Issues & Next Steps**

While cleaning and analyzing the data, I identified several data quality concerns that could impact our ability to generate accurate insights:
- `Users` dataset has missing values, inconsistent `GENDER` labels, and `LANGUAGE` codes that could be improved.
- `Transactions` dataset lacks a Primary Key, has missing and inconsistent `FINAL_SALE` and `FINAL_QUANTITY` values, and contains duplicates.
- `Products dataset` has significant missing values, duplicate PRIMARY KEY `BARCODEs`, and placeholder `MANUFACTURER` data.

I have already cleaned the data and generated a list of Fetch’s power users, and I will collaborate with the data engineering team to resolve these issues at the source to ensure data quality for future analyses.

2. **Key Insight: Age Differentiates Power Users**

One interesting finding is that age appears to be a key differentiator among power users:
- The average age of power users is 46.7 years, while non-power users are older on average (55.1 years).
- Power users tend to be between 30-40 years old, while non-power users cluster around 50-65 years old.

This suggests that younger users are more engaged with receipt scanning, which may inform targeted engagement strategies for future growth.

Other available data—such as `ACCOUNT_AGE`, `STATE`, `LANGUAGE`, and `GENDER`—show no significant differences between power and non-power users.

However, due to data limitations, further validation is required before drawing firm conclusions:
- The `TRANSACTIONS` dataset contains 17,694 unique users, and the `USERS` dataset contains 100,000 unique users, but only 91 users remain after joining the two datasets.
- This small sample size makes it difficult to generalize trends accurately.

3. **Additional Data Needed for Deeper Insights**

To refine our understanding of power users and optimize engagement strategies, I recommend gathering the following data:
- **More Comprehensive Data**
  - Ensure we have complete datasets, as the current one only allows us to analyze 91 users.
- **User Engagement Data**
  - How often do users open the app?
  - What activities do they have after opening the app?
  - What other Fetch products (e.g., cashback, rewards) do they engage with?
- **Retention rates**
  - Do users stay engaged over time, or do they churn quickly?
- **Acquisition Channel**
  - Where did power users come from (e.g., ads, referrals, organic downloads)?
  - Do different marketing channels produce more power users?
- **Spending & Loyalty Data**
  - Do power users spend more per transaction, or do they scan more frequently?
  - Are they more likely to engage with exlusive rewards than other users?

This additional data would significantly enhance our ability to identify, engage, and grow our most valuable users.

Would you be able to help facilitate access to these data points, or connect me with the right team to discuss further?

Looking forward to your thoughts!

Best,  
Yu  
Data Analyst
