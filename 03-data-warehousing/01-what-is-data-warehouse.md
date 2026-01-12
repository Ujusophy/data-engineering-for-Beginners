# What is a Data Warehouse?

## The Simple Explanation

Imagine you run a pizza restaurant with an app for orders:

- **Database (Operational)**: Stores today's orders, customer info, pizza inventory
  - "Is this customer's order ready?"
  - "Do we have pepperoni in stock?"
  - Fast reads and writes for running the business

- **Data Warehouse (Analytical)**: Stores historical data for analysis
  - "What was our best-selling pizza last year?"
  - "Which customers order most frequently?"
  - "How do sales compare month-over-month?"
  - Optimized for reading and analyzing large amounts of data

**Key Difference**: Database = Run the business day-to-day. Data Warehouse = Understand the business over time.

## Why Do We Need Data Warehouses?

### Problem 1: Performance

```
Your database:
- Handling 1000s of customer orders per minute
- Someone runs a report: "Show me sales trends for the last 5 years"
- Report slows down the entire system
- Customers can't place orders while report runs
```

**Solution**: Copy data to a separate data warehouse for reporting.

### Problem 2: Data is Scattered

```
Your company has:
- Orders database
- Customer database  
- Inventory database
- Marketing database

Question: "Which marketing campaigns led to the most sales?"
Problem: Data is in different systems!
```

**Solution**: Data warehouse combines all data in one place.

### Problem 3: Historical Data

```
Your operational database:
- Keeps last 90 days of data
- Old data gets deleted to save space
- Can't answer: "How did we do 2 years ago?"
```

**Solution**: Data warehouse keeps all historical data.

## Real-World Example

### E-commerce Company

**Operational Databases (OLTP):**
```
orders_db:
- New order comes in → Insert into orders table
- User updates address → Update users table
- Inventory decreases → Update products table
- Optimized for: Fast writes, quick lookups

Current data: Last 90 days
```

**Data Warehouse (OLAP):**
```
warehouse_db:
- Sales data from orders_db (all time)
- Customer data from users_db (all time)
- Product data from inventory_db (all time)
- Website analytics from Google Analytics
- Marketing data from email system
- Optimized for: Complex analysis, reporting

Historical data: All time (years of data)
```

**Business Questions Answered:**
- "Show me monthly revenue for the last 3 years"
- "Which products are most profitable?"
- "What's our customer retention rate?"
- "Which marketing channels drive the most sales?"

## How Data Flows

```
Source Systems          ETL Process              Data Warehouse
┌──────────────┐       ┌──────────┐             ┌────────────────┐
│ Orders DB    │──────>│ Extract  │             │                │
│              │       │    ↓     │             │  Star Schema   │
│ Customers DB │──────>│Transform │────────────>│  - Dimensions  │
│              │       │    ↓     │             │  - Facts       │
│ Products DB  │──────>│  Load    │             │  - Aggregates  │
│              │       └──────────┘             │                │
│ Web Analytics│──────>                         └────────────────┘
└──────────────┘                                         ↓
                                                  ┌────────────┐
                                                  │ BI Tools   │
                                                  │ Dashboards │
                                                  │ Reports    │
                                                  └────────────┘
```

