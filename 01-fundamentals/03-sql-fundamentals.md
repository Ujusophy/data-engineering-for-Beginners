
# SQL Fundamentals

## Introduction

SQL (Structured Query Language) is the standard language for working with relational databases. As a data engineer, you'll use SQL daily to query, transform, and manage data.

## Why SQL for Data Engineering?

- **Universal**: Works across all relational databases
- **Declarative**: Describe what you want, not how to get it
- **Powerful**: Can handle complex data operations
- **Efficient**: Databases are optimized for SQL queries
- **Essential**: Required skill for data engineering roles

## Basic SQL Concepts

### Tables

Tables store data in rows and columns, like a spreadsheet.

```sql
-- Example: users table
| id | name    | age | city          | email                |
|----|---------|-----|---------------|----------------------|
| 1  | Alice   | 25  | New York      | alice@example.com    |
| 2  | Bob     | 30  | San Francisco | bob@example.com      |
| 3  | Charlie | 35  | Los Angeles   | charlie@example.com  |
```

### Schemas

A schema defines the structure of a table:
- Column names
- Data types (INTEGER, VARCHAR, DATE, etc.)
- Constraints (PRIMARY KEY, NOT NULL, etc.)

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INTEGER,
    city VARCHAR(100),
    email VARCHAR(255) UNIQUE
);
```

## SELECT Statement

The SELECT statement retrieves data from tables.

### Basic SELECT

```sql
-- Select all columns
SELECT * FROM users;

-- Select specific columns
SELECT name, email FROM users;

-- Select with aliases
SELECT 
    name AS user_name,
    email AS contact_email
FROM users;
```

### WHERE Clause

Filter rows based on conditions.

```sql
-- Simple condition
SELECT * FROM users
WHERE age > 25;

-- Multiple conditions (AND)
SELECT * FROM users
WHERE age > 25 AND city = 'New York';

-- Multiple conditions (OR)
SELECT * FROM users
WHERE city = 'New York' OR city = 'Los Angeles';

-- IN operator
SELECT * FROM users
WHERE city IN ('New York', 'Los Angeles', 'Chicago');

-- LIKE operator (pattern matching)
SELECT * FROM users
WHERE email LIKE '%@gmail.com';

-- BETWEEN operator
SELECT * FROM users
WHERE age BETWEEN 25 AND 35;

-- NULL checks
SELECT * FROM users
WHERE city IS NULL;

SELECT * FROM users
WHERE city IS NOT NULL;
```

### ORDER BY

Sort results.

```sql
-- Ascending order (default)
SELECT * FROM users
ORDER BY age;

-- Descending order
SELECT * FROM users
ORDER BY age DESC;

-- Multiple columns
SELECT * FROM users
ORDER BY city, age DESC;
```

### LIMIT

Limit the number of results.

```sql
-- Get first 10 rows
SELECT * FROM users
LIMIT 10;

-- Skip first 10, get next 10 (pagination)
SELECT * FROM users
LIMIT 10 OFFSET 10;
```

## Aggregate Functions

Perform calculations on groups of rows.

### Common Aggregates

```sql
-- Count rows
SELECT COUNT(*) FROM users;

-- Count non-null values
SELECT COUNT(city) FROM users;

-- Count distinct values
SELECT COUNT(DISTINCT city) FROM users;

-- Sum
SELECT SUM(age) FROM users;

-- Average
SELECT AVG(age) FROM users;

-- Min and Max
SELECT MIN(age), MAX(age) FROM users;
```

### GROUP BY

Group rows and apply aggregates.

```sql
-- Count users by city
SELECT city, COUNT(*) as user_count
FROM users
GROUP BY city;

-- Average age by city
SELECT city, AVG(age) as avg_age
FROM users
GROUP BY city;

-- Multiple aggregates
SELECT 
    city,
    COUNT(*) as user_count,
    AVG(age) as avg_age,
    MIN(age) as youngest,
    MAX(age) as oldest
FROM users
GROUP BY city;
```

### HAVING

Filter groups (like WHERE but for aggregates).

```sql
-- Cities with more than 5 users
SELECT city, COUNT(*) as user_count
FROM users
GROUP BY city
HAVING COUNT(*) > 5;

-- Cities where average age is over 30
SELECT city, AVG(age) as avg_age
FROM users
GROUP BY city
HAVING AVG(age) > 30;
```

## JOINS

Combine data from multiple tables.

### Sample Tables

```sql
-- orders table
| order_id | user_id | product   | amount |
|----------|---------|-----------|--------|
| 1        | 1       | Laptop    | 1200   |
| 2        | 1       | Mouse     | 25     |
| 3        | 2       | Keyboard  | 75     |
| 4        | 3       | Monitor   | 300    |
```

### INNER JOIN

Returns only matching rows from both tables.

```sql
SELECT 
    users.name,
    orders.product,
    orders.amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- Result: Only users with orders
```

### LEFT JOIN

Returns all rows from left table, matching rows from right table.

```sql
SELECT 
    users.name,
    orders.product,
    orders.amount
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Result: All users, even those without orders (NULL for order fields)
```

### RIGHT JOIN

Returns all rows from right table, matching rows from left table.

```sql
SELECT 
    users.name,
    orders.product,
    orders.amount
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;

