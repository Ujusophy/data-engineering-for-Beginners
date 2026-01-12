# Relational Databases Overview

## What is a Relational Database?

A relational database stores data in tables with rows and columns. Tables can be connected (related) to each other using keys. It's like having multiple spreadsheets that reference each other.

## Why Relational Databases?

**For Data Engineers:**
- Most common database type in production
- Excellent for structured data
- ACID guarantees (data integrity)
- Powerful query capabilities with SQL
- Battle-tested and reliable

## Core Concepts

### Tables

Tables store data in rows and columns.

```
users table:
+----+----------+------------------+
| id | name     | email            |
+----+----------+------------------+
| 1  | Alice    | alice@email.com  |
| 2  | Bob      | bob@email.com    |
| 3  | Charlie  | charlie@email.com|
+----+----------+------------------+
```

### Primary Keys

A unique identifier for each row.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,  -- Auto-incrementing unique ID
    name VARCHAR(100),
    email VARCHAR(255)
);
```

**Why it matters:** Every table should have a primary key so you can uniquely identify each record.

### Foreign Keys

Links to another table's primary key.

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),  -- Foreign key
    total DECIMAL(10, 2)
);
```

**Why it matters:** Foreign keys maintain relationships and data integrity between tables.

### Relationships

**One-to-Many**
Most common relationship type.

```
One user → Many orders

users:
1 | Alice

orders:
1 | 1 | 100.00  (Alice's order)
2 | 1 | 50.00   (Alice's another order)
3 | 2 | 75.00   (Bob's order)
```

**Many-to-Many**
Requires a junction table.

```
Students ↔ Courses

students:           enrollments:        courses:
1 | Alice          1 | 1 | 1          1 | Math
2 | Bob            2 | 1 | 2          2 | Science
                   3 | 2 | 1
```

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    PRIMARY KEY (student_id, course_id)
);
```

**One-to-One**
Less common, used for optional data.

```
One user → One profile (optional detailed info)
```

## ACID Properties

What makes relational databases reliable:

**A - Atomicity**
- Transactions succeed completely or fail completely
- No partial updates

**C - Consistency**
- Data follows all rules and constraints
- Foreign keys are valid

**I - Isolation**
- Multiple transactions don't interfere with each other
- Each transaction acts like it's alone

**D - Durability**
- Once committed, data persists even if system crashes
- Data is saved to disk

**Why it matters for data engineers:** You need ACID when data accuracy is critical (financial transactions, inventory, user accounts).

## Constraints

Rules that ensure data quality:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,              -- Cannot be empty
    price DECIMAL(10, 2) CHECK (price > 0),  -- Must be positive
    sku VARCHAR(50) UNIQUE,                  -- No duplicates
    category_id INTEGER REFERENCES categories(id)  -- Must exist
);
```

Common constraints:
- `NOT NULL`: Field must have a value
- `UNIQUE`: No duplicate values
- `CHECK`: Custom validation rule
- `FOREIGN KEY`: Must reference existing record
- `PRIMARY KEY`: Combines UNIQUE and NOT NULL

## Indexes

Speed up queries by creating a lookup structure.

```sql
-- Create index on frequently searched column
CREATE INDEX idx_users_email ON users(email);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

**When to use indexes:**
- Columns in WHERE clauses
- Columns in JOIN conditions
- Columns in ORDER BY

**Trade-off:** Indexes speed up reads but slow down writes.

## Transactions

Group multiple operations into one unit.

```sql
BEGIN;

-- Transfer money between accounts
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- If everything succeeded
COMMIT;

-- If there was an error
-- ROLLBACK;
```

**Why it matters:** Ensures all-or-nothing operations. Critical for data consistency.

## Normalization (Simplified)

Organizing data to reduce redundancy.

**Bad (Denormalized):**
```
orders table:
order_id | user_name | user_email      | product  | price
1        | Alice     | alice@email.com | Laptop   | 1000
2        | Alice     | alice@email.com | Mouse    | 25
3        | Bob       | bob@email.com   | Laptop   | 1000
```
Problems: Duplicate user data, inconsistent prices

**Good (Normalized):**
```
users table:
user_id | name  | email
1       | Alice | alice@email.com
2       | Bob   | bob@email.com

products table:
product_id | name   | price
1          | Laptop | 1000
2          | Mouse  | 25

orders table:
order_id | user_id | product_id | quantity
1        | 1       | 1          | 1
2        | 1       | 2          | 1
3        | 2       | 1          | 1
```

**Basic normalization rules:**
1. Each table has a primary key
2. No repeating groups (no comma-separated values)
3. Store data in one place only
4. Use foreign keys to connect tables

**Note:** Data warehouses sometimes denormalize for performance. We'll cover this in Module 3.

## Common Database Operations

### Creating a Database

```sql
CREATE DATABASE my_database;
```

### Connecting to a Database

```bash
# Using psql (PostgreSQL command line)
psql -U username -d database_name

