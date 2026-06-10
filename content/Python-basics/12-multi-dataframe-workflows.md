---
title: "Working with Collections of DataFrames"
tags: [python, data-handling, pandas, numpy]
chapter: 12
theme: "Working with Collections of DataFrames"
---

## Why Collections of DataFrames?

Real-world data is rarely contained in a single neat file. You often deal with multiple monthly sales reports, per-subject experiment logs, or daily log dumps from different cities. In these scenarios, you have two primary design decisions: 

1. Keep the data separate and process them in parallel.
2. Merge them into one large DataFrame with a key column to track the source.

Keeping them separate is advantageous when each slice of data represents a distinct logical unit that must be individually exported or analyzed without memory bloat. Merging them into a single structure (such as a MultiIndex DataFrame) is often preferred when you plan to perform global aggregations across all sources.

```python
import pandas as pd
import numpy as np
import glob
from pathlib import Path

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

## Storage Patterns

### Dictionary of DataFrames (Primary Pattern)

A dictionary of DataFrames is the most idiomatic way to manage collections in pandas. It binds a descriptive name (key) to each DataFrame (value), preserving the source identity throughout transformations.

**Construction Methods:**

```python
# Method 1: From groupby
dfs_from_groupby = dict(tuple(df.groupby(df['date'].dt.month_name().str.lower())))

# Method 2: Manual assignment
dfs_manual = {
    'january': df_jan,
    'february': df_feb
}

# Method 3: From files using pathlib
# {f.stem: pd.read_csv(f) for f in Path('data_folder').glob('*.csv')}
```

**Accessing and Iterating:**

```python
# Access a specific DataFrame
jan_df = dfs['january']

# Iterating (essential for bulk processing)
for month_name, month_df in dfs.items():
    print(f"{month_name}: {len(month_df)} records")

# Adding, Updating, and Removing entries
dfs['april'] = df_jan.copy()     # Add a new entry
dfs['january'] = df_jan.copy()   # Update an existing entry
del dfs['april']                 # Remove an entry (or use dfs.pop('april'))

# Nested dicts (useful for complex hierarchies like year -> quarter -> df)
nested_dfs = {'2024': {'Q1': dfs['january']}}
```

### List of DataFrames

A list is useful when order matters or when the datasets do not have natural distinct names (e.g., anonymized batches).

```python
# Construction
dfs_list = [dfs['january'], dfs['february'], dfs['march']]
# Or from files: [pd.read_csv(f) for f in sorted(glob.glob('*.csv'))]

# Iterating
for i, month_df in enumerate(dfs_list):
    print(f"Batch {i}: {len(month_df)} records")

# Combining multiple lists for parallel iteration using zip
dfs_targets = [dfs['january'].copy(), dfs['february'].copy(), dfs['march'].copy()]
for df_actual, df_target in zip(dfs_list, dfs_targets):
    # e.g., compare_to_target(df_actual, df_target)
    pass
```

> [!NOTE] When to Prefer Which?
> Use a **dictionary** when the identity of the dataset matters (e.g., "sales_january"). Use a **list** for anonymous collections or strictly ordered batches where you intend to concatenate them immediately.

### `pd.concat` with Keys

You can combine a dictionary of DataFrames directly into a MultiIndex DataFrame, seamlessly bridging the gap between separate data and a single unified DataFrame.

```python
# The keys of the dictionary become the outermost index level
master_df = pd.concat(dfs)

# Accessing a specific source after concatenation
jan_reconstructed = master_df.loc['january']

# Alternative Pattern: Adding a key column before concatenation
for month, mdf in dfs.items():
    mdf['source_month'] = month

master_df_alt = pd.concat(list(dfs.values()), ignore_index=True)
# The identity is now stored in a regular column rather than an index level
```

### I/O for Collections: Excel, HDF5, Parquet, Pickle

When saving collections, you should aim to write them into formats that natively support grouped or multi-table structures.

**Saving multiple DataFrames:**

```python
# 1. Excel multi-sheet (Great for business reports)
with pd.ExcelWriter('monthly_sales.xlsx') as writer:
    for month, mdf in dfs.items():
        mdf.to_excel(writer, sheet_name=month, index=False)

