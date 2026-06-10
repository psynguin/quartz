---
title: "Adding, Modifying, Renaming, and Removing Data"
tags: [python, data-handling, pandas, numpy]
chapter: 6
theme: "Adding, Modifying, Renaming, and Removing Data"
---

# Adding, Modifying, Renaming, and Removing Data

Data rarely comes exactly as you need it. You will often need to compute new metrics, remove irrelevant features, rename columns for clarity, and reorder data for presentation. Mastering these operations allows you to transform raw, messy data into a clean structure ready for analysis.

> [!NOTE] In-Place vs. Copy
> Understanding the difference between operations that modify a DataFrame *in-place* versus those that return a *new copy* is critical. We strongly recommend the functional approach (returning copies) as it prevents accidental data corruption and fits perfectly into method chaining pipelines.

## Running Dataset

We will use the following monthly sales dataset throughout this chapter. This dataset serves as our baseline.

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

## Adding and Modifying Columns

Adding new columns or modifying existing ones is a daily task in data analysis. Pandas offers several ways to achieve this, each with specific trade-offs.

### Progressive Example: Computing Metrics

Let's start by calculating a simple revenue column and build up to a categorized output.

**Method 1: Direct Assignment**

The most straightforward way to add or overwrite a column is using bracket notation.

```python
# Create a copy so we do not mutate the baseline 'df'
df_demo = df.copy()

# Minimal: Compute a new column
df_demo['revenue'] = df_demo['units_sold'] * df_demo['unit_price']

# Overwrite an existing column (e.g., convert discount to percentage)
df_demo['discount'] = df_demo['discount'] * 100
```

**Method 2: The `.assign()` Method**

The `.assign()` method returns a **new** DataFrame with the requested columns added or modified. It is highly recommended because it supports method chaining.

```python
# Realistic scenario: Compute revenue and net revenue in one pass using lambdas
# Lambdas refer to the DataFrame at that exact step in the chain
df_computed = df.assign(
    revenue=lambda x: x['units_sold'] * x['unit_price'],
    net_revenue=lambda x: x['revenue'] * (1 - x['discount'])
)
```

**Method 3: Conditional Columns with `np.where`**

When you need to create a column based on a condition (like an IF-ELSE statement), `np.where` from NumPy is the most performant approach.

```python
# Flag high volume sales
df_computed['high_volume'] = np.where(df_computed['units_sold'] > 20, 'Yes', 'No')
```

**Method 4: Binning with `pd.cut` and `pd.qcut`**

Sometimes you need to convert continuous numeric data into discrete categories or "bins". 
- `pd.cut()` creates bins based on **exact value ranges**.
- `pd.qcut()` creates bins based on **quantiles** (ensuring roughly equal number of items per bin).

```python
# Binning by specific boundaries (0-50, 50-200, 200-1000)
df_computed['price_tier'] = pd.cut(
    df_computed['unit_price'],
    bins=[0, 50, 200, 1000],
    labels=['Low', 'Medium', 'High']
)

# Binning into equal-sized quantiles (e.g., 2 bins = median split)
df_computed['price_quantile'] = pd.qcut(df_computed['unit_price'], q=2, labels=['Bottom 50%', 'Top 50%'])
```

**Method 5: One-Hot Encoding with `pd.get_dummies`**

When preparing data for machine learning, categorical variables often need to be converted into binary indicator columns.

```python
# Create indicator columns for the 'region' category
df_encoded = pd.get_dummies(df_computed, columns=['region'], prefix='loc')
```

### Common Errors and Gotchas

**SettingWithCopyWarning**

When you use direct assignment on a subset of data (a slice), pandas throws a warning because it's ambiguous whether you are modifying a copy or the original data.

```python
# The Error Trigger:
df_north = df[df['region'] == 'North']
df_north['new_col'] = 1 
# SettingWithCopyWarning: A value is trying to be set on a copy of a slice from a DataFrame.
```

**The Fix:** Use `.copy()` when creating the subset if you intend to modify it.

```python
df_north = df[df['region'] == 'North'].copy()
df_north['new_col'] = 1  # No warning
```

### When to Use

| Method | Returns | Best For |
|--------|---------|----------|
| Direct `df['col'] =` | Modifies existing DataFrame | Quick exploratory scripts, single assignments |
| `.assign()` | New DataFrame | Data pipelines, method chaining, functional programming |
| `np.where()` | Array / Column | Fast conditional logic (IF-ELSE logic) |
| `pd.cut()` / `pd.qcut()` | Categorical Column | Converting continuous numbers to discrete categories |
| `pd.get_dummies()` | New DataFrame | Machine learning data preparation |

