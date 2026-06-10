---
title: "Aggregation and the GroupBy Paradigm"
tags: [python, data-handling, pandas, numpy]
chapter: 9
theme: "Aggregation and the GroupBy Paradigm"
---

## Understanding Aggregation and GroupBy

Aggregation is the process of turning multiple values into a single summary statistic—like taking thousands of sales records and computing total revenue. While aggregating an entire dataset is useful, data science often requires summarizing data *by category*.

This is where the **GroupBy** paradigm comes in. It follows a mental model known as **Split-Apply-Combine**:
1. **Split**: Break the dataset into distinct groups based on the values of one or more keys (e.g., group by `store_id`).
2. **Apply**: Compute a metric or perform a transformation on each individual group independently (e.g., sum the `units_sold`).
3. **Combine**: Stitch the results back together into a new, unified DataFrame.

Whenever you find yourself asking "What is the average X per Y?" or "How many Z did each W produce?", you need a GroupBy operation.

Let's define our running dataset used for these examples:

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

## Direct Column Aggregations

Before grouping data, you can aggregate the entire DataFrame or specific columns. Pandas provides standard statistical methods (`sum`, `mean`, `min`, `max`, `count`, `std`, `var`) that can be called directly or via the `.agg()` method.

```python
# Method 1: Single metric via direct function call
total_units = df['units_sold'].sum()

# Method 2: Multiple metrics on a Series using .agg()
unit_stats = df['units_sold'].agg(['sum', 'mean', 'max'])

# Method 3: Multiple metrics on multiple columns using .agg()
multi_stats = df[['units_sold', 'unit_price']].agg(['min', 'max'])

# Method 4: Dictionary mapping distinct columns to specific functions using .agg()
custom_stats = df.agg({
    'units_sold': 'sum',
    'unit_price': 'mean',
    'discount': ['min', 'max']
})

# Method 5: Descriptive summary of all numeric columns
summary = df.describe()
```

### Direct Aggregation Method Comparison

| Method | Syntax | Best For | Output Type |
|--------|--------|----------|-------------|
| **Direct Method** | `df['col'].sum()` | Quick, single-metric checks on a single column | Scalar |
| **List with `.agg()`** | `df['col'].agg(['sum', 'mean'])` | Multiple statistics applied uniformly | Series (1 col) or DataFrame |
| **Dict with `.agg()`** | `df.agg({'col1': 'sum', 'col2': 'mean'})` | Different logic for different columns | Series |
| **`.describe()`** | `df.describe()` | Full descriptive summary of all numeric/object columns | DataFrame |

## GroupBy Fundamentals

When you call `df.groupby('column')`, pandas does not immediately compute anything. Instead, it returns a `DataFrameGroupBy` object—a lazy evaluator holding the grouping metadata.

### Creating the GroupBy Object

```python
# Single column groupby
grouped_store = df.groupby('store_id')

# Multi-column groupby (creates a hierarchical group key)
grouped_store_cat = df.groupby(['store_id', 'category'])
```

### Inspecting the GroupBy Object

You can peer inside the object to understand how pandas split your data:

```python
# View the dictionary mapping group keys to row index locations
print(grouped_store.groups)

# Get the total number of distinct groups
print(grouped_store.ngroups)

# Get the number of rows in each group (returns a Series)
print(grouped_store.size())

# Extract a specific group as a distinct DataFrame
s01_data = grouped_store.get_group('S01')
```

### Iterating Over Groups

Sometimes you need to apply custom logic or save out each group. You can iterate directly over a GroupBy object, yielding a tuple of `(group_name, group_dataframe)`:

```python
for store, group_df in df.groupby('store_id'):
    print(f"Processing store: {store}")
    print(f"Average price in {store}: {group_df['unit_price'].mean():.2f}")
```

## Aggregation After GroupBy

The most common operation after splitting data is computing summary statistics for each distinct group.

```python
# Method 1: Direct function call on a grouped Series
df.groupby('store_id')['units_sold'].sum()

# Method 2: Multiple aggregations using .agg() with a list
df.groupby('store_id')[['units_sold', 'unit_price']].agg(['sum', 'mean'])

# Method 3: Dictionary-based per-column aggregation
df.groupby('store_id').agg({
    'units_sold': 'sum',
    'unit_price': 'mean'
})

# Method 4: Named Aggregations (Pandas 0.25+)
# Syntax: new_column_name=('source_column', 'aggregation_function')
df.groupby('store_id').agg(
    total_sold=('units_sold', 'sum'),
    avg_price=('unit_price', 'mean'),
    max_discount=('discount', 'max')
)
```

### GroupBy Aggregation Method Comparison

