# Data Formats

## Introduction

Data comes in many different formats. As a data engineer, you need to understand these formats to efficiently store, process, and transfer data. Choosing the right format can significantly impact performance, storage costs, and processing speed.

## Common Data Formats

### 1. CSV (Comma-Separated Values)

**What is it?**
Plain text format where values are separated by commas (or other delimiters).

**Example:**
```csv
id,name,age,city
1,Alice,25,New York
2,Bob,30,San Francisco
3,Charlie,35,Los Angeles
```

**Pros:**
- Human-readable
- Universal support
- Simple to create and parse
- Works with Excel and spreadsheets
- Small file size for text data

**Cons:**
- No data type information
- No schema enforcement
- Inefficient for large datasets
- Issues with special characters and delimiters
- No support for nested data

**When to Use:**
- Small to medium datasets
- Data exchange between systems
- Quick data exploration
- Reports and exports

**Python Example:**
```python
import csv

# Reading CSV
with open('data.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['name'], row['age'])

# Writing CSV
with open('output.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()
    writer.writerow({'name': 'Alice', 'age': 25})

# Using Pandas
import pandas as pd
df = pd.read_csv('data.csv')
df.to_csv('output.csv', index=False)
```

### 2. JSON (JavaScript Object Notation)

**What is it?**
Text-based format for representing structured data as key-value pairs.

**Example:**
```json
{
  "users": [
    {
      "id": 1,
      "name": "Alice",
      "age": 25,
      "address": {
        "city": "New York",
        "state": "NY"
      }
    },
    {
      "id": 2,
      "name": "Bob",
      "age": 30,
      "address": {
        "city": "San Francisco",
        "state": "CA"
      }
    }
  ]
}
```

**Pros:**
- Human-readable
- Supports nested structures
- Native web support
- Flexible schema
- Widely used in APIs

**Cons:**
- Larger file size than binary formats
- Slower to parse than binary formats
- No compression by default
- Redundant field names in arrays

**When to Use:**
- API responses and requests
- Configuration files
- Semi-structured data
- Data with nested objects
- Web applications

**Python Example:**
```python
import json

# Reading JSON
with open('data.json', 'r') as f:
    data = json.load(f)
    print(data['users'][0]['name'])

# Writing JSON
data = {'name': 'Alice', 'age': 25}
with open('output.json', 'w') as f:
    json.dump(data, f, indent=2)

# Using Pandas
df = pd.read_json('data.json')
df.to_json('output.json', orient='records', indent=2)
```

### 3. Parquet

**What is it?**
Columnar storage format designed for big data processing. Developed by Apache.

**Key Features:**
- Columnar storage (stores data by column, not row)
- Built-in compression
- Schema stored in file
- Supports complex nested data types
- Efficient for analytics queries

**Pros:**
- Excellent compression (10-100x smaller than CSV)
- Fast query performance
- Supports predicate pushdown
- Schema evolution support
- Ideal for analytical workloads
- Read only columns you need

**Cons:**
- Not human-readable
- Requires specific tools to read
- Write-once format (not ideal for frequent updates)
- More complex than CSV/JSON

**When to Use:**
- Data lakes and warehouses
- Big data processing (Spark, Hive)
- Analytical workloads
- Long-term storage
- Large datasets (GB to TB)

**Python Example:**
```python
import pandas as pd
import pyarrow.parquet as pq

# Write Parquet
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [25, 30, 35]
})
df.to_parquet('data.parquet')

# Read Parquet
df = pd.read_parquet('data.parquet')

# Read specific columns (efficient!)
df = pd.read_parquet('data.parquet', columns=['name'])

# Using PyArrow
table = pq.read_table('data.parquet')
df = table.to_pandas()
```

### Performance Characteristics

| Format   | Read Speed | Write Speed | Random Access | Compression | Human Readable |
|----------|------------|-------------|---------------|-------------|----------------|
| CSV      | Slow       | Fast        | Poor          | None        | ✓              |
| JSON     | Slow       | Fast        | Good          | None        | ✓              |
| Parquet  | Fast       | Slow        | Excellent     | Excellent   | ✗              |

## Choosing the Right Format

### For Data Storage

**Analytics/Data Warehouse:**
- Use: **Parquet** 
- Why: Columnar format optimized for queries

**Archive/Backup:**
- Use: **Parquet**
- Why: Best compression, self-contained

### For Data Transfer

**APIs:**
- Use: **JSON**
- Why: Web-friendly, human-readable

**Data Exchange:**
- Use: **CSV** or **JSON**
- Why: Universal compatibility

### For Development

**Exploration/Testing:**
- Use: **CSV** or **JSON**
- Why: Easy to create and inspect

**Production Pipelines:**
- Use: **Parquet** 
- Why: Performance and efficiency

## Compression

Most formats support additional compression:

### Common Compression Algorithms

```python
import pandas as pd

# Parquet with compression
df.to_parquet('data.parquet', compression='snappy')  # Default
df.to_parquet('data.parquet', compression='gzip')    # Smaller, slower
df.to_parquet('data.parquet', compression='brotli')  # Best compression

# CSV with gzip
df.to_csv('data.csv.gz', compression='gzip')

# JSON with gzip
df.to_json('data.json.gz', compression='gzip')
```