### Applying to Multiple DataFrames

When working with collections of DataFrames—such as our `dfs` dictionary of monthly files—you can apply column additions uniformly using a dictionary comprehension.

```python
# Add the computed revenue column to all DataFrames in the dictionary
dfs_computed = {
    month: data.assign(revenue=lambda x: x['units_sold'] * x['unit_price'])
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list = [dfs['january'], dfs['february'], dfs['march']]
dfs_list_computed = [
    data.assign(revenue=lambda x: x['units_sold'] * x['unit_price']) 
    for data in dfs_list
]
```

---

## Removing Data

Cleaning data inevitably involves dropping columns you don't need or removing rows that are invalid. 

### Progressive Example: Cleaning the Dataset

**Method 1: `.drop(columns=[...])`**

The safest and most explicit way to remove columns. By default, it returns a new DataFrame.

```python
# Minimal: Drop a single column
df_clean = df.drop(columns=['discount'])

# Realistic: Drop multiple columns using the axis parameter syntax
df_clean = df.drop(['discount', 'category'], axis=1)
```

**Method 2: `.drop(index=[...])`**

To remove specific rows by their index label.

```python
# Drop specific rows using their index labels
df_clean = df_clean.drop(index=[0, 1, 2])
```

**Method 3: The `del` Statement**

The native Python `del` statement removes a column directly from memory in-place. Use this with caution.

```python
df_demo = df.copy()

# Mutates the DataFrame immediately
del df_demo['region']
```

**In-Place vs. Returning a Copy**

Many pandas methods accept an `inplace=True` argument.

```python
# Functional approach: returns a new DataFrame, original is unchanged
df_new = df.drop(columns=['discount'])

# Mutational approach: modifies the original DataFrame, returns None
df_demo = df.copy()
df_demo.drop(columns=['discount'], inplace=True)
```

> [!TIP] Best Practice
> The pandas core team strongly discourages using `inplace=True`. It rarely saves memory (under the hood, pandas often makes a copy anyway), and it breaks method chaining. Stick to the functional style: `df = df.drop(...)`.

### Common Errors and Gotchas

**KeyError when Dropping Non-Existent Columns**

```python
# The Error Trigger:
# df.drop(columns=['non_existent_column'])
# KeyError: "['non_existent_column'] not found in axis"
```

**The Fix:** Pass `errors='ignore'` to safely attempt dropping columns that might not exist. This is especially useful in loops.

```python
df.drop(columns=['non_existent_column'], errors='ignore')  # Succeeds silently
```

### When to Use

| Method | Axis | In-Place? | Best For |
|--------|------|-----------|----------|
| `.drop(columns=...)` | Columns | No (unless `inplace=True`) | Functional data pipelines |
| `.drop(index=...)` | Rows | No (unless `inplace=True`) | Removing specific row labels |
| `del df['col']` | Columns | Yes (Always) | Immediate memory cleanup of large columns |

### Applying to Multiple DataFrames

To drop columns across multiple DataFrames safely, always use `errors='ignore'` to prevent the loop from crashing if a column is missing in one of the months.

```python
# Drop the discount column from all DataFrames in the dictionary
dfs_dropped = {
    month: data.drop(columns=['discount'], errors='ignore')
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list_dropped = [
    data.drop(columns=['discount'], errors='ignore') 
    for data in dfs_list
]
```

---

## Renaming Columns and Axes

Consistently named columns make your code easier to read. 

### Progressive Example: Standardizing Column Names

**Method 1: `.rename(columns={...})`**

Pass a dictionary mapping old names to new names.

```python
# Minimal: Rename a single column
df_renamed = df.rename(columns={'store_id': 'Store'})
```

**Method 2: Function-based Rename**

You can pass a function to transform all column names at once.

```python
# Realistic: Standardize all column names to uppercase
df_upper = df.rename(columns=str.upper)

# Or clean up messy names with a custom lambda
df_clean_names = df.rename(columns=lambda x: x.strip().replace('_', ' ').title())
```

**Method 3: `.set_axis()`**

To completely replace all column names at once without mapping.

```python
new_names = ['Date', 'Store', 'Category', 'Quantity', 'Price', 'Discount', 'Region']
df_renamed_all = df.set_axis(new_names, axis=1)
```

**Method 4: Direct `.columns` Assignment**

You can overwrite the columns attribute directly.

```python
df_demo = df.copy()
df_demo.columns = ['Date', 'Store', 'Category', 'Quantity', 'Price', 'Discount', 'Region']
```

**Method 5: `.rename_axis()`**

Sometimes the index or columns have a "name" attribute that needs renaming.