# 2. HDF5 store (Great for fast numeric storage of multiple keys)
with pd.HDFStore('sales_data.h5') as store:
    for month, mdf in dfs.items():
        store[f'sales/{month}'] = mdf

# 3. Parquet folder pattern (Write one file per key in a directory)
# Path('sales_data').mkdir(exist_ok=True)
# for month, mdf in dfs.items():
#     mdf.to_parquet(f'sales_data/{month}.parquet')

# 4. Pickle dict (Quick Python-only persistence)
import pickle
with open('dfs.pkl', 'wb') as f:
    pickle.dump(dfs, f)

# 5. SQLite store
import sqlite3
with sqlite3.connect('sales_data.db') as conn:
    for month, mdf in dfs.items():
        mdf.to_sql(f'sales_{month}', conn, if_exists='replace', index=False)

# 6. Joblib (Often better than pickle for large numeric arrays)
import joblib
joblib.dump(dfs, 'dfs.joblib')
```

**Loading multiple DataFrames:**

```python
# 1. From Excel multi-sheet: sheet_name=None automatically returns a dict of DataFrames
loaded_dfs = pd.read_excel('monthly_sales.xlsx', sheet_name=None)

# 2. From HDF5
with pd.HDFStore('sales_data.h5') as store:
    loaded_h5_dfs = {key: store[key] for key in store.keys()}

# 3. From Parquet directory (loads all parquets as a single DataFrame)
# combined_parquet = pd.read_parquet('sales_data/')

# 4. From Pickle
with open('dfs.pkl', 'rb') as f:
    loaded_pickle_dfs = pickle.load(f)

# 5. From SQLite
with sqlite3.connect('sales_data.db') as conn:
    tables = pd.read_sql("SELECT name FROM sqlite_master WHERE type='table';", conn)['name']
    loaded_sql_dfs = {t: pd.read_sql(f"SELECT * FROM {t}", conn) for t in tables}

# 6. From Joblib
loaded_joblib_dfs = joblib.load('dfs.joblib')
```

## Transformation Patterns

When you have a collection of DataFrames, you typically want to apply identical data-cleaning steps to all of them.

### For-Loop Reassignment Pitfalls

A common mistake is modifying the iteration variable without updating the dictionary:

```python
# ❌ INCORRECT: This creates a new local object `mdf` but does not update the dict!
for month, mdf in dfs.items():
    mdf = mdf.dropna() 

# ✅ CORRECT: Explicit reassignment
for month, mdf in dfs.items():
    dfs[month] = mdf.dropna()
```

> [!WARNING] Mutability Gotcha
> Operations like `.dropna()` or `.assign()` return a *copy*. If you do not reassign the result back to the dictionary or list, your changes are lost. While `inplace=True` sometimes bypasses this, it is actively discouraged in modern pandas.

### Dict and List Comprehensions (Functional Style)

Comprehensions are the most Pythonic and robust way to process collections. They avoid mutation side-effects by creating an entirely new collection.

```python
# Dict comprehension (Preferred)
dfs_clean = {month: mdf.dropna() for month, mdf in dfs.items()}

# List comprehension
dfs_list_clean = [mdf.dropna() for mdf in dfs_list]
```

### The Pipeline Function Pattern

If your transformations involve more than a simple `.dropna()`, encapsulate the logic inside a function. This guarantees uniform application and is easiest to test.

```python
def clean_monthly_data(df):
    """A standard cleaning pipeline."""
    return (df
            .dropna(subset=['unit_price', 'units_sold'])
            .assign(revenue=lambda x: x['units_sold'] * x['unit_price'] * (1 - x['discount']))
            .rename(columns=str.lower))

# Apply uniformly via dict comprehension
dfs_processed = {month: clean_monthly_data(mdf) for month, mdf in dfs.items()}

# Apply to list via map
dfs_list_processed = list(map(clean_monthly_data, dfs_list))
```

### Sequential Accumulations with `functools.reduce`

While mapping applies a function to each DataFrame independently, sometimes you need to combine a list of DataFrames iteratively (e.g., sequentially merging them).

```python
from functools import reduce

