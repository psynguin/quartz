---
title: "MultiIndex and Hierarchical Indexing"
tags: [python, data-handling, pandas, numpy]
chapter: 14
theme: "MultiIndex and Hierarchical Indexing"
---

# MultiIndex and Hierarchical Indexing

Hierarchical indexing—commonly referred to as a **MultiIndex** in pandas—allows you to work with higher-dimensional data using standard two-dimensional DataFrames. It lets you store and manipulate data with an arbitrary number of index levels (row labels) or column levels. 

In data science, hierarchical indexing arises naturally. Whenever you group data by multiple variables (e.g., summarizing sales by `region` and then by `store_id`), the resulting summary will have a MultiIndex. Mastering MultiIndex operations will enable you to navigate complex aggregations, reshape data fluidly, and avoid frustrating indexing errors.

> [!NOTE] Mental Model
> Think of a MultiIndex as a nested outline or an interactive tree directory. Just as a file lives inside a `folder/subfolder/` hierarchy, your data points live inside a `level_1/level_2/` hierarchy.

Let's initialize our running example dataset:

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

## Creating a MultiIndex

You can create a MultiIndex implicitly through operations like `set_index` and `.groupby()`, or explicitly using constructors from the `pd.MultiIndex` class.

### Method 1: Using `set_index()`
The most common way to create a MultiIndex row hierarchy from a flat DataFrame is by passing a list of columns to `.set_index()`.

```python
# Creates a 2-level MultiIndex using 'region' and 'store_id'
df_mi = df.set_index(['region', 'store_id'])
df_mi.head(3)
```

### Method 2: Via GroupBy Aggregation
When you group by multiple columns, pandas automatically returns a DataFrame with a MultiIndex containing the group keys.

```python
# Grouping by multiple columns
df_grouped = df.groupby(['region', 'store_id'])[['units_sold', 'unit_price']].mean()
```

### Method 3: Explicit Constructors
Sometimes you need to construct a MultiIndex directly, especially when building DataFrames from scratch.

```python
# From Arrays: pass a list of arrays (one per level)
arrays = [['North', 'North', 'South'], ['S01', 'S02', 'S01']]
mi_arrays = pd.MultiIndex.from_arrays(arrays, names=['region', 'store_id'])

# From Tuples: pass a list of tuples (one tuple per row)
tuples = [('North', 'S01'), ('North', 'S02'), ('South', 'S01')]
mi_tuples = pd.MultiIndex.from_tuples(tuples, names=['region', 'store_id'])

# From Product: Cartesian product of sets of labels
regions = ['North', 'South', 'East']
stores = ['S01', 'S02', 'S03']
mi_product = pd.MultiIndex.from_product([regions, stores], names=['region', 'store_id'])
```

**When to Use:**
- Use **`set_index()`** or **`groupby()`** when transforming an existing, populated flat dataset.
- Use **`from_product()`** when you need to generate every possible combination of categories (useful for detecting missing combinations or "filling out" a matrix).
- Use **`from_arrays()`** or **`from_tuples()`** when piecing together an index for a dynamically built DataFrame.

---

## Accessing MultiIndex Data

Selecting data inside a MultiIndex demands precision. First, let's create a solid hierarchical DataFrame to query. 

```python
# Create a summarized MultiIndex DataFrame
df_summary = df.groupby(['region', 'store_id']).sum(numeric_only=True)
```

### Common Error: `UnsortedIndexError`

A frequent mistake when slicing a MultiIndex is forgetting to sort it first.

```python
# Let's create an unsorted DataFrame intentionally
df_unsorted = pd.DataFrame(
    {'val': [1, 2, 3]}, 
    index=pd.MultiIndex.from_tuples([('South', 'S01'), ('North', 'S02'), ('East', 'S03')], names=['region', 'store'])
)

# ❌ THIS WILL RAISE AN ERROR
# df_unsorted.loc['East':'North']
# Raises: UnsortedIndexError: 'Key length (1) was greater than MultiIndex lexsort depth (0)'
```