-- Result: All orders, even if user is deleted
```

### Multiple Joins

```sql
-- Add products table
SELECT 
    users.name,
    orders.order_id,
    products.name as product_name,
    products.price
FROM users
INNER JOIN orders ON users.id = orders.user_id
INNER JOIN products ON orders.product_id = products.id;
```

## Subqueries

Queries within queries.

```sql
-- Users who spent more than average
SELECT name, email
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
    GROUP BY user_id
    HAVING SUM(amount) > (SELECT AVG(total) FROM (
        SELECT SUM(amount) as total
        FROM orders
        GROUP BY user_id
    ) as user_totals)
);

-- Cleaner with WITH clause (see CTEs below)
```

## Common Table Expressions (CTEs)

Named temporary result sets.

```sql
-- Basic CTE
WITH user_totals AS (
    SELECT 
        user_id,
        SUM(amount) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT 
    users.name,
    user_totals.total_spent
FROM users
JOIN user_totals ON users.id = user_totals.user_id;

-- Multiple CTEs
WITH 
user_totals AS (
    SELECT user_id, SUM(amount) as total
    FROM orders
    GROUP BY user_id
),
avg_spending AS (
    SELECT AVG(total) as avg_total
    FROM user_totals
)
SELECT 
    users.name,
    user_totals.total,
    avg_spending.avg_total
FROM users
JOIN user_totals ON users.id = user_totals.user_id
CROSS JOIN avg_spending;
```

## Window Functions

Perform calculations across rows related to the current row.

### ROW_NUMBER

```sql
-- Assign row numbers
SELECT 
    name,
    age,
    ROW_NUMBER() OVER (ORDER BY age DESC) as rank
FROM users;
```

### RANK and DENSE_RANK

```sql
-- Rank users by age (with gaps)
SELECT 
    name,
    age,
    RANK() OVER (ORDER BY age DESC) as rank
FROM users;

-- Dense rank (no gaps)
SELECT 
    name,
    age,
    DENSE_RANK() OVER (ORDER BY age DESC) as rank
FROM users;
```

### Partition By

```sql
-- Rank users within each city
SELECT 
    name,
    city,
    age,
    ROW_NUMBER() OVER (PARTITION BY city ORDER BY age DESC) as city_rank
FROM users;
```

### Running Totals

```sql
-- Running total of orders by user
SELECT 
    user_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id 
        ORDER BY order_date
    ) as running_total
FROM orders;
```

### LAG and LEAD

```sql
-- Compare with previous row
SELECT 
    order_date,
    amount,
    LAG(amount) OVER (ORDER BY order_date) as prev_amount,
    amount - LAG(amount) OVER (ORDER BY order_date) as difference
FROM orders;

-- Look ahead to next row
SELECT 
    order_date,
    amount,
    LEAD(amount) OVER (ORDER BY order_date) as next_amount
FROM orders;
```

## Data Manipulation

### INSERT

```sql
-- Insert single row
INSERT INTO users (name, age, city, email)
VALUES ('Frank', 27, 'Boston', 'frank@example.com');

-- Insert multiple rows
INSERT INTO users (name, age, city, email)
VALUES 
    ('Grace', 29, 'Seattle', 'grace@example.com'),
    ('Henry', 31, 'Portland', 'henry@example.com');

-- Insert from SELECT
INSERT INTO users_backup
SELECT * FROM users
WHERE created_date < '2023-01-01';
```

### UPDATE

```sql
-- Update single column
UPDATE users
SET age = 26
WHERE id = 1;

-- Update multiple columns
UPDATE users
SET 
    age = 26,
    city = 'Boston'
WHERE id = 1;

-- Update with calculation
UPDATE products
SET price = price * 1.1
WHERE category = 'Electronics';

-- Update from another table
UPDATE users
SET last_order_date = (
    SELECT MAX(order_date)
    FROM orders
    WHERE orders.user_id = users.id
);
```

### DELETE

```sql
-- Delete specific rows
DELETE FROM users
WHERE id = 1;

-- Delete based on condition
DELETE FROM orders
WHERE order_date < '2023-01-01';

-- Delete all rows (keep table structure)
DELETE FROM users;

-- Better for clearing table
TRUNCATE TABLE users;
```

## Data Definition

### CREATE TABLE

```sql
-- Basic table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2),
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table with foreign key
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    order_date DATE DEFAULT CURRENT_DATE
);

