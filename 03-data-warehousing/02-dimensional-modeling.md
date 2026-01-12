# Dimensional Modeling Basics

## What is Dimensional Modeling?

Dimensional modeling is a simple way to organize data in a warehouse so it's easy to understand and query. Think of it as organizing your data for humans, not just computers.

**The Goal**: Make it easy to answer business questions.

## The Problem with Normal Databases

### Normalized Database (OLTP)
For our pizza restaurant here's how orders might look:

```
orders table:
order_id | customer_id | pizza_id | quantity | order_date

customers table:
customer_id | name | email | city | state

pizzas table:
pizza_id | name | size | price | category_id

categories table:
category_id | name
```

**To answer**: "How much revenue did we make from large pizzas in California last month?"

```sql
-- Complex query with many joins
SELECT SUM(o.quantity * p.price) as revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN pizzas p ON o.pizza_id = p.pizza_id
JOIN categories cat ON p.category_id = cat.category_id
WHERE p.size = 'Large'
  AND c.state = 'California'
  AND o.order_date >= '2024-01-01'
  AND o.order_date < '2024-02-01';
```

**Problems:**
- Too many joins
- Slow for analysis
- Hard to understand
- Analysts make mistakes

### Dimensional Model (OLAP)
Same data, organized differently:

```
fact_sales (the numbers/metrics):
sale_id | customer_id | product_id | date_id | quantity | revenue

dim_customers (who):
customer_id | name | email | city | state | region

dim_products (what):
product_id | name | size | category | price

dim_dates (when):
date_id | date | day | month | year | quarter
```

**Same question, simpler query:**

```sql
-- Much simpler!
SELECT SUM(revenue) as total_revenue
FROM fact_sales fs
JOIN dim_customers c ON fs.customer_id = c.customer_id
JOIN dim_products p ON fs.product_id = p.product_id
JOIN dim_dates d ON fs.date_id = d.date_id
WHERE p.size = 'Large'
  AND c.state = 'California'
  AND d.year = 2024
  AND d.month = 1;
```

**Benefits:**
- Fewer joins
- Faster queries
- Easier to understand
- Less chance of errors

## Star Schema - The Most Common Pattern

### What Does It Look Like?

```
         dim_customers              dim_products
         ┌──────────────┐          ┌──────────────┐
         │customer_id PK│          │product_id  PK│
         │name          │          │name          │
         │city          │          │category      │
         │state         │          │price         │
         └──────┬───────┘          └──────┬───────┘
                │                         │
                │                         │
         ┌──────┴─────────────────────────┴───────┐
         │         fact_sales                      │
         │  ────────────────────────────────       │
         │  sale_id          PK                    │
         │  customer_id      FK ────────────────┐  │
         │  product_id       FK ───────────────┐│  │
         │  date_id          FK ──────────────┐││  │
         │  quantity                          │││  │
         │  revenue                           │││  │
         │  cost                              │││  │
         └────────────────────────────────────┴┴┴──┘
                │                            
         ┌──────┴───────┐
         │dim_dates     │
         │date_id     PK│
         │date          │
         │month         │
         │year          │
         │day_of_week   │
         └──────────────┘
```

**Why "Star"?** Because it looks like a star! ⭐
- **Center** = Fact table (the numbers)
- **Points** = Dimension tables (the context)

## The Two Types of Tables

### 1. Fact Tables (The Numbers)

**What it stores**: Measurements, metrics, events that happened

**Think of it as**: The transaction records, the "what happened"

**Example - Sales Fact Table:**
```
fact_sales:
┌─────────┬─────────────┬────────────┬─────────┬──────────┬─────────┬──────┐
│sale_id  │customer_id  │product_id  │date_id  │quantity  │revenue  │cost  │
├─────────┼─────────────┼────────────┼─────────┼──────────┼─────────┼──────┤
│1        │101          │5001        │20240115 │2         │19.98    │8.00  │
│2        │102          │5002        │20240115 │1         │24.99    │10.00 │
│3        │101          │5001        │20240116 │1         │9.99     │4.00  │
└─────────┴─────────────┴────────────┴─────────┴──────────┴─────────┴──────┘
```

**Characteristics:**
- Usually has MANY rows (millions, billions)
- Contains foreign keys to dimensions
- Contains numeric measurements (quantity, revenue, cost)
- Represents business events
- Grows over time

**Types of Measurements:**
- **Additive**: Can sum them up (revenue, quantity)
- **Semi-additive**: Can't always sum (account balance - can't sum across time)
- **Non-additive**: Never sum (ratios, percentages)

### 2. Dimension Tables (The Context)

