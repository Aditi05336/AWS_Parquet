# AWS Athena Lab: CSV to Snappy Parquet with Partitioning (CTAS)

**Dataset:** `Activity03_Athena_sales.csv` (20,000 Rows | ~1.54 MB)  
**Goal:** Convert raw uncompressed CSV into partitioned Snappy Parquet using Amazon Athena CTAS, reducing query data scan from **1.54 MB to 2.36 KB (99.85% reduction)**.

---

## 🗺️ Quick Flowchart

```
[IAM User Group] -> [S3 Bucket: raw, parquet, results] -> [Athena Settings]
                          │
                   [Upload CSV to raw/]
                          │
          [Step 1: Create Database & Raw Table]
                          │
          [Step 2: Baseline Query (1.54 MB Scanned)]
                          │
      [Step 3: CTAS Parquet Conversion & Partitioning]
                          │
      [Step 4: Optimized Parquet Query (2.36 KB Scanned)]
                          │
                 [99.85% Cost & Scan Saved!]
```

---

## Step 0: IAM Permissions (Setup Lab Group)

1. Go to **IAM Console** -> **User groups** -> Click **Create group**.
2. **User group name:** `Athena-Lab-Group`
3. Attach Policies:
   - `AmazonAthenaFullAccess`
   - `AmazonS3FullAccess`
4. Click **Create user group**.
5. Go to **Users** -> Click your IAM User -> **User groups** tab -> **Add user to groups** -> Select `Athena-Lab-Group` -> **Add to groups**.

---

## Step 1: Amazon S3 Bucket & Upload CSV

1. Open **Amazon S3** -> Click **Create bucket**.
   - **Bucket name:** `sales-analytics-lab-<rollno>` (e.g. `practice2-916633759478-us-east-1-an`)
   - Region: `us-east-1` (or your default region)
   - Click **Create bucket**.
2. Open your bucket and create 3 folders:
   - `raw/`
   - `parquet/`
   - `results/`
3. Open `raw/` folder -> Click **Upload** -> Select `Activity03_Athena_sales.csv` -> Click **Upload**.

---

## Step 2: Athena Query Result Location Setup

> **Note:** Athena requires an S3 bucket location to store query output results before running any query.

1. Open **Amazon Athena** -> Go to **Query editor**.
2. Click the **Settings** tab at the top.
3. Click **Manage**.
4. In **Location of query result**, enter your S3 results path:
   ```text
   s3://<YOUR-BUCKET-NAME>/results/
   ```
5. Click **Save** and switch back to the **Editor** tab.

---

## Step 3: Create Athena Database

Run in Athena Query Editor:

```sql
CREATE DATABASE IF NOT EXISTS sales_db;
```

*Select `sales_db` from the Database dropdown on the left.*

---

## Step 4: Create Raw External Table (CSV)

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS sales_db.raw_sales (
    order_id string,
    order_date string,
    customer_id string,
    product string,
    category string,
    quantity string,
    unit_price string,
    total_amount string,
    city string,
    payment_mode string,
    year string,
    month string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
   'separatorChar' = ',',
   'quoteChar' = '\"',
   'escapeChar' = '\\'
)
LOCATION 's3://<YOUR-BUCKET-NAME>/raw/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

Verify rows:
```sql
SELECT * FROM sales_db.raw_sales LIMIT 5;
```

---

## Step 5: Baseline Query on Raw CSV (Before Optimization)

We run an analytical query to calculate category performance for October 2025 (`year = '2025'` and `month = '10'`):

```sql
SELECT
    category,
    COUNT(*) AS total_orders,
    ROUND(SUM(CAST(total_amount AS DOUBLE)), 2) AS total_revenue,
    SUM(CAST(quantity AS INT)) AS total_quantity
FROM sales_db.raw_sales
WHERE year = '2025' 
  AND month = '10'
GROUP BY category
ORDER BY total_revenue DESC;
```

### Baseline Performance Metrics:
- **Data Scanned:** `1.54 MB` (Athena scans the entire raw CSV file)
- **Run Time:** `~850 ms`

---

## Step 6: CTAS Conversion (CSV to Partitioned Snappy Parquet)

Convert raw CSV data into compressed Apache Parquet format partitioned by `year` and `month`:

