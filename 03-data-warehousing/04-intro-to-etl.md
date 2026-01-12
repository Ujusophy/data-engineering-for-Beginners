# Introduction to ETL

## What is ETL?

**ETL = Extract, Transform, Load**

It's the process of moving data from source systems into your data warehouse.

### Simple Example

You have a pizza restaurant with:
- Orders database (PostgreSQL)
- Website analytics (CSV files)
- Customer reviews (MongoDB)

You want all this data in your data warehouse for analysis.

**ETL Process:**
1. **Extract** → Get data from each source
2. **Transform** → Clean and reshape the data
3. **Load** → Put it into the warehouse

```
Sources              ETL Pipeline           Warehouse
┌─────────┐         ┌──────────┐          ┌─────────────┐
│Orders DB│────────>│ Extract  │          │             │
│         │         │    ↓     │          │   Star      │
│Website  │────────>│Transform │─────────>│   Schema    │
│Analytics│         │    ↓     │          │             │
│         │         │   Load   │          │ • Facts     │
│Reviews  │────────>│          │          │ • Dimensions│
└─────────┘         └──────────┘          └─────────────┘
```

## The Three Stages Explained

### 1. Extract - Getting the Data

**Goal**: Pull data from source systems without breaking them.

#### Common Source Types

**Databases:**
```python
import psycopg2

# Extract from PostgreSQL
conn = psycopg2.connect("dbname=orders ...")
query = "SELECT * FROM orders WHERE order_date >= '2024-01-01'"
cursor = conn.cursor()
cursor.execute(query)
orders = cursor.fetchall()
```

**CSV Files:**
```python
import pandas as pd

# Extract from CSV
orders = pd.read_csv('daily_orders.csv')
```

**APIs:**
```python
import requests

# Extract from API
response = requests.get('https://api.example.com/orders')
orders = response.json()
```

**Databases (Different Systems):**
```python
# PostgreSQL
# MySQL
# SQL Server
# Oracle
# MongoDB
```

#### Extraction Strategies

**Full Extraction** - Get everything every time
```python
# Simple but inefficient
orders = pd.read_sql("SELECT * FROM orders", conn)
```

**Pros**: Simple, guarantees completeness
**Cons**: Slow, lots of data transfer

**Incremental Extraction** - Get only new/changed data
```python
# Much more efficient
last_run = '2024-01-15 00:00:00'
query = f"""
    SELECT * FROM orders 
    WHERE updated_at > '{last_run}'
"""
orders = pd.read_sql(query, conn)
```

**Pros**: Fast, less data transfer
**Cons**: Need to track what's been extracted

**How to track**: Store last extraction time
```python
# Track last successful run
last_run_file = 'last_run.txt'

# Read last run time
with open(last_run_file, 'r') as f:
    last_run = f.read()

# Extract new data
new_orders = extract_orders_since(last_run)

# Update last run time
with open(last_run_file, 'w') as f:
    f.write(datetime.now().isoformat())
```

### 2. Transform - Cleaning and Reshaping

**Goal**: Convert data into the format needed for the warehouse.

#### Common Transformations

**Data Cleaning:**
```python
import pandas as pd

# Remove duplicates
df = df.drop_duplicates(subset=['order_id'])

# Handle missing values
df['city'].fillna('Unknown', inplace=True)
df['discount'].fillna(0, inplace=True)

# Fix data types
df['order_date'] = pd.to_datetime(df['order_date'])
df['amount'] = df['amount'].astype(float)

# Remove invalid data
df = df[df['amount'] > 0]  # Remove negative amounts
df = df[df['quantity'] > 0]  # Remove negative quantities
```

**Data Standardization:**
```python
# Standardize formats
df['phone'] = df['phone'].str.replace(r'\D', '')  # Remove non-digits
df['email'] = df['email'].str.lower()  # Lowercase emails
df['state'] = df['state'].str.upper()  # Uppercase state codes

# Standardize values
df['country'] = df['country'].replace({
    'USA': 'United States',
    'US': 'United States',
    'U.S.A.': 'United States'
})
```

**Data Enrichment:**
```python
# Add calculated fields
df['revenue'] = df['quantity'] * df['unit_price']
df['profit'] = df['revenue'] - df['cost']
df['order_year'] = df['order_date'].dt.year
df['order_month'] = df['order_date'].dt.month

# Categorize
df['customer_segment'] = df['total_orders'].apply(
    lambda x: 'VIP' if x > 50 else 'Regular' if x > 10 else 'New'
)
```

**Joining Data:**
```python
# Combine data from multiple sources
orders = pd.read_csv('orders.csv')
customers = pd.read_csv('customers.csv')

# Join
orders_enriched = orders.merge(
    customers[['customer_id', 'city', 'state']],
    on='customer_id',
    how='left'
)
```

