---
title: "Selecting and Accessing Data"
tags: [python, data-handling, pandas, numpy]
chapter: 5
theme: "Selecting and Accessing Data in DataFrames"
---

Extracting specific subsets of data from a DataFrame is the most common operation in pandas. Because a DataFrame has both an index (row labels) and column labels, pandas provides multiple accessors optimized for different access patterns. The mental model for selection is a grid: you can slice it vertically (columns), horizontally (rows), or use criteria to filter specific cells.

````python
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
````

## Column Selection

Column selection narrows down the features in your dataset while retaining all observations (rows). You use this when you want to focus on specific metrics or remove irrelevant variables from memory.

### Methods for Column Selection

````python
# Method 1: Bracket notation (returns a Series for a single column)
category_series = df['category']

# Method 2: Attribute access (returns a Series)
category_series = df.category

# Method 3: Bracket notation with a list (returns a DataFrame)
subset_df = df[['category', 'units_sold']]

# Using a list variable
cols = ['category', 'units_sold']
subset_df_var = df[cols]

# Method 4: Select by data type
numeric_cols = df.select_dtypes(include=['number'])
non_numeric = df.select_dtypes(exclude=['object', 'datetime64[ns]'])

# Method 5: Filter by regex pattern
unit_cols = df.filter(regex='^unit_')

# Method 6: Filter by explicit items
# Safely ignores 'non_existent' instead of raising a KeyError
safe_cols = df.filter(items=['region', 'discount', 'non_existent'])
````

**When to Use**:
- **Bracket notation**: This is the safest and most standard way to select a column. Always prefer this.
- **Attribute access**: Useful for quick interactive exploration, but avoid it in production scripts. It fails if your column name has spaces, begins with a number, or collides with a pandas method (e.g., `df.count`).
- **List of columns**: Use when you need to extract a sub-DataFrame.
- **`.select_dtypes()`**: Indispensable in data pipelines when applying transformations only to numeric or categorical data.
- **`.filter()`**: Excellent for working with wide datasets where columns share a naming convention, or when you want to safely attempt to select columns without triggering errors if they are missing.

### Common Errors and Gotchas

**KeyError**: Attempting to access a column that doesn't exist using bracket notation raises a `KeyError`.
````python
df['sales']  # KeyError: 'sales'
````
*Fix*: Check `.columns` or use `.filter(items=['sales'])` if the column's presence is optional.

**Series vs. DataFrame**: Double brackets return a DataFrame, single brackets return a Series.
````python
type(df['category'])    # pandas.core.series.Series
type(df[['category']])  # pandas.core.frame.DataFrame
````

### Applying to Multiple DataFrames

To standardize the columns across a dictionary of DataFrames, use a dict comprehension to iterate and subset:

````python
# Extract only the 'date' and 'units_sold' columns for each monthly DataFrame
sales_only_dfs = {k: d[['date', 'units_sold']] for k, d in dfs.items()}

# Select only numeric columns for all DataFrames
numeric_dfs = {k: d.select_dtypes(include=['number']) for k, d in dfs.items()}
````

## Row Selection — `.loc[]`

The `.loc[]` accessor performs **label-based** selection. It looks at the explicit index labels (row names) and column names. Use `.loc[]` when you know the specific named identifier of the row you want.

> [!WARNING] .loc Endpoints are Inclusive
> When slicing with `.loc['A':'C']`, the result includes both label 'A' and label 'C'. This is a major departure from standard Python slicing syntax.

### Methods for `.loc[]` Selection

````python
# Create a DataFrame with a meaningful index for demonstration
df_idx = df.set_index('date')

# Method 1: Single label (returns a Series)
row = df_idx.loc['2024-01-12']

# Method 2: List of labels (returns a DataFrame)
rows = df_idx.loc[['2024-01-12', '2024-02-14']]

# Method 3: Slice of labels (inclusive on both ends)
date_range = df_idx.loc['2024-01-05':'2024-02-03']

# Method 4: Row and column together (loc[row_indexer, column_indexer])
# Select row by date, column by name
cell = df_idx.loc['2024-01-12', 'store_id']

# Select row slice and column list
subset = df_idx.loc['2024-01-05':'2024-02-03', ['category', 'units_sold']]

# Method 5: Boolean Series with .loc
high_sales = df.loc[df['units_sold'] > 30, ['store_id', 'units_sold']]
````

**When to Use**:
- **`.loc[]`**: Always use `.loc` when you are retrieving data by its index label, or when you are using a boolean mask and want to select specific columns at the same time.

### Common Errors and Gotchas

**KeyError**: Passing a label that is not in the index.
````python
df_idx.loc['2025-01-01']  # KeyError: '2025-01-01'
````

**SettingWithCopyWarning**: Modifying data directly on a slice without `.loc` can cause pandas to warn you that you might be modifying a copy rather than the original DataFrame.
````python
# Bad: chained indexing
df[df['units_sold'] > 30]['discount'] = 0.5  # SettingWithCopyWarning

# Good: .loc indexing
df.loc[df['units_sold'] > 30, 'discount'] = 0.5
````

### Applying to Multiple DataFrames