**Why it happens:** Slicing relies on labels being contiguous in memory. If the categories are scattered (e.g., 'South' comes before 'East' and 'North'), pandas cannot safely return a range.

**The Fix:** Always apply `.sort_index()` before slicing.

```python
# ✅ The Fix
df_sorted = df_unsorted.sort_index()
df_sorted.loc['East':'North']
```

Let's apply this fix to our `df_summary` dataset before continuing with our examples:
```python
df_summary = df_summary.sort_index()
```

### Method 1: Tuple-based Slicing with `.loc`
The simplest method to access a specific nested location is to pass a tuple into `.loc[]`.

```python
# Select a single inner row by providing a tuple of labels
row = df_summary.loc[('North', 'S01')]

# Select multiple explicit inner rows
rows = df_summary.loc[[('North', 'S01'), ('South', 'S02')]]

# Select row and column simultaneously using a tuple for the row
units = df_summary.loc[('North', 'S01'), 'units_sold']
```

### Method 2: `pd.IndexSlice`
When you need to slice *across* levels—for example, getting all regions but only store `S01`—tuples become messy. `pd.IndexSlice` is a cleaner, dedicated tool for this.

```python
# Initialize the index slicer object
idx = pd.IndexSlice

# Syntax: df.loc[idx[level_0_slice, level_1_slice], column_slice]
# Example: All regions, but ONLY store 'S01', for all columns
df_s01_only = df_summary.loc[idx[:, 'S01'], :]

# Example: Regions from 'East' to 'North', all stores, specific column
df_subset = df_summary.loc[idx['East':'North', :], 'units_sold']
```

### Method 3: Cross-Section with `.xs()`
`.xs()` is a highly readable way to grab data from a single value at a specific level. Unlike `.loc`, it automatically drops the level you searched on.

```python
# Get all 'S01' stores, regardless of region
# Note that 'store_id' is dropped from the resulting index
df_xs = df_summary.xs('S01', level='store_id')
```

### Method 4: Swapping and Sorting
Sometimes it is easier to swap the levels of the MultiIndex so that your target level is on the outside (level 0).

```python
# Swaps 'region' and 'store_id', then sorts to make it sliceable
df_swapped = df_summary.swaplevel('region', 'store_id').sort_index()
df_swapped.loc['S01']
```

### Access Method Comparison

| Method | Syntax Readability | Slicing Capabilities | Best For |
|--------|--------------------|----------------------|----------|
| `tuple` | High | Poor | Exact lookups where you know the full hierarchy. |
| `pd.IndexSlice` | Medium | Excellent | Complex, multi-level slicing or partial selections. |
| `.xs()` | High | None (exact match only) | Grabbing a cross-section of a specific level easily. |

---

## MultiIndex Columns

Hierarchies are not just for rows; columns can have them too. They most commonly appear after grouped aggregations with multiple functions.

```python
# Applying multiple aggregation functions creates MultiIndex columns
df_col_mi = df.groupby('store_id').agg({
    'units_sold': ['sum', 'mean'],
    'unit_price': ['max']
})

df_col_mi.head()
```
*Visualizing the structure:*
```text
           units_sold         unit_price
                  sum   mean         max
store_id                                
S01                34  11.33      299.99
S02                87  29.00      149.99
S03                95  47.50        9.99
```

### Accessing MultiIndex Columns

```python
# Accessing the top-level column returns a DataFrame containing all sub-levels
df_units = df_col_mi['units_sold']

# Accessing a specific sub-level requires a tuple
mean_units = df_col_mi[('units_sold', 'mean')]
```

### Flattening MultiIndex Columns
MultiIndex columns can complicate subsequent analysis (especially when exporting to CSV or machine learning pipelines). It is standard practice to "flatten" them into a single-level string index.

