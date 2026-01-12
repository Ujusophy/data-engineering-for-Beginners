# Fact and Dimension Tables

## Deep Dive into Facts and Dimensions

Now that you understand star schema basics, let's explore fact and dimension tables in more detail with practical examples.

## Fact Tables - The Details

### What Makes a Good Fact Table?

**Remember**: Fact tables store measurements/metrics from business events.

### Types of Fact Tables

#### 1. Transaction Fact Table
**Most common type**. One row = one business event.

**Example - Retail Sales:**
```sql
CREATE TABLE fact_sales (
    sale_id SERIAL PRIMARY KEY,
    date_id INTEGER NOT NULL,
    customer_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    store_id INTEGER NOT NULL,
    
    -- Facts (measurements)
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    revenue DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2) NOT NULL,
    profit DECIMAL(10,2) NOT NULL
);
```

**Grain**: One row per line item in a sale

**Characteristics:**
- Grows continuously (new transactions added)
- Very large (millions to billions of rows)
- Immutable (rarely updated after insert)

#### 2. Periodic Snapshot Fact Table
One row = summary for a time period.

**Example - Daily Inventory:**
```sql
CREATE TABLE fact_inventory_daily (
    snapshot_id SERIAL PRIMARY KEY,
    date_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    warehouse_id INTEGER NOT NULL,
    
    -- Facts (measurements at end of day)
    units_on_hand INTEGER NOT NULL,
    units_on_order INTEGER NOT NULL,
    units_sold_today INTEGER NOT NULL,
    units_received_today INTEGER NOT NULL,
    
    UNIQUE(date_id, product_id, warehouse_id)
);
```

**Grain**: One row per product per warehouse per day

**Characteristics:**
- Regular snapshots (daily, weekly, monthly)
- Predictable size
- Shows status at a point in time

**Use when**: You need to know the state of something regularly (inventory levels, account balances)

#### 3. Accumulating Snapshot Fact Table
One row = entire lifecycle of an event.

**Example - Order Fulfillment:**
```sql
CREATE TABLE fact_order_fulfillment (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    
    -- Multiple date foreign keys for each milestone
    order_date_id INTEGER NOT NULL,
    payment_date_id INTEGER,
    shipping_date_id INTEGER,
    delivery_date_id INTEGER,
    
    -- Facts
    order_amount DECIMAL(10,2),
    days_to_ship INTEGER,
    days_to_deliver INTEGER,
    
    -- Status
    current_status VARCHAR(50)
);
```

**Grain**: One row per order (updated as order progresses)

**Characteristics:**
- Row is updated as process moves forward
- Multiple date dimensions (one per milestone)
- Less common than transaction facts

**Use when**: Tracking a process with multiple stages (orders, support tickets, loans)

### Fact Table Best Practices

#### 1. Use Surrogate Keys for Dimensions

```sql
-- Don't use natural keys
fact_sales (
    customer_email VARCHAR(255),  -- Bad!
    ...
)

-- Use surrogate keys
fact_sales (
    customer_id INTEGER,  -- Good! Links to dim_customers
    ...
)
```

**Why?**
- Smaller storage (integer vs string)
- Faster joins
- Handles changes in dimension

#### 2. Store Additive Facts

```sql
CREATE TABLE fact_sales (
    -- Additive (can be summed)
    quantity INTEGER,           -- ✓
    revenue DECIMAL(10,2),      -- ✓
    cost DECIMAL(10,2),         -- ✓
    
    -- Semi-additive (can sum across some dimensions)
    account_balance DECIMAL(10,2),  -- Can't sum across time
    
    -- Non-additive (never sum)
    unit_price DECIMAL(10,2),   -- Calculate from revenue/quantity
    profit_margin DECIMAL(5,2)  -- Calculate from profit/revenue
);
```

#### 3. Pre-calculate Derived Facts

```sql
-- Store calculated values to avoid runtime calculations
CREATE TABLE fact_sales (
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    
    -- Pre-calculated
    revenue DECIMAL(10,2),      -- quantity * unit_price - discount
    cost DECIMAL(10,2),
    profit DECIMAL(10,2)        -- revenue - cost
);
```

**Why?** Queries are much faster when values are pre-calculated.

#### 4. Keep Fact Tables Narrow

```sql
-- Bad - too many columns
CREATE TABLE fact_sales (
    sale_id INTEGER,
    customer_name VARCHAR(100),     -- Should be in dimension
    customer_city VARCHAR(100),     -- Should be in dimension
    product_name VARCHAR(200),      -- Should be in dimension
    product_category VARCHAR(100),  -- Should be in dimension
    ...
);

-- Good - only keys and facts
CREATE TABLE fact_sales (
    sale_id INTEGER,
    customer_id INTEGER,   -- Link to dimension
    product_id INTEGER,    -- Link to dimension
    date_id INTEGER,
    quantity INTEGER,
    revenue DECIMAL(10,2),
    ...
);
```