````python
# Set 'store_id' as the index for all monthly DataFrames
dfs_by_store = {k: d.set_index('store_id') for k, d in dfs.items()}

# Extract rows for store 'S01' and specifically their sales volume across all months
s01_sales = {k: d.loc[['S01'], ['units_sold']] for k, d in dfs_by_store.items()}
````

## Row Selection — `.iloc[]`

The `.iloc[]` accessor performs **position-based** selection. It operates strictly using 0-based integer positions, behaving exactly like standard Python lists and NumPy arrays. 

> [!NOTE] .iloc Endpoints are Exclusive
> When slicing with `.iloc[0:3]`, pandas returns rows at index 0, 1, and 2. It excludes the endpoint, just like Python's `list[0:3]`.

### Methods for `.iloc[]` Selection

````python
# Method 1: Single position (returns a Series)
first_row = df.iloc[0]

# Method 2: List of positions
specific_rows = df.iloc[[0, 2, 4]]

# Method 3: Slice of positions (exclusive endpoint)
first_three = df.iloc[0:3]

# Method 4: Last N rows using negative indexing
last_two = df.iloc[-2:]

# Method 5: Row and column by position (iloc[row_pos, col_pos])
# Select first 3 rows, and the 2nd to 4th columns
subset_pos = df.iloc[0:3, 1:4]
````

**When to Use**:
- **`.iloc[]`**: Use this when your DataFrame order matters logically, such as extracting the top 5 rows after sorting, or blindly splitting a DataFrame in half, regardless of what the index labels are.

### Common Errors and Gotchas

**IndexError**: Providing an integer position that is out of bounds.
````python
df.iloc[100]  # IndexError: single positional indexer is out-of-bounds
````

### Applying to Multiple DataFrames

````python
# Extract the first two rows from each monthly dataset regardless of the index
first_two_records = {k: d.iloc[0:2] for k, d in dfs.items()}
````

## Boolean/Mask Access

Boolean masking filters rows based on a true/false condition. The expression generates a Series of booleans, and pandas returns only the rows corresponding to `True`.

### Methods for Boolean Access

````python
# Method 1: Single condition mask
high_sales = df[df['units_sold'] > 20]

# Method 2: Compound conditions using & (AND)
# Parentheses around each condition are MANDATORY
clothing_sales = df[(df['category'] == 'Clothing') & (df['units_sold'] >= 30)]

# Method 3: Compound conditions using | (OR) and ~ (NOT)
north_or_east = df[(df['region'] == 'North') | (df['region'] == 'East')]
not_s01 = df[~(df['store_id'] == 'S01')]

# Method 4: .isin() for checking membership against a list
food_clothing = df[df['category'].isin(['Food', 'Clothing'])]

# Method 5: .between() for numeric or date ranges
mid_range = df[df['unit_price'].between(20.0, 150.0)]

# Method 6: .query() with string expressions
query_res = df.query("units_sold > 20 and category == 'Electronics'")

# .query() with local variables using the @ prefix
target_store = 'S02'
store_res = df.query("store_id == @target_store")
````

**When to Use**:
- **Bracket masks (`df[mask]`)**: Best for simple, single conditions.
- **`.isin()` and `.between()`**: Dramatically cleaner than chaining multiple `==` or `>=`/`<=` operators.
- **`.query()`**: Highly recommended when chaining methods or when dealing with complex compound conditions, as it prevents the visual clutter of repeating `df[...]`.

### Common Errors and Gotchas

**ValueError/TypeError with Compound Masks**: Forgetting parentheses around compound conditions. Python's bitwise operators (`&`, `|`) have higher precedence than comparison operators (`>`, `==`), causing pandas to evaluate `5 & df['b']` first.
````python
# Raises TypeError or ValueError
df[df['units_sold'] > 20 & df['region'] == 'North']

# Fix
df[(df['units_sold'] > 20) & (df['region'] == 'North')]
````

### Applying to Multiple DataFrames

````python
# Filter for 'Electronics' across all monthly DataFrames using bracket notation
electronics_dfs = {k: d[d['category'] == 'Electronics'] for k, d in dfs.items()}

# Same operation using query
electronics_dfs_q = {k: d.query("category == 'Electronics'") for k, d in dfs.items()}
````

## Scalar Access

If you need to retrieve or set exactly one cell in the DataFrame, pandas provides optimized accessors: `.at[]` (label-based) and `.iat[]` (position-based).

### Methods for Scalar Access

````python
df_idx = df.set_index('date')

# Method 1: .at[] for label-based scalar access
val_at = df_idx.at[pd.Timestamp('2024-01-12'), 'units_sold']

# Method 2: .iat[] for position-based scalar access
# 0th row, 3rd column
val_iat = df.iat[0, 3] 
````

**When to Use**:
- Use `.at[]` and `.iat[]` strictly when speed is the priority (e.g., inside a loop or a custom algorithm iterating over a large DataFrame). They bypass the overhead of `.loc` and `.iloc` and return Python scalars immediately.

### Common Errors and Gotchas

**ValueError**: Passing lists or slices to `.at` or `.iat`. They are strictly for single values.
````python
df.iat[0:2, 3]  # ValueError: iat requires integer locators, not slice
````

