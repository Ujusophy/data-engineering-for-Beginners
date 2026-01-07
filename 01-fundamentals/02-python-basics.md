# Python for Data Engineering

## Why Python?

Python is the most popular language for data engineering because it's:
- **Easy to learn**: Clear, readable syntax
- **Powerful**: Rich ecosystem of data libraries
- **Versatile**: Works for scripts, APIs, and applications
- **Well-supported**: Large community and documentation

## Python Basics Review

### Variables and Data Types

```python
# Numbers
age = 25
price = 19.99

# Strings
name = "Alice"
email = 'alice@example.com'

# Booleans
is_active = True
has_error = False

# None (null value)
result = None
```

### Lists

```python
# Creating lists
numbers = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Charlie"]
mixed = [1, "hello", True, 3.14]

# Accessing elements
first = numbers[0]  # 1
last = numbers[-1]  # 5

# Adding elements
numbers.append(6)
numbers.extend([7, 8, 9])

# List operations
length = len(numbers)
total = sum(numbers)
```

### Dictionaries

```python
# Creating dictionaries
user = {
    "name": "Alice",
    "age": 25,
    "email": "alice@example.com"
}

# Accessing values
name = user["name"]
age = user.get("age")  # Safer - returns None if key doesn't exist

# Adding/updating
user["city"] = "New York"
user["age"] = 26

# Iterating
for key, value in user.items():
    print(f"{key}: {value}")
```

### Control Flow

```python
# If statements
if age >= 18:
    print("Adult")
elif age >= 13:
    print("Teenager")
else:
    print("Child")

# For loops
for i in range(5):
    print(i)

for name in names:
    print(f"Hello, {name}!")

# While loops
count = 0
while count < 5:
    print(count)
    count += 1
```

### Functions

```python
# Basic function
def greet(name):
    return f"Hello, {name}!"

# Function with default parameter
def greet_with_title(name, title="Mr."):
    return f"Hello, {title} {name}!"

# Multiple return values
def get_user_info():
    return "Alice", 25, "alice@example.com"

name, age, email = get_user_info()
```

## Essential Libraries for Data Engineering

### 1. Pandas - Data Manipulation

```python
import pandas as pd

# Reading data
df = pd.read_csv('data.csv')
df = pd.read_json('data.json')
df = pd.read_parquet('data.parquet')

# Basic operations
print(df.head())  # First 5 rows
print(df.info())  # Column info
print(df.describe())  # Statistics

# Selecting data
names = df['name']
subset = df[['name', 'age']]
filtered = df[df['age'] > 25]

# Transforming data
df['age_group'] = df['age'].apply(lambda x: 'adult' if x >= 18 else 'minor')
grouped = df.groupby('city').agg({'age': 'mean'})

# Writing data
df.to_csv('output.csv', index=False)
df.to_json('output.json', orient='records')
```

### 2. Working with Files

```python
# Reading files
with open('data.txt', 'r') as f:
    content = f.read()
    # File automatically closes

# Writing files
with open('output.txt', 'w') as f:
    f.write("Hello, world!\n")

# Reading CSV manually
import csv

with open('data.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['name'], row['age'])

# Writing CSV
with open('output.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()
    writer.writerow({'name': 'Alice', 'age': 25})
```

### 3. JSON Handling

```python
import json

# Reading JSON
with open('data.json', 'r') as f:
    data = json.load(f)

# Writing JSON
data = {'name': 'Alice', 'age': 25}
with open('output.json', 'w') as f:
    json.dump(data, f, indent=2)

# JSON string conversion
json_string = json.dumps(data)
parsed_data = json.loads(json_string)
```

### 4. Working with Dates

```python
from datetime import datetime, timedelta

# Current time
now = datetime.now()
today = datetime.today()

# Parsing dates
date = datetime.strptime('2024-01-15', '%Y-%m-%d')

# Formatting dates
formatted = date.strftime('%Y-%m-%d %H:%M:%S')

# Date arithmetic
tomorrow = today + timedelta(days=1)
week_ago = today - timedelta(weeks=1)
```

## Error Handling

