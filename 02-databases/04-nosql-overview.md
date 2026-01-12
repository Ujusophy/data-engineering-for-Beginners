# NoSQL Overview

## What is NoSQL?

NoSQL (Not Only SQL) databases are alternatives to traditional relational databases. They trade some relational features for flexibility, scalability, and performance in specific use cases.

**Important:** As a data engineer, you'll mostly work with relational databases, but you need to understand NoSQL for:
- Storing unstructured data
- High-volume write scenarios
- Caching layers
- Real-time applications

## When to Use NoSQL vs SQL

### Use Relational (SQL) when:
- Data has clear relationships
- Need ACID transactions
- Complex queries with joins
- Financial data, user accounts
- **This is your default choice**

### Use NoSQL when:
- Unstructured or semi-structured data
- Extremely high write volumes
- Need horizontal scaling
- Simple key-value lookups
- Flexible/changing schema

## Types of NoSQL Databases

### 1. Document Databases (MongoDB)

**What:** Store data as JSON-like documents

**Example:**
```javascript
// MongoDB document
{
  "_id": "507f1f77bcf86cd799439011",
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "city": "New York",
    "state": "NY"
  },
  "orders": [
    { "id": 1, "total": 100 },
    { "id": 2, "total": 50 }
  ]
}
```

**Good for:**
- Content management systems
- User profiles
- Product catalogs
- Event logging

**Python example:**
```python
from pymongo import MongoClient

# Connect
client = MongoClient('mongodb://localhost:27017/')
db = client['my_database']
collection = db['users']

# Insert
collection.insert_one({
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': 25
})

# Query
user = collection.find_one({'email': 'alice@example.com'})

# Update
collection.update_one(
    {'email': 'alice@example.com'},
    {'$set': {'age': 26}}
)
```

**Pros:**
- Flexible schema
- Easy to scale horizontally
- Natural for nested data

**Cons:**
- No joins (must denormalize)
- Eventual consistency
- Harder to ensure data integrity

### 2. Key-Value Stores (Redis)

**What:** Simple key-value pairs, very fast

**Example:**
```
Key: "user:1:name"     Value: "Alice"
Key: "user:1:email"    Value: "alice@example.com"
Key: "session:abc123"  Value: {"user_id": 1, "expires": "2024-01-15"}
```

**Good for:**
- Caching
- Session storage
- Rate limiting
- Real-time analytics

**Python example:**
```python
import redis

# Connect
r = redis.Redis(host='localhost', port=6379, db=0)

# Set values
r.set('user:1:name', 'Alice')
r.setex('session:abc123', 3600, 'user_data')  # Expires in 1 hour

# Get values
name = r.get('user:1:name')

# Increment counter
r.incr('page:views')

# Lists
r.lpush('recent_orders', 'order_123')
recent = r.lrange('recent_orders', 0, 9)  # Get last 10
```

**Pros:**
- Extremely fast
- Simple to use
- Built-in expiration

**Cons:**
- Very limited querying
- No complex data types
- Everything in memory (can be expensive)

### 3. Column-Family Stores (Cassandra)

**What:** Store data in column families, optimized for write-heavy workloads

**Good for:**
- Time-series data
- IoT sensor data
- Event logging at scale
- High-volume writes

**When data engineers use it:**
- Storing billions of events
- Real-time analytics
- When PostgreSQL is too slow for writes

**Note:** We won't cover Cassandra in detail as it's more advanced. Just know it exists for massive scale.

### 4. Graph Databases (Neo4j)

**What:** Store data as nodes and relationships

**Example:**
```
(Alice)-[:FRIENDS_WITH]->(Bob)
(Alice)-[:BOUGHT]->(Laptop)
(Bob)-[:BOUGHT]->(Laptop)
```

**Good for:**
- Social networks
- Recommendation engines
- Fraud detection
- Network analysis

**When data engineers use it:**
- Complex relationship queries
- Recommendation systems
- When joins in SQL become too complex

**Note:** Specialized use case. Only learn if your project needs it.

## Practical Comparison

### Storing User Data

**PostgreSQL (Relational):**
```sql
-- Normalized tables
users table: id, name, email
addresses table: id, user_id, city, state

-- Query with join
SELECT u.name, a.city 
FROM users u 
JOIN addresses a ON u.id = a.user_id;
```

**MongoDB (Document):**
```javascript
// Embedded document
{
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "city": "New York",
    "state": "NY"
  }
}

// Query (no join needed)
db.users.find({"address.city": "New York"})
```

**Redis (Key-Value):**
```
# Separate keys
user:1:name = "Alice"
user:1:email = "alice@example.com"
user:1:city = "New York"

# Or JSON value
user:1 = {"name": "Alice", "email": "...", "city": "..."}
```