```python
# Method 1: String join with list comprehension (Preferred)
# Joins level 0 ('units_sold') and level 1 ('sum') into 'units_sold_sum'
df_flat = df_col_mi.copy()
df_flat.columns = ['_'.join(col).strip() for col in df_flat.columns.values]

# Method 2: Dropping the top level entirely
# Useful if the top level is redundant
df_drop = df_col_mi.copy()
df_drop.columns = df_drop.columns.droplevel(0) 
```

> [!TIP] The `droplevel` Approach
> Only use `.droplevel(0, axis=1)` if you are absolutely sure that the remaining sub-level names won't conflict (e.g., if you only aggregated a single column). Otherwise, joining the levels with `'_'.join()` is much safer.

---

## Round-Trip: Stack, Unstack, and Reset

The `.stack()` and `.unstack()` methods move index levels between rows and columns. They are the primary tools for pivoting hierarchical data.

*Mental model:*
- **Stacking** compresses columns down into rows, making the DataFrame longer and narrower.
- **Unstacking** pushes row indices up into columns, making the DataFrame shorter and wider.

```python
# Start with our grouped MultiIndex data
df_summary = df.groupby(['region', 'store_id'])['units_sold'].sum()

# UNSTACK: Moves 'store_id' from the row index to the column index
df_unstacked = df_summary.unstack(level='store_id')

# STACK: Reverses the operation, moving the columns back into the row index
df_stacked = df_unstacked.stack()

# RESET_INDEX: Converts all index levels back into standard columns
df_reset = df_summary.reset_index()
```

---

### Applying to Multiple DataFrames

When working with collections of DataFrames, MultiIndex results frequently appear inside your processing loops. A common workflow is performing a grouped aggregation across all DataFrames, flattening the resulting hierarchical columns, and saving the clean tables.

Because MultiIndex creation and flattening involves multiple steps, it's best to encapsulate the logic in a custom pipeline function before applying it in bulk:

```python
def summarize_and_flatten(df_in):
    # 1. Group by multiple categories and aggregate
    mi_df = df_in.groupby(['region', 'category']).agg({
        'units_sold': ['sum', 'count'],
        'unit_price': ['mean']
    })
    
    # 2. Flatten the MultiIndex columns
    mi_df.columns = [
        f"{col[0]}_{col[1]}" if col[1] else col[0] 
        for col in mi_df.columns.values
    ]
    
    # 3. Reset the row MultiIndex to turn 'region' and 'category' into regular columns
    return mi_df.reset_index()
```

Now we can apply this function uniformly across our collections using either dictionary or list comprehensions.

**Pattern 1: Dictionary comprehension**
```python
monthly_summaries = {k: summarize_and_flatten(month_df) for k, month_df in dfs.items()}
```

**Pattern 2: List comprehension**
```python
# Assuming we have a list of DataFrames instead of a dictionary
dfs_list = list(dfs.values())
summaries_list = [summarize_and_flatten(month_df) for month_df in dfs_list]
```

**Caveats for Bulk MultiIndex Operations:**
- **Inconsistent Levels:** If your DataFrames have different columns or data types, `.agg()` might silently drop un-aggregatable columns in some DataFrames but not others. This results in inconsistent MultiIndex structures across your collection, which will break subsequent concatenation attempts.
- **Empty Groups:** If a DataFrame is missing a specific group combination (e.g., no "Food" sales in February), the resulting MultiIndex for that month will lack that row. Always use `pd.concat()` after flattening or carefully `.reindex()` your aggregated results to a common MultiIndex to ensure structural uniformity.

---

## Related Chapters
- To see how grouping creates these MultiIndexes, review [[09-aggregation-and-groupby]].
- To learn other ways to pivot and reshape flat data, see [[10-reshaping-dataframes]].
- To learn how `pd.concat` can generate MultiIndexes from collections of DataFrames, check [[11-combining-dataframes]].
- For applying operations on flattened DataFrames across dictionaries, see [[12-multi-dataframe-workflows]].