**ETL = Extract, Transform, Load** (we'll learn this in detail later)

## Key Characteristics

### Operational Database
- **Purpose**: Run the business
- **Users**: Customers, employees
- **Operations**: Insert, Update, Delete often
- **Data**: Current state
- **Speed**: Milliseconds
- **Structure**: Normalized (many tables, joins)

### Data Warehouse
- **Purpose**: Analyze the business
- **Users**: Analysts, managers, data scientists
- **Operations**: Mostly SELECT queries
- **Data**: Historical snapshots
- **Speed**: Seconds to minutes (complex queries)
- **Structure**: Denormalized (fewer tables, easy to query)

## Simple Comparison

**Question**: "What did customer #123 order today?"
- **Answer from**: Operational Database ✓
- Fast lookup, current data needed

**Question**: "What were our top 10 products last quarter?"
- **Answer from**: Data Warehouse ✓
- Need to analyze 3 months of data across all products

## Data Warehouse Architecture (Simplified)

### Basic Setup

```
┌─────────────────────────────────────────────────┐
│           Data Warehouse Database               │
│                                                 │
│  ┌──────────────┐    ┌────────────────────┐     │
│  │  Staging     │───>│  Warehouse Schema  │     │
│  │  (Raw Data)  │    │  (Star Schema)     │     │
│  └──────────────┘    └────────────────────┘     │
│                               │                 │
│                               ↓                 │
│                      ┌────────────────┐         │
│                      │  Data Marts    │         │
│                      │  (Departments) │         │
│                      └────────────────┘         │
└─────────────────────────────────────────────────┘
```

**Layers:**
1. **Staging**: Raw data as extracted from sources
2. **Warehouse**: Cleaned, transformed data in star schema
3. **Data Marts**: Department-specific views (optional)

### Simple Version (What We'll Build)

```
PostgreSQL Database = Our Data Warehouse

Tables:
- dim_customers (dimension)
- dim_products (dimension)
- dim_dates (dimension)
- fact_sales (fact table - the metrics)
```

## OLTP vs OLAP

### OLTP (Online Transaction Processing)
**Example**: Pizza ordering app

```sql
-- Insert new order
INSERT INTO orders (customer_id, pizza_id, quantity) 
VALUES (123, 5, 2);

-- Update order status
UPDATE orders SET status = 'delivered' WHERE id = 456;

-- Look up customer
SELECT * FROM customers WHERE id = 123;
```

- Many small, fast transactions
- Insert, Update, Delete frequently
- Current data
- Normalized schema

### OLAP (Online Analytical Processing)
**Example**: Business intelligence dashboard

```sql
-- Analyze sales trends
SELECT 
    EXTRACT(YEAR FROM order_date) as year,
    EXTRACT(MONTH FROM order_date) as month,
    SUM(total_amount) as revenue,
    COUNT(*) as order_count
FROM fact_sales
JOIN dim_dates ON fact_sales.date_id = dim_dates.date_id
GROUP BY year, month
ORDER BY year, month;
```

- Few large, complex queries
- Mostly SELECT statements
- Historical data
- Denormalized schema (star/snowflake)

## Benefits of Data Warehouses

### 1. Don't Slow Down Operations
```
✓ Reports run on warehouse
✓ Operational database stays fast
✓ Customers can place orders while reports run
```

### 2. All Data in One Place
```
✓ Combine data from multiple sources
✓ Answer cross-system questions
✓ Single source of truth
```

### 3. Keep History
```
✓ Store years of data
✓ Track changes over time
✓ Year-over-year comparisons
```

### 4. Optimized for Analytics
```
✓ Fast reporting queries
✓ Pre-aggregated data
✓ Easy-to-understand structure
```

### 5. Data Quality
```
✓ Clean data before loading
✓ Standardize formats
✓ Validate and fix errors
```

## Common Use Cases

### Retail
- Sales trends by store, region, product
- Inventory turnover analysis
- Customer purchasing patterns
- Seasonal demand forecasting

### E-commerce
- Conversion funnel analysis
- Customer lifetime value
- Product recommendation effectiveness
- Marketing campaign ROI

### Finance
- Transaction analysis
- Fraud detection
- Risk assessment
- Regulatory reporting

### Healthcare
- Patient outcome analysis
- Treatment effectiveness
- Resource utilization
- Cost analysis

## Types of Data Warehouses

### 1. Enterprise Data Warehouse (EDW)
- Central warehouse for entire company
- All departments
- Large scale

### 2. Data Mart
- Department-specific
- Subset of EDW
- Sales data mart, Marketing data mart

### 3. Operational Data Store (ODS)
- Near real-time
- Bridge between OLTP and warehouse
- Less common for beginners

**For this course**: We'll build a simple enterprise warehouse with star schema.

## Modern Cloud Data Warehouses

### Traditional (What we'll use)
- PostgreSQL
- MySQL
- SQL Server
- Run on your own servers

### Modern Cloud (Industry uses)
- Snowflake
- BigQuery (Google)
- Redshift (Amazon)
- Synapse (Microsoft Azure)

**Why cloud warehouses?**
- Scale automatically
- Pay for what you use
- Managed by provider
- Very fast for huge data

**Why we use PostgreSQL?**
- Free to learn
- Same concepts apply
- Easy to set up locally
- Good for understanding fundamentals

## What You DON'T Need to Know Yet

- Advanced optimization techniques
- Partitioning strategies (we'll cover basics)
- Complex ETL tools (we'll use Python)
- Real-time streaming (later module)
- Data lake architectures (advanced topic)

## Simple Mental Model

```
Think of it like a library:

Operational Database = Librarian's desk
- Check out books
- Return books
- Update records
- Fast, current transactions

Data Warehouse = Archive room
- Historical collection
- Research and analysis
- "How many books were checked out last year?"
- "Which genres are most popular?"
```

## Summary

**Key Takeaways:**
- Data warehouse = Separate database for analysis and reporting
- Don't slow down operational systems
- Combines data from multiple sources
- Keeps historical data
- Optimized for reading, not writing
- Uses different structure (star schema vs normalized)

**Remember:**
- Operational Database = Run the business (OLTP)
- Data Warehouse = Understand the business (OLAP)

## Practice Exercise

Answer these questions:

1. You have a restaurant app. Where should you:
   - Store today's orders? (Database or Warehouse?)
   - Analyze "Top 10 dishes of 2023"? (Database or Warehouse?)
   - Check if a table is available? (Database or Warehouse?)
   - Generate monthly revenue report? (Database or Warehouse?)

2. Your company has 3 databases: Orders, Customers, Inventory.
   - Where should you combine them for reporting?
   - Why not just query all 3 databases directly?

3. List 3 benefits of having a data warehouse.

4. Explain in your own words: Why do we need a separate database for analytics?

**Think about it, then check your understanding with the next lesson!**

## Next Steps

Learn about [Dimensional Modeling Basics](02-dimensional-modeling.md) to understand how to structure a data warehouse.

## Additional Resources

- [Data Warehouse Fundamentals for Beginners](https://www.holistics.io/books/setup-analytics/)
- [Kimball Group: What is a Data Warehouse?](https://www.kimballgroup.com/)
- [Simple Guide to Star Schema](https://www.vertabelo.com/blog/star-schema-data-warehouse/)

---

**Next**: [Dimensional Modeling Basics](02-dimensional-modeling.md)