**What it stores**: Descriptive attributes, the "who, what, where, when, why"

**Think of it as**: The context about the facts

**Example - Customer Dimension:**
```
dim_customers:
┌─────────────┬─────────────┬──────────────────────┬──────────────┬───────┬────────┐
│customer_id  │name         │email                 │city          │state  │region  │
├─────────────┼─────────────┼──────────────────────┼──────────────┼───────┼────────┤
│101          │Alice Smith  │alice@email.com       │Los Angeles   │CA     │West    │
│102          │Bob Jones    │bob@email.com         │New York      │NY     │East    │
│103          │Carol White  │carol@email.com       │Chicago       │IL     │Central │
└─────────────┴─────────────┴──────────────────────┴──────────────┴───────┴────────┘
```

**Characteristics:**
- Usually has FEWER rows (thousands, not millions)
- Contains descriptive attributes (names, categories, dates)
- Wide tables (many columns)
- Changes slowly (Slowly Changing Dimensions - we'll cover this)

## Building a Simple Star Schema

### Example: Online Store

**Business Questions:**
- How much did we sell each day?
- Which products are most popular?
- Which customers spend the most?
- How do sales vary by region?

### Step 1: Identify the Facts (Measurements)

What are we measuring? **Sales!**

Metrics we care about:
- Quantity sold
- Revenue
- Cost
- Profit

### Step 2: Identify the Dimensions (Context)

What context do we need?
- **Customer**: Who bought?
- **Product**: What was bought?
- **Date**: When was it bought?
- **Store**: Where was it bought? (if applicable)

### Step 3: Design Fact Table

```sql
CREATE TABLE fact_sales (
    sale_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    date_id INTEGER NOT NULL,
    store_id INTEGER,
    
    -- Measurements (the facts)
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    revenue DECIMAL(10, 2) NOT NULL,
    cost DECIMAL(10, 2) NOT NULL,
    profit DECIMAL(10, 2) NOT NULL,
    
    -- Foreign keys
    FOREIGN KEY (customer_id) REFERENCES dim_customers(customer_id),
    FOREIGN KEY (product_id) REFERENCES dim_products(product_id),
    FOREIGN KEY (date_id) REFERENCES dim_dates(date_id)
);

CREATE INDEX idx_fact_sales_customer ON fact_sales(customer_id);
CREATE INDEX idx_fact_sales_product ON fact_sales(product_id);
CREATE INDEX idx_fact_sales_date ON fact_sales(date_id);
```

### Step 4: Design Dimension Tables

**Customer Dimension:**
```sql
CREATE TABLE dim_customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(20),
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50),
    region VARCHAR(50),
    customer_since DATE,
    customer_segment VARCHAR(50)  -- VIP, Regular, New
);
```

**Product Dimension:**
```sql
CREATE TABLE dim_products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_cost DECIMAL(10, 2),
    unit_price DECIMAL(10, 2),
    is_active BOOLEAN DEFAULT TRUE
);
```

**Date Dimension** (very important!)
```sql
CREATE TABLE dim_dates (
    date_id INTEGER PRIMARY KEY,  -- Format: YYYYMMDD (20240115)
    date DATE NOT NULL UNIQUE,
    day INTEGER,
    day_name VARCHAR(10),  -- Monday, Tuesday
    week INTEGER,
    month INTEGER,
    month_name VARCHAR(10),  -- January, February
    quarter INTEGER,
    year INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(100)
);
```

**Why a Date Dimension?**
Instead of just storing dates, we pre-calculate useful attributes:
- Easy to filter by quarter, month, year
- Easy to find weekends vs weekdays
- Easy to exclude holidays
- Much faster queries

## Querying the Star Schema

### Example 1: Total Sales by Month

```sql
SELECT 
    d.year,
    d.month_name,
    SUM(fs.revenue) as total_revenue,
    SUM(fs.quantity) as total_quantity
FROM fact_sales fs
JOIN dim_dates d ON fs.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.year, d.month, d.month_name
ORDER BY d.month;
```

### Example 2: Top 10 Customers

```sql
SELECT 
    c.name,
    c.city,
    c.state,
    SUM(fs.revenue) as total_spent,
    COUNT(*) as order_count
FROM fact_sales fs
JOIN dim_customers c ON fs.customer_id = c.customer_id
GROUP BY c.customer_id, c.name, c.city, c.state
ORDER BY total_spent DESC
LIMIT 10;
```

### Example 3: Sales by Product Category

```sql
SELECT 
    p.category,
    SUM(fs.revenue) as total_revenue,
    SUM(fs.profit) as total_profit,
    SUM(fs.quantity) as units_sold
FROM fact_sales fs
JOIN dim_products p ON fs.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;
```

### Example 4: Weekend vs Weekday Sales

```sql
SELECT 
    CASE 
        WHEN d.is_weekend THEN 'Weekend'
        ELSE 'Weekday'
    END as day_type,
    SUM(fs.revenue) as total_revenue,
    COUNT(*) as transaction_count,
    AVG(fs.revenue) as avg_transaction
FROM fact_sales fs
JOIN dim_dates d ON fs.date_id = d.date_id
GROUP BY d.is_weekend;
```

## Snowflake Schema (Briefly)

Sometimes dimensions are normalized further:

```
dim_products ──> dim_categories ──> dim_category_types
```

**Star Schema**: All attributes in dimension (denormalized)
**Snowflake Schema**: Dimensions are normalized (split into sub-dimensions)

**For beginners**: Stick with star schema. It's simpler and faster to query.

## Why This Design Works

### 1. Easy to Understand
```
"Show me sales by customer, product, and month"
= Join fact table with those 3 dimensions
```

### 2. Fast Queries
```
- Fewer joins than normalized schema
- Indexes on foreign keys
- Pre-calculated attributes in dimensions
```

### 3. Flexible Analysis
```
- Add new dimensions easily
- Analyze from any angle
- Combine dimensions as needed
```

### 4. Consistent
```
- One fact table = one business process
- Dimensions shared across facts
- Same customer dimension for sales and returns
```

## Common Patterns

### Grain: What Does One Row Represent?

**Important**: Define the grain clearly!

```
Examples:
- "One row = One sale transaction"
- "One row = One line item in an order"
- "One row = Daily sales summary per product"
- "One row = Monthly sales per customer per product"
```

**Rule**: All facts in the table must be at the same grain!

### Degenerate Dimensions

Sometimes we store dimension attributes directly in the fact table:

```sql
CREATE TABLE fact_sales (
    ...
    order_number VARCHAR(50),  -- Degenerate dimension
    ...
);
```

Use when:
- The attribute doesn't belong to any dimension
- It's a unique identifier (order number, invoice number)

### Junk Dimensions

Low-cardinality flags grouped together:

```sql
CREATE TABLE dim_transaction_type (
    transaction_type_id SERIAL PRIMARY KEY,
    is_online BOOLEAN,
    is_discounted BOOLEAN,
    payment_method VARCHAR(50),
    shipping_method VARCHAR(50)
);
```

Instead of 4 columns in fact table → 1 foreign key

## Design Process Summary

1. **Choose a business process** (e.g., sales, orders, shipments)
2. **Declare the grain** (what does one row represent?)
3. **Identify dimensions** (who, what, where, when, why)
4. **Identify facts** (the measurements/metrics)
5. **Build dimension tables** (denormalized, descriptive)
6. **Build fact table** (foreign keys + measurements)
7. **Add indexes** (on foreign keys)

## Practice Exercise

Design a star schema for a **Movie Streaming Service**:

**Business Process**: Track what movies users watch

**Questions to answer:**
- How many hours of content is watched per day?
- Which movies are most popular?
- Which users watch the most?
- How does viewing vary by time of day?
- Which genres are trending?

**Your Tasks:**
1. Define the grain (what is one row in the fact table?)
2. List all dimensions needed
3. List all facts (measurements) needed
4. Design the fact table (write CREATE TABLE)
5. Design 3 dimension tables (write CREATE TABLE)
6. Write 3 SQL queries to answer business questions

**Hint**: You'll probably need these dimensions:
- Users (who)
- Movies (what)
- Date/Time (when)
- Maybe more?

**Measurements might include**:
- Watch duration (minutes)
- Did they finish the movie?
- Video quality
- Device type

## Summary

**Key Concepts:**
- Star schema = Fact table in center + Dimension tables around it
- Fact table = Measurements (revenue, quantity, etc.)
- Dimension tables = Context (who, what, when, where)
- Denormalize dimensions for easier queries
- Always define the grain clearly

**Remember:**
- Keep it simple
- Focus on answering business questions
- Denormalize for readability
- Pre-calculate useful attributes in dimensions

## Next Steps

Learn about [Fact and Dimension Tables](03-fact-dimension-tables.md) in more detail.

## Additional Resources

- [Star Schema Complete Guide](https://www.holistics.io/books/setup-analytics/dimensional-modeling/star-schema/)
- [Kimball Dimensional Modeling Techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)

---

**Next**: [Fact and Dimension Tables](03-fact-dimension-tables.md)
