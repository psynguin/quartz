---
title: "Performance, Memory Optimization, and Scaling"
tags: [python, data-handling, pandas, numpy]
chapter: 17
theme: "Performance, Memory Optimization, and Scaling"
---

Performance and memory management differentiate functional pandas code from robust, production-ready pipelines. As datasets grow from thousands of rows to millions, inefficient operations that once ran instantly can bottleneck workflows, and unoptimized memory usage can trigger system crashes. The core mental model for pandas performance centers on prioritizing vectorized operations, managing memory actively, employing fast conditional logic, and using chunking strategies for data that exceeds available RAM.

```python
import pandas as pd
import numpy as np

# Monthly sales dataset — base dataset used throughout the book
df = pd.DataFrame({
    'date': pd.to_datetime(['2024-01-05', '2024-01-12', '2024-01-20',
                            '2024-02-03', '2024-02-14', '2024-02-28',
                            '2024-03-07', '2024-03-19', '2024-03-25']),
    'store_id': ['S01', 'S02', 'S01', 'S03', 'S02', 'S01', 'S03', 'S02', 'S01'],
    'category': ['Electronics', 'Clothing', 'Electronics',
                 'Food', 'Electronics', 'Clothing',
                 'Food', 'Clothing', 'Electronics'],
    'units_sold': [12, 35, 8, 50, 22, 18, 45, 30, 14],
    'unit_price': [299.99, 49.99, 299.99, 9.99, 149.99, 49.99, 9.99, 49.99, 299.99],
    'discount': [0.10, 0.0, 0.15, 0.05, 0.0, 0.20, 0.05, 0.10, 0.0],
    'region': ['North', 'South', 'North', 'East', 'South', 'North', 'East', 'South', 'North']
})

# Multi-DataFrame version (dict of monthly DataFrames — used in multi-df sections)
dfs = {
    'january': df[df['date'].dt.month == 1].reset_index(drop=True),
    'february': df[df['date'].dt.month == 2].reset_index(drop=True),
    'march': df[df['date'].dt.month == 3].reset_index(drop=True),
}
```

## Vectorization First

The single most important rule for writing fast pandas code is to use **vectorized operations**. Under the hood, pandas stores data in contiguous blocks of memory (using NumPy arrays or Apache Arrow) and passes operations down to highly optimized C or Cython code. 

When you use a `for` loop over the rows of a DataFrame, you bypass this optimization. The Python interpreter has to process each row individually, unpack the variables, perform the calculation, and repack the result—a process that is orders of magnitude slower.

### The Anti-Pattern: For-Loops over Rows

Suppose you want to compute the final revenue for each row (`units_sold * unit_price * (1 - discount)`).

```python
# Method 1: The For-Loop Anti-Pattern (VERY SLOW)
revenues = []
for index, row in df.iterrows():
    rev = row['units_sold'] * row['unit_price'] * (1 - row['discount'])
    revenues.append(rev)
df['revenue_loop'] = revenues
```

While `df.iterrows()` works, it is an anti-pattern. If you must loop, `df.itertuples()` is slightly faster because it returns namedtuples rather than Series objects, but it is still fundamentally a loop.

### The Solution: Vectorized Operations

By contrast, a vectorized operation computes the result for the entire column simultaneously in C. 

```python
# Method 2: Vectorized Arithmetic (FASTEST)
df['revenue_vectorized'] = df['units_sold'] * df['unit_price'] * (1 - df['discount'])
```

**When to Use**: Always default to vectorized operations whenever possible. Use `for` loops or `itertuples()` only as a last resort when the logic fundamentally cannot be vectorized.

This concept extends to string and datetime operations as well. Instead of iterating through a column to process text, use the `.str` accessor. Instead of parsing dates in a loop, use the `.dt` accessor.

```python
# Vectorized string and datetime extraction
df['year_month'] = df['date'].dt.to_period('M')
df['region_upper'] = df['region'].str.upper()
```

