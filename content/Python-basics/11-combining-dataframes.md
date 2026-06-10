---
title: "Combining DataFrames — Concat, Merge, and Join"
tags: [python, data-handling, pandas, numpy]
chapter: 11
theme: "Combining DataFrames — Concat, Merge, and Join"
---

# Combining DataFrames — Concat, Merge, and Join

A core part of any data pipeline is bringing data together. Whether you are stacking monthly transaction files on top of one another or enriching sales data with store information from a lookup table, pandas provides specialized tools for the job.

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

# Supplementary store information dataset for merge/join examples
df_stores = pd.DataFrame({
    'store_id': ['S01', 'S02', 'S03', 'S04'],
    'location': ['New York', 'Los Angeles', 'Chicago', 'Houston'],
    'manager': ['Alice', 'Bob', 'Charlie', 'Diana']
})
```

## `pd.concat`: Stacking DataFrames

Concatenation is used to stitch DataFrames together either vertically (adding rows) or horizontally (adding columns). It relies purely on the axes and indexes, without attempting to match specific column values like a database join.

### Syntactic Variants for `pd.concat`

```python
# We will use two subsets of our main DataFrame for demonstration
df_top = df.head(3)
df_bottom = df.tail(3)

# Method 1: Vertical (row-wise) concatenation (default)
pd.concat([df_top, df_bottom], axis=0)

# Method 2: Vertical concat with a fresh RangeIndex
pd.concat([df_top, df_bottom], ignore_index=True)

# Method 3: Vertical concat adding a key level to track the source (creates a MultiIndex)
pd.concat([df_top, df_bottom], keys=['Top', 'Bottom'])

# Method 4: Horizontal (column-wise) concatenation
# (Note: this aligns on the index. Since df_top and df_bottom have different 
# indexes, this will result in NaNs unless we reset indexes first)
pd.concat([df_top.reset_index(drop=True), df_bottom.reset_index(drop=True)], axis=1)

# Method 5: Handling mismatched columns/indexes with 'join'
# inner: keeps only overlapping items. outer (default): keeps all items, filling NaNs
pd.concat([df_top, df_bottom], join='inner')

# Method 6: Concatenating a dictionary of DataFrames
# This automatically uses the dictionary keys to create a hierarchical index
pd.concat(dfs)
```

**When to Use**: Use `pd.concat` when you have structurally similar datasets that need to be stacked together, such as combining multiple files of daily server logs into a single master dataset.

> [!TIP] Performance Tip: Collect, Then Concat
> Never call `pd.concat` inside a `for` loop. Because DataFrames are immutable in structure, calling `concat` creates a brand new DataFrame in memory each time. Instead, append your DataFrames to a Python list, and call `pd.concat(list_of_dfs)` exactly once at the end of the loop.

## `pd.merge`: Database-Style Joins

`pd.merge` aligns DataFrames based on the actual values inside specific columns, replicating SQL `JOIN` behavior. 

**Join Types (`how=`)**:
- **Inner Join** (Intersection): Keeps only rows with matching keys in *both* DataFrames.
- **Left Join** (Left set): Keeps all rows from the *left* DataFrame. Fills missing matches from the right with NaN.
- **Right Join** (Right set): Keeps all rows from the *right* DataFrame.
- **Outer Join** (Union): Keeps all rows from *both* DataFrames.

### Syntactic Variants for `pd.merge`

```python
# Method 1: Merge on a common column name (defaults to 'inner')
pd.merge(df, df_stores, on='store_id')

# Method 2: Left merge (keeping all sales, even if store info is missing)
pd.merge(df, df_stores, on='store_id', how='left')

# Method 3: Merge when column names differ
# Pretend our store dataframe has the column named 'id' instead of 'store_id'
df_stores_renamed = df_stores.rename(columns={'store_id': 'id'})
pd.merge(df, df_stores_renamed, left_on='store_id', right_on='id').drop(columns='id')

# Method 4: Merge using indexes
df_stores_indexed = df_stores.set_index('store_id')
pd.merge(df, df_stores_indexed, left_on='store_id', right_index=True)

# Method 5: Suffix handling for overlapping columns
# If both DataFrames had a 'region' column, they'd be renamed region_x and region_y
pd.merge(df, df_stores, on='store_id', suffixes=('_sales', '_info'))