**Reshaping for Star Schema:**
```python
# Transform to dimensional model

# Create dimension records
dim_customers = orders[['customer_id', 'customer_name', 'city', 'state']].drop_duplicates()

dim_products = orders[['product_id', 'product_name', 'category', 'price']].drop_duplicates()

# Create fact records
fact_sales = orders[[
    'order_id',
    'customer_id',
    'product_id', 
    'order_date',
    'quantity',
    'revenue',
    'cost'
]]
```

#### Handling Slowly Changing Dimensions

**Type 1 (Overwrite):**
```python
def upsert_customer(customer_data):
    """Update if exists, insert if new"""
    query = """
        INSERT INTO dim_customers (customer_id, name, city, state)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (customer_id)
        DO UPDATE SET
            name = EXCLUDED.name,
            city = EXCLUDED.city,
            state = EXCLUDED.state
    """
    cursor.execute(query, customer_data)
```

**Type 2 (Add New Row):**
```python
def handle_customer_change(new_customer_data):
    """Expire old record, insert new record"""
    
    # Expire old record
    expire_query = """
        UPDATE dim_customers
        SET is_current = FALSE,
            expiration_date = CURRENT_DATE
        WHERE customer_id = %s
          AND is_current = TRUE
    """
    cursor.execute(expire_query, (new_customer_data['customer_id'],))
    
    # Insert new record
    insert_query = """
        INSERT INTO dim_customers 
        (customer_id, name, city, state, effective_date, is_current)
        VALUES (%s, %s, %s, %s, CURRENT_DATE, TRUE)
    """
    cursor.execute(insert_query, new_customer_data)
```

### 3. Load - Putting Data in the Warehouse

**Goal**: Efficiently load transformed data into destination tables.

#### Loading Strategies

**Full Refresh** - Delete and reload everything
```python
def full_refresh_load(df, table_name, conn):
    """Delete all data and reload"""
    
    # Truncate table
    cursor = conn.cursor()
    cursor.execute(f"TRUNCATE TABLE {table_name}")
    
    # Load new data
    df.to_sql(table_name, conn, if_exists='append', index=False)
    
    conn.commit()
```

**Pros**: Simple, guarantees freshness
**Cons**: Slow, downtime during load

**Incremental Load** - Add only new/changed records
```python
def incremental_load(df, table_name, conn):
    """Load only new records"""
    
    # Filter to only new records
    existing_ids = pd.read_sql(
        f"SELECT order_id FROM {table_name}",
        conn
    )
    
    new_records = df[~df['order_id'].isin(existing_ids['order_id'])]
    
    # Load new records
    new_records.to_sql(table_name, conn, if_exists='append', index=False)
```

**Upsert** - Update if exists, insert if new
```python
def upsert_load(df, table_name, key_column, conn):
    """Update existing or insert new"""
    
    for _, row in df.iterrows():
        query = f"""
            INSERT INTO {table_name} 
            ({', '.join(df.columns)})
            VALUES ({', '.join(['%s'] * len(df.columns))})
            ON CONFLICT ({key_column})
            DO UPDATE SET
                {', '.join([f"{col} = EXCLUDED.{col}" for col in df.columns if col != key_column])}
        """
        cursor.execute(query, tuple(row))
    
    conn.commit()
```

#### Bulk Loading (Fastest Method)

**PostgreSQL COPY:**
```python
from io import StringIO

def bulk_load_postgres(df, table_name, conn):
    """Fast bulk load using COPY"""
    
    # Create CSV in memory
    buffer = StringIO()
    df.to_csv(buffer, index=False, header=False)
    buffer.seek(0)
    
    # Use COPY command
    cursor = conn.cursor()
    cursor.copy_from(buffer, table_name, sep=',', columns=df.columns.tolist())
    conn.commit()
```

**Much faster than row-by-row inserts!**

## Complete ETL Example

### Scenario: Load Daily Sales into Warehouse