# Sequentially merge a list of DataFrames on 'store_id'
dfs_to_merge = [dfs['january'], dfs['february'], dfs['march']]
merged_all = reduce(lambda left, right: pd.merge(left, right, on='store_id', how='outer'), dfs_to_merge)
```

### Combine-Then-Split

Sometimes it is easier (and faster) to concatenate all DataFrames first, apply vectorized pandas operations on the massive master table, and then split them back.

```python
# Combine
master_df = pd.concat(dfs, names=['month', 'original_index']).reset_index(level='month')

# Operate on everything at once (highly efficient)
master_df['discounted_price'] = master_df['unit_price'] * (1 - master_df['discount'])

# Split back
dfs_split_back = dict(tuple(master_df.groupby('month')))
```

> [!TIP] When to use Combine-Then-Split?
> Use this when memory allows and you need global statistics (like a grand mean normalization) or when looping in Python over thousands of small DataFrames is creating an unacceptable performance bottleneck.

## Parallel Operations Reference

Below is a quick reference showing how to perform common tasks uniformly across a collection.

| Operation | Dict Comprehension `dfs` | List Comprehension `dfs_list` |
|-----------|--------------------------|-------------------------------|
| **Rename Columns** | `{k: v.rename(columns={'old':'new'}) for k,v in dfs.items()}` | `[df.rename(columns={'old':'new'}) for df in dfs_list]` |
| **Drop Columns** | `{k: v.drop(columns=['col']) for k,v in dfs.items()}` | `[df.drop(columns=['col']) for df in dfs_list]` |
| **Cast Types** | `{k: v.astype({'col':'float'}) for k,v in dfs.items()}` | `[df.astype({'col':'float'}) for df in dfs_list]` |
| **Filter Rows** | `{k: v[v['col'] > 0] for k,v in dfs.items()}` | `[df[df['col'] > 0] for df in dfs_list]` |
| **Add Computed Column** | `{k: v.assign(new_col=v['a']*v['b']) for k,v in dfs.items()}` | `[df.assign(new_col=df['a']*df['b']) for df in dfs_list]` |
| **Handle Missing (Fill)** | `{k: v.fillna(0) for k,v in dfs.items()}` | `[df.fillna(0) for df in dfs_list]` |
| **Normalize Column** | `{k: v.assign(n=v['a']/v['a'].max()) for k,v in dfs.items()}` | `[df.assign(n=df['a']/df['a'].max()) for df in dfs_list]` |
| **Sort Values** | `{k: v.sort_values('date') for k,v in dfs.items()}` | `[df.sort_values('date') for df in dfs_list]` |
| **Reset Index** | `{k: v.reset_index(drop=True) for k,v in dfs.items()}` | `[df.reset_index(drop=True) for df in dfs_list]` |

> [!NOTE] Per-DataFrame vs. Global Normalization
> If you execute `{k: v.assign(norm=v['sales'] / v['sales'].max()) for k, v in dfs.items()}`, each DataFrame is normalized against **its own local maximum**. If you want to normalize against the **global maximum** across all DataFrames, you must find the global maximum first or use the combine-then-split pattern.

## Common Errors and Gotchas

When working with collections of DataFrames, certain Python and pandas errors surface frequently.

### 1. `KeyError` on Missing Dictionary Keys

**The Error:**
```python
# Assuming 'april' is not in our dfs dictionary
april_df = dfs['april']
```
```text
KeyError: 'april'
```

**Why it happens:**
You tried to access a DataFrame by a key that doesn't exist in the dictionary. This often occurs when a data file was missing from a folder you processed, or a grouping operation yielded no results for a specific category.

**The Fix:**
Use `.get()` with a fallback, or explicitly check for the key.
```python
# Safe access: returns an empty DataFrame (or None) if 'april' is missing
april_df = dfs.get('april', pd.DataFrame())
```

### 2. `ValueError: No objects to concatenate`

**The Error:**
```python
# Filtering our list to only huge dataframes (none match)
huge_dfs = [df for df in dfs_list if df['units_sold'].sum() > 1000000]

# Attempting to concatenate the empty list
pd.concat(huge_dfs)
```
```text
ValueError: No objects to concatenate
```

**Why it happens:**
`pd.concat()` requires at least one DataFrame or Series in the list/dictionary. If your filtering logic removes all items (or your directory read found zero files), pandas cannot deduce what columns or index to build.

**The Fix:**
Always check if the collection is empty before concatenating.
```python
if huge_dfs:
    master_huge_df = pd.concat(huge_dfs)
