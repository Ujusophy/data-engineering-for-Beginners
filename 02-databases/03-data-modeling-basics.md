# Data Modeling Basics

## What is Data Modeling?

Data modeling is designing how data will be stored in a database. As a data engineer, you'll design schemas for applications, data pipelines, and analytics systems.

## Why Data Modeling Matters

- **Good model**: Fast queries, easy maintenance, data integrity
- **Bad model**: Slow performance, hard to update, data inconsistencies

## The Data Modeling Process

### Step 1: Understand Requirements

Ask questions:
- What data needs to be stored?
- How will it be queried?
- What are the relationships?
- What are the business rules?

**Example:** E-commerce system
- Store: Users, Products, Orders
- Queries: "Show user's orders", "Popular products"
- Rules: Users can have many orders, orders contain multiple products

### Step 2: Identify Entities

Entities are the "things" you need to store.

**Example entities:**
- Users
- Products
- Orders
- Categories

### Step 3: Define Attributes

What information do you need about each entity?

```
Users:
- id
- name
- email
- password_hash
- created_at

Products:
- id
- name
- description
- price
- stock_quantity
- category_id

Orders:
- id
- user_id
- order_date
- total_amount
- status
```

### Step 4: Determine Relationships

How do entities connect?

```
User ─── has many ───> Orders
Product ─── belongs to ───> Category
Order ─── contains many ───> Products (through OrderItems)
```

### Step 5: Create Schema

Turn your design into SQL.

## Common Patterns

### Pattern 1: One-to-Many

**Use case:** One user has many orders

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total DECIMAL(10, 2)
);

-- Add index for faster lookups
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**Query pattern:**
```sql
-- Get user with their orders
SELECT 
    u.name,
    o.id as order_id,
    o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id = 1;
```

### Pattern 2: Many-to-Many

**Use case:** Orders contain multiple products, products appear in multiple orders

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Junction table
CREATE TABLE order_items (
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price_at_time DECIMAL(10, 2) NOT NULL,  -- Store price when ordered
    PRIMARY KEY (order_id, product_id)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
```

**Query pattern:**
```sql
-- Get order with all items
SELECT 
    o.id as order_id,
    p.name as product_name,
    oi.quantity,
    oi.price_at_time
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.id = 1;
```

### Pattern 3: Self-Referencing

**Use case:** Employees have managers (who are also employees)

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),  -- Self-reference
    title VARCHAR(100)
);
```

**Query pattern:**
```sql
-- Get employee with their manager
SELECT 
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### Pattern 4: Hierarchical Data

**Use case:** Categories with subcategories

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories(id)
);

-- Example data
-- Electronics (1, NULL)
--   ├── Laptops (2, 1)
--   └── Phones (3, 1)
```

**Query pattern (get all descendants):**
```sql
-- Recursive query
WITH RECURSIVE category_tree AS (
    -- Base case: top-level category
    SELECT id, name, parent_id, 0 as level
    FROM categories
    WHERE id = 1
    
    UNION ALL
    
    -- Recursive case: children
    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

## Normalization Rules

### First Normal Form (1NF)
- Each cell contains single value (no arrays or lists)
- Each row is unique

**Bad:**
```
users table:
id | name  | emails
1  | Alice | alice@email.com, alice@work.com
```

**Good:**
```
users table:
id | name

user_emails table:
user_id | email
1       | alice@email.com
1       | alice@work.com
```

### Second Normal Form (2NF)
- Must be in 1NF
- No partial dependencies (all non-key columns depend on entire primary key)

**Bad:**
```
order_items table:
order_id | product_id | product_name | quantity
1        | 1          | Laptop       | 2
```
Problem: product_name depends only on product_id, not the full key

**Good:**
```
products table:
product_id | product_name

order_items table:
order_id | product_id | quantity
```

### Third Normal Form (3NF)
- Must be in 2NF
- No transitive dependencies (non-key columns don't depend on other non-key columns)

**Bad:**
```
employees table:
id | name  | department | dept_location
1  | Alice | Sales      | Building A
```
Problem: dept_location depends on department, not employee

**Good:**
```
departments table:
id | name  | location

employees table:
id | name  | department_id
```

**When to denormalize:** Data warehouses often denormalize for query performance. We'll cover this in Module 3.

## Naming Conventions

### Tables
```sql
-- Plural, lowercase, underscores
users
orders
order_items
product_categories
```

### Columns
```sql
-- Lowercase, underscores
user_id
first_name
created_at
is_active
```

### Keys
```sql
-- Primary keys: id or table_name_id
id
user_id

-- Foreign keys: referenced_table_id
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)  -- Clear foreign key name
);
```

### Indexes
```sql
-- idx_table_column or idx_table_column1_column2
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

## Data Types Best Practices