```python
# Name the row index and column axis
df_axis_named = df.rename_axis('transaction_id').rename_axis('features', axis=1)
```

### Common Errors and Gotchas

**ValueError with `.set_axis()` Mismatch**

```python
# The Error Trigger:
# Dataset has 7 columns, but we provide only 6 names
# df.set_axis(['Date', 'Store', 'Category', 'Quantity', 'Price', 'Discount'], axis=1)
# ValueError: Length mismatch: Expected axis has 7 elements, new values have 6 elements
```

**The Fix:** Ensure the list of new names perfectly matches the number of existing columns, or use `.rename(columns={...})` if you only want to rename specific ones.

### When to Use

| Method | Scope | Best For |
|--------|-------|----------|
| `.rename(columns=dict)` | Specific columns | Explicit renaming of a few targeted columns |
| `.rename(columns=func)` | All columns | Algorithmic standardization (e.g., lowercase all) |
| `.set_axis(list)` | All columns | Wholesale replacement of every column name |
| `.rename_axis(name)` | Axis metadata | Naming the index for later MultiIndex/groupby use |

### Applying to Multiple DataFrames

Applying standardized naming conventions across all files before merging them is critical.

```python
# Apply string lowercase to all column names in the dictionary
dfs_renamed = {
    month: data.rename(columns=str.lower)
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list_renamed = [
    data.rename(columns=str.lower) 
    for data in dfs_list
]
```

---

## Reordering Columns

Often, you want your most important identifiers (like dates and IDs) at the front of your DataFrame.

### Progressive Example: Organizing Output

**Method 1: Manual Reorder with Bracket Notation**

Pass a list of all column names in the exact order you want them.

```python
# Minimal: Bring primary identifiers to the front
df_reordered = df[['date', 'store_id', 'region', 'category', 'units_sold', 'unit_price', 'discount']]
```

**Method 2: `.reindex(columns=[...])`**

Reorder columns while allowing for missing columns to be filled with `NaN` rather than throwing an error.

```python
# Realistic: Ensure output strictly conforms to a target schema
target_schema = ['date', 'store_id', 'target_sales', 'units_sold']
df_schema_aligned = df.reindex(columns=target_schema)
```

**Method 3: Alphabetical Sort**

```python
# Sort columns alphabetically
df_sorted_cols = df.sort_index(axis=1)
```

### Common Errors and Gotchas

**KeyError in Bracket Reordering**

```python
# The Error Trigger:
# df[['date', 'non_existent_column']]
# KeyError: "['non_existent_column'] not in index"
```

**The Fix:** If you aren't absolutely sure a column exists, use `.reindex(columns=...)` instead, which safely inserts `NaN`s for missing columns without crashing.

### When to Use

| Method | Missing Columns Handling | Best For |
|--------|--------------------------|----------|
| `df[[cols]]` | Raises KeyError | Strict, manual reordering where exact columns are known |
| `.reindex()` | Fills with NaN | Conforming a DataFrame to a rigid external schema |
| `.sort_index(axis=1)` | N/A | Cleaning up wide datasets alphabetically |

### Applying to Multiple DataFrames

To strictly enforce column ordering across many files:

```python
target_order = ['date', 'store_id', 'region', 'category', 'units_sold', 'unit_price', 'discount']

dfs_reordered = {
    month: data.reindex(columns=target_order)
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list_reordered = [
    data.reindex(columns=target_order) 
    for data in dfs_list
]
```

---

## Index Operations

The row index in pandas is powerful, acting as a natural alignment key and lookup mechanism. 

### Progressive Example: Setting and Realigning Time Series

**Method 1: `.set_index()`**

Moves one or more columns out of the data space and into the index.

```python
# Minimal: Set date as index
df_indexed = df.set_index('date')

# Create a MultiIndex
df_multi = df.set_index(['region', 'store_id'])
```

**Method 2: `.reindex(new_index)`**

Aligns the DataFrame to a completely new set of index labels. If a label exists in the new index but not the original DataFrame, the row will be filled with `NaN`.

```python
# Create a DataFrame with unique dates to safely reindex
df_unique_dates = df.drop_duplicates(subset=['date']).set_index('date')
expected_dates = pd.date_range(start='2024-01-05', end='2024-01-12')

# Align the dataframe to this new index (missing dates will have NaN)
df_aligned = df_unique_dates.reindex(expected_dates)
```

**Method 3: `.reset_index()`**

Restores the index back to a regular column and creates a new default numeric `RangeIndex`.

```python
# Move index back to columns after time-series operations
df_reset = df_aligned.reset_index()

# Discard the old index entirely without adding it as a column
df_dropped = df_unique_dates.reset_index(drop=True)
```