| Method | Syntax | Output Columns | Best For |
|--------|--------|----------------|----------|
| **Direct Call** | `.sum()` | Original names | Quick, single-metric checks |
| **List `.agg()`** | `.agg(['sum', 'mean'])` | MultiIndex (`col`, `func`) | Uniform stats across columns |
| **Dict `.agg()`** | `.agg({'c1': 'sum', 'c2': 'mean'})` | Original names | Different logic per column |
| **Named Aggs** | `.agg(new_name=('c1', 'sum'))` | Flat, custom names | Production code, clean outputs |

### Common Errors and Gotchas

```python
try:
    # Attempting to aggregate the whole DataFrame without isolating numeric columns
    df.groupby('store_id').mean()
except TypeError as e:
    print(f"TypeError: {e}")
```

> [!WARNING] Gotcha: Aggregating Non-Numeric Columns
> **Why it happens**: In modern pandas (2.0+), operations like `.mean()` or `.sum()` will raise a `TypeError` if applied to a GroupBy object containing datetime or string columns (like `category`, `region`, `date`), unless you explicitly tell pandas to ignore them.
> **The Fix**: Either pre-select numeric columns: `df.groupby('store_id')[['units_sold', 'unit_price']].mean()`, or pass the `numeric_only=True` parameter: `df.groupby('store_id').mean(numeric_only=True)`.

## Transform: Broadcasting Group Stats

While `.agg()` returns one row per group, `.transform()` returns an object of the **same shape as the original data**. It computes the group statistic and broadcasts it back to every original row.

```python
# 1. Computing Within-Group Percentages
# Compute total sales per store and broadcast it to a new column
df['store_total_units'] = df.groupby('store_id')['units_sold'].transform('sum')

# Now calculate what percentage each transaction contributed to its store's total
df['pct_of_store_sales'] = df['units_sold'] / df['store_total_units']

# 2. Group Normalization (Z-score)
# Standardize prices relative to other items in the SAME category
df['price_zscore'] = df.groupby('category')['unit_price'].transform(
    lambda x: (x - x.mean()) / x.std()
)

# 3. Filling NaNs with Group Means
# If we had missing discounts, we could fill them using the average discount for that region
# df['discount'] = df['discount'].fillna(
#     df.groupby('region')['discount'].transform('mean')
# )
```

### Common Transform Errors

```python
try:
    # Attempting to return a shape that cannot be broadcast back to the original group size
    df.groupby('store_id')['units_sold'].transform(lambda x: [1, 2])
except ValueError as e:
    print(f"ValueError: {e}")
```

> [!WARNING] Gotcha: Transform Shape Mismatch
> **Why it happens**: The function passed to `.transform()` must return either a single scalar value (which gets broadcast to all rows in the group) or an object of the exact same size as the group. If it returns an irregularly shaped array or list, pandas raises a `ValueError: Length mismatch`.
> **The Fix**: Ensure your transform function either aggregates to a single scalar (`x.sum()`) or applies an element-wise operation that preserves length (`x - x.mean()`).

> [!TIP] Transform vs. Agg
> Use `.agg()` when you want to reduce your dataset (fewer rows). Use `.transform()` when you want to keep your original data scale but append group-level context to each row.

## Filter: Keeping or Dropping Entire Groups

The `.filter()` method applies a boolean function to each group as a whole. If the function evaluates to `True`, the entire group is kept. If `False`, all rows belonging to that group are dropped from the resulting DataFrame.

```python
# Keep only stores that have sold more than 40 units in total across all transactions
high_volume_stores = df.groupby('store_id').filter(
    lambda g: g['units_sold'].sum() > 40
)
```

> [!WARNING] Common Error
> Do not confuse `df.groupby(...).filter()` with `df.filter()`. The GroupBy filter drops *rows* conditionally based on group-level statistics. The standard DataFrame filter drops *columns* based on their string names.

## Apply: The Most Flexible Method

If a group operation doesn't fit neatly into `.agg()`, `.transform()`, or `.filter()`, you can use `.apply()`. It passes each group sub-DataFrame into a custom Python function. Depending on what your function returns, pandas intelligently constructs the output.

```python
# 1. Returning a DataFrame (e.g., top N rows per group)
def top_transaction(group):
    return group.nlargest(1, 'units_sold')

best_sales = df.groupby('category').apply(top_transaction)

# 2. Returning a Series (e.g., multiple computed metrics per group)
def complex_metrics(group):
    profit = (group['units_sold'] * group['unit_price'] * (1 - group['discount'])).sum()
    avg_volume = group['units_sold'].mean()
    return pd.Series({'total_profit': profit, 'avg_volume': avg_volume})

category_metrics = df.groupby('category').apply(complex_metrics)

# 3. Returning a Scalar (e.g., a single custom metric per group)
def sales_range(group):
    return group['units_sold'].max() - group['units_sold'].min()

sales_spread = df.groupby('store_id').apply(sales_range)
```

### Common Apply Errors