else:
    # Create empty DataFrame with expected schema
    master_huge_df = pd.DataFrame(columns=['date', 'store_id', 'units_sold'])
```

## Aggregating Across DataFrames


Often, you want to collect summary statistics from each separate dataset to compare them across groups.

```python
# Compute describe() for each DataFrame and collect into a dictionary
summaries = {month: mdf['units_sold'].describe() for month, mdf in dfs.items()}

# Concatenate these Series/DataFrames into a single comparison table
comparison_table = pd.concat(summaries, axis=1)
print(comparison_table)
```

Output:
```text
           january   february      march
count     3.000000   3.000000   3.000000
mean     18.333333  30.000000  29.666667
std      14.571662  16.000000  15.502688
min       8.000000  18.000000  14.000000
25%      10.000000  20.000000  22.000000
50%      12.000000  22.000000  30.000000
75%      23.500000  36.000000  37.500000
max      35.000000  50.000000  45.000000
```

Alternatively, you can use a key column added before concatenation to identify the source:
```python
# Alternative pattern: Using a key column added before concat to identify source
dfs_with_keys = []
for month, mdf in dfs.items():
    summary_df = mdf.groupby('category', observed=True)['units_sold'].sum().reset_index()
    summary_df['source_month'] = month  # Add key column to identify source
    dfs_with_keys.append(summary_df)

cross_df_summary = pd.concat(dfs_with_keys, ignore_index=True)
print(cross_df_summary.head(2))
```

Output:
```text
      category  units_sold source_month
0     Clothing          35      january
1  Electronics          20      january
```

## Full Running Example: The 12-Step Workflow

This is a capstone scenario putting everything together: loading data, systematically cleaning multiple datasets, summarizing them, performing cross-dataset comparisons, and exporting the final results.

```python
# 1. Load: (Using our `dfs` dict defined at the top)

# 2. Inspect each df:
for m, mdf in dfs.items():
    print(f"{m.upper()}: {mdf.shape[0]} rows")

# Build a pipeline function to handle steps 3-7 uniformly
def process_month(df):
    return (df
        # 3. Standardize column names
        .rename(columns=str.lower)
        # 4. Cast types consistently
        .assign(category=lambda x: x['category'].astype('category'))
        # 5. Handle missing values
        .fillna({'discount': 0.0})
        # 6. Add computed column
        .assign(revenue=lambda x: x['units_sold'] * x['unit_price'] * (1 - x['discount']))
        # 7. Filter rows (remove invalid negative sales)
        .query('units_sold > 0')
    )

# Apply pipeline uniformly
dfs_clean = {month: process_month(mdf) for month, mdf in dfs.items()}

# 8. GroupBy + Aggregate each df: Monthly revenue by category
category_summaries = {
    month: mdf.groupby('category', observed=True)['revenue'].sum() 
    for month, mdf in dfs_clean.items()
}

# 9. Collect summaries into a comparison table
revenue_comparison = pd.concat(category_summaries, axis=1).fillna(0)
print("\n--- Revenue Comparison ---")
print(revenue_comparison)

# 10. Concatenate into master df with month key
master_df = pd.concat(dfs_clean, names=['month', 'original_id']).reset_index('month')

# 11. Cross-month analysis on master df
total_revenue_by_month = master_df.groupby('month')['revenue'].sum()
print("\n--- Total Revenue by Month ---")
print(total_revenue_by_month)

# 12. Save outputs
# Save individual cleaned CSVs per df
# for month, mdf in dfs_clean.items():
#     mdf.to_csv(f'cleaned_{month}.csv', index=False)

# Save summary to Excel multi-sheet
# with pd.ExcelWriter('monthly_report.xlsx') as writer:
#     revenue_comparison.to_excel(writer, sheet_name='Category_Comparison')
#     total_revenue_by_month.to_excel(writer, sheet_name='Total_Revenue')

# Save master data to Parquet for fast analytics later
# master_df.to_parquet('master_sales_data.parquet')
```

## Related Chapters
- [[01-python-native-data-structures]]
- [[04-dataframe-creation-inspection]]
- [[06-modifying-dataframes]]
- [[09-aggregation-and-groupby]]
- [[11-combining-dataframes]]
- [[15-method-chaining-and-pipelines]]
- [[16-input-output]]
