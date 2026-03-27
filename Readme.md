# NovoCartGlobal Data Warehouse - Complete Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Data Architecture](#data-architecture)
3. [Bronze Layer: Data Ingestion](#bronze-layer-data-ingestion)
4. [Silver Layer: Data Transformation & Cleaning](#silver-layer-data-transformation--cleaning)
5. [Gold Layer: Dimensional Model](#gold-layer-dimensional-model)
6. [Data Cube: Pre-Joined Analytics Layer](#data-cube-pre-joined-analytics-layer)
7. [KPIs (Key Performance Indicators)](#kpis-key-performance-indicators)
8. [Data Quality Framework](#data-quality-framework)
9. [Storage and Catalogs](#storage-and-catalogs)

---

## Project Overview

**NovoCartGlobal** is an end-to-end data warehouse solution built on Databricks using the **Medallion Architecture** (Bronze → Silver → Gold). The project processes e-commerce data from multiple sources, performs comprehensive data cleaning and transformation, creates a star schema dimensional model, builds a pre-joined data cube for analytics, and generates business KPIs for reporting.

**Key Technologies:**
* **Platform:** Databricks on AWS
* **Storage:** Unity Catalog with Delta Lake format
* **Catalog Name:** `novacart_dev`
* **Data Source:** AWS S3 (CSV files)
* **Processing:** Apache Spark with PySpark

---

## Data Architecture

### Medallion Architecture with Data Cube Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  S3 Bucket (CSV Files)                                          │
│  s3://databricks-project-jman/NovoCartGlobal/                   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │  BRONZE LAYER   │         │  Raw Data       │                │
│  │  (Raw Ingestion)│────────▶│  Tables (5)     │                │
│  └─────────────────┘         └─────────────────┘                │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │  SILVER LAYER   │         │  Cleaned &      │                │
│  │  (Cleaned Data) │────────▶│  Transformed    │                │
│  └─────────────────┘         │  Tables (5)     │                │
│           │                   └─────────────────┘               │
│           ▼                                                     │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │   GOLD LAYER    │         │  Dimensional    │                │
│  │  (Star Schema)  │────────▶│  Model          │                │
│  │                 │         │  • 4 Dimensions │                │
│  │                 │         │  • 1 Fact Table │                │
│  └─────────────────┘         └─────────────────┘                │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────┐                        │
│  │      DATA CUBE (Pre-Joined)         │                        │
│  │  Fact + All Dimensions Combined     │                        │
│  │  • Single Table: data_cube          │                        │
│  │  • 51 Columns                       │                        │
│  │  • 2,048 Records                    │                        │
│  └─────────────────────────────────────┘                        │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────┐                        │
│  │     KPI TABLES (10 Metrics)         │                        │
│  │  Calculated from Data Cube          │                        │
│  └─────────────────────────────────────┘                        │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────┐                        │
│  │     BI DASHBOARDS & ANALYTICS       │                        │
│  └─────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Bronze Layer: Data Ingestion

### Purpose
The Bronze layer ingests raw data from AWS S3 CSV files into Unity Catalog with **minimal transformation**. This layer serves as the **source of truth** for all downstream processing.

### Notebook
* **Path:** `/Users/madhavkkp@gmail.com/NovoCartGlobal/Bronze/Bronze_raw_data`
* **Operation:** Read CSV from S3 → Write to Unity Catalog

### Data Sources & Tables

| Source File | Bronze Table | Schema Location | Row Count |
|-------------|--------------|-----------------|----------|
| `customers.csv` | `novacart_dev.bronze.customers` | Unity Catalog | 500 |
| `exchange_rates.csv` | `novacart_dev.bronze.exchange_rates` | Unity Catalog | Variable |
| `order_items.csv` | `novacart_dev.bronze.order_items` | Unity Catalog | 2,048 |
| `orders.csv` | `novacart_dev.bronze.orders` | Unity Catalog | 800 |
| `products.csv` | `novacart_dev.bronze.products` | Unity Catalog | 100 |

### Ingestion Configuration

```python
# Example: Customers Table Ingestion
bronze_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("s3://databricks-project-jman/NovoCartGlobal/customers.csv")

catalog_name = "novacart_dev"
schema_name = "bronze"
table_name = "customers"

bronze_df.write.mode("overwrite").saveAsTable(f"{catalog_name}.{schema_name}.{table_name}")
```

### Key Characteristics
* **Format:** Delta Lake
* **Schema Inference:** Automatic from CSV headers
* **Write Mode:** Overwrite (full refresh)
* **No Transformations:** Data is loaded as-is

---

## Silver Layer: Data Transformation & Cleaning

### Purpose
The Silver layer performs **data cleaning, standardization, validation, and enrichment**. This layer ensures data quality and prepares data for analytics.

### Notebooks

| Notebook | Source Table | Target Table | Purpose |
|----------|--------------|--------------|----------|
| `01_silver_customers_cleaning` | `bronze.customers` | `silver.slv_customers` | Clean customer data |
| `02_silver_orders_cleaning` | `bronze.orders` | `silver.slv_orders` | Clean order data |
| `03_silver_exchange_rate_cleaning` | `bronze.exchange_rates` | `silver.slv_exchange_rates` | Clean exchange rates |
| `04_silver_products_cleaning` | `bronze.products` | `silver.slv_products` | Clean product data |
| `05_silver_orders_items_cleaning` | `bronze.order_items` | `silver.slv_order_items` | Clean order line items |

### Transformation Steps (Common Across All Tables)

#### 1. Column Standardization
* Convert all column names to **lowercase**
* Replace spaces with **underscores**
* Example: `Customer ID` → `customer_id`

#### 2. Null Handling
* Replace various null representations with actual `NULL`
* Patterns replaced: `"\\N"`, `"?"`, `""`, `"null"`, `"NULL"`

#### 3. String Cleaning
* **Trim** whitespace from all string columns
* Convert strings to **lowercase** for consistency

```python
string_cols = [c for c, t in df.dtypes if t == "string"]
for col in string_cols:
    df = df.withColumn(col, F.trim(F.lower(F.col(col))))
```

#### 4. Date Parsing with Multiple Format Support
* Handle multiple date formats: `yyyy-MM-dd`, `yyyy/MM/dd`, `dd/MM/yyyy`
* Use `coalesce` to try multiple formats

```python
df = df.withColumn(
    "registration_date",
    F.coalesce(
        F.to_date(F.expr("try_to_timestamp(registration_date, 'yyyy-MM-dd')")),
        F.to_date(F.expr("try_to_timestamp(registration_date, 'yyyy/MM/dd')")),
        F.to_date(F.expr("try_to_timestamp(registration_date, 'dd/MM/yyyy')"))
    )
)
```

#### 5. Duplicate Removal
* Remove duplicates based on primary key
* Example: `dropDuplicates(["customer_id"])`

#### 6. Data Validation & Standardization

**Customers Table:**
* Validate `channel` field (only "web" or "mobile" allowed)
* Add data quality flags:
  * `dq_missing_customer_id`
  * `dq_missing_email`
  * `dq_invalid_channel`

**Orders Table:**
* Status standardization with multi-language support
  * Mapping: "已完成" → "completed", "versandt" → "shipped", etc.
  * Created `order_status_clean` column
* Validate and clean `total_amount` (reject negative values)
* Uppercase `country_code` for consistency
* Validate foreign key relationships with customers
* Data quality flags:
  * `dq_orphan_customer`
  * `dq_missing_order_id`
  * `dq_invalid_order_date`
  * `dq_invalid_status`
  * `dq_invalid_amount`

**Products Table:**
* Validate `price` (reject negative values)
* Uppercase `country_code`
* Validate `category` field
* Data quality flags:
  * `dq_missing_product_id`
  * `dq_missing_product_name`
  * `dq_invalid_price`

**Order Items Table:**
* Validate line item calculations
* Check for orphan records (missing orders/products)
* Data quality flags:
  * `dq_line_total_mismatch`
  * `dq_orphan_order`
  * `dq_orphan_product`
  * `dq_missing_order_item_id`
  * `dq_invalid_quantity`
  * `dq_invalid_price`

**Exchange Rates Table:**
* Validate date formats
* Validate currency codes
* Validate exchange rate values (positive, non-zero)
* Data quality flags:
  * `dq_invalid_date`
  * `dq_invalid_currency`
  * `dq_invalid_exchange`

#### 7. Metadata Addition
* Add `load_timestamp` column with current timestamp
* Tracks when data was processed into Silver layer

### Silver Layer Output

| Table | Schema | Purpose |
|-------|--------|----------|
| `slv_customers` | `novacart_dev.silver` | Cleaned customer master data |
| `slv_orders` | `novacart_dev.silver` | Cleaned order header data |
| `slv_products` | `novacart_dev.silver` | Cleaned product catalog |
| `slv_order_items` | `novacart_dev.silver` | Cleaned order line items |
| `slv_exchange_rates` | `novacart_dev.silver` | Cleaned currency exchange rates |

---

## Gold Layer: Dimensional Model

### Purpose
The Gold layer implements a **Star Schema** optimized for analytical queries and BI reporting. It consists of **dimension tables** and a **fact table** that joins all entities.

### Star Schema Design

```
                    ┌──────────────────┐
                    │  dim_dates       │
                    │  - date_key (PK) │
                    │  - year, month   │
                    │  - quarter       │
                    │  - day_name      │
                    │  - is_weekend    │
                    └──────────────────┘
                             │
                             │
                             ▼
┌──────────────────┐   ┌─────────────────────────┐   ┌──────────────────┐
│  dim_customers   │   │   fact_order_items      │   │  dim_products    │
│  - customer_key  │◀──│  - order_item_key (PK)  │──▶│  - product_key   │
│  - reg_date      │   │  - order_key (FK)       │   │  - product_name  │
│  - reg_year      │   │  - product_key (FK)     │   │  - category      │
│  - reg_month     │   │  - customer_key (FK)    │   │  - price         │
└──────────────────┘   │  - date_key (FK)        │   │  - currency      │
                       │                         │   └──────────────────┘
                       │  MEASURES:              │
                       │  - quantity             │
                       │  - unit_price           │   ┌──────────────────┐
                       │  - line_total           │   │  dim_orders      │
                       │  - revenue_usd          │◀──│  - order_key     │
                       │                         │   │  - customer_id   │
                       │  ATTRIBUTES:            │   │  - order_date    │
                       │  - order_status_clean   │   │  - status        │
                       │  - channel              │   │  - total_amount  │
                       │  - order_country        │   │  - channel       │
                       │  - category             │   └──────────────────┘
                       │  - product_currency     │
                       │                         │
                       │  DQ FLAGS (20):         │
                       │  - All quality checks   │
                       └─────────────────────────┘
```

### Dimension Tables

#### 1. dim_customers
**Notebook:** `/NovoCartGlobal/Gold/dim/dim_customers`

**Source:** `novacart_dev.silver.slv_customers`

**Columns:**
* `customer_key` (Primary Key) - Customer identifier
* `registration_date` - Date customer registered
* `registration_year` - Year extracted from registration date
* `registration_month` - Month extracted from registration date
* `load_timestamp` - ETL timestamp

**Grain:** One row per unique customer

**Storage:** `novacart_dev.gold.dim_customers`

---

#### 2. dim_products
**Notebook:** `/NovoCartGlobal/Gold/dim/dim_products`

**Source:** `novacart_dev.silver.slv_products`

**Columns:**
* `product_key` (Primary Key) - Product identifier
* `product_name` - Product name
* `category` - Product category (laptops, accessories, peripherals, etc.)
* `price` - Product price in local currency
* `currency` - Product currency (INR, EUR, GBP, CNY)
* `product_origin_country` - Country where product is sold
* `load_timestamp` - ETL timestamp

**Grain:** One row per unique product (100 products)

**Storage:** `novacart_dev.gold.dim_products`

---

#### 3. dim_orders
**Notebook:** `/NovoCartGlobal/Gold/dim/dim_orders`

**Source:** `novacart_dev.silver.slv_orders`

**Columns:**
* `order_key` (Primary Key) - Order identifier
* `customer_id` - Foreign key to customer
* `order_date` - Date order was placed
* `order_status` - Original status value
* `order_status_clean` - Standardized status (completed, pending, shipped, cancelled, returned)
* `channel` - Order channel (web, mobile)
* `country_code` - Country where order originated (IN, UK, ES, CN)
* `total_amount` - Total order amount in local currency
* `currency` - Order currency
* `load_timestamp` - ETL timestamp

**Grain:** One row per unique order (800 orders)

**Storage:** `novacart_dev.gold.dim_orders`

---

#### 4. dim_dates
**Notebook:** `/NovoCartGlobal/Gold/dim/dim_dates`

**Source:** Generated from order date range

**Columns:**
* `date_key` (Primary Key) - Date in YYYYMMDD format (integer)
* `full_date` - Full date value
* `year` - Year (2023-2025)
* `quarter` - Quarter (1-4)
* `month` - Month (1-12)
* `month_name` - Month name (January, February, etc.)
* `day` - Day of month (1-31)
* `day_of_week` - Day of week (1=Monday, 7=Sunday)
* `day_name` - Day name (Monday, Tuesday, etc.)
* `is_weekend` - Weekend flag (1=weekend, 0=weekday)

**Grain:** One row per date (755 dates from 2023-01-03 to 2025-01-26)

**Storage:** `novacart_dev.gold.dim_dates`

---

### Fact Table

#### fact_order_items
**Notebook:** `/NovoCartGlobal/Gold/fact/fact_order_items`

**Sources:**
* `novacart_dev.silver.slv_order_items`
* `novacart_dev.silver.slv_orders`
* `novacart_dev.silver.slv_products`
* `novacart_dev.silver.slv_customers`
* `novacart_dev.silver.slv_exchange_rates`

**Grain:** One row per order line item (2,048 records)

**Keys (Foreign Keys):**
* `order_item_key` (Primary Key)
* `order_key` → `dim_orders.order_key`
* `product_key` → `dim_products.product_key`
* `customer_key` → `dim_customers.customer_key`
* `date_key` → `dim_dates.date_key`

**Measures (Numeric Facts):**
* `quantity` - Number of units ordered
* `unit_price` - Price per unit in local currency
* `line_total` - Total line amount (quantity × unit_price) in local currency
* `revenue_usd` - **Converted revenue in USD** (line_total × exchange_rate_to_usd)

**Denormalized Attributes (for query performance):**
* `order_status_clean` - Standardized order status
* `channel` - Order channel (web/mobile)
* `order_country` - Country where order was placed
* `category` - Product category
* `product_currency` - Product's local currency

**Data Quality Flags (20 flags):**

*Order Items (6 flags):*
* `dq_line_total_mismatch`
* `dq_orphan_order`
* `dq_orphan_product`
* `dq_missing_order_item_id`
* `dq_invalid_quantity`
* `dq_invalid_item_price`

*Orders (5 flags):*
* `dq_orphan_customer`
* `dq_missing_order_id`
* `dq_invalid_order_date`
* `dq_invalid_status`
* `dq_invalid_amount`

*Products (3 flags):*
* `dq_missing_product_id`
* `dq_missing_product_name`
* `dq_invalid_product_price`

*Customers (3 flags):*
* `dq_missing_customer_id`
* `dq_missing_email`
* `dq_invalid_customer_channel`

*Exchange Rates (3 flags):*
* `dq_invalid_exchange_date`
* `dq_invalid_exchange_currency`
* `dq_invalid_exchange`

**Metadata:**
* `etl_timestamp` - Timestamp when record was created

**Partitioning:**
* Partitioned by `date_key` for query performance

**Storage:** `novacart_dev.gold.fact_order_items`

**Currency Handling:**
* Product prices are stored in local currency (INR, EUR, GBP, CNY)
* `line_total` is in product's local currency
* `revenue_usd` is calculated by joining with exchange rates and converting to USD
* Exchange rate conversion:
  ```python
  revenue_usd = line_total * exchange_rate_to_usd
  ```

**Currency Conversion Summary:**
| Currency | Line Items | Total Local Currency | Total USD |
|----------|------------|---------------------|----------|
| CNY | 528 | 569,583.68 | 79,741.72 |
| EUR | 726 | 1,106,856.23 | 1,217,542.29 |
| GBP | 401 | 495,630.30 | 629,450.61 |
| INR | 393 | 689,087.11 | 8,269.19 |

---

## Data Cube: Pre-Joined Analytics Layer

### Purpose
The **Data Cube** is a unified, pre-joined table that combines the fact table with all dimension tables. It serves as a **single source of truth for all KPI calculations and analytics**, eliminating the need to repeatedly join fact and dimension tables.

**Notebook:** `/NovoCartGlobal/Gold/Reporting/NovoCart Data Cube Builder`

**Table:** `novacart_dev.gold.data_cube`

### What is the Data Cube?

A data cube is a **denormalized, pre-joined table** that combines:
* **Fact table** (`fact_order_items`): Transaction-level metrics (revenue, quantity, prices)
* **Dimension tables**: Descriptive attributes for analysis
  * `dim_products`: Product details (name, category, origin)
  * `dim_customers`: Customer registration information
  * `dim_dates`: Time-based attributes (year, quarter, month, day of week)
  * `dim_orders`: Order-level details (status, channel, country)

### Architecture Flow

```
┌─────────────────┐
│ fact_order_items│
└────────┬────────┘
         │
         ├─────────JOIN────────┐
         │                     │
         ▼                     ▼
┌─────────────┐         ┌─────────────┐
│dim_products │         │dim_customers│
└──────┬──────┘         └──────┬──────┘
       │                       │
       └──────┐    ┌───────────┘
              │    │
              ▼    ▼
         ┌─────────────┐
         │  dim_dates  │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ dim_orders  │
         └──────┬──────┘
                │
                ▼
      ┌──────────────────┐
      │   DATA CUBE      │
      │ (Pre-Joined)     │
      │ 51 Columns       │
      │ 2,048 Records    │
      └──────────────────┘
                │
                ▼
         ┌──────────┐
         │   KPIs   │
         └──────────┘
```

### Schema Details

**Total Columns:** 51

**Column Categories:**

1. **Keys (5 columns):**
   * `order_item_key`, `order_key`, `product_key`, `customer_key`, `date_key`

2. **Metrics (4 columns):**
   * `revenue_usd` - Revenue converted to USD
   * `quantity` - Number of units
   * `unit_price` - Price per unit
   * `line_total` - Line total in local currency

3. **Dimension Attributes (29 columns):**
   * **From fact table:** `order_status_clean`, `fact_channel`, `order_country`, `fact_category`, `product_currency`
   * **From dim_products:** `product_name`, `product_category`, `product_price`, `product_dim_currency`, `product_origin_country`
   * **From dim_customers:** `registration_date`, `registration_year`, `registration_month`
   * **From dim_dates:** `full_date`, `order_year`, `order_quarter`, `order_month`, `order_month_name`, `order_day`, `day_of_week`, `day_name`, `is_weekend`
   * **From dim_orders:** `customer_id`, `order_date`, `order_status`, `order_channel`, `country_code`, `order_total_amount`, `order_currency`

4. **Data Quality Flags (13 columns):**
   * All `dq_*` flags from the fact table for quality monitoring

### Key Join Logic

```python
data_cube = fact \
    .join(dim_products, fact.product_key == dim_products.product_key, "left") \
    .join(dim_customers, fact.customer_key == dim_customers.customer_key, "left") \
    .join(dim_dates, fact.date_key == dim_dates.full_date, "left") \
    .join(dim_orders, fact.order_key == dim_orders.order_key, "left")
```

### Benefits

✓ **Simplified Queries:** No need to repeatedly join fact and dimension tables  
✓ **Better Performance:** Single table scan instead of multiple joins  
✓ **Consistency:** All KPIs use the same underlying data structure  
✓ **Easier Maintenance:** Update joins in one place instead of across multiple notebooks  
✓ **Self-Service Analytics:** Business users can query a single table with all context  
✓ **BI Tool Friendly:** Direct connection to pre-joined data

### Usage Example

```python
# Load the data cube
cube = spark.table("novacart_dev.gold.data_cube")

# Example: Calculate revenue by product category
revenue_by_category = cube.filter(F.col("order_status_clean") == "completed") \
    .groupBy("product_category") \
    .agg(F.sum("revenue_usd").alias("total_revenue"))
```

### Storage
* **Table:** `novacart_dev.gold.data_cube`
* **Format:** Delta Lake
* **Records:** 2,048 (one per order line item)
* **Columns:** 51

---

## KPIs (Key Performance Indicators)

### Purpose
Pre-calculated business metrics stored as separate tables for dashboard and reporting use. **All KPIs are calculated from the pre-joined data cube** for consistency and performance.

**Notebook:** `/NovoCartGlobal/Gold/Reporting/NovoCart Gold KPIs`

**Source:** `novacart_dev.gold.data_cube` (pre-joined fact + dimensions)

### KPI Tables

#### KPI 1: Total Revenue
**Table:** `novacart_dev.gold.kpi_total_revenue`

**Metric:** Total revenue from completed orders in USD

**Calculation:**
```python
cube.filter(F.col("order_status_clean") == "completed") \
    .agg(F.sum("revenue_usd").alias("total_revenue_usd"))
```

**Result:** $1,210,553.90 USD

---

#### KPI 2: Revenue by Country
**Table:** `novacart_dev.gold.kpi_revenue_by_country`

**Columns:**
* `order_country` - Country code
* `revenue_usd` - Total revenue in USD
* `order_items` - Number of line items

**Source:** Data cube filtered by `order_status_clean = 'completed'`

**Top Country:** India (IN) with $294,969.82 USD and 297 items

---

#### KPI 3: Revenue by Channel
**Table:** `novacart_dev.gold.kpi_revenue_by_channel`

**Columns:**
* `channel` - Sales channel (web/mobile)
* `revenue_usd` - Total revenue in USD
* `order_items` - Number of line items
* `orders` - Number of orders

**Source:** Data cube filtered by `order_status_clean = 'completed'`, grouped by `fact_channel`

**Top Channel:** Mobile with $638,780.82 USD and 266 orders

---

#### KPI 3b: Revenue by Product Currency
**Table:** `novacart_dev.gold.kpi_revenue_by_currency`

**Purpose:** Shows original currency totals vs USD-converted totals

**Columns:**
* `product_currency` - Currency code
* `total_local_currency` - Total in original currency
* `total_usd` - Total converted to USD
* `order_items` - Number of line items
* `orders` - Number of orders
* `avg_exchange_rate` - Average exchange rate used

**Source:** Data cube grouped by `product_currency`

---

#### KPI 4: Completed Order Count
**Table:** `novacart_dev.gold.kpi_completed_order_count`

**Metric:** Number of orders with status = 'completed'

**Calculation:**
```python
cube.filter(F.col("order_status_clean") == "completed") \
    .select("order_key").distinct().count()
```

**Result:** 519 completed orders

---

#### KPI 5: Completed Order Rate
**Table:** `novacart_dev.gold.kpi_completed_order_rate`

**Columns:**
* `total_orders` - Total number of orders
* `completed_orders` - Number of completed orders
* `completed_order_rate` - Percentage of completed orders

**Calculation:**
```python
completed_order_rate = (completed_orders / total_orders) * 100
```

**Source:** Data cube distinct order keys

**Result:** 64.875% (519 completed out of 800 total)

---

#### KPI 6: Average Order Value (AOV)
**Table:** `novacart_dev.gold.kpi_average_order_value`

**Columns:**
* `avg_order_value` - Average order value in USD
* `min_order_value` - Minimum order value
* `max_order_value` - Maximum order value

**Calculation:**
```python
cube.filter(F.col("order_status_clean") == "completed") \
    .groupBy("order_key").agg(F.sum("revenue_usd").alias("order_total")) \
    .agg(F.avg("order_total"), F.min("order_total"), F.max("order_total"))
```

**Results:**
* Average: $2,332.47 USD
* Min: $0.64 USD
* Max: $26,345.67 USD

---

#### KPI 7: Top 5 Products by Revenue
**Table:** `novacart_dev.gold.kpi_top_products`

**Columns:**
* `product_key` - Product identifier
* `product_name` - Product name
* `category` - Product category
* `total_revenue` - Total revenue in USD
* `total_quantity_sold` - Total units sold
* `avg_price` - Average selling price

**Source:** Data cube grouped by product, filtered by completed orders

**Top Product:** "laptops product 78" with $104,870.25 USD and 45 units sold

---

#### KPI 8: Active Customers Count
**Table:** `novacart_dev.gold.kpi_active_customers_count`

**Columns:**
* `total_customers` - Total registered customers
* `active_customers` - Customers with at least one completed order
* `active_rate` - Percentage of active customers

**Calculation:**
```python
active_customers = cube.filter(F.col("order_status_clean") == "completed") \
    .select("customer_key").distinct().count()
total_customers = cube.select("customer_key").distinct().count()
active_rate = (active_customers / total_customers) * 100
```

**Results:**
* Total Customers: 500
* Active Customers: 331
* Active Rate: 66.2%

---

#### KPI 9: Customer Acquisition by Month
**Table:** `novacart_dev.gold.kpi_customer_acquisition`

**Columns:**
* `registration_year` - Year
* `registration_month` - Month
* `new_customers` - Number of new customers registered

**Source:** Data cube grouped by registration year and month

**Purpose:** Track customer growth over time

---

#### KPI 10: Data Quality Score
**Table:** `novacart_dev.gold.kpi_data_quality_score`

**Columns:**
* `total_records` - Total fact table records
* `clean_records` - Records with no data quality issues
* `records_with_issues` - Records with at least one DQ flag
* `data_quality_score` - Percentage of clean records
* `total_dq_checks` - Number of DQ checks performed

**Source:** Data cube with all DQ flags analyzed

**Results:**
* Total Records: 2,048
* Clean Records: 1,730 (84.47%)
* Records with Issues: 318
* Total DQ Checks: 20

**Detailed Breakdown Table:** `novacart_dev.gold.kpi_dq_breakdown`

**Top 5 Data Quality Issues:**
1. `dq_invalid_status` - 156 records (7.62%)
2. `dq_orphan_customer` - 50 records (2.44%)
3. `dq_invalid_order_date` - 48 records (2.34%)
4. `dq_missing_email` - 43 records (2.10%)
5. `dq_missing_product_name` - 40 records (1.95%)

---

## Data Quality Framework

### Purpose
Comprehensive data quality monitoring with 20 different checks across all data sources.

### Data Quality Flags

Each record in the fact table (and subsequently the data cube) has 20 boolean flags (1 = issue present, 0 = no issue).

### Flag Categories

#### Missing Data Flags (6)
* `dq_missing_customer_id`
* `dq_missing_email`
* `dq_missing_order_id`
* `dq_missing_order_item_id`
* `dq_missing_product_id`
* `dq_missing_product_name`

#### Invalid Data Flags (8)
* `dq_invalid_channel` - Channel not in (web, mobile)
* `dq_invalid_status` - Invalid order status
* `dq_invalid_order_date` - Date parsing failed
* `dq_invalid_amount` - Negative or null amounts
* `dq_invalid_quantity` - Invalid quantity values
* `dq_invalid_item_price` - Invalid item price
* `dq_invalid_product_price` - Invalid product price
* `dq_invalid_exchange` - Invalid exchange rate

#### Referential Integrity Flags (3)
* `dq_orphan_customer` - Order references non-existent customer
* `dq_orphan_order` - Order item references non-existent order
* `dq_orphan_product` - Order item references non-existent product

#### Exchange Rate Quality Flags (3)
* `dq_invalid_exchange_date` - Invalid date in exchange rates
* `dq_invalid_exchange_currency` - Invalid currency code
* `dq_invalid_exchange` - Invalid exchange rate value

#### Calculation Validation Flags (1)
* `dq_line_total_mismatch` - Line total doesn't match quantity × unit_price

### Quality Reporting

**Overall Score:** 84.47% of records are clean (no DQ issues)

**Monitoring Tables:**
* `kpi_data_quality_score` - Summary statistics
* `kpi_dq_breakdown` - Detailed breakdown by flag type

---

## Storage and Catalogs

### Unity Catalog Structure

```
novacart_dev (Catalog)
│
├── bronze (Schema)
│   ├── customers
│   ├── exchange_rates
│   ├── order_items
│   ├── orders
│   └── products
│
├── silver (Schema)
│   ├── slv_customers
│   ├── slv_exchange_rates
│   ├── slv_order_items
│   ├── slv_orders
│   └── slv_products
│
└── gold (Schema)
    ├── dim_customers
    ├── dim_dates
    ├── dim_orders
    ├── dim_products
    ├── fact_order_items
    ├── data_cube  ◀── Pre-joined analytics layer
    ├── kpi_total_revenue
    ├── kpi_revenue_by_country
    ├── kpi_revenue_by_channel
    ├── kpi_revenue_by_currency
    ├── kpi_completed_order_count
    ├── kpi_completed_order_rate
    ├── kpi_average_order_value
    ├── kpi_top_products
    ├── kpi_active_customers_count
    ├── kpi_customer_acquisition
    ├── kpi_data_quality_score
    └── kpi_dq_breakdown
```

### Storage Format

* **Format:** Delta Lake (ACID transactions, time travel, schema evolution)
* **Location:** Managed tables in Unity Catalog (default storage location)
* **Partitioning:** Fact table partitioned by `date_key` for performance

### Table Count Summary

| Layer | Schema | Table Count |
|-------|--------|-------------|
| Bronze | `bronze` | 5 tables |
| Silver | `silver` | 5 tables |
| Gold - Dimensions | `gold` | 4 tables |
| Gold - Facts | `gold` | 1 table |
| Gold - Data Cube | `gold` | 1 table |
| Gold - KPIs | `gold` | 12 tables |
| **Total** | | **28 tables** |

---

## Column Details by Table

### Silver Layer Column Additions

All Silver tables include:
* Original columns from Bronze (cleaned and standardized)
* Data quality flags (specific to each table)
* `load_timestamp` - When data was loaded to Silver

### Gold Layer Column Specifications

#### fact_order_items (Main Fact Table)

**Total Columns:** 43 columns
* Keys: 5
* Measures: 4
* Attributes: 5
* Data Quality Flags: 20
* Metadata: 1

**Complete Column List:**
```
order_item_key, order_key, product_key, customer_key, date_key,
quantity, unit_price, line_total, revenue_usd,
order_status_clean, channel, order_country, category, product_currency,
dq_line_total_mismatch, dq_orphan_order, dq_orphan_product,
dq_missing_order_item_id, dq_invalid_quantity, dq_invalid_item_price,
dq_orphan_customer, dq_missing_order_id, dq_invalid_order_date,
dq_invalid_status, dq_invalid_amount,
dq_missing_product_id, dq_missing_product_name, dq_invalid_product_price,
dq_missing_customer_id, dq_missing_email, dq_invalid_customer_channel,
dq_invalid_exchange_date, dq_invalid_exchange_currency, dq_invalid_exchange,
etl_timestamp
```

#### data_cube (Pre-Joined Analytics Table)

**Total Columns:** 51 columns
* Keys: 5
* Metrics: 4
* Dimension Attributes: 29
* Data Quality Flags: 13

**Purpose:** Single table combining fact_order_items with all dimension tables for simplified analytics

---

## Project Summary

### What Was Built

✅ **Data Ingestion:** 5 CSV files from S3 → Bronze layer

✅ **Data Cleaning:** 5 Silver transformation notebooks with comprehensive cleaning logic

✅ **Dimensional Model:** Star schema with 4 dimensions + 1 fact table

✅ **Data Cube:** Pre-joined analytics table combining fact + all dimensions (51 columns, 2,048 records)

✅ **KPIs:** 10 business metrics calculated from the data cube

✅ **Data Quality:** 20 quality checks across all data sources

✅ **Dashboard:** Lakeview dashboard for visualization

### Key Achievements

1. **Multi-Language Support:** Order status standardization across Chinese, German, Spanish, Hindi, and English

2. **Currency Conversion:** Automatic conversion from 4 currencies (INR, EUR, GBP, CNY) to USD

3. **Data Quality Monitoring:** Comprehensive DQ framework with 84.47% clean data rate

4. **Optimized for Analytics:** 
   * Partitioned fact table
   * Star schema design
   * Pre-joined data cube for fast queries
   * Denormalized attributes for performance

5. **Scalable Architecture:** Medallion architecture supports incremental updates and data lineage

6. **Single Source of Truth:** Data cube eliminates redundant joins and ensures KPI consistency

### Business Insights Enabled

* Revenue analysis by country, channel, currency, and product
* Customer acquisition and activation trends
* Order completion rates and patterns
* Product performance and top sellers
* Data quality monitoring and alerting
* Multi-currency financial reporting in USD

---

## Notebook Locations

### Bronze Layer
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Bronze/Bronze_raw_data`

### Silver Layer
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Silver/01_silver_customers_cleaning`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Silver/02_silver_orders_cleaning`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Silver/03_silver_exchange_rate_cleaning`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Silver/04_silver_products_cleaning`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Silver/05_silver_orders_items_cleaning`

### Gold Layer - Dimensions
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/dim/dim_customers`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/dim/dim_dates`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/dim/dim_orders`
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/dim/dim_products`

### Gold Layer - Fact
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/fact/fact_order_items`

### Gold Layer - Data Cube & Reporting
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/Reporting/NovoCart Data Cube Builder` ◀── **Creates pre-joined data cube**
* `/Users/madhavkkp@gmail.com/NovoCartGlobal/Gold/Reporting/NovoCart Gold KPIs` ◀── **Calculates KPIs from data cube**

### Dashboard
* `/Users/madhavkkp@gmail.com/NovoCart KPI Dashboard.lvdash.json`

---

## Next Steps & Recommendations

1. **Incremental Loading:** Implement incremental updates instead of full refresh
2. **Data Lineage:** Use Delta Lake time travel for historical analysis
3. **Automated Scheduling:** Schedule notebooks using Databricks Jobs
   * Run data cube builder after fact table updates
   * Run KPI notebook after data cube refresh
4. **Alerts:** Set up data quality alerts when thresholds are breached
5. **Additional KPIs:** Consider adding customer lifetime value, churn rate, etc.
6. **Slowly Changing Dimensions:** Implement SCD Type 2 for dimension tracking
7. **Data Cataloging:** Add table descriptions and column comments in Unity Catalog
8. **Query Optimization:** Monitor data cube query performance and add indexes if needed

---

**Documentation Last Updated:** January 2025

**Project Owner:** madhavkkp@gmail.com

**Platform:** Databricks on AWS