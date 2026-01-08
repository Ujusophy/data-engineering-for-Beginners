# PostgreSQL Essentials

## Getting Started with PostgreSQL

PostgreSQL (often called "Postgres") is a powerful, open-source relational database. This guide covers what data engineers need to know.

## Installation

### macOS
```bash
# Using Homebrew
brew install postgresql@15
brew services start postgresql@15
```

### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
```

### Windows
Download installer from [postgresql.org](https://www.postgresql.org/download/windows/)

### Verify Installation
```bash
psql --version
# Should show: psql (PostgreSQL) 15.x
```

## Basic PostgreSQL Tools

### psql (Command Line)

Connect to PostgreSQL:
```bash
# Connect to default database
psql postgres

# Connect to specific database
psql -d my_database

# Connect with username
psql -U username -d database_name
```

Useful psql commands:
```sql
\l              -- List databases
\c database     -- Connect to database
\dt             -- List tables
\d table_name   -- Describe table
\du             -- List users
\q              -- Quit
```

### pgAdmin (GUI Tool)

Free graphical interface for PostgreSQL. Good for beginners.
- Download: [pgadmin.org](https://www.pgadmin.org/)
- Visual database management
- Query editor
- Easy table browsing

### DBeaver (Universal Client)

Works with many database types.
- Download: [dbeaver.io](https://dbeaver.io/)
- Free community edition
- Good for data engineers working with multiple databases

## Creating Your First Database

```sql
-- Create database
CREATE DATABASE my_first_db;

-- Connect to it
\c my_first_db

-- Create a simple table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert data
INSERT INTO users (name, email) 
VALUES ('Alice', 'alice@example.com');

-- Query data
SELECT * FROM users;
```

## PostgreSQL Data Types

### Numeric Types
```sql
-- Integers
SMALLINT        -- -32,768 to 32,767
INTEGER or INT  -- -2 billion to 2 billion
BIGINT          -- Very large numbers
SERIAL          -- Auto-incrementing integer

-- Decimals
DECIMAL(10, 2)  -- Exact: 10 digits, 2 decimal places
NUMERIC(10, 2)  -- Same as DECIMAL
REAL            -- Approximate floating point
DOUBLE PRECISION -- More precise floating point
```

**Usage:**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    price DECIMAL(10, 2),  -- Use for money
    weight REAL,           -- Use for measurements
    quantity INTEGER       -- Use for counts
);
```

### String Types
```sql
CHAR(n)         -- Fixed length (padded with spaces)
VARCHAR(n)      -- Variable length, max n characters
TEXT            -- Unlimited length
```

**Usage:**
```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),     -- Short text with limit
    content TEXT,           -- Long text, no limit
    code CHAR(10)          -- Fixed-length codes
);
```

### Date/Time Types
```sql
DATE            -- Date only (2024-01-15)
TIME            -- Time only (14:30:00)
TIMESTAMP       -- Date and time
TIMESTAMPTZ     -- Timestamp with timezone (recommended)
INTERVAL        -- Time duration
```

**Usage:**
```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_date DATE,
    event_time TIME,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Query examples
SELECT * FROM events WHERE event_date >= CURRENT_DATE - INTERVAL '7 days';
SELECT * FROM events WHERE created_at > NOW() - INTERVAL '1 hour';
```

### Boolean
```sql
BOOLEAN or BOOL  -- TRUE, FALSE, or NULL
```

**Usage:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    email_verified BOOLEAN DEFAULT FALSE
);
```

### JSON Types
```sql
JSON     -- Stores JSON text
JSONB    -- Binary JSON (faster, recommended)
```

**Usage:**
```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_data JSONB
);

-- Insert JSON
INSERT INTO events (event_data) VALUES 
('{"user_id": 123, "action": "login", "ip": "192.168.1.1"}');

-- Query JSON
SELECT event_data->>'action' as action FROM events;
SELECT * FROM events WHERE event_data->>'user_id' = '123';
```

### Arrays
```sql
INTEGER[]       -- Array of integers
TEXT[]          -- Array of text
```

**Usage:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[]
);

INSERT INTO users (name, tags) VALUES 
('Alice', ARRAY['developer', 'python', 'postgres']);

SELECT * FROM users WHERE 'python' = ANY(tags);
```