## Dimension Tables - The Details

### What Makes a Good Dimension Table?

**Remember**: Dimensions provide context about the facts.

### Dimension Design Patterns

#### 1. Descriptive Attributes

Add as many descriptive attributes as useful:

```sql
CREATE TABLE dim_products (
    product_id SERIAL PRIMARY KEY,
    
    -- Identifiers
    sku VARCHAR(50) UNIQUE NOT NULL,
    
    -- Names and descriptions
    product_name VARCHAR(200) NOT NULL,
    short_description TEXT,
    long_description TEXT,
    
    -- Categorization
    department VARCHAR(100),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    
    -- Product attributes
    size VARCHAR(50),
    color VARCHAR(50),
    material VARCHAR(100),
    weight_kg DECIMAL(8,2),
    
    -- Pricing
    unit_cost DECIMAL(10,2),
    list_price DECIMAL(10,2),
    
    -- Flags
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    is_on_sale BOOLEAN DEFAULT FALSE,
    
    -- Dates
    launch_date DATE,
    discontinued_date DATE
);
```

**Principle**: Denormalize! Store redundant data for easy querying.

#### 2. Date Dimension (Special and Important)

```sql
CREATE TABLE dim_dates (
    date_id INTEGER PRIMARY KEY,  -- Format: YYYYMMDD (20240115)
    
    -- The actual date
    date DATE NOT NULL UNIQUE,
    
    -- Day attributes
    day_of_month INTEGER,
    day_of_week INTEGER,           -- 1-7
    day_name VARCHAR(10),           -- Monday, Tuesday...
    day_of_year INTEGER,            -- 1-365
    
    -- Week attributes
    week_of_year INTEGER,
    iso_week INTEGER,
    week_start_date DATE,
    week_end_date DATE,
    
    -- Month attributes
    month_number INTEGER,           -- 1-12
    month_name VARCHAR(10),         -- January, February...
    month_abbr VARCHAR(3),          -- Jan, Feb...
    days_in_month INTEGER,
    month_start_date DATE,
    month_end_date DATE,
    
    -- Quarter attributes
    quarter INTEGER,                -- 1-4
    quarter_name VARCHAR(10),       -- Q1, Q2...
    quarter_start_date DATE,
    quarter_end_date DATE,
    
    -- Year attributes
    year INTEGER,
    year_month VARCHAR(7),          -- 2024-01
    year_quarter VARCHAR(7),        -- 2024-Q1
    
    -- Business attributes
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(100),
    is_business_day BOOLEAN,
    
    -- Fiscal period (if different from calendar)
    fiscal_year INTEGER,
    fiscal_quarter INTEGER,
    fiscal_month INTEGER
);
```

**Populate date dimension:**
```sql
-- Generate 10 years of dates
INSERT INTO dim_dates (date_id, date, day_of_month, month_number, year, ...)
SELECT 
    TO_CHAR(date, 'YYYYMMDD')::INTEGER as date_id,
    date,
    EXTRACT(DAY FROM date) as day_of_month,
    EXTRACT(MONTH FROM date) as month_number,
    EXTRACT(YEAR FROM date) as year,
    ...
FROM generate_series(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) as date;
```

**Why so detailed?** Makes queries much easier:
```sql
-- Without date dimension
SELECT SUM(revenue)
FROM fact_sales
WHERE EXTRACT(QUARTER FROM order_date) = 1
  AND EXTRACT(YEAR FROM order_date) = 2024
  AND EXTRACT(DOW FROM order_date) NOT IN (0, 6);  -- Exclude weekends

-- With date dimension
SELECT SUM(fs.revenue)
FROM fact_sales fs
JOIN dim_dates d ON fs.date_id = d.date_id
WHERE d.year = 2024
  AND d.quarter = 1
  AND NOT d.is_weekend;
```

#### 3. Hierarchies in Dimensions

Store hierarchies denormalized in the dimension:

```sql
CREATE TABLE dim_geography (
    geography_id SERIAL PRIMARY KEY,
    
    -- Lowest level (most granular)
    city VARCHAR(100),
    postal_code VARCHAR(20),
    
    -- Middle levels
    county VARCHAR(100),
    state VARCHAR(100),
    state_code VARCHAR(2),
    
    -- Highest level
    country VARCHAR(100),
    country_code VARCHAR(3),
    region VARCHAR(100),  -- North America, Europe, Asia...
    
    -- Latitude/Longitude for mapping
    latitude DECIMAL(10, 7),
    longitude DECIMAL(10, 7)
);
```