```sql
CREATE TABLE sales_db.parquet_sales
WITH (
    format = 'PARQUET',
    write_compression = 'SNAPPY',
    external_location = 's3://<YOUR-BUCKET-NAME>/parquet/sales_data/',
    partitioned_by = ARRAY['year', 'month']
)
AS
SELECT
    order_id,
    date(date_parse(order_date, '%Y-%m-%d')) AS order_date,
    customer_id,
    product,
    category,
    CAST(quantity AS INT) AS quantity,
    CAST(unit_price AS DOUBLE) AS unit_price,
    CAST(total_amount AS DOUBLE) AS total_amount,
    city,
    payment_mode,
    year,
    month
FROM sales_db.raw_sales;
```

*(Note: In CTAS queries, partition columns `year` and `month` must be listed last in the `SELECT` clause).*

---

## Step 7: Verify Partitions & Row Integrity

1. **Check Partitions in Athena:**
   ```sql
   SHOW PARTITIONS sales_db.parquet_sales;
   ```
2. **Verify Row Counts:**
   ```sql
   SELECT COUNT(*) FROM sales_db.raw_sales;
   SELECT COUNT(*) FROM sales_db.parquet_sales;
   ```
   *Both return exactly `20,000` rows.*  
   *(Notice: `COUNT(*)` on Parquet scans `0 MB` because Athena reads directly from metadata footers).*

3. **Check S3 Directory Structure:**
   S3 will show partition folders created automatically:
   ```text
   s3://<YOUR-BUCKET-NAME>/parquet/sales_data/
   ├── year=2025/
   │   ├── month=1/
   │   ├── month=10/
   │   └── ...
   └── year=2026/
       ├── month=1/
       └── ...
   ```

---

## Step 8: Optimized Query on Partitioned Parquet (After Optimization)

Run the exact same analytical query on the new Parquet table:

```sql
SELECT
    category,
    COUNT(*) AS total_orders,
    ROUND(SUM(total_amount), 2) AS total_revenue,
    SUM(quantity) AS total_quantity
FROM sales_db.parquet_sales
WHERE year = '2025' 
  AND month = '10'
GROUP BY category
ORDER BY total_revenue DESC;
```

### Optimized Performance Metrics:
- **Data Scanned:** `2.36 KB` (Only scanned the `year=2025/month=10` folder and only requested columns)
- **Run Time:** `~586 ms`
- **Result Output:** 100% Identical to Baseline CSV Query!

---

## 📊 Final Performance & Cost Comparison

| Metric | Raw CSV Table (`raw_sales`) | Partitioned Parquet (`parquet_sales`) | Improvement / Benefit |
| :--- | :--- | :--- | :--- |
| **Storage Format** | Uncompressed Text CSV | Snappy Compressed Columnar Parquet | Highly compressed |
| **Partitioning** | None (Single file) | Hive-style (`year`, `month`) | Partition Pruning enabled |
| **Data Scanned** | **1.54 MB** (1,577 KB) | **2.36 KB** | **99.85% Reduction!** |
| **Run Time** | ~850 ms | ~586 ms | **~31% Faster** |
| **Row Count** | 20,000 | 20,000 | 100% Data Integrity |
| **Athena Query Cost** | Higher ($5/TB scanned) | 99.85% Cheaper | Direct cost savings |

### Data Scanned Reduction Formula:
$$\text{Reduction (\%)} = \frac{\text{CSV Data Scanned} - \text{Parquet Data Scanned}}{\text{CSV Data Scanned}} \times 100$$

$$\text{Reduction (\%)} = \frac{1577\text{ KB} - 2.36\text{ KB}}{1577\text{ KB}} \times 100 = \mathbf{99.85\%}$$

---

## 🎓 Viva / Examiner Q&A Guide

1. **Q: Why convert CSV to Apache Parquet?**  
   **A:** CSV is row-based and uncompressed. Every query must scan all columns and rows. Parquet is a columnar storage format with Snappy compression; Athena only scans the specific columns referenced in the query.

2. **Q: What is CTAS in Athena?**  
   **A:** CTAS stands for *Create Table As Select*. It creates a new table in Athena and simultaneously transforms, converts, compresses, and writes data to Amazon S3 in a single query.

3. **Q: What is Partition Pruning?**  
   **A:** Athena skips reading entire folders/partitions that do not match the `WHERE` clause filters (`year='2025' AND month='10'`), drastically reducing data scanned and query runtime.

4. **Q: Why use `OpenCSVSerde` for the raw table?**  
   **A:** OpenCSVSerde properly handles quotes, commas inside fields, and escapes without corrupting columns.

5. **Q: Why did `COUNT(*)` on Parquet scan 0 MB data?**  
   **A:** Parquet files store record counts and metadata in their file footers, allowing Athena to answer `COUNT(*)` queries without scanning data records.
