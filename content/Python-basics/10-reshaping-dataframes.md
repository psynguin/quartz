---
title: "Reshaping DataFrames"
tags: [python, data-handling, pandas, numpy]
chapter: 10
theme: "Reshaping: Pivoting, Melting, and Structural Transformations"
---

## Introduction to Reshaping

Data reshaping is the process of altering the structural representation of your data—its rows and columns—without changing the actual information it contains. In real-world data science, you frequently receive data in a "wide" format (many columns, fewer rows) but need it in a "long" format (fewer columns, many rows) for visualization or machine learning, or vice versa. 

Think of reshaping like rearranging a deck of cards: you still have the same 52 cards, but you can lay them out in a single line, stack them by suit, or group them by face value.

Throughout this chapter, we will use the following running dataset of monthly sales:

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

---

## 1. Pivot

The `pivot` method reshapes data from "long" format to "wide" format. It uses unique values from specified index/columns to form the axes of the resulting DataFrame.

```text
Original:             Pivot(index='date', columns='store_id', values='units_sold'):
date        store  units      store        S01  S02  S03
2024-01-05  S01    12   --->  date 
2024-01-12  S02    35         2024-01-05   12   NaN  NaN
2024-01-20  S01    8          2024-01-12   NaN  35   NaN
                              2024-01-20   8    NaN  NaN
```

### Syntactic Variants

```python
# Method 1: Specify index, columns, and values
df.pivot(index='date', columns='store_id', values='units_sold')

# Method 2: Omit 'values' to pivot all remaining columns (creates a MultiIndex column)
df.pivot(index='date', columns='store_id')

# Method 3: Using multiple columns for index or columns (pandas 1.1.0+)
df.pivot(index=['date', 'region'], columns='store_id', values='units_sold')
```

**When to Use**: Use `pivot` when you want to reshape data purely structurally without any aggregation. This is best when every combination of `index` and `columns` has exactly zero or one value in your dataset.

### Worked Examples

**Minimal Example**: Pivoting just the dates and stores.
```python
# Simple pivot of units sold
minimal_pivot = df.pivot(index='date', columns='store_id', values='units_sold')
```

**Realistic Example**: Pivoting multiple value columns at once to create a comprehensive daily store report.
```python
# Creates a DataFrame with a MultiIndex on the columns (e.g., ('units_sold', 'S01'))
daily_report = df.pivot(index='date', columns='store_id', values=['units_sold', 'unit_price'])
# Fill NaN with 0 for units sold, but leave NaN for prices where no sale occurred
daily_report['units_sold'] = daily_report['units_sold'].fillna(0)
```

### Common Errors and Gotchas

> [!WARNING] The Duplicate Entries ValueError
> If there are multiple rows with the same `index` and `columns` combination, `pivot` does not know how to combine them and will throw a `ValueError`.

```python
try:
    df.pivot(index='region', columns='category', values='units_sold')
except ValueError as e:
    print(f"Error: {type(e).__name__} - {e}")
    # Error: ValueError - Index contains duplicate entries, cannot reshape
```
**Why it happens:** In our dataset, the "North" region has multiple sales in "Electronics". `pivot` cannot decide which value to place in the new `(North, Electronics)` cell.
**The fix:** Use `pivot_table` instead, which aggregates duplicate entries.

---

## 2. Pivot Table

`pivot_table` is the robust sibling of `pivot`. It behaves exactly like an Excel PivotTable: it reshapes the data *and* aggregates duplicate entries using an aggregation function (like `mean`, `sum`, or `count`).

```text
Original (with duplicates):   Pivot_table(index='region', columns='category', aggfunc='sum'):
region  category  units       category   Clothing  Electronics  Food
North   Elec      12    --->  region
North   Elec      8           North      18        34           NaN
North   Clothing  18          South      65        22           NaN
```

### Syntactic Variants

```python
# Method 1: Basic mean aggregation (default aggfunc is 'mean')
df.pivot_table(index='region', columns='category', values='units_sold')

# Method 2: Adding margins (grand totals) and replacing NaNs
df.pivot_table(index='region', columns='category', values='units_sold', aggfunc='sum', margins=True, fill_value=0)

# Method 3: Multiple aggregation functions at once
df.pivot_table(index='region', columns='category', values='units_sold', aggfunc=['sum', 'mean'])

# Method 4: Different aggregation functions for different values
df.pivot_table(index='region', columns='category', values=['units_sold', 'unit_price'], aggfunc={'units_sold': 'sum', 'unit_price': 'mean'})
```