## Working with Python

### Installing psycopg2

```bash
pip install psycopg2-binary
```

### Basic Connection

```python
import psycopg2

# Connect to database
conn = psycopg2.connect(
    host="localhost",
    database="my_database",
    user="postgres",
    password="your_password",
    port=5432
)

# Create cursor
cur = conn.cursor()

# Execute query
cur.execute("SELECT * FROM users")

# Fetch results
rows = cur.fetchall()
for row in rows:
    print(row)

# Close connection
cur.close()
conn.close()
```

### Using Context Managers

```python
import psycopg2

# Better: Auto-closes connection
with psycopg2.connect(
    host="localhost",
    database="my_database",
    user="postgres",
    password="your_password"
) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        results = cur.fetchall()
        print(results)
```

### Insert Data

```python
# Single insert
cur.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("Alice", "alice@example.com")
)
conn.commit()

# Multiple inserts
users = [
    ("Bob", "bob@example.com"),
    ("Charlie", "charlie@example.com")
]
cur.executemany(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    users
)
conn.commit()
```

### Query with Parameters

```python
# NEVER do this (SQL injection risk):
email = "user@example.com"
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")  # DANGEROUS!

# ALWAYS do this (safe):
cur.execute(
    "SELECT * FROM users WHERE email = %s",
    (email,)  # Note the comma for single parameter
)
```

### Fetch Results

```python
# Fetch all rows
cur.execute("SELECT * FROM users")
all_users = cur.fetchall()  # List of tuples

# Fetch one row
cur.execute("SELECT * FROM users WHERE id = %s", (1,))
user = cur.fetchone()  # Single tuple

# Fetch many rows
cur.execute("SELECT * FROM users")
batch = cur.fetchmany(10)  # First 10 rows

# Using DictCursor for named columns
from psycopg2.extras import DictCursor

with conn.cursor(cursor_factory=DictCursor) as cur:
    cur.execute("SELECT * FROM users")
    for row in cur:
        print(row['name'], row['email'])  # Access by column name
```

### Using Pandas

```python
import pandas as pd
import psycopg2

conn = psycopg2.connect(...)

# Read query into DataFrame
df = pd.read_sql_query("SELECT * FROM users", conn)

# Write DataFrame to table
df.to_sql('users_backup', conn, if_exists='replace', index=False)

conn.close()
```

## User Management

### Create User

```sql
-- Create user
CREATE USER data_engineer WITH PASSWORD 'secure_password';

-- Grant permissions
GRANT CONNECT ON DATABASE my_database TO data_engineer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO data_engineer;

-- Grant on future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO data_engineer;
```

### Common Permission Patterns

```sql
-- Read-only user
CREATE USER analyst WITH PASSWORD 'password';
GRANT CONNECT ON DATABASE my_database TO analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;

-- Application user (full access)
CREATE USER app_user WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE my_database TO app_user;

-- Revoke permissions
REVOKE INSERT, UPDATE, DELETE ON users FROM analyst;
```

## Database Maintenance

### Backup Database

```bash
# Full backup
pg_dump my_database > backup.sql

# Compressed backup
pg_dump my_database | gzip > backup.sql.gz

# Backup specific tables
pg_dump my_database -t users -t orders > tables_backup.sql
```

### Restore Database

```bash
# Create database first
createdb my_database_restored

# Restore
psql my_database_restored < backup.sql

# Or from compressed
gunzip -c backup.sql.gz | psql my_database_restored
```

### Analyze Query Performance

```sql
-- See query execution plan
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';

-- With actual execution stats
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';
```

### Vacuum and Analyze

```sql
-- Clean up and update statistics
VACUUM ANALYZE users;

-- Full vacuum (more thorough, slower)
VACUUM FULL users;

-- Analyze only (update statistics)
ANALYZE users;
```

## Common Patterns for Data Engineers