### IDs
```sql
-- Use SERIAL for auto-increment
id SERIAL PRIMARY KEY

-- Or BIGSERIAL for very large tables
id BIGSERIAL PRIMARY KEY

-- UUID for distributed systems
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

### Money
```sql
-- Always use DECIMAL, never FLOAT
price DECIMAL(10, 2)  -- 10 digits, 2 decimal places
```

### Timestamps
```sql
-- Use TIMESTAMPTZ (with timezone)
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
```

### Boolean Flags
```sql
-- Use BOOLEAN
is_active BOOLEAN DEFAULT TRUE
is_verified BOOLEAN DEFAULT FALSE
```

### Text
```sql
-- Short, limited text
name VARCHAR(100)
email VARCHAR(255)

-- Long text
description TEXT
content TEXT
```

## Common Schema Patterns

### Audit Columns

Add to most tables:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    -- ... other columns ...
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER REFERENCES users(id),
    updated_by INTEGER REFERENCES users(id)
);
```

### Soft Deletes

Don't actually delete records:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    deleted_at TIMESTAMPTZ,  -- NULL means active
    is_deleted BOOLEAN DEFAULT FALSE
);

-- Query only active records
SELECT * FROM users WHERE deleted_at IS NULL;

-- "Delete" a record
UPDATE users SET deleted_at = CURRENT_TIMESTAMP WHERE id = 1;
```

### Status Fields

Use ENUM or CHECK constraint:
```sql
-- Option 1: ENUM type
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status DEFAULT 'pending'
);

-- Option 2: CHECK constraint
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status VARCHAR(20) CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'))
);
```

### Version Control

Track changes to records:
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10, 2),
    version INTEGER DEFAULT 1
);

-- Increment version on update
UPDATE products 
SET price = 99.99, version = version + 1 
WHERE id = 1 AND version = 1;  -- Optimistic locking
```

## Real-World Example: Blog System

```sql
-- Users
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Posts
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    author_id INTEGER NOT NULL REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Comments
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id INTEGER NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    parent_comment_id INTEGER REFERENCES comments(id),  -- For nested comments
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Tags
CREATE TABLE tags (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL
);

-- Post-Tag relationship (many-to-many)
CREATE TABLE post_tags (
    post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
    tag_id INTEGER REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Indexes
CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_published ON posts(published_at);
CREATE INDEX idx_comments_post ON comments(post_id);
CREATE INDEX idx_comments_author ON comments(author_id);
CREATE INDEX idx_post_tags_post ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag ON post_tags(tag_id);
```

## Schema Design Checklist

- [ ] Every table has a primary key
- [ ] Foreign keys have indexes
- [ ] Use appropriate data types
- [ ] Add NOT NULL where appropriate
- [ ] Add UNIQUE constraints where needed
- [ ] Include created_at/updated_at timestamps
- [ ] Consider soft deletes vs hard deletes
- [ ] Add CHECK constraints for valid values
- [ ] Use meaningful table and column names
- [ ] Document complex relationships

## Common Mistakes

### 1. No Primary Key
```sql
-- Bad
CREATE TABLE logs (
    message TEXT,
    created_at TIMESTAMP
);

-- Good
CREATE TABLE logs (
    id BIGSERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Using VARCHAR without Limit
```sql
-- Bad (no practical limit)
name VARCHAR

-- Good (reasonable limit)
name VARCHAR(100)
```

### 3. Storing Derived Data
```sql
-- Bad
CREATE TABLE orders (
    subtotal DECIMAL(10, 2),
    tax DECIMAL(10, 2),
    total DECIMAL(10, 2)  -- This is calculated
);

-- Good (calculate on read)
SELECT 
    subtotal,
    tax,
    subtotal + tax as total
FROM orders;
```

### 4. No Indexes on Foreign Keys
```sql
-- Bad
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)
);  -- Missing index!

-- Good
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)
);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

## Summary

**Key Concepts:**
- Identify entities and relationships
- Normalize to reduce redundancy
- Use appropriate data types
- Add indexes on foreign keys
- Include audit columns
- Follow naming conventions
- Plan for common query patterns

**For Data Engineers:**
- You'll design schemas for data pipelines
- You'll optimize schemas for analytics
- You'll migrate and transform schemas
- Good design = better performance

## Next Steps

Learn about [NoSQL Overview](04-nosql-overview.md) to understand when to use non-relational databases.

## Additional Resources

- [Database Design for Mere Mortals](https://www.amazon.com/Database-Design-Mere-Mortals-Hands/dp/0321884493)
- [PostgreSQL Schema Design Best Practices](https://www.postgresql.org/docs/current/ddl.html)
- [Data Modeling Best Practices](https://www.sqlshack.com/database-design-best-practices/)

---

**Next**: [NoSQL Overview](04-nosql-overview.md)