**When to Use**: Use `pivot_table` whenever you are reshaping wide but expect duplicates in the key combinations, or when you are explicitly trying to summarize or group your data visually. 

### Worked Examples

**Minimal Example**: Sum of units sold by region.
```python
# Equivalent to grouping by region and summing units_sold
region_sales = df.pivot_table(index='region', values='units_sold', aggfunc='sum')
```

**Realistic Example**: Building a full executive summary grid.
```python
# Complex executive summary: Total units and average prices, grouped by region and category, with grand totals
exec_summary = df.pivot_table(
    index='region',
    columns='category',
    values=['units_sold', 'unit_price'],
    aggfunc={'units_sold': 'sum', 'unit_price': 'mean'},
    margins=True,
    fill_value=0
)
```

### Common Errors and Gotchas

> [!WARNING] Aggregating Non-Numeric Data
> Passing columns to `pivot_table` that contain strings or dates without specifying a compatible aggregation function can cause errors.

```python
try:
    # 'store_id' is a string. The default aggfunc='mean' will fail on strings.
    df.pivot_table(index='region', values='store_id')
except Exception as e: # Will raise TypeError or DataError depending on pandas version
    print(f"Error: {type(e).__name__} - {e}")
    # Error: DataError - No numeric types to aggregate
```
**Why it happens:** The default `aggfunc` is `np.mean`. Pandas cannot calculate the mean of string values like `'S01'`.
**The fix:** Specify an aggregation function that works on categorical/string data, such as `'count'`, or `'nunique'`, or a custom lambda like `lambda x: ', '.join(x)`.

### Applying to Multiple DataFrames

When working with collections of datasets (like our `dfs` dictionary of monthly sales), you often need to reshape all of them identically before further analysis.

#### The Dict-Based Pattern (Preferred)

```python
# Create a summarized wide-format table for each month
monthly_pivots = {
    month: month_df.pivot_table(
        index='store_id', 
        columns='category', 
        values='units_sold', 
        aggfunc='sum', 
        fill_value=0
    ) 
    for month, month_df in dfs.items()
}

# Access January's pivoted data
print(monthly_pivots['january'].head())
```

#### The List-Based Pattern

If your DataFrames are in a list instead of a dictionary:

```python
dfs_list = list(dfs.values())

pivoted_list = [
    d.pivot_table(index='store_id', columns='category', values='units_sold', fill_value=0)
    for d in dfs_list
]
```

#### Caveats: Misaligned Categories

> [!WARNING] Misaligned Categories
> When using `pivot_table` across multiple DataFrames, some DataFrames might lack certain categories (e.g., maybe no "Food" was sold in February). This will result in different columns for different DataFrames. If you plan to concatenate them later, Pandas will fill the missing columns with `NaN`s, but it's best to explicitly `.reindex()` the columns inside your loop to enforce a uniform shape.

```python
# Enforcing consistent columns across all pivoted DataFrames
expected_categories = ['Clothing', 'Electronics', 'Food']

uniform_pivots = {
    m: d.pivot_table(index='store_id', columns='category', values='units_sold', fill_value=0)
       .reindex(columns=expected_categories, fill_value=0)
    for m, d in dfs.items()
}
```

---

## 3. Melt (Wide to Long)

`melt` is the exact inverse of `pivot`. It takes multiple columns and collapses them down into two columns: a "variable" column (containing the old column names) and a "value" column (containing the data).

```text
Original (Wide):              Melt(id_vars='store', value_vars=['Q1', 'Q2']):
store   Q1   Q2               store  variable  value
S01     10   20      --->     S01    Q1        10
S02     30   NaN              S02    Q1        30
                              S01    Q2        20
                              S02    Q2        NaN
```

To demonstrate this, let's temporarily create a wide DataFrame:
```python
wide_df = df.pivot_table(index='store_id', columns='category', values='units_sold', fill_value=0).reset_index()
# wide_df columns: ['store_id', 'Clothing', 'Electronics', 'Food']
```