### Compression Trade-offs

| Algorithm | Compression Ratio | Speed    | CPU Usage |
|-----------|-------------------|----------|-----------|
| Snappy    | Good (3-5x)       | Very Fast| Low       |
| GZIP      | Better (5-10x)    | Fast     | Medium    |
| ZSTD      | Better (5-10x)    | Fast     | Medium    |
| BROTLI    | Best (10-15x)     | Slow     | High      |
| LZ4       | Good (3-5x)       | Very Fast| Very Low  |

**General Rule:**
- Use **Snappy** for hot data (frequent access)
- Use **GZIP/ZSTD** for cold data (archival)
- Use **LZ4** for streaming (low latency)

## Schema Management

### Schema Evolution

How formats handle schema changes:

**Parquet:**
```python
# Original schema: name, age
df1 = pd.DataFrame({'name': ['Alice'], 'age': [25]})
df1.to_parquet('data.parquet')

# New schema: name, age, city (added column)
df2 = pd.DataFrame({
    'name': ['Bob'], 
    'age': [30], 
    'city': ['NYC']
})
df2.to_parquet('data.parquet', engine='fastparquet', append=True)

# Reading back - missing values filled with NULL
df = pd.read_parquet('data.parquet')
```

### Schema Definition

**Parquet Schema:**
```python
import pyarrow as pa

schema = pa.schema([
    ('name', pa.string()),
    ('age', pa.int64()),
    ('salary', pa.float64()),
    ('hire_date', pa.timestamp('ms'))
])

table = pa.table(data, schema=schema)
```

## Working with Large Files

### Chunked Reading

```python
# Read CSV in chunks
for chunk in pd.read_csv('large_file.csv', chunksize=10000):
    process(chunk)

# Read Parquet by row groups
import pyarrow.parquet as pq

parquet_file = pq.ParquetFile('large_file.parquet')
for batch in parquet_file.iter_batches(batch_size=10000):
    df = batch.to_pandas()
    process(df)
```

### Partitioning

```python
# Write partitioned Parquet
df.to_parquet(
    'output/',
    partition_cols=['year', 'month'],
    engine='pyarrow'
)

# Creates structure:
# output/
#   year=2024/
#     month=01/
#       part-0.parquet
#     month=02/
#       part-0.parquet

# Read specific partitions
df = pd.read_parquet('output/', filters=[('year', '=', 2024)])
```

## Best Practices

### 1. Choose Based on Use Case

```
Raw Data Ingestion → CSV/JSON
Intermediate Processing → Avro
Analytics Storage → Parquet
Long-term Archive → Parquet (compressed)
API Communication → JSON
High-Performance APIs → Protobuf
```

### 2. Use Compression

```python
# Always compress for storage
df.to_parquet('data.parquet', compression='snappy')

# Consider compression level for CSV
df.to_csv('data.csv.gz', compression='gzip')
```

### 3. Partition Large Datasets

```python
# Partition by frequently filtered columns
df.to_parquet('data/', partition_cols=['date', 'region'])
```

### 4. Include Metadata

```python
# Add metadata to Parquet
metadata = {'source': 'sales_db', 'version': '1.0'}
df.to_parquet('data.parquet', metadata=metadata)
```

### 5. Test Format Performance

```python
import time

# Test write performance
start = time.time()
df.to_csv('data.csv')
csv_time = time.time() - start

start = time.time()
df.to_parquet('data.parquet')
parquet_time = time.time() - start

print(f"CSV: {csv_time:.2f}s, Parquet: {parquet_time:.2f}s")
```

## Format Conversion

### Common Conversions

```python
# CSV to Parquet
df = pd.read_csv('data.csv')
df.to_parquet('data.parquet')

# JSON to Parquet
df = pd.read_json('data.json')
df.to_parquet('data.parquet')

# Parquet to CSV (for analysis)
df = pd.read_parquet('data.parquet')
df.to_csv('data.csv', index=False)

# Batch conversion
import glob

csv_files = glob.glob('data/*.csv')
for file in csv_files:
    df = pd.read_csv(file)
    output = file.replace('.csv', '.parquet')
    df.to_parquet(output)
```

## Summary

**Key Takeaways:**
- CSV: Simple, human-readable, best for small datasets
- JSON: Flexible, nested structures, APIs
- Parquet: Columnar, compressed, analytics
- Choose format based on: size, access patterns, performance needs

**Decision Tree:**
```
Need human-readable? → CSV/JSON
Need best compression? → Parquet
Need web compatibility? → JSON
Need analytics performance? → Parquet
Need legacy compatibility? → CSV
```

## Practice Exercises
You can use any of the [csv files](../dataset/) and [OpenWeatherMap API](https://openweathermap.org/)
1. Convert a CSV file to Parquet and compare file sizes
2. Read a JSON API response and save as Parquet
3. Implement a format converter that handles CSV, JSON, and Parquet
4. Test read/write performance of different formats
5. Create a partitioned Parquet dataset

## Additional Resources

- [Apache Parquet Documentation](https://parquet.apache.org/)
- [Pandas I/O Tools](https://pandas.pydata.org/docs/user_guide/io.html)

---

**Next**: [Module 2: Databases](../02-databases/)