# Using Python
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    database="my_database",
    user="username",
    password="password"
)
```

### Backing Up

```bash
# Backup
pg_dump database_name > backup.sql

# Restore
psql database_name < backup.sql
```

## Popular Relational Databases

### PostgreSQL (Our Choice)
- Open source
- Feature-rich
- Great for data engineering
- JSON support
- Strong data types

### MySQL/MariaDB
- Popular for web apps
- Good performance
- Large community

### SQL Server
- Microsoft ecosystem
- Enterprise features
- Windows-focused

### Oracle
- Enterprise-grade
- Expensive
- Legacy systems

**Why we use PostgreSQL:** Free, powerful, excellent for learning, widely used in industry.

## When to Use Relational Databases

**Good fit:**
- Structured data with clear relationships
- Need for data integrity (ACID)
- Complex queries and joins
- Financial transactions
- User management
- Inventory systems

**Not ideal for:**
- Unstructured data (logs, documents)
- Extremely high write volumes
- Data without relationships
- Frequently changing schema

## Basic Performance Tips

1. **Use indexes wisely**
   ```sql
   CREATE INDEX ON users(email);  -- For lookups
   ```

2. **Avoid SELECT ***
   ```sql
   -- Bad
   SELECT * FROM users;
   
   -- Good
   SELECT id, name, email FROM users;
   ```

3. **Use LIMIT for large results**
   ```sql
   SELECT * FROM users LIMIT 100;
   ```

4. **Add appropriate constraints**
   ```sql
   ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);
   ```

5. **Use connection pooling** (in production)
   ```python
   from psycopg2 import pool
   connection_pool = pool.SimpleConnectionPool(1, 20, ...)
   ```

## Common Pitfalls

**1. No indexes on foreign keys**
```sql
-- Bad: No index
CREATE TABLE orders (
    user_id INTEGER REFERENCES users(id)
);

-- Good: Add index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**2. Not using transactions**
```python
# Bad
cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")

# Good
conn.begin()
cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
conn.commit()
```

**3. Storing calculated values**
```sql
-- Bad: Storing total
CREATE TABLE orders (
    subtotal DECIMAL(10,2),
    tax DECIMAL(10,2),
    total DECIMAL(10,2)  -- This should be calculated
);

-- Good: Calculate on read
SELECT subtotal + tax as total FROM orders;
```

## Practice Exercise

Design a database schema for a **Library Management System**. Your schema should handle:

**Requirements:**
1. Track books (title, ISBN, author, publication year, available copies)
2. Track members (name, email, membership date)
3. Track book loans (which member borrowed which book, when, return date)
4. Track authors (books can have multiple authors)
5. Track book categories (books can belong to multiple categories)

**Dataset Provided:**
Use the sample data in the `datasets/` folder:
- `books.csv` - 15 programming books
- `authors.csv` - 18 authors
- `book_authors.csv` - Book-author relationships
- `members.csv` - 10 library members
- `loans.csv` - 15 loan records (some books still borrowed, some overdue)
- `categories.csv` - 8 book categories
- `book_categories.csv` - Book-category relationships

**Tasks:**
1. Identify the entities needed
2. Define the relationships between entities
3. Write CREATE TABLE statements with:
   - Appropriate primary keys
   - Foreign keys with proper references
   - Necessary constraints (NOT NULL, UNIQUE, CHECK)
   - Appropriate data types
4. Create indexes on foreign keys
5. Import the provided CSV data into your tables
6. Write queries to answer:
   - List all books borrowed by a specific member
   - Find all available books in a category
   - Show overdue books (return_date is NULL and due_date < today)
   - Which member has borrowed the most books?
   - Which authors have written multiple books?
   - What's the most popular category?

**Hints:**
- ISBN is the primary key for books
- Use DATE type for dates
- Empty return_date means the book is still borrowed
- Consider using ON DELETE CASCADE for loan records

**Bonus:**
- Add audit columns (created_at, updated_at)
- Implement a reservation system (members can reserve books)
- Track book reviews and ratings
- Calculate late fees for overdue books

## Summary

**Key Concepts:**
- Tables store data in rows and columns
- Primary keys uniquely identify rows
- Foreign keys create relationships
- ACID ensures data integrity
- Indexes speed up queries
- Transactions group operations
- Constraints enforce data quality

**For Data Engineers:**
- You'll query databases daily
- You'll design schemas for data pipelines
- You'll optimize queries for performance
- You'll handle migrations and transformations

## Next Steps

Learn about [PostgreSQL Essentials](02-postgresql-essentials.md) to start working with a real database.

## Additional Resources

- [Database Design for Mere Mortals](https://www.amazon.com/Database-Design-Mere-Mortals-Hands/dp/0321884493)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

**Next**: [PostgreSQL Essentials](02-postgresql-essentials.md)