```python
# Basic try-except
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero!")

# Multiple exceptions
try:
    file = open('data.txt', 'r')
    data = json.load(file)
except FileNotFoundError:
    print("File not found")
except json.JSONDecodeError:
    print("Invalid JSON")
finally:
    file.close()  # Always runs

# Re-raising exceptions
try:
    # Some operation
    pass
except Exception as e:
    print(f"Error occurred: {e}")
    raise  # Re-raise the exception
```

## Logging

```python
import logging

# Basic logging
logging.basicConfig(level=logging.INFO)
logging.info("Processing started")
logging.warning("This is a warning")
logging.error("An error occurred")

# Better logging setup
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    filename='app.log'
)

logger = logging.getLogger(__name__)
logger.info("Application started")
```

## Virtual Environments

Virtual environments keep project dependencies isolated.

```bash
# Create virtual environment
python -m venv venv

# Activate (Linux/Mac)
source venv/bin/activate

# Activate (Windows)
venv\Scripts\activate

# Install packages
pip install pandas

# Save dependencies
pip freeze > requirements.txt

# Install from requirements
pip install -r requirements.txt

# Deactivate
deactivate
```

## Best Practices for Data Engineering

### 1. Write Clean Code

```python
# Bad
def f(x):
    return x*2

# Good
def double_value(number):
    """Double the input number."""
    return number * 2
```

### 2. Use Type Hints

```python
def process_data(filename: str, limit: int = 100) -> pd.DataFrame:
    """
    Process data from a file.
    
    Args:
        filename: Path to the data file
        limit: Maximum number of rows to process
        
    Returns:
        Processed DataFrame
    """
    df = pd.read_csv(filename)
    return df.head(limit)
```

### 3. Handle Errors Gracefully

```python
def read_json_file(filepath: str) -> dict:
    """Read JSON file with error handling."""
    try:
        with open(filepath, 'r') as f:
            return json.load(f)
    except FileNotFoundError:
        logging.error(f"File not found: {filepath}")
        return {}
    except json.JSONDecodeError as e:
        logging.error(f"Invalid JSON in {filepath}: {e}")
        return {}
```

### 4. Add Logging

```python
def process_users(users: list) -> list:
    """Process list of users."""
    logging.info(f"Processing {len(users)} users")
    
    processed = []
    for user in users:
        try:
            # Process user
            processed.append(user)
        except Exception as e:
            logging.error(f"Error processing user {user.get('id')}: {e}")
            
    logging.info(f"Successfully processed {len(processed)} users")
    return processed
```

### 5. Keep Functions Small

```python
# Bad - does too much
def process_everything(filename):
    # Read, validate, transform, load - all in one function
    pass

# Good - single responsibility
def read_data(filename):
    """Read data from file."""
    pass

def validate_data(data):
    """Validate data quality."""
    pass

def transform_data(data):
    """Apply transformations."""
    pass

def load_data(data, destination):
    """Load data to destination."""
    pass
```

## Common Patterns in Data Engineering

### 1. Reading Multiple Files

```python
import glob

# Get all CSV files
files = glob.glob('data/*.csv')

# Read and combine
dfs = []
for file in files:
    df = pd.read_csv(file)
    dfs.append(df)

combined = pd.concat(dfs, ignore_index=True)
```

### 2. Batch Processing

```python
def process_in_batches(items, batch_size=1000):
    """Process items in batches."""
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        yield batch

# Usage
for batch in process_in_batches(large_list, batch_size=100):
    # Process batch
    pass
```

### 3. Retry Logic

```python
import time

def retry_operation(func, max_retries=3, delay=1):
    """Retry operation on failure."""
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            logging.warning(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(delay * (attempt + 1))
```

## Practice Exercise
Use this dataset [dataset/orders.csv](../dataset/orders.csv) to create a Python script that:
1. Reads a CSV file
2. Filters rows based on a condition
3. Adds a new calculated column
4. Writes the result to a new CSV file
5. Includes error handling and logging

## Summary

You now know:
- Python basics for data manipulation
- Essential libraries (Pandas, JSON)
- File handling and error management
- Logging and best practices
- Common data engineering patterns

## Next Steps

Move on to [SQL Fundamentals](03-sql-fundamentals.md).

## Additional Resources

- [Python Official Documentation](https://docs.python.org/3/)
- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [Real Python Tutorials](https://realpython.com/)

---

**Next**: [SQL Fundamentals](03-sql-fundamentals.md)