**Query at any level:**
```sql
-- City level
SELECT g.city, SUM(fs.revenue)
FROM fact_sales fs
JOIN dim_geography g ON fs.geography_id = g.geography_id
GROUP BY g.city;

-- State level
SELECT g.state, SUM(fs.revenue)
FROM fact_sales fs
JOIN dim_geography g ON fs.geography_id = g.geography_id
GROUP BY g.state;

-- Region level
SELECT g.region, SUM(fs.revenue)
FROM fact_sales fs
JOIN dim_geography g ON fs.geography_id = g.geography_id
GROUP BY g.region;
```

### Slowly Changing Dimensions (SCD)

Dimensions change over time. How do we handle this?

#### Type 0: No Changes Allowed
Original values never change. Rare.

#### Type 1: Overwrite (Most Common)
Update the dimension, no history kept.

**Example:**
```sql
-- Customer moves
UPDATE dim_customers
SET city = 'Seattle', state = 'WA'
WHERE customer_id = 123;
```

**Pros:**
- Simple
- No extra storage

**Cons:**
- Lose history
- Can't analyze "where was customer when they made this purchase?"

**Use when:** History doesn't matter (typo fixes, phone number updates)

#### Type 2: Add New Row (Common for Important Changes)
Keep history by adding new row with new version.

**Example:**
```sql
CREATE TABLE dim_customers (
    customer_key SERIAL PRIMARY KEY,  -- Surrogate key
    customer_id VARCHAR(50),          -- Natural key
    name VARCHAR(100),
    city VARCHAR(100),
    state VARCHAR(50),
    
    -- SCD Type 2 columns
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_current BOOLEAN DEFAULT TRUE,
    
    version INTEGER DEFAULT 1
);

-- Customer 123 moves from LA to Seattle
-- Old row
customer_key: 1
customer_id: '123'
city: 'Los Angeles'
effective_date: '2023-01-01'
expiration_date: '2024-01-15'
is_current: FALSE

-- New row (inserted)
customer_key: 2  -- New surrogate key!
customer_id: '123'
city: 'Seattle'
effective_date: '2024-01-15'
expiration_date: NULL
is_current: TRUE
```

**Pros:**
- Full history preserved
- Can analyze past behavior

**Cons:**
- More complex
- Dimension table grows

**Use when:** Need to track important changes (customer segments, product pricing, employee departments)

#### Type 3: Add New Column (Rare)
Store old and new value in separate columns.

**Example:**
```sql
CREATE TABLE dim_products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(200),
    current_category VARCHAR(100),
    previous_category VARCHAR(100),
    category_change_date DATE
);
```

**Use when:** Only need to track one previous value (rarely used)

### Special Dimension Types

#### Junk Dimension
Combine low-cardinality flags into one dimension.

```sql
-- Instead of many boolean columns in fact table
CREATE TABLE dim_transaction_flags (
    flag_key SERIAL PRIMARY KEY,
    is_online BOOLEAN,
    is_gift BOOLEAN,
    is_expedited_shipping BOOLEAN,
    is_first_time_customer BOOLEAN,
    payment_method VARCHAR(50)
);

-- Fact table references this dimension
CREATE TABLE fact_sales (
    sale_id INTEGER,
    flag_key INTEGER REFERENCES dim_transaction_flags,
    ...
);
```

**Benefit:** Reduces fact table width, easier to query combinations of flags.

#### Degenerate Dimension
Dimension attribute stored directly in fact table (no separate dimension table).

```sql
CREATE TABLE fact_sales (
    sale_id INTEGER,
    customer_id INTEGER,
    product_id INTEGER,
    
    -- Degenerate dimension
    order_number VARCHAR(50),  -- No dim_orders table
    invoice_number VARCHAR(50),
    
    quantity INTEGER,
    revenue DECIMAL(10,2)
);
```

**Use when:**
- Unique identifiers (order number, transaction ID)
- Don't need other attributes
- Not used for grouping/filtering

#### Role-Playing Dimension
Same dimension used multiple times with different context.

**Example - Multiple Dates:**
```sql
CREATE TABLE fact_orders (
    order_id INTEGER,
    
    -- Same dim_dates table, different roles
    order_date_id INTEGER REFERENCES dim_dates(date_id),
    ship_date_id INTEGER REFERENCES dim_dates(date_id),
    delivery_date_id INTEGER REFERENCES dim_dates(date_id),
    
    ...
);

-- Query using different date roles
SELECT 
    order_dates.month_name as order_month,
    ship_dates.month_name as ship_month,
    COUNT(*) as order_count
FROM fact_orders fo
JOIN dim_dates order_dates ON fo.order_date_id = order_dates.date_id
JOIN dim_dates ship_dates ON fo.ship_date_id = ship_dates.date_id
GROUP BY order_dates.month_name, ship_dates.month_name;
```