## Common Use Cases for Data Engineers

### 1. Caching with Redis

```python
import redis
import psycopg2
import json

r = redis.Redis()

def get_user(user_id):
    # Check cache first
    cached = r.get(f'user:{user_id}')
    if cached:
        return json.loads(cached)
    
    # Not in cache, query database
    conn = psycopg2.connect(...)
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cur.fetchone()
    
    # Store in cache for 1 hour
    r.setex(f'user:{user_id}', 3600, json.dumps(user))
    
    return user
```

### 2. Storing Logs in MongoDB

```python
from pymongo import MongoClient
import datetime

client = MongoClient()
db = client['app_logs']
logs = db['events']

# Log event
logs.insert_one({
    'timestamp': datetime.datetime.utcnow(),
    'level': 'INFO',
    'message': 'User logged in',
    'user_id': 123,
    'ip': '192.168.1.1',
    'metadata': {
        'browser': 'Chrome',
        'os': 'Windows'
    }
})

# Query logs
recent_errors = logs.find({
    'level': 'ERROR',
    'timestamp': {'$gte': datetime.datetime.utcnow() - datetime.timedelta(hours=1)}
})
```

### 3. Session Storage with Redis

```python
import redis
import json
import uuid

r = redis.Redis()

def create_session(user_id):
    session_id = str(uuid.uuid4())
    session_data = {
        'user_id': user_id,
        'created': datetime.datetime.utcnow().isoformat()
    }
    
    # Store session for 24 hours
    r.setex(f'session:{session_id}', 86400, json.dumps(session_data))
    return session_id

def get_session(session_id):
    data = r.get(f'session:{session_id}')
    return json.loads(data) if data else None
```

## Polyglot Persistence

Modern applications often use multiple database types:

```
Web Application:
├── PostgreSQL - User accounts, orders, products
├── Redis - Session storage, caching
├── MongoDB - User activity logs, analytics events
└── Elasticsearch - Full-text search
```

**As a data engineer, you might:**
- Extract data from PostgreSQL (source of truth)
- Cache frequently accessed data in Redis
- Store raw events in MongoDB
- Load aggregated data back to PostgreSQL for reporting

## Quick Decision Tree

```
Need strong consistency & transactions? → PostgreSQL
Need caching / session storage? → Redis
Need flexible schema for logs/events? → MongoDB
Need graph relationships? → Neo4j
Need massive write throughput? → Cassandra
Need full-text search? → Elasticsearch

When in doubt → Start with PostgreSQL
```

## NoSQL in Data Pipelines

### Extract from NoSQL

```python
# Extract MongoDB data to Parquet
from pymongo import MongoClient
import pandas as pd

client = MongoClient()
collection = client['app']['events']

# Get all documents
documents = list(collection.find())

# Convert to DataFrame
df = pd.DataFrame(documents)

# Save as Parquet
df.to_parquet('events.parquet')
```

### Load to NoSQL

```python
# Load CSV to MongoDB
import pandas as pd
from pymongo import MongoClient

df = pd.read_csv('users.csv')

client = MongoClient()
collection = client['app']['users']

# Convert DataFrame to list of dicts
records = df.to_dict('records')

# Insert into MongoDB
collection.insert_many(records)
```

## Best Practices

1. **Don't abandon SQL lightly**
   - PostgreSQL is very capable
   - NoSQL adds complexity
   - Start with SQL unless you have a specific reason

2. **Use Redis for caching**
   - Speeds up read-heavy applications
   - Easy to add to existing systems

3. **MongoDB for logs and events**
   - Flexible schema
   - Easy to dump JSON data
   - Good for high-volume writes

4. **Keep source of truth in SQL**
   - Use NoSQL for derived data
   - SQL databases have better guarantees

5. **Learn PostgreSQL deeply first**
   - More important for data engineers
   - NoSQL is secondary

## What You DON'T Need to Learn (Yet)

- Cassandra internals
- Neo4j query language
- Elasticsearch architecture
- Distributed systems theory

**Learn these only when your project specifically needs them.**

## Summary

**Key Takeaways:**
- NoSQL = trade-offs for specific use cases
- Document DB (MongoDB): Flexible schema, nested data
- Key-Value (Redis): Fast caching, sessions
- Use PostgreSQL by default
- Add NoSQL when you have a specific need

**For Data Engineers:**
- You'll extract data from various NoSQL sources
- You'll use Redis for caching pipelines
- You'll store logs/events in MongoDB
- But PostgreSQL is still your main tool

## Additional Resources

- [MongoDB University (Free)](https://university.mongodb.com/)
- [Redis Documentation](https://redis.io/documentation)
- [When to Use NoSQL](https://www.mongodb.com/nosql-explained/when-to-use-nosql)