### Syntactic Variants

```python
# Method 1: DataFrame method (preferred)
wide_df.melt(id_vars='store_id', value_vars=['Clothing', 'Electronics', 'Food'], var_name='product_category', value_name='items_sold')

# Method 2: Pandas top-level function
pd.melt(wide_df, id_vars=['store_id']) # Unspecified value_vars defaults to all other columns

# Method 3: pd.wide_to_long (for structured prefix names like 'sales_Q1', 'sales_Q2')
# pd.wide_to_long(df, stubnames='sales', i='store_id', j='quarter', sep='_')
```

**When to Use**: Use `melt` to tidy up data where variable names are used as column headers (e.g., columns named `2020`, `2021`, `2022`). This "long" format is generally required by plotting libraries like Seaborn and for machine learning models.

### Worked Examples

**Minimal Example**: Melting all columns.
```python
# This will melt every column into rows, which is rarely what you want!
full_melt = wide_df.melt() 
```

**Realistic Example**: Reconstructing a tidy dataset from a wide summary report.
```python
# Keeping store_id as the identifier, melting the categories into rows, and dropping the 0s
tidy_sales = wide_df.melt(
    id_vars='store_id',
    var_name='category',
    value_name='units_sold'
)
tidy_sales = tidy_sales[tidy_sales['units_sold'] > 0].sort_values('store_id')
```

### Common Errors and Gotchas

> [!WARNING] Missing Columns in id_vars or value_vars
> Spelling errors in column names passed to `id_vars` or `value_vars` will raise a KeyError.

```python
try:
    wide_df.melt(id_vars='Store_ID') # Typo: capital letters
except KeyError as e:
    print(f"Error: {type(e).__name__} - {e}")
    # Error: KeyError - "The following 'id_vars' are not present in the DataFrame: ['Store_ID']"
```
**Why it happens:** Pandas strictly checks that the columns you are asking to retain or melt actually exist in the DataFrame.
**The fix:** Double-check your column names, often by printing `wide_df.columns`, and ensure exact matches.

### Applying to Multiple DataFrames

If you have a collection of wide-format DataFrames (such as the `monthly_pivots` generated in the pivot section above), you can reshape all of them back into a long format using dictionary or list comprehensions.

#### The Dict-Based Pattern (Preferred)

```python
# Melt them back to long format, assuming `monthly_pivots` is already generated
monthly_long = {
    month: pivot_df.reset_index().melt(
        id_vars='store_id', 
        value_name='units_sold'
    )
    for month, pivot_df in monthly_pivots.items()
}
```

#### The List-Based Pattern

```python
# If you had a list of pivoted DataFrames instead
long_list = [
    pivot_df.reset_index().melt(id_vars='store_id', value_name='units_sold')
    for pivot_df in pivoted_list
]
```

---

## 4. Stack and Unstack

`stack` and `unstack` manipulate the structure of indexes. `stack` "compresses" columns into the row index (making the DataFrame taller and narrower), while `unstack` "expands" the row index into columns (making it shorter and wider).

```text
Unstack: row index -> columns
Stack: columns -> row index

Index: [North, South]         Unstack:
Columns: [Clothing, Elec] ---> Index: [North]
Data: 2x2 grid                 Columns: [(Clothing, North), (Clothing, South)...]
```

### Syntactic Variants

```python
# First, let's create a DataFrame with a MultiIndex using groupby
multi_df = df.groupby(['region', 'category'])[['units_sold']].sum()

# Method 1: unstack() moves the innermost row index ('category') to the columns
unstacked = multi_df.unstack()

# Method 2: unstack(level) specifying which level to move
unstacked_region = multi_df.unstack(level='region')

# Method 3: stack() moves the innermost column index back to the row index
restacked = unstacked.stack()
```

**When to Use**: These are most commonly used immediately after a `.groupby()` aggregation that produces a MultiIndex. They are the programmatic primitives that power `pivot` and `melt` under the hood.

### Worked Examples

**Minimal Example**: Unstacking a groupby result.
```python
# Moves 'category' (the innermost index) to be columns
category_columns = df.groupby(['region', 'category'])['units_sold'].sum().unstack()
```