> [!TIP] The Benchmark Mindset
> Always assume that a built-in pandas or NumPy method exists for your operation. If you catch yourself writing a `for` loop or using `apply()`, pause and check the documentation. A vectorized solution is almost always available and will be 100x to 1000x faster.

## Apply Trade-Offs

When a native vectorized operation doesn't exist, you might turn to `.apply()`. However, not all `.apply()` calls are created equal.

- **`Series.apply(func)`**: Applies a Python function to each element of a single column. It acts as a glorified `for` loop but is somewhat optimized.
- **`DataFrame.apply(func, axis=1)`**: Applies a function to every row across multiple columns. This is extremely slow because pandas must assemble a new Series for every single row.

```python
# Method 1: DataFrame.apply with axis=1 (SLOW)
def calculate_tier(row):
    if row['units_sold'] > 30 and row['unit_price'] < 100:
        return 'Volume Sales'
    return 'Standard'

df['tier_apply'] = df.apply(calculate_tier, axis=1)
```

### Element-wise Mapping: `.map()` vs `.applymap()`

When you need to apply a function or dictionary mapping to *every single element* in a structure:
- **`Series.map(dict_or_func)`**: Extremely fast for dictionary lookups on a single column.
- **`DataFrame.map(func)`** (formerly `DataFrame.applymap()` before pandas 2.1): Applies a function element-by-element across the entire DataFrame.

```python
# Fast mapping using a dictionary
category_codes = {'Electronics': 'E', 'Clothing': 'C', 'Food': 'F'}
df['cat_code'] = df['category'].map(category_codes)

# Element-wise formatting across the entire DataFrame
# df_str = df.select_dtypes(include=['object']).map(lambda x: str(x).strip())
```

### When is `.apply()` Unavoidable?

**When to Use**: Use `Series.map()` for fast dictionary lookups. Use `DataFrame.map()` to alter every single cell independently. Use `.apply()` only when the logic relies on external libraries that do not support vectorization (e.g., API calls, complex regex matching, Natural Language Processing functions, or deeply nested custom business logic). If it can be expressed via arithmetic or simple conditions, avoid `.apply()`.

## Fast Vectorized Conditionals: `np.where` and `np.select`

If you are using `.apply()` to implement if-else logic, you should replace it with NumPy's vectorized conditional functions.

### Two Branches: `np.where`

For simple if-else logic, use `np.where(condition, true_value, false_value)`. It operates on the entire array at once.

```python
# Fast vectorized equivalent of an if/else block
df['high_value'] = np.where(df['unit_price'] > 100, 'Yes', 'No')
```

### Multiple Branches: `np.select`

For logic resembling an `if-elif-else` chain, use `np.select(condlist, choicelist, default)`.

```python
# Define the conditions
conditions = [
    (df['units_sold'] > 40),
    (df['units_sold'] > 20) & (df['units_sold'] <= 40),
    (df['units_sold'] <= 20)
]

# Define the corresponding outputs
choices = ['High Volume', 'Medium Volume', 'Low Volume']

# Vectorized multiple branching
df['volume_tier'] = np.select(conditions, choices, default='Unknown')
```

**When to Use**: Use `np.where` for simple two-way (if-else) conditional logic. Use `np.select` when you have three or more branches (if-elif-else) to keep your code clean and avoid deeply nested `np.where` statements.

### Applying to Multiple DataFrames

To apply vectorized conditional logic uniformly across a collection of DataFrames:

```python
# Using a dict comprehension to apply np.select to all monthly dataframes
def add_tiers(data):
    conds = [data['units_sold'] > 40, data['units_sold'] > 20]
    vals = ['High', 'Medium']
    return data.assign(
        tier = np.select(conds, vals, default='Low')
    )

dfs_tiered = {month: add_tiers(d) for month, d in dfs.items()}
```

## Memory Optimization

Pandas is an in-memory tool, meaning the size of your dataset is limited by your computer's RAM. Pandas often defaults to memory-heavy data types (like 64-bit integers and floats) which consume more RAM than necessary.