# Method 6: Validating the merge relationship
# Prevents accidental cartesian products if the relationship isn't what you expect
pd.merge(df, df_stores, on='store_id', validate='many_to_one')

# Method 7: Adding an indicator column to trace the source
pd.merge(df, df_stores, on='store_id', how='outer', indicator=True)
# Outputs a '_merge' column with values 'both', 'left_only', or 'right_only'
```

**When to Use**: Use `pd.merge` when you need to enrich a dataset with attributes from another dataset based on matching identifiers (e.g., matching customer IDs to a customer demographics table).

> [!WARNING] The Many-to-Many Explosion Gotcha
> If the column you merge on has duplicate values in *both* DataFrames, pandas will create a Cartesian product for those rows. This drastically increases the number of rows in your output and inflates your aggregations. Use the `validate='one_to_one'` or `validate='many_to_one'` argument to have pandas catch this error and raise a `MergeError` before polluting your data.

## `df.join`: Index-Based Combination

`df.join` is a convenience method that combines DataFrames directly on their indexes. Under the hood, it utilizes `pd.merge`, but the syntax is streamlined for index alignment.

### Syntactic Variants for `df.join`

```python
# Setup: indexing DataFrames
df_indexed = df.set_index('store_id')
df_stores_indexed = df_stores.set_index('store_id')

# Method 1: Standard join on index (defaults to 'left')
df_indexed.join(df_stores_indexed)

# Method 2: Join on the calling DataFrame's column to the other DataFrame's index
df.join(df_stores_indexed, on='store_id')

# Method 3: Joining multiple DataFrames at once
# df_extra would be another DataFrame with 'store_id' as its index
# df_indexed.join([df_stores_indexed, df_extra])
```

**Practical difference from merge**: 
1. `df.join` operates on the index by default, while `pd.merge` defaults to using common columns.
2. `df.join` defaults to a `left` join; `pd.merge` defaults to an `inner` join.
3. `df.join` can accept a list of multiple DataFrames to combine at once without chaining multiple commands.

**When to Use**: Use `df.join` when your DataFrames are already neatly indexed by the same keys or when you need to combine three or more indexed DataFrames simultaneously.

## `df.compare`: Finding Differences

When you have two identically structured DataFrames and you want to see exactly what has changed between them, use `df.compare`.

```python
# Create a modified copy of the dataset
df_modified = df.copy()
df_modified.loc[0, 'unit_price'] = 199.99  # Change price
df_modified.loc[1, 'discount'] = 0.50      # Change discount

# Compare the original against the modified
df.compare(df_modified)

# Compare with custom names for clarity
df.compare(df_modified, result_names=('Original', 'Modified'))
```

**When to Use**: This is exceptionally useful for data auditing, validation pipelines, or inspecting the before-and-after effects of a risky transformation.

## Common Errors and Gotchas

### 1. Merge Validation Errors
When combining datasets, a common mistake is assuming a one-to-one relationship when the data actually contains duplicates, leading to silent data duplication (a cartesian product).

**The Error:**
```python
# Introduce a duplicate into df_stores to simulate dirty data
df_stores_dirty = df_stores.copy()
df_stores_dirty.loc[4] = ['S01', 'Boston', 'Eve']  # Duplicate S01

# Enforcing validation raises an exception rather than silently duplicating data
pd.merge(df, df_stores_dirty, on='store_id', validate='many_to_one')
```
```text
MergeError: Merge keys are not unique in right dataset; not a many-to-one merge
```

**Why it happens:**
The `validate='many_to_one'` argument strictly checks that the merge keys in the right DataFrame (`df_stores_dirty`) are unique. Because `S01` appears twice, pandas raises a `MergeError` to prevent the silent many-to-many explosion that would otherwise duplicate your sales records.

**The Fix:**
Clean or deduplicate the right DataFrame before merging.
```python
# Drop duplicates based on the merge key before merging
df_stores_clean = df_stores_dirty.drop_duplicates(subset=['store_id'], keep='first')
pd.merge(df, df_stores_clean, on='store_id', validate='many_to_one')
```

### 2. Concatenating with Misaligned Indexes
When performing a horizontal concatenation (`axis=1`), pandas aligns data based on the row index.

**The Error:**
```python
df_a = df.head(2)
# Create a second DataFrame with a completely different index
df_b = df.tail(2)