### Applying to Multiple DataFrames

````python
# Extract the first unit price recorded in each monthly DataFrame
first_prices = {k: d.iat[0, 4] for k, d in dfs.items()}
````

## Filtering Rows

Pandas provides specialized methods for applying conditions to strings, dates, and nulls.

### Methods for Filtering

````python
# Method 1: String accessors (.str.contains, .str.startswith, .str.endswith, .str.match)
# Supports regex by default
s_stores = df[df['store_id'].str.startswith('S')]
s1_stores = df[df['store_id'].str.endswith('1')]
s_regex = df[df['store_id'].str.match(r'^S0[12]$')]

# Method 2: Date filtering via the .dt accessor
feb_data = df[df['date'].dt.month == 2]

# Method 3: Conditional column creation post-filter using np.where
df_flag = df.copy()
df_flag['high_sales'] = np.where(df_flag['units_sold'] > 30, 'Yes', 'No')

# Method 4: Filtering for non-null values
# (Assuming 'discount' might contain NaNs in real data)
valid_discounts = df[df['discount'].notna()]

# Method 5: Top N and Bottom N values
top_3_sales = df.nlargest(3, 'units_sold')
lowest_2_prices = df.nsmallest(2, 'unit_price')
````

**When to Use**:
- **`.str` filtering**: Essential for unstructured text matching.
- **`.nlargest()` and `.nsmallest()`**: Significantly faster and more readable than sorting the entire DataFrame and using `.head(n)`.

### Common Errors and Gotchas

**ValueError with `.str.contains`**: If a column has `NaN` values, `.str.contains` will return `NaN` instead of a boolean, which breaks the boolean mask evaluation.
````python
# Fix: Force NaN values to evaluate to False
df[df['category'].str.contains('Elec', na=False)]
````

### Applying to Multiple DataFrames

````python
# Retrieve the top selling record for each month
top_sales_per_month = {k: d.nlargest(1, 'units_sold') for k, d in dfs.items()}
````

## Selecting Specific Rows After Operations

When working with time-series data or hierarchically indexed data (MultiIndex), pandas provides highly specialized methods to extract specific subsets of rows without relying purely on position or standard labels.

### Methods for Time-Series and MultiIndex Selection

````python
# Setup 1: Time-series DatetimeIndex (ensure it is sorted)
df_time = df.set_index('date').sort_index()

# Extract the first 3 days of observations
first_3_days = df_time.first('3D')

# Extract the last 5 days of observations
last_5_days = df_time.last('5D')

# Setup 2: MultiIndex (Hierarchical Index)
# Group by store and category to create a MultiIndex DataFrame
df_multi = df.groupby(['store_id', 'category'])['units_sold'].sum().to_frame()

# Extract a cross-section (.xs)
# Select all rows where category is 'Electronics', regardless of the store_id
electronics_only = df_multi.xs('Electronics', level='category')
````

**When to Use**:
- **`.first()` and `.last()`**: Crucial for time-series workflows. Instead of manually finding the first date and computing an offset, `.first('3D')` elegantly grabs all records within 3 calendar days of the earliest timestamp.
- **`.xs()`**: Use when you need to select data from a specific inner level of a MultiIndex. It allows you to bypass the need to provide tuples for the outer index levels.

### Common Errors and Gotchas

**TypeError / AttributeError**: Using `.first()` or `.last()` requires the index to be a DatetimeIndex. If the index is a string or integer, it will fail.
````python
# df has a default integer index
df.first('3D')  # Raises TypeError: 'first' only supports a DatetimeIndex...
````
*Fix*: Ensure you `set_index('date_column')` first.

### Applying to Multiple DataFrames

````python
# Assume dfs is a dictionary of DataFrames, and we want to compute grouped MultiIndex stats
# and then extract only the 'Food' category from each DataFrame.
grouped_dfs = {k: d.groupby(['store_id', 'category'])[['units_sold']].sum() for k, d in dfs.items()}

# Apply .xs() to extract 'Food' sales per store across all months
food_sales_dfs = {k: d.xs('Food', level='category') for k, d in grouped_dfs.items() if 'Food' in d.index.get_level_values('category')}
````

## Method Comparison Table

| Method | Index Type | Endpoint Behavior | Returns | Speed | Best For |
|--------|-----------|------------------|---------|-------|----------|
| `df[mask]` | Boolean | N/A | DataFrame | Fast | Simple filters |
| `.loc` | Label | Inclusive | Scalar/Series/DataFrame | Medium | Label-based selection |
| `.iloc` | Integer | Exclusive | Scalar/Series/DataFrame | Medium | Position-based selection |
| `.at` | Label | N/A | Scalar | Fastest | Single value |
| `.iat` | Integer | N/A | Scalar | Fastest | Single value |
| `.query` | String expr | N/A | DataFrame | Medium | Readable chains & complex filters |

## Related Chapters
- [[04-dataframe-creation-inspection]]
- [[06-modifying-dataframes]]
- [[12-multi-dataframe-workflows]]
- [[15-method-chaining-and-pipelines]]