**Realistic Example**: Swapping hierarchy to compare regions side-by-side.
```python
# Get a MultiIndex series
sales_stats = df.groupby(['region', 'category'])['units_sold'].mean()
# By unstacking the 0th level ('region'), we make 'region' the columns, allowing easy side-by-side category comparison
region_comparison = sales_stats.unstack(level=0).fillna(0)
```

### Common Errors and Gotchas

> [!WARNING] Unstacking Non-Unique Hierarchical Indexes
> If your MultiIndex has duplicate entries for the same hierarchical combination, `unstack` will fail.

```python
# Creating an artificial duplicate in the index
dup_df = pd.DataFrame({'val': [1, 2]}, index=pd.MultiIndex.from_tuples([('A', 'x'), ('A', 'x')]))

try:
    dup_df.unstack()
except ValueError as e:
    print(f"Error: {type(e).__name__} - {e}")
    # Error: ValueError - Index contains duplicate entries, cannot reshape
```
**Why it happens:** Like `pivot`, `unstack` does no aggregation. If there are two values for `('A', 'x')`, it doesn't know which one to put in the column `x` for row `A`.
**The fix:** Apply an aggregation like `.groupby(level=[0,1]).sum()` before calling `unstack()`.

---

## 5. Crosstab

`pd.crosstab` creates a cross-tabulation of two (or more) factors. By default, it computes a frequency table (counts) of the factors.

```text
Original:                       Crosstab(index=region, columns=category):
region  category                category   Clothing   Electronics   Food
North   Elec           --->     region
South   Clothing                North      0          2             0
North   Elec                    South      1          0             0
```

### Syntactic Variants

```python
# Method 1: Basic frequency count
pd.crosstab(index=df['region'], columns=df['category'])

# Method 2: Adding margins (row and column totals)
pd.crosstab(index=df['region'], columns=df['category'], margins=True)

# Method 3: Normalizing to get proportions instead of counts
pd.crosstab(index=df['region'], columns=df['category'], normalize='index')

# Method 4: Crosstab with values and an aggregation function
pd.crosstab(index=df['region'], columns=df['category'], values=df['units_sold'], aggfunc='mean')
```

**When to Use**: Use `crosstab` when you need to quickly check the frequency or percentage distribution of categorical variables.

### Worked Examples

**Minimal Example**: Counting transactions per region and category.
```python
transaction_counts = pd.crosstab(df['region'], df['category'])
```

**Realistic Example**: Creating a probability matrix (percentage of total sales within each region).
```python
# normalize='index' means each row will sum to 1.0 (100%)
# This shows us the breakdown of category popularity *within* each region
category_share_by_region = pd.crosstab(
    index=df['region'], 
    columns=df['category'], 
    normalize='index'
) * 100 # Multiply by 100 for percentages
```

### Common Errors and Gotchas

> [!WARNING] Passing Strings Instead of Arrays/Series
> Unlike `pivot_table` (which is a DataFrame method and takes column names as strings), `crosstab` is a top-level pandas function and expects actual array-like objects (like a pandas Series).

```python
try:
    # Incorrectly passing strings instead of Series
    pd.crosstab(index='region', columns='category')
except ValueError as e:
    print(f"Error: {type(e).__name__} - {e}")
    # Error: ValueError - If passing a list/tuple/array of values, they must be 1-dimensional
```
**Why it happens:** `pd.crosstab` does not know which DataFrame "region" belongs to, because you didn't pass it the DataFrame.
**The fix:** Pass the actual Series objects: `pd.crosstab(index=df['region'], columns=df['category'])`.

---

## 6. Explode

`explode` transforms each element of a list-like column into a separate row, replicating the index values.

```text
Original:             Explode('items'):
id  items             id  items
1   [A, B]   --->     1   A
2   [C]               1   B
                      2   C
```

### Syntactic Variants

```python
# Let's create a DataFrame with lists
list_df = pd.DataFrame({
    'order_id': ['O-1', 'O-2'],
    'items': [['Apple', 'Banana'], ['Carrot']]
})

# Method 1: Explode a single column
list_df.explode('items')

# Method 2: Exploding multiple columns simultaneously (pandas 1.3.0+)
# If multiple columns are exploded, they must contain lists of the exact same length in every row.
```