# Concatenating horizontally
pd.concat([df_a, df_b], axis=1)
```
*Result:* A DataFrame with 4 rows and many `NaN` values, because the indexes `[0, 1]` and `[7, 8]` do not align.

**Why it happens:**
`pd.concat` strictly matches the index labels. Since the top rows have indexes 0 and 1, and the bottom rows have 7 and 8, pandas places them on separate rows.

**The Fix:**
Reset the index on both DataFrames before horizontal concatenation if you simply want to staple them side-by-side.
```python
pd.concat([df_a.reset_index(drop=True), df_b.reset_index(drop=True)], axis=1)
```

## Method Comparison Table

| Method | Axis | Key Type | Aggregation | Typical Use |
|--------|------|----------|-------------|-------------|
| `concat` | Both | Optional | No | Stacking similar datasets |
| `merge` | Column | Column values | No | Database-style joins |
| `join` | Column | Index | No | Index-based combination |

---

## Applying to Multiple DataFrames

Working with collections of DataFrames often requires combining them into a single master table for analysis. Here is the full workflow pattern for loading, tagging, and concatenating multiple datasets.

### Dict-Based Pattern

If your DataFrames are stored in a dictionary (e.g., keys are filenames or month names), you can add a tracking column to each and then concatenate them. 

```python
# Step 1: Add a 'month' column to each DataFrame to track the source
dfs_with_month = {}
for name, month_df in dfs.items():
    # .assign() creates a copy with the new column
    dfs_with_month[name] = month_df.assign(month=name)

# Alternatively, using dictionary comprehension (Functional style)
dfs_with_month = {name: month_df.assign(month=name) for name, month_df in dfs.items()}

# Step 2: Concatenate all dictionary values into a single DataFrame
df_master_dict = pd.concat(dfs_with_month.values(), ignore_index=True)
```

Alternatively, if you pass the raw dictionary directly to `pd.concat`, pandas will automatically build a MultiIndex using the dict keys:
```python
df_multi_index = pd.concat(dfs)
# The index will be ('january', 0), ('january', 1), etc.
```

### List-Based Pattern

If you store your DataFrames in a list, you lose the explicit dictionary keys, so you must define the tracking variable sequentially if you wish to track sources.

```python
dfs_list = [dfs['january'], dfs['february'], dfs['march']]
month_names = ['january', 'february', 'march']

dfs_processed = []
for idx, month_df in enumerate(dfs_list):
    processed_df = month_df.assign(month=month_names[idx])
    dfs_processed.append(processed_df)

df_master_list = pd.concat(dfs_processed, ignore_index=True)
```

### The "Concatenate Monthly Files" Worked Example

Here is a full, real-world workflow. Imagine you have loaded three monthly files representing sales. You want to track the source month, stack them together, and verify the operation.

```python
# 1. Load data (we use our `dfs` dict here as the stand-in for pd.read_csv)
monthly_files = {
    'january': dfs['january'],
    'february': dfs['february'],
    'march': dfs['march']
}

# 2. Add 'month' column to trace data origin
prepared_dfs = []
for month_name, data in monthly_files.items():
    data = data.assign(month=month_name)
    prepared_dfs.append(data)

# 3. Concat exactly once
df_master = pd.concat(prepared_dfs, ignore_index=True)

# 4. Verify rows matched up properly
print(df_master['month'].value_counts())
# Expected output: 
# february    3
# march       3
# january     3
# Name: month, dtype: int64
```

### Caveats for Bulk Concatenation
1. **Misaligned Columns**: If one DataFrame has a column spelled `unit_price` and another has `Unit_Price`, `pd.concat` will create two separate columns, populating the gaps with NaNs. Always standardize column names across your collection *before* concatenating.
2. **Type Coercion**: If a column is strictly `int64` in one DataFrame and `float64` (perhaps due to a missing value) in another, the final concatenated column will be upcast to `float64` for all rows.

---

## Related Chapters
- [[05-selecting-and-accessing-data]]
- [[10-reshaping-dataframes]]
- [[12-multi-dataframe-workflows]]
- [[14-multiindex-hierarchical-indexing]]