### Inspecting Memory Usage

To accurately measure the memory footprint, use `.info(memory_usage='deep')` or `.memory_usage(deep=True)`. The `deep=True` parameter ensures that pandas inspects the true size of object (string) columns.

```python
# Check memory usage per column in bytes
usage = df.memory_usage(deep=True)
print(usage)
```

### Downcasting Numeric Types

By default, numeric columns are cast as `int64` or `float64`. If a column only contains small integers (e.g., `units_sold` from 0 to 100), it can safely be downcast to `int8` or `int16`.

```python
# Downcast float64 to float32
df['unit_price'] = pd.to_numeric(df['unit_price'], downcast='float')

# Downcast int64 to the smallest possible integer type
df['units_sold'] = pd.to_numeric(df['units_sold'], downcast='integer')
```

### Converting Object Types to Category

Columns containing strings (`object` type) are extremely memory-intensive because Python stores each string individually. If a column has a low number of unique values (low cardinality), convert it to the `category` type. Under the hood, pandas stores each unique string once and uses small integers to reference them in the array.

```python
# Convert categorical strings to category dtype
df['category'] = df['category'].astype('category')
df['region'] = df['region'].astype('category')
df['store_id'] = df['store_id'].astype('category')

# Check memory usage again after optimizations
usage_after = df.memory_usage(deep=True)
print("Savings:\n", usage - usage_after)
```

> [!TIP] When to use Categories
> A good rule of thumb is to convert an object column to `category` if the number of unique values is less than 50% of the total number of rows.

### Applying to Multiple DataFrames

You can write a pipeline function to optimize memory across a dictionary of DataFrames:

```python
def optimize_memory(data):
    d = data.copy()
    # Downcast numerics
    for col in d.select_dtypes(include=['float64']).columns:
        d[col] = pd.to_numeric(d[col], downcast='float')
    for col in d.select_dtypes(include=['int64']).columns:
        d[col] = pd.to_numeric(d[col], downcast='integer')
        
    # Convert low-cardinality objects to category
    for col in d.select_dtypes(include=['object']).columns:
        if d[col].nunique() / len(d) < 0.5:
            d[col] = d[col].astype('category')
    return d

dfs_optimized = {month: optimize_memory(d) for month, d in dfs.items()}
```

## Copy vs. View and the SettingWithCopyWarning

One of the most confusing errors in pandas is the `SettingWithCopyWarning`. This occurs when you attempt to modify a DataFrame that might be a *view* (a reference to the original data) rather than a *copy* (an independent block of memory). 

### The Cause

When you filter a DataFrame, pandas decides dynamically whether to return a view or a copy to save memory. If you then assign a value to that filtered slice, pandas warns you because it isn't sure whether you meant to modify the original DataFrame or just the temporary slice.

```python
# ⚠️ This triggers SettingWithCopyWarning
south_stores = df[df['region'] == 'South']
south_stores['discount'] = 0.50 
```

### The Fix

If you are extracting a subset of data with the intention of modifying it independently, use `.copy()` to explicitly create an independent memory block.

```python
# ✅ The proper fix: explicit copy
south_stores = df[df['region'] == 'South'].copy()
south_stores['discount'] = 0.50 
```

Alternatively, if you want to modify the original DataFrame, use `.loc[]` in a single step:

```python
# ✅ The proper fix: modify original via .loc
df.loc[df['region'] == 'South', 'discount'] = 0.50
```

> [!NOTE] Copy-on-Write (CoW)
> Starting in pandas 2.0, a new mode called "Copy-on-Write" was introduced to eliminate this confusion entirely. When enabled, any DataFrame derived from another behaves like a view until you modify it, at which point pandas automatically creates a copy behind the scenes. 
> You can enable it globally: `pd.options.mode.copy_on_write = True`.

## Chunking Large Files