```python
try:
    # Attempting to mutate the group inplace inside apply can lead to unpredictable results
    def bad_apply(group):
        group['new_col'] = group['units_sold'] * 2
        return group
    
    # Can raise TypeError or yield unexpected behavior depending on pandas version
    df.groupby('store_id').apply(bad_apply)
except Exception as e:
    print(f"Error: {e}")
```

> [!WARNING] Gotcha: Mutating Groups within Apply
> **Why it happens**: Functions passed to `.apply()` should not mutate the group DataFrame in-place. Pandas may evaluate the first group twice to infer whether the function is returning a Series, DataFrame, or scalar. If you mutate the group, it can cause duplicate operations or a `TypeError`.
> **The Fix**: Always return a new object or a copy, e.g., `return group.assign(new_col=...)`.

### Core GroupBy Methods Comparison

| Method | Purpose | Output Shape | Performance |
|--------|---------|--------------|-------------|
| **`.agg()`** | Compute summary statistics | One row per group | Fast (C-optimized) |
| **`.transform()`** | Broadcast group stats back to rows | Same shape as original data | Fast |
| **`.filter()`** | Keep/drop entire groups conditionally | Subset of original rows | Medium |
| **`.apply()`** | Complex custom logic | Depends on return type | Slow (Python loop) |

> [!TIP] Apply Trade-offs
> `.apply()` is incredibly flexible, but it is the slowest GroupBy method. Always prefer `.agg()` or `.transform()` with built-in functions when possible.

## Advanced GroupBy Patterns

### Time-Based Grouping with `pd.Grouper`
When working with time series, you can group data into frequencies (e.g., monthly) natively using `pd.Grouper`.

```python
# Group by month using the 'date' column
# Note: 'M' or 'ME' (Month End) specifies the frequency
monthly_sales = df.groupby(pd.Grouper(key='date', freq='M'))['units_sold'].sum()
```

### Grouping Categorical Data with `observed=True`
If you group by a `category` dtype, pandas traditionally creates an output row for *every possible category* defined in the data type, even if it doesn't appear in the current subset of data. To prevent this bloat, use `observed=True`.

```python
# Convert region to categorical
df['region'] = df['region'].astype('category')

# Filter the data to just one region
south_df = df[df['region'] == 'South']

# By default, grouping will show NaNs for North and East. 
# Using observed=True suppresses the unobserved categories.
south_totals = south_df.groupby('region', observed=True)['units_sold'].sum()
```

### Flattening MultiIndex Columns After Aggregation
If you used standard `.agg()` with a list of functions (Method 2), you will end up with hierarchical MultiIndex columns. You often need to flatten them to proceed smoothly.

```python
agg_df = df.groupby('store_id').agg({'units_sold': ['sum', 'mean']})

# Flatten the hierarchical columns: joins ('units_sold', 'sum') into 'units_sold_sum'
agg_df.columns = ['_'.join(col).strip() for col in agg_df.columns.values]
agg_df = agg_df.reset_index()
```

### Applying to Multiple DataFrames

When working with collections of DataFrames (like our `dfs` dict of monthly data), you frequently need to aggregate each dataset independently and consolidate the results to compare metrics across periods.

**1. The Dict-Based Pattern (Preferred)**
```python
# Compute total units sold per store, for each month's DataFrame
monthly_summaries = {
    month: month_df.groupby('store_id')['units_sold'].sum()
    for month, month_df in dfs.items()
}

# Combine the dictionary of Series into a single DataFrame.
# The index (store_id) aligns the rows automatically.
comparison_table = pd.DataFrame(monthly_summaries)
print(comparison_table)
```

**2. The List-Based Pattern**
If your DataFrames sit in a list, you collect the aggregations and concatenate them:
```python
dfs_list = list(dfs.values())

# Aggregate each DataFrame
list_summaries = [
    d.groupby('store_id')['units_sold'].sum() 
    for d in dfs_list
]

# Combine using concat (requires manual tracking of what each column represents)
comparison_list_table = pd.concat(list_summaries, axis=1, keys=['Jan', 'Feb', 'Mar'])
```

**Caveats When Applying GroupBy in Bulk**
- **Missing Keys:** If a category exists in one DataFrame but not another (e.g., store `S03` had no sales in January), computing the groupby independently results in mismatched indices. Fortunately, when combining them via `pd.DataFrame(dict)` or `pd.concat(axis=1)`, pandas seamlessly aligns the indices and inserts `NaN` where data was absent.
- **Categorical Mismatches:** If the grouping column is a `category` dtype, ensure all DataFrames share the exact same categories (via `pd.CategoricalDtype`) before grouping, or use `observed=True` to avoid mismatched category errors.

## Related Chapters
- [[05-selecting-and-accessing-data]]
- [[08-missing-data-handling]]
- [[10-reshaping-dataframes]]
- [[12-multi-dataframe-workflows]]
- [[14-multiindex-hierarchical-indexing]]