```python
import pandas as pd
import psycopg2
from datetime import datetime, timedelta
import logging

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def extract_daily_sales(source_conn, date):
    """Extract sales from source database"""
    logger.info(f"Extracting sales for {date}")
    
    query = """
        SELECT 
            o.order_id,
            o.customer_id,
            o.product_id,
            o.order_date,
            o.quantity,
            o.unit_price,
            o.discount_amount,
            c.name as customer_name,
            c.city,
            c.state,
            p.name as product_name,
            p.category
        FROM orders o
        JOIN customers c ON o.customer_id = c.id
        JOIN products p ON o.product_id = p.id
        WHERE DATE(o.order_date) = %s
    """
    
    df = pd.read_sql(query, source_conn, params=(date,))
    logger.info(f"Extracted {len(df)} records")
    
    return df

def transform_sales(df):
    """Transform and clean sales data"""
    logger.info("Transforming data")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['order_id'])
    
    # Calculate revenue and profit
    df['revenue'] = df['quantity'] * df['unit_price'] - df['discount_amount']
    df['revenue'] = df['revenue'].round(2)
    
    # Handle missing values
    df['discount_amount'].fillna(0, inplace=True)
    
    # Create date_id (YYYYMMDD format)
    df['date_id'] = df['order_date'].dt.strftime('%Y%m%d').astype(int)
    
    # Prepare dimension data
    dim_customers = df[['customer_id', 'customer_name', 'city', 'state']].drop_duplicates()
    dim_products = df[['product_id', 'product_name', 'category']].drop_duplicates()
    
    # Prepare fact data
    fact_sales = df[[
        'order_id',
        'customer_id',
        'product_id',
        'date_id',
        'quantity',
        'revenue'
    ]]
    
    logger.info("Transformation complete")
    
    return dim_customers, dim_products, fact_sales

def load_dimensions(dim_customers, dim_products, warehouse_conn):
    """Load dimension tables"""
    logger.info("Loading dimensions")
    
    cursor = warehouse_conn.cursor()
    
    # Load customers (Type 1 - overwrite)
    for _, row in dim_customers.iterrows():
        query = """
            INSERT INTO dim_customers (customer_id, name, city, state)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (customer_id)
            DO UPDATE SET
                name = EXCLUDED.name,
                city = EXCLUDED.city,
                state = EXCLUDED.state
        """
        cursor.execute(query, tuple(row))
    
    # Load products (Type 1 - overwrite)
    for _, row in dim_products.iterrows():
        query = """
            INSERT INTO dim_products (product_id, name, category)
            VALUES (%s, %s, %s)
            ON CONFLICT (product_id)
            DO UPDATE SET
                name = EXCLUDED.name,
                category = EXCLUDED.category
        """
        cursor.execute(query, tuple(row))
    
    warehouse_conn.commit()
    logger.info("Dimensions loaded")

def load_facts(fact_sales, warehouse_conn):
    """Load fact table"""
    logger.info(f"Loading {len(fact_sales)} fact records")
    
    # Use bulk load for speed
    from io import StringIO
    
    buffer = StringIO()
    fact_sales.to_csv(buffer, index=False, header=False)
    buffer.seek(0)
    
    cursor = warehouse_conn.cursor()
    cursor.copy_from(
        buffer,
        'fact_sales',
        sep=',',
        columns=fact_sales.columns.tolist()
    )
    
    warehouse_conn.commit()
    logger.info("Facts loaded")

def run_etl_pipeline(date):
    """Run complete ETL pipeline"""
    logger.info(f"Starting ETL for {date}")
    
    try:
        # Connect to source
        source_conn = psycopg2.connect("dbname=source_db ...")
        
        # Connect to warehouse
        warehouse_conn = psycopg2.connect("dbname=warehouse_db ...")
        
        # EXTRACT
        sales_data = extract_daily_sales(source_conn, date)
        
        # TRANSFORM
        dim_customers, dim_products, fact_sales = transform_sales(sales_data)
        
        # LOAD
        load_dimensions(dim_customers, dim_products, warehouse_conn)
        load_facts(fact_sales, warehouse_conn)
        
        logger.info("ETL completed successfully")
        
    except Exception as e:
        logger.error(f"ETL failed: {e}")
        raise
    
    finally:
        source_conn.close()
        warehouse_conn.close()

# Run for yesterday
if __name__ == "__main__":
    yesterday = (datetime.now() - timedelta(days=1)).date()
    run_etl_pipeline(yesterday)
```

## ETL Best Practices

### 1. Idempotent Pipelines
**Idempotent** = Running twice gives same result

```python
# Bad - adds duplicate data
def load_data(df):
    df.to_sql('sales', conn, if_exists='append')  # Duplicates if run twice!

# Good - prevents duplicates
def load_data(df):
    # Delete existing data for this date first
    cursor.execute("DELETE FROM sales WHERE date = %s", (date,))
    df.to_sql('sales', conn, if_exists='append')  # Safe to rerun
```

### 2. Error Handling

```python
def etl_with_error_handling():
    try:
        # Extract
        data = extract()
        
        # Transform
        transformed = transform(data)
        
        # Load
        load(transformed)
        
        # Log success
        log_success()
        
    except Exception as e:
        # Log error
        logger.error(f"ETL failed: {e}")
        
        # Alert team
        send_alert(f"ETL pipeline failed: {e}")
        
        # Don't silently fail!
        raise
```