**When to Use**: Very common when dealing with JSON or NoSQL data dumps where a column contains lists or arrays of items that need to be normalized into individual records.

### Worked Examples

**Minimal Example**: Unpacking the items list.
```python
exploded_df = list_df.explode('items')
```

**Realistic Example**: Exploding a list, then calculating costs.
```python
# Imagine we have lists of items and their corresponding quantities
cart_df = pd.DataFrame({
    'order_id': [1, 2],
    'items': [['Laptop', 'Mouse'], ['Monitor']],
    'qty': [[1, 2], [3]]
})

# Explode both columns at the same time
# (Requires pandas 1.3+ and matching list lengths in each row)
flat_cart = cart_df.explode(['items', 'qty'])
```

### Common Errors and Gotchas

> [!WARNING] Exploding Multiple Columns with Mismatched Lengths
> When using `explode` on multiple columns, the lists inside the same row must be exactly the same length.

```python
bad_cart = pd.DataFrame({
    'order_id': [1],
    'items': [['Laptop', 'Mouse']], # Length 2
    'qty': [[1]]                    # Length 1
})

try:
    bad_cart.explode(['items', 'qty'])
except ValueError as e:
    print(f"Error: {type(e).__name__} - {e}")
    # Error: ValueError - columns must have matching element counts
```
**Why it happens:** Pandas pairs up the elements row-by-row. If one list has 2 items and the other has 1, it cannot align them into flat rows.
**The fix:** Ensure your upstream data aligns properly before exploding, or explode one column, then merge/join the secondary data.

---

## 7. Indicator Variables (One-Hot Encoding)

`get_dummies` converts categorical variables into dummy/indicator variables (binary 0 or 1). This is often called "one-hot encoding" in machine learning.

```text
Original:             get_dummies(df, columns=['category']):
id  cat               id   cat_A   cat_B
1   A        --->     1    1       0
2   B                 2    0       1
```

### Syntactic Variants

```python
# Method 1: Basic get_dummies on specified columns
pd.get_dummies(df, columns=['category'])

# Method 2: drop_first=True to avoid the dummy variable trap
pd.get_dummies(df, columns=['category'], drop_first=True)

# Method 3: Customizing the prefix separator
pd.get_dummies(df, columns=['category'], prefix_sep='==')

# Method 4: Reversing the operation (Pandas 1.5.0+)
# dummies = pd.get_dummies(df[['category']], prefix='cat')
# pd.from_dummies(dummies, sep='_')
```

**When to Use**: This is an essential preprocessing step for many machine learning algorithms (like linear regression or standard neural networks) that require numerical inputs and cannot handle raw categorical text.

### Worked Examples

**Minimal Example**: Encoding the 'region' column.
```python
# Converts 'region' into 'region_East', 'region_North', 'region_South'
features = pd.get_dummies(df, columns=['region'])
```

**Realistic Example**: Preparing a DataFrame for linear regression.
```python
# We one-hot encode categorical features, drop the first category to ensure linear independence, 
# and cast the resulting bools to integers (1/0).
ml_dataset = pd.get_dummies(
    df[['units_sold', 'unit_price', 'category', 'region']], 
    columns=['category', 'region'], 
    drop_first=True, 
    dtype=int
)
```

### Common Errors and Gotchas

> [!WARNING] Exploding Memory Usage
> Calling `get_dummies` on a high-cardinality continuous or text column will create a massive DataFrame and crash your memory.

```python
try:
    # Attempting to get dummies for a unique continuous float column
    # If the column has 1,000 unique values, this creates 1,000 new columns!
    # In a real large dataset, this often raises MemoryError
    _ = pd.get_dummies(df, columns=['unit_price'])
except MemoryError as e:
    print(f"Error: {type(e).__name__} - {e}")
```
**Why it happens:** `get_dummies` creates a new column for *every unique value* in the target column. If applied to continuous numbers, text, or IDs, the column count explodes.
**The fix:** Only apply `get_dummies` to strictly categorical data with a limited set of discrete values. Use `pd.cut()` to bin continuous data first if you really need to dummy encode it.

---

## Related Chapters
- [[09-aggregation-and-groupby]]
- [[11-combining-dataframes]]
- [[12-multi-dataframe-workflows]]
- [[14-multiindex-hierarchical-indexing]]