If you attempt to read a 10GB CSV file on a machine with 8GB of RAM, pandas will crash with a `MemoryError`. To bypass this, you can process the file in chunks using the `chunksize` parameter in `pd.read_csv()`.

This turns `read_csv` into an iterator that yields DataFrames of the specified row count.

```python
# Example processing pattern for large files using a context manager
chunk_size = 100000
total_units = 0

# The iterator loop using 'with' to ensure the file handle closes safely
# with pd.read_csv('massive_sales_data.csv', chunksize=chunk_size) as reader:
#     for chunk in reader:
#         # Process only the chunk currently in memory
#         chunk_units = chunk['units_sold'].sum()
#         total_units += chunk_units

# print(f"Total units sold across all chunks: {total_units}")
```

### Aggregating Across Chunks

Often, you want to filter out unnecessary rows and accumulate a smaller, filtered result.

```python
# result_chunks = []
# with pd.read_csv('massive_sales_data.csv', chunksize=chunk_size) as reader:
#     for chunk in reader:
#         # Keep only electronics
#         filtered_chunk = chunk[chunk['category'] == 'Electronics']
#         result_chunks.append(filtered_chunk)
# 
# # Combine all filtered chunks into a single manageable DataFrame
# final_df = pd.concat(result_chunks, ignore_index=True)
```

### Applying to Multiple DataFrames

To process a large collection of massive files, you can wrap the chunking logic in a pipeline function and apply it to a list of file paths.

```python
# import glob
# file_paths = glob.glob('data/monthly_sales_*.csv')
# 
# def process_large_file(file_path):
#     """Reads a single large file in chunks and returns a filtered summary."""
#     chunks = []
#     with pd.read_csv(file_path, chunksize=100000) as reader:
#         for chunk in reader:
#             # Filter and aggregate
#             summary = chunk[chunk['region'] == 'North'].groupby('category')['units_sold'].sum()
#             chunks.append(summary)
#     
#     # Combine chunks for this single file
#     return pd.concat(chunks).groupby(level=0).sum()
#
# # Process the entire collection
# dfs_summaries = {path: process_large_file(path) for path in file_paths}
```

## Other Speed Tips

If you've optimized your dtypes and fully vectorized your code, but performance is still lacking, consider these advanced techniques:

- **`df.eval()` and `df.query()`**: For DataFrames with millions of rows, `df.eval('units_sold * unit_price')` and `df.query('units_sold > 30')` can evaluate expressions faster than standard bracket notation by leveraging the `numexpr` library (which uses multi-threading and avoids allocating intermediate arrays).
- **Faster I/O with PyArrow**: When reading massive CSVs or Parquet files, use the PyArrow engine for significantly faster parsing: `pd.read_csv('file.csv', engine='pyarrow')`.
- **Numba for Rolling Apply**: For custom functions used inside a `.rolling().apply()` window, enable the Numba engine to compile your Python function into machine code: `df['col'].rolling(3).apply(custom_func, engine='numba', raw=True)`.

### Method Comparison Table

| Technique | Used For | Speed Impact | Effort to Implement |
|-----------|----------|--------------|---------------------|
| **Vectorized Math** | Arithmetic operations | Extreme (100x+) | Low |
| **`np.where`** | If-else logic | High | Low |
| **`.copy()`** | Safe memory separation | Minimal (fixes bugs) | Low |
| **`category` dtype** | Memory reduction | High (RAM savings) | Low |
| **Chunking** | Reading files > RAM | Makes the impossible possible | Medium |
| **`df.query()`** | Complex filters on large data | Medium (on 1M+ rows) | Low |
| **`apply(axis=1)`** | Complex logic unsuited for vectorization | Negative (very slow) | Low |

## Related Chapters
- [[02-numpy-arrays]] — Deep dive into NumPy mechanics and boolean arrays
- [[07-data-types-and-conversion]] — Exploring the categorical dtype
- [[12-multi-dataframe-workflows]] — Using pipeline functions across dictionary collections
- [[16-input-output]] — PyArrow engine and efficient I/O formats like Parquet