### 3. Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('etl.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

logger.info("Starting ETL")
logger.info(f"Extracted {len(df)} records")
logger.warning("Missing values found in column X")
logger.error("Database connection failed")
```

### 4. Data Quality Checks

```python
def validate_data(df):
    """Check data quality before loading"""
    
    # Check for required columns
    required_cols = ['order_id', 'customer_id', 'product_id', 'quantity', 'revenue']
    missing = set(required_cols) - set(df.columns)
    if missing:
        raise ValueError(f"Missing columns: {missing}")
    
    # Check for nulls
    null_counts = df[required_cols].isnull().sum()
    if null_counts.any():
        logger.warning(f"Null values found: {null_counts}")
    
    # Check for duplicates
    dupe_count = df['order_id'].duplicated().sum()
    if dupe_count > 0:
        logger.warning(f"Found {dupe_count} duplicate orders")
    
    # Check for invalid values
    if (df['quantity'] <= 0).any():
        raise ValueError("Found negative or zero quantities")
    
    if (df['revenue'] < 0).any():
        raise ValueError("Found negative revenue")
    
    logger.info("Data quality checks passed")
```

### 5. Incremental vs Full Load

```python
def smart_load_strategy(table_name, date):
    """Choose load strategy based on data volume"""
    
    # Count records for this date
    record_count = count_records_for_date(date)
    
    if record_count < 10000:
        # Small load - just append
        return incremental_load(table_name, date)
    else:
        # Large load - partition and load in batches
        return batch_load(table_name, date, batch_size=5000)
```

## ELT (Modern Alternative)

**ELT = Extract, Load, Transform**

Load data first, transform in the warehouse.

```
Sources         Warehouse              Analytics
┌────────┐     ┌─────────────┐       ┌──────────┐
│Sources │────>│ Raw Tables  │──────>│Transform │
└────────┘     │ (Staging)   │       │ in DB    │
               └─────────────┘       └──────────┘
                                           │
                                           v
                                     ┌──────────┐
                                     │  Star    │
                                     │  Schema  │
                                     └──────────┘
```

**Why ELT?**
- Cloud warehouses are powerful (can transform)
- Keep raw data for reprocessing
- Faster initial load

**When to use:**
- Cloud data warehouses (Snowflake, BigQuery)
- Complex transformations
- Need to keep raw data

**When to use ETL:**
- Traditional databases
- Simple transformations
- Privacy/security (transform before loading)

## Common Tools

**Python Libraries:**
- Pandas - Data manipulation
- SQLAlchemy - Database connections
- psycopg2 - PostgreSQL
- pymongo - MongoDB
- requests - APIs

**ETL Tools (Advanced):**
- Apache Airflow - Orchestration
- dbt - Transformations
- Airbyte - Extract and Load
- Fivetran - Managed ELT
- Talend - Enterprise ETL

**For learning**: Start with Python scripts!

## Practice Exercise

Build an ETL pipeline for a **Weather Data Warehouse**:

**Source Data** (CSV files):
- `weather_daily.csv` - Daily weather readings
  - Columns: station_id, date, temp_high, temp_low, precipitation, humidity
- `stations.csv` - Weather station info
  - Columns: station_id, name, city, state, latitude, longitude

**Requirements:**
1. Extract data from CSV files
2. Transform:
   - Calculate average temperature
   - Convert Fahrenheit to Celsius
   - Handle missing values
   - Create date_id (YYYYMMDD format)
3. Load into star schema:
   - dim_stations
   - dim_dates
   - fact_weather

**Your Tasks:**
1. Design the star schema (write CREATE TABLE statements)
2. Write Python ETL script with three functions:
   - extract()
   - transform()
   - load()
3. Add error handling and logging
4. Make it idempotent (safe to rerun)
5. Add data quality checks
6. Write SQL queries to analyze:
   - Average temperature by state
   - Rainiest cities
   - Temperature trends over time

## Summary

**Key Concepts:**
- ETL = Extract, Transform, Load
- Extract: Get data from sources
- Transform: Clean, reshape, enrich
- Load: Put into warehouse
- Make pipelines idempotent
- Add logging and error handling
- Validate data quality

**Remember:**
- Start simple (Python scripts)
- Incremental loading when possible
- Bulk load for performance
- Transform for star schema
- Log everything
- Handle errors gracefully

## Additional Resources

- [ETL Best Practices](https://www.startdataengineering.com/post/etl-best-practices/)
- [Pandas for Data Engineering](https://pandas.pydata.org/docs/)
- [Python ETL Tutorial](https://realpython.com/python-data-engineering/)