-- Table with constraints
CREATE TABLE inventory (
    product_id INTEGER PRIMARY KEY,
    quantity INTEGER NOT NULL CHECK (quantity >= 0),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ALTER TABLE

```sql
-- Add column
ALTER TABLE users
ADD COLUMN phone VARCHAR(20);

-- Modify column
ALTER TABLE users
ALTER COLUMN age TYPE INTEGER;

-- Add constraint
ALTER TABLE users
ADD CONSTRAINT unique_email UNIQUE (email);

-- Drop column
ALTER TABLE users
DROP COLUMN phone;
```

### DROP TABLE

```sql
-- Drop table
DROP TABLE users;

-- Drop if exists
DROP TABLE IF EXISTS users;

-- Drop with dependencies
DROP TABLE users CASCADE;
```

## Indexes

Indexes speed up queries but slow down writes.

```sql
-- Create index
CREATE INDEX idx_users_city ON users(city);

-- Create unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Composite index
CREATE INDEX idx_users_city_age ON users(city, age);

-- Drop index
DROP INDEX idx_users_city;
```

## Common Data Engineering Patterns

### Incremental Loading

```sql
-- Get records updated since last load
SELECT *
FROM source_table
WHERE updated_at > '2024-01-01 00:00:00';
```

### Deduplication

```sql
-- Find duplicates
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Remove duplicates (keep latest)
DELETE FROM users
WHERE id NOT IN (
    SELECT MAX(id)
    FROM users
    GROUP BY email
);
```

### Data Quality Checks

```sql
-- Check for NULL values
SELECT COUNT(*) as null_emails
FROM users
WHERE email IS NULL;

-- Check for invalid data
SELECT COUNT(*) as negative_prices
FROM products
WHERE price < 0;

-- Check referential integrity
SELECT COUNT(*) as orphaned_orders
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;
```

### Slowly Changing Dimensions (Type 2)

```sql
-- Track historical changes
CREATE TABLE users_history (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    name VARCHAR(100),
    city VARCHAR(100),
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,
    is_current BOOLEAN DEFAULT TRUE
);

-- Insert new version
INSERT INTO users_history (user_id, name, city, valid_from)
VALUES (1, 'Alice Smith', 'Boston', CURRENT_TIMESTAMP);

-- Mark old version as inactive
UPDATE users_history
SET 
    valid_to = CURRENT_TIMESTAMP,
    is_current = FALSE
WHERE user_id = 1 AND is_current = TRUE;
```

## Query Optimization Tips

### 1. Use EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE city = 'New York';

EXPLAIN ANALYZE SELECT * FROM users WHERE city = 'New York';
```

### 2. Avoid SELECT *

```sql
-- Bad
SELECT * FROM users;

-- Good
SELECT id, name, email FROM users;
```

### 3. Use Indexes Wisely

```sql
-- Index columns used in WHERE, JOIN, ORDER BY
CREATE INDEX idx_users_city ON users(city);
```

### 4. Limit Results Early

```sql
-- Bad
SELECT * FROM large_table ORDER BY date LIMIT 10;

-- Better (if date is indexed)
SELECT * FROM large_table 
WHERE date >= '2024-01-01'
ORDER BY date 
LIMIT 10;
```

### 5. Use EXISTS Instead of IN

```sql
-- Slower
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Faster
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

## PostgreSQL-Specific Features

### COPY Command

```sql
-- Import CSV
COPY users FROM '/path/to/users.csv' 
WITH (FORMAT csv, HEADER true);

-- Export to CSV
COPY (SELECT * FROM users WHERE city = 'New York')
TO '/path/to/ny_users.csv'
WITH (FORMAT csv, HEADER true);
```

### JSON Support

```sql
-- JSON column
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- Query JSON
SELECT data->>'name' as name
FROM events
WHERE data->>'type' = 'click';
```

### UPSERT (INSERT ON CONFLICT)

```sql
INSERT INTO users (id, name, email)
VALUES (1, 'Alice', 'alice@example.com')
ON CONFLICT (id) 
DO UPDATE SET 
    name = EXCLUDED.name,
    email = EXCLUDED.email;
```

## Practice Exercises

### Exercise 1: Basic Queries
Use this dataset [dataset/users.csv](../dataset/users.csv)
Write queries to:
1. Get all users from New York
2. Count users by city
3. Find the oldest user in each city
4. Get users who registered in the last 30 days

### Exercise 2: Joins
Use this dataset [dataset/orders.csv](../dataset/orders.csv)
1. List all users with their total order count
2. Find users who never placed an order
3. Get top 5 customers by total spending

### Exercise 3: Window Functions
Use this dataset [dataset/products.csv](../dataset/products.csv)
1. Rank products by price within each category
2. Calculate running total of daily sales
3. Find the difference between each order and the previous order

### Exercise 4: Data Quality
Write queries to:
1. Find duplicate email addresses
2. Identify orders with invalid user_ids
3. Check for products with NULL prices

## Summary

You now know:
- Basic SQL syntax (SELECT, WHERE, ORDER BY)
- Aggregate functions and GROUP BY
- Different types of JOINs
- Subqueries and CTEs
- Window functions
- Data manipulation (INSERT, UPDATE, DELETE)
- Table creation and modification
- Indexes and optimization
- Common data engineering patterns

## Next Steps

Learn about [Data Formats](04-data-formats.md).

## Additional Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [SQLZoo - Interactive SQL Tutorial](https://sqlzoo.net/)
- [Mode SQL Tutorial](https://mode.com/sql-tutorial/)
- [SQL Style Guide](https://www.sqlstyle.guide/)

---

**Next**: [Data Formats](04-data-formats.md)