### Common Errors and Gotchas

**ValueError on Duplicate Indices during Reindex**

```python
# The Error Trigger:
# df_indexed has duplicate dates in the index (e.g. '2024-01-05' appears multiple times)
# df_indexed.reindex(expected_dates)
# ValueError: cannot reindex on an axis with duplicate labels
```

**The Fix:** Ensure the index is strictly unique before reindexing, typically by aggregating the data first (e.g., using `.groupby('date').sum()`).

### When to Use

| Method | Purpose | Best For |
|--------|---------|----------|
| `.set_index()` | Column -> Index | Preparing for time-series analysis or fast `.loc` lookups |
| `.reset_index()` | Index -> Column | Flattening data before saving to CSV or SQL |
| `.reindex()` | Align Index | Filling gaps in time series or aligning two DataFrames |

### Applying to Multiple DataFrames

If you change the index inside a bulk pipeline, remember that downstream steps (like horizontal concatenation) will align on this new index. If you don't want this, reset the index before finishing the loop.

```python
# Set the 'date' column as the index for all datasets
dfs_indexed = {
    month: data.set_index('date')
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list_indexed = [
    data.set_index('date') 
    for data in dfs_list
]
```

---

## Sorting Data

Sorting is essential for time-series data or identifying top performers.

### Progressive Example: Finding Top Performers

**Method 1: `.sort_values()`**

Sort by one or more columns.

```python
# Minimal: Sort ascending by default
df_sorted = df.sort_values(by='units_sold')
```

**Method 2: Multiple Columns with Mixed Ascending**

You can sort by multiple columns and define the direction for each independently.

```python
# Realistic: Sort by region (A-Z), then by highest units sold (High-Low)
df_multi_sort = df.sort_values(
    by=['region', 'units_sold'], 
    ascending=[True, False]
)
```

**Method 3: Handling Missing Values (`na_position`)**

By default, `NaN` values are sent to the bottom of the sorted DataFrame. You can change this behavior.

```python
df_demo = df.copy()
# Intentionally add a NaN for demonstration
df_demo.loc[0, 'units_sold'] = np.nan

# Force NaNs to appear at the top
df_nan_first = df_demo.sort_values(by='units_sold', na_position='first')
```

**Method 4: Sorting by Row Index with `.sort_index()`**

When a DataFrame has been grouped or has had a new index assigned, sorting the index ensures that row lookups remain fast and chronological order is maintained.

```python
# Create a scrambled index scenario
df_shuffled = df.sample(frac=1, random_state=42)
df_indexed_shuffled = df_shuffled.set_index('date')

# Sort by the row index to restore chronological order
df_chronological = df_indexed_shuffled.sort_index()

# Sort descending
df_recent_first = df_indexed_shuffled.sort_index(ascending=False)
```

### Common Errors and Gotchas

**ValueError when Mismatching Sorting Arrays**

```python
# The Error Trigger:
# df.sort_values(by=['region', 'units_sold'], ascending=[True])
# ValueError: Length of ascending (1) != length of by (2)
```

**The Fix:** Ensure the `ascending` list has the exact same number of boolean values as the `by` list.

```python
df.sort_values(by=['region', 'units_sold'], ascending=[True, False])
```

### When to Use

| Method / Param | Purpose | Best For |
|----------------|---------|----------|
| `.sort_values()` | Sort by data | Ranking, finding top/bottom records |
| `.sort_index()` | Sort by index | Reordering after a groupby or set_index to enable fast slicing |
| `na_position` | Missing data placement | Controlling whether incomplete records appear first or last |

### Applying to Multiple DataFrames

Sorting individually across multiple DataFrames changes their row alignment.

```python
dfs_sorted = {
    month: data.sort_values(by='units_sold', ascending=False)
    for month, data in dfs.items()
}

# The list-based equivalent:
dfs_list_sorted = [
    data.sort_values(by='units_sold', ascending=False) 
    for data in dfs_list
]
```

#### Caveats When Applying in Bulk
- **Data-Dependent Sorting:** If you sort each DataFrame independently by a computed column, be aware that the row orders will no longer match across DataFrames. This matters if you plan to combine them horizontally later.
- **Index Misalignment:** If you use `.set_index()` or `.sort_values()` inside the loop, consider appending `.reset_index(drop=True)` at the end of the chain if your downstream operations expect a clean numeric index.

---

## Related Chapters
- [[05-selecting-and-accessing-data]]
- [[07-data-types-and-conversion]]
- [[12-multi-dataframe-workflows]]
- [[15-method-chaining-and-pipelines]]