### Loading CSV Data

```sql
-- Import CSV file
COPY users(name, email, age)
FROM '/path/to/users.csv'
DELIMITER ','
CSV HEADER;

-- Export to CSV
COPY (SELECT * FROM users WHERE age > 25)
TO '/path/to/export.csv'
DELIMITER ','
CSV HEADER;
```

### Batch Inserts

```python
import psycopg2
from psycopg2.extras import execute_values

conn = psycopg2.connect(...)
cur = conn.cursor()

# Fast batch insert
data = [
    ("Alice", "alice@example.com"),
    ("Bob", "bob@example.com"),
    # ... thousands of rows
]

execute_values(
    cur,
    "INSERT INTO users (name, email) VALUES %s",
    data
)
conn.commit()
```

### Upsert (Insert or Update)

```sql
-- Insert or update if exists
INSERT INTO users (id, name, email) 
VALUES (1, 'Alice', 'alice@example.com')
ON CONFLICT (id) 
DO UPDATE SET 
    name = EXCLUDED.name,
    email = EXCLUDED.email,
    updated_at = CURRENT_TIMESTAMP;
```

### Creating Materialized Views

```sql
-- Create materialized view (pre-computed results)
CREATE MATERIALIZED VIEW daily_sales AS
SELECT 
    DATE(order_date) as date,
    COUNT(*) as order_count,
    SUM(total) as total_sales
FROM orders
GROUP BY DATE(order_date);

-- Refresh when data changes
REFRESH MATERIALIZED VIEW daily_sales;
```

## Connection Configuration

### Environment Variables

```python
import os
import psycopg2

conn = psycopg2.connect(
    host=os.getenv('DB_HOST', 'localhost'),
    database=os.getenv('DB_NAME', 'my_database'),
    user=os.getenv('DB_USER', 'postgres'),
    password=os.getenv('DB_PASSWORD')
)
```

### Configuration File

```python
# config.py
DATABASE_CONFIG = {
    'host': 'localhost',
    'database': 'my_database',
    'user': 'postgres',
    'password': 'password',
    'port': 5432
}

# main.py
from config import DATABASE_CONFIG
import psycopg2

conn = psycopg2.connect(**DATABASE_CONFIG)
```

## Troubleshooting

### Can't Connect
```bash
# Check if PostgreSQL is running
# macOS
brew services list

# Linux
sudo systemctl status postgresql

# Check if port is listening
netstat -an | grep 5432
```

### Permission Denied
```sql
-- Check current user
SELECT current_user;

-- Check permissions
\du

-- Grant necessary permissions
GRANT ALL PRIVILEGES ON DATABASE my_database TO username;
```

### Slow Queries
```sql
-- Check for missing indexes
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';

-- Add index if needed
CREATE INDEX idx_users_email ON users(email);

-- Check table statistics
SELECT * FROM pg_stat_user_tables WHERE relname = 'users';
```

## Best Practices

1. **Always use parameterized queries**
   ```python
   # Good
   cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
   ```

2. **Close connections**
   ```python
   # Use context managers or always close
   conn.close()
   ```

3. **Use transactions for multiple operations**
   ```python
   try:
       cur.execute("INSERT INTO ...")
       cur.execute("UPDATE ...")
       conn.commit()
   except:
       conn.rollback()
       raise
   ```

4. **Create indexes on foreign keys**
   ```sql
   CREATE INDEX idx_orders_user_id ON orders(user_id);
   ```

5. **Use appropriate data types**
   ```sql
   -- Use DECIMAL for money, not FLOAT
   price DECIMAL(10, 2)
   ```

## Summary

You now know:
- How to install and connect to PostgreSQL
- Essential PostgreSQL data types
- Working with PostgreSQL from Python
- User management basics
- Database backup and restore
- Common data engineering patterns
- Performance optimization basics

## Next Steps

Learn about [Data Modeling Basics](03-data-modeling-basics.md) to design better database schemas.

## Additional Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/)

---

**Next**: [Data Modeling Basics](03-data-modeling-basics.md)