## Practical Examples

### Example 1: E-commerce Data Warehouse

**Dimensions:**
```sql
-- Customer Dimension
CREATE TABLE dim_customers (
    customer_id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50),
    customer_since DATE,
    loyalty_tier VARCHAR(50)
);

-- Product Dimension
CREATE TABLE dim_products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50),
    name VARCHAR(200),
    category VARCHAR(100),
    brand VARCHAR(100),
    unit_price DECIMAL(10,2)
);

-- Date Dimension (as shown earlier)
```

**Fact Table:**
```sql
CREATE TABLE fact_orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES dim_customers,
    product_id INTEGER REFERENCES dim_products,
    order_date_id INTEGER REFERENCES dim_dates,
    ship_date_id INTEGER REFERENCES dim_dates,
    
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    shipping_cost DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2)
);
```

**Sample Queries:**
```sql
-- Monthly revenue by category
SELECT 
    d.year_month,
    p.category,
    SUM(fo.total_amount) as revenue
FROM fact_orders fo
JOIN dim_dates d ON fo.order_date_id = d.date_id
JOIN dim_products p ON fo.product_id = p.product_id
GROUP BY d.year_month, p.category
ORDER BY d.year_month, revenue DESC;

-- Top customers by state
SELECT 
    c.state,
    c.name,
    SUM(fo.total_amount) as lifetime_value
FROM fact_orders fo
JOIN dim_customers c ON fo.customer_id = c.customer_id
GROUP BY c.state, c.name, c.customer_id
ORDER BY c.state, lifetime_value DESC;
```

## Common Mistakes to Avoid

### 1. Don't Store Everything in Fact Table
```sql
-- Bad
CREATE TABLE fact_sales (
    customer_name VARCHAR(100),  -- Belongs in dimension
    product_name VARCHAR(200),   -- Belongs in dimension
    quantity INTEGER,
    revenue DECIMAL(10,2)
);

-- Good
CREATE TABLE fact_sales (
    customer_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    revenue DECIMAL(10,2)
);
```

### 2. Don't Normalize Dimensions
```sql
-- Bad (normalized)
dim_products → dim_categories → dim_departments

-- Good (denormalized)
dim_products (includes category, department in same table)
```

### 3. Don't Forget Indexes
```sql
-- Always index foreign keys in fact tables
CREATE INDEX idx_fact_sales_customer ON fact_sales(customer_id);
CREATE INDEX idx_fact_sales_product ON fact_sales(product_id);
CREATE INDEX idx_fact_sales_date ON fact_sales(date_id);
```

### 4. Don't Mix Grains
```sql
-- Bad - mixed grain
fact_sales:
Row 1: Daily summary for Product A
Row 2: Individual transaction for Product B

-- Good - consistent grain
All rows: Individual transactions
OR
All rows: Daily summaries
```

## Practice Exercise

Build a fact and dimension model for a **Gym Membership System**:

**Requirements:**
1. Track member check-ins (when members come to the gym)
2. Track membership payments
3. Analyze:
   - Most popular times to visit
   - Which members visit most frequently
   - Revenue by membership type
   - Retention rates

**Your Tasks:**
1. Identify what fact tables you need (hint: probably 2)
2. Define the grain for each fact table
3. List dimensions needed
4. Design:
   - dim_members (with SCD Type 2 for membership_type changes)
   - dim_dates
   - dim_time_of_day (for check-ins)
   - fact_checkins
   - fact_payments
5. Write CREATE TABLE statements
6. Write 5 analysis queries

**Think about:**
- What's the grain of each fact table?
- Should membership_type changes be tracked historically?
- How to handle time of day for check-ins?
- What measurements belong in each fact table?

## Summary

**Fact Tables:**
- Store measurements/metrics
- Transaction, Snapshot, or Accumulating
- Keep narrow (only keys and facts)
- Pre-calculate derived values
- Index foreign keys

**Dimension Tables:**
- Store descriptive context
- Denormalize for easy querying
- Handle changes with SCD types
- Date dimension is critical
- Add as many useful attributes as possible

**Remember:**
- Facts = Numbers (what happened)
- Dimensions = Context (who, what, when, where, why)
- Denormalize dimensions
- Normalize is for OLTP, Denormalize is for OLAP

## Next Steps

Learn about [Introduction to ETL](04-intro-to-etl.md) to understand how to load data into your warehouse!

## Additional Resources

- [Kimball's Fact Table Types](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [Slowly Changing Dimensions Explained](https://www.holistics.io/blog/slowly-changing-dimensions/)

---

**Next**: [Introduction to ETL](04-intro-to-etl.md)
