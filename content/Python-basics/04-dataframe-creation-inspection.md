---
title: "DataFrame Construction and Inspection"
tags: [python, data-handling, pandas, numpy]
chapter: 4
theme: "DataFrame Construction and Inspection"
---

# DataFrame Construction and Inspection

The pandas `DataFrame` is the cornerstone of data handling in Python. It is a two-dimensional, size-mutable, potentially heterogeneous tabular data structure with labeled axes (rows and columns). Understanding the various ways to construct a DataFrame is crucial because your data will come from a myriad of sources—web APIs (JSON-like nested dictionaries), database query results (lists of tuples), or direct user input. Once data is loaded, knowing how to immediately inspect its shape, types, and summary statistics prevents silent errors and informs your entire downstream workflow.

## Base Dataset

We will use the following monthly sales dataset throughout our examples.

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

## DataFrame Creation Methods

Data can be ingested into pandas from almost any native Python or NumPy data structure.

### From a Dictionary of Lists

This is the most common programmatic way to define a DataFrame manually. The keys become column headers, and the lists become the column values. All lists must be the exact same length.

```python
# Dictionary of lists
data_dict = {
    'store_id': ['S01', 'S02', 'S03'],
    'region': ['North', 'South', 'East']
}

df_from_dict = pd.DataFrame(data_dict)
```

### From a List of Dictionaries (Records Format)

This format frequently arises when consuming JSON from web APIs or document databases. Each dictionary represents one row. Missing keys in specific dictionaries are automatically filled with `NaN`.

```python
# List of dictionaries
data_records = [
    {'store_id': 'S01', 'region': 'North', 'manager': 'Alice'},
    {'store_id': 'S02', 'region': 'South'}, # Missing 'manager' will become NaN
    {'store_id': 'S03', 'region': 'East', 'manager': 'Charlie'}
]

df_from_records = pd.DataFrame(data_records)
```

### From a List of Lists

When data is strictly tabular but lacks headers (e.g., raw CSV reading without a header row or SQL fetchall results), you can create a DataFrame from a list of lists. You must provide the column names explicitly.

```python
# List of lists
data_lists = [
    ['S01', 'North'],
    ['S02', 'South'],
    ['S03', 'East']
]

df_from_lists = pd.DataFrame(data_lists, columns=['store_id', 'region'])
```

### From a NumPy 2D Array

Because pandas DataFrames are backed by NumPy arrays, creating a DataFrame from an existing `ndarray` is straightforward and fast.

```python
# 2D NumPy array
array_2d = np.array([
    [100, 0.1],
    [150, 0.2],
    [200, 0.15]
])

df_from_array = pd.DataFrame(array_2d, columns=['sales', 'margin'])
```

### From Another DataFrame

You can create a new DataFrame from an existing one. By default, pandas simply references the underlying data. If you want an independent copy, use the `copy=True` argument.

```python
# Create from another DataFrame
df_copy = pd.DataFrame(df, copy=True)
```

### Advanced Creation: `from_dict` and `from_records`

Pandas provides alternate constructor methods for more granular control over how the dictionary structure is interpreted.

**Method 1: `pd.DataFrame.from_dict()`**
This method is especially useful when your dictionary's keys should be the *row index* rather than the columns. Control this with the `orient` parameter.

```python
# Nested dictionary
nested_dict = {
    'S01': {'region': 'North', 'tier': 1},
    'S02': {'region': 'South', 'tier': 2}
}

# orient='index' makes the outer keys ('S01', 'S02') the row index
df_oriented = pd.DataFrame.from_dict(nested_dict, orient='index')

# orient='columns' is the default behavior, similar to pd.DataFrame()
df_default = pd.DataFrame.from_dict(nested_dict, orient='columns')
```

**Method 2: `pd.DataFrame.from_records()`**
If your data comes as a list of tuples (like SQL output), `from_records` is slightly more efficient than the default constructor. You can also specify which field should become the index.

```python
sql_results = [('S01', 'North'), ('S02', 'South')]
df_sql = pd.DataFrame.from_records(sql_results, columns=['store_id', 'region'])
```

### Empty DataFrame with Predefined Columns

Sometimes you need to initialize an empty structure before appending rows in a loop.

> [!WARNING] Performance Warning
> Appending rows to a DataFrame in a loop is extremely inefficient. It is always better to accumulate data in a native Python list first, and then build the DataFrame once at the end.

```python
empty_df = pd.DataFrame(columns=['date', 'store_id', 'units_sold'])
```

### Choosing a Creation Method

| Method | Source Data Structure | When to Use |
|--------|----------------------|-------------|
| `pd.DataFrame()` | Dict of lists | Defining explicit column-based data. |
| `pd.DataFrame()` | List of dicts | Parsing API JSON responses (row-based). |
| `pd.DataFrame()` | List of lists / NumPy array | Processing purely matrix-like tabular data. |
| `.from_dict(orient='index')` | Nested dict | Using the outer dictionary keys as row indexes. |
| `.from_records()` | List of tuples | Loading database `fetchall()` results. |

---

## Inspecting and Exploring the DataFrame

> [!TIP] Best Practice
> Always run `.info()` and `.describe()` immediately after loading any new dataset before doing any transformation. This habit catches unexpected data types, missing values, and bizarre anomalies instantly.

### Viewing Rows: `.head()`, `.tail()`, and `.sample()`

These methods allow you to glimpse subsets of your dataset without printing thousands of rows to your console.

```python
# View the first 5 rows (default is 5)
df.head()

# View the last 3 rows
df.tail(3)

# View a random sample of 4 rows
df.sample(4)

# View a reproducible random sample
df.sample(4, random_state=42)
```

**When to use:** Use `.head()` to verify column headers and general data formats. Use `.tail()` to check if the dataset has unwanted summary rows at the bottom (common in Excel exports). Use `.sample()` to avoid the bias of chronologically sorted data, ensuring you see a true mix of records.

### Metadata and Structure: `.info()`

The `.info()` method is your primary diagnostic tool. It prints a concise summary including the index dtype, column names, non-null counts, column dtypes, and memory usage.

```python
df.info()
```

*Interpreting the output:*
- **RangeIndex:** Shows the total number of rows (e.g., `9 entries, 0 to 8`).
- **Non-Null Count:** If this number is lower than the total entries, the column contains missing data (`NaN`).
- **Dtype:** Confirms whether pandas correctly identified numeric fields (`int64`, `float64`) or parsed dates (`datetime64[ns]`). If a numeric column is labeled `object`, it likely contains strings or mixed types.

### Summary Statistics: `.describe()`

The `.describe()` method generates descriptive statistics. By default, it only processes numeric columns.

```python
# Numeric statistics (count, mean, std, min, 25%, 50%, 75%, max)
df.describe()

# Include all columns (numeric, objects, datetimes, categoricals)
df.describe(include='all')

# Target specific dtypes
df.describe(include=['object'])
```

*Interpreting the output:*
- For **numeric** columns: Look for impossibly high maximums or negative minimums where there shouldn't be any. Check if the mean is wildly skewed compared to the median (50%).
- For **categorical/object** columns (`include='all'`): Provides `unique` (number of distinct values), `top` (most frequent value), and `freq` (frequency of the top value).

### Core Attributes

Instead of printing methods, DataFrames have numerous attributes (no parentheses) that return specific metadata properties.

```python
# Dimensionality
df.shape     # Returns a tuple: (rows, columns) e.g., (9, 7)
df.ndim      # Returns 2 (DataFrames are always 2D)
df.size      # Returns total number of elements (rows * columns)

# Axes
df.columns   # Returns an Index object with column names
df.index     # Returns the row labels/Index
df.dtypes    # Returns a Series with data types for each column

# Underlying Data
df.values    # Returns a 2D NumPy array of the DataFrame's data
```

### Memory Profiling: `.memory_usage()`

When working with large datasets, understanding the memory footprint is critical.

```python
# Returns memory used by each column in bytes
df.memory_usage()

# deep=True interrogates object dtypes to calculate exact string sizes
df.memory_usage(deep=True)
```

### Cardinality and Frequencies

To quickly understand the distinct values within specific columns, use cardinality and counting methods.

```python
# Returns the number of unique elements in each column
df.nunique()

# Returns the frequency count of unique values in a specific column
df['category'].value_counts()

# Returns the relative frequencies (percentages)
df['category'].value_counts(normalize=True)

# Includes NaN values in the count (by default they are excluded)
df['category'].value_counts(dropna=False)
```

### Choosing an Inspection Method

| Method | What it Returns | When to Use |
|--------|----------------|-------------|
| `.info()` | Console output of schema | Instantly checking for missing values or wrong data types (e.g., numbers loaded as objects). |
| `.describe()` | DataFrame of statistics | Checking summary distributions, outliers, and ranges. |
| `.shape` / `.ndim` | Tuple / Integer | Programmatically checking dimensions. |
| `.head()` / `.tail()` | DataFrame subset | Verifying column names and eyeballing actual values. |
| `.value_counts()` | Series of counts | Checking frequency of categories or finding corrupt strings. |

---

## Common Errors and Gotchas

### Passing Scalar Values to a Dictionary Without an Index

**The Error:**
Raises a `ValueError: If using all scalar values, you must pass an index`.
```python
# This will raise a ValueError
bad_df = pd.DataFrame({'store_id': 'S01', 'region': 'North'})
```
**Why it happens:** Pandas expects iterables (like lists) to define the rows for each column. If you give it single strings or numbers, it doesn't know how many rows to construct.
**The Fix:** Wrap the scalars in a list, or explicitly provide an index.
```python
# Fix 1: Wrap in a list
good_df = pd.DataFrame({'store_id': ['S01'], 'region': ['North']})

# Fix 2: Provide an explicit index
good_df = pd.DataFrame({'store_id': 'S01', 'region': 'North'}, index=[0])
```

### Value Counts vs Count

**The Error:** Using `.count()` when you meant `.value_counts()`.
**Why it happens:** `.count()` returns the number of *non-null* entries in a column or DataFrame. `.value_counts()` returns the frequency of *each distinct value*.
**The Fix:**
```python
df['category'].count()          # Output: 9
df['category'].value_counts()   # Output: Electronics 4, Clothing 3, Food 2
```

### Object Dtypes Hiding Corrupt Data

**The Gotcha:** You load a CSV, run `.info()`, and notice your `unit_price` column is an `object` dtype instead of `float64`.
**Why it happens:** One or more values in that column could not be parsed as a number. Often this is due to a stray currency symbol (e.g., `$299.99`), a comma (`2,000.00`), or text (`"N/A"`).
**The Fix:** You will need to clean the string and convert the type, which involves string replacement and explicit casting before running numerical operations.

---

## Related Chapters
- To understand the underlying arrays powering DataFrames, see [[02-numpy-arrays]].
- To learn how to extract individual columns (which are Series) to inspect them closer, see [[03-pandas-series]].
- To dive into extracting specific rows and subsets of your newly created DataFrame, see [[05-selecting-and-accessing-data]].
- To learn how to load DataFrames directly from files instead of Python objects, see [[16-input-output]].
