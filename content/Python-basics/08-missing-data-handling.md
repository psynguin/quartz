---
title: "Missing Data Handling"
tags: [python, data-handling, pandas, numpy]
chapter: 8
theme: "Detecting, Removing, and Imputing Missing Data"
---

## Introduction

Missing data is an unavoidable reality in data science. Whether it stems from system errors, user omissions, or disjointed data sources, knowing how to detect, remove, and impute missing values is critical. Pandas represents missing data primarily using NumPy's `np.nan` (Not a Number) for numerical data, though you may also encounter `None` or `pd.NaT` (Not a Time). 

In this chapter, you will learn how to quantify missingness, safely discard incomplete records, and apply various imputation strategies to fill in the gaps.

Let's start by initializing our running dataset and explicitly introducing some missing values to work with so we can demonstrate these methods:

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

# Introduce missing data for demonstration
df.loc[[2, 5], 'units_sold'] = np.nan
df.loc[[1, 7], 'discount'] = np.nan
df.loc[8, 'category'] = np.nan

# Multi-DataFrame version (dict of monthly DataFrames — used in multi-df sections)
dfs = {
    'january': df[df['date'].dt.month == 1].reset_index(drop=True),
    'february': df[df['date'].dt.month == 2].reset_index(drop=True),
    'march': df[df['date'].dt.month == 3].reset_index(drop=True),
}
```

## Detecting Missing Data

Before deciding how to handle missing data, you must know where it exists and in what quantities. Real-world datasets often have hidden gaps, and finding them early prevents downstream calculations from producing silent `NaN` results.

Pandas provides several boolean methods for detecting missing values. Note that `.isnull()` and `.isna()` are functionally identical aliases, as are `.notnull()` and `.notna()`. The `.isna()` / `.notna()` variants are slightly preferred because they align with the naming conventions in other libraries (like R's `is.na()`).

### Syntactic Variants

```python
# Method 1: Element-wise boolean mask of missing values
df.isna()  # or df.isnull()

# Method 2: Element-wise boolean mask of non-missing values
df.notna() # or df.notnull()

# Method 3: Count of missing values per column
df.isna().sum()

# Method 4: Count of missing values per row
df.isna().sum(axis=1)

# Method 5: Check if ANY value in a column is missing
df.isna().any()

# Method 6: Check if ALL values in a column are missing
df.isna().all()
```

**When to Use:**
- Use `df.isna().sum()` to quickly assess data quality.
- Use `df.isna().any()` in automated checks to flag columns needing imputation.
- Use `df[df['col'].isna()]` to filter and inspect the specific rows where data is missing.

> [!TIP] Visualizing Missingness
> While tabular counts are useful, integrating the `missingno` library gives you a visual "heatmap" of where gaps occur in large datasets. `import missingno as msno; msno.matrix(df)` provides an immediate sense of missing data patterns.

### Worked Examples

Let's begin with a minimal reproducible example, and then progressively explore the missingness in our running dataset.

```python
# Minimal example
s_mini = pd.Series([1, np.nan, 3])
print(s_mini.isna())
# 0    False
# 1     True
# 2    False
# dtype: bool

# Realistic scenario using the running dataset
# 1. Tallying missing values per column
missing_counts = df.isna().sum()
print("Missing values per column:\n", missing_counts[missing_counts > 0])

# 2. Inspecting the exact rows with a missing 'discount'
missing_discount_rows = df[df['discount'].isna()]
print("\nRows with missing discount:\n", missing_discount_rows[['store_id', 'date', 'discount']])
```

### Common Errors and Gotchas

**Error:** Confusing string `"NaN"` or `"None"` with the actual floating-point `np.nan`.
- **Why it happens:** When loading data from a CSV or external system, literal string representations of nulls may be imported as normal text. The `.isna()` method only looks for the specialized float `np.nan` (or `None`/`pd.NaT`), not strings.
- **The fix:** Specify your null strings at load time using `pd.read_csv(..., na_values=['NaN', 'None', '??'])` or replace them before checking for missingness.

```python
df_bad = pd.DataFrame({'val': [1, "NaN", 3]})
print("Before fix:", df_bad['val'].isna().sum())  # Output: 0!

df_bad['val'] = df_bad['val'].replace("NaN", np.nan)
print("After fix:", df_bad['val'].isna().sum())   # Output: 1
```

## Dropping Missing Data

Sometimes, the safest approach is to simply drop rows or columns containing missing values. This is appropriate when the proportion of missing data is small, or when filling it might introduce unwanted bias (e.g., guessing a target variable in machine learning).

The `.dropna()` method gives you granular control over what gets discarded.

### Syntactic Variants

```python
# Method 1: Drop rows with ANY missing values (default behavior)
df.dropna()

# Method 2: Drop rows where ALL values are missing
df.dropna(how='all')

# Method 3: Drop rows requiring a minimum of k non-NaN values
df.dropna(thresh=5)

# Method 4: Drop rows ONLY if specific columns have missing values
df.dropna(subset=['units_sold', 'discount'])

# Method 5: Drop columns containing ANY missing values
df.dropna(axis=1)
```

**When to Use:**
- Use the default `.dropna()` when you have an abundance of data and missingness is rare.
- Use `subset=[...]` when certain columns are critical for your analysis, but you tolerate missingness in secondary columns.
- Use `thresh=k` when dealing with sparse datasets where some missingness is expected, but rows lacking sufficient data are useless.

### Worked Examples

```python
# Minimal example
df_mini = pd.DataFrame({'a': [1, np.nan], 'b': [np.nan, 2]})
# Drops all rows because both rows contain at least one NaN
print(df_mini.dropna()) 

# Realistic scenario using the running dataset
# We decide we can't calculate revenue without 'units_sold'. 
# We'll drop rows missing 'units_sold', but tolerate missing 'discount' or 'category'.
df_clean = df.dropna(subset=['units_sold'])

print(f"Original shape: {df.shape}")
print(f"Clean shape: {df_clean.shape}")
```

### Common Errors and Gotchas

**Error:** `KeyError: "['units_sold'] not found in axis"` when trying to use `subset` with `axis=1`.
- **Why it happens:** The `subset` parameter looks for row index labels when `axis=1` is specified, not column names. You cannot specify column names in `subset` while simultaneously dropping columns (`axis=1`).
- **The fix:** Only use `subset` with the default `axis=0` to check specific columns before dropping rows.

```python
try:
    df.dropna(axis=1, subset=['units_sold'])
except KeyError as e:
    print(f"KeyError: {e}")

# Correct usage (drops rows):
df.dropna(subset=['units_sold'])
```

> [!WARNING] Dropping Too Much
> Using `df.dropna()` without specifying a `subset` on a wide dataset can result in dropping almost all of your rows, because almost every row might have at least one missing value somewhere. Always check `df.shape` before and after dropping to ensure you haven't destroyed your dataset.

## Filling Missing Data

When dropping data is not viable, you must fill (`impute`) the missing values. Pandas supports constant filling, statistical imputation, positional filling (forward/backward), and mathematical interpolation.

### Syntactic Variants

#### Constant and Statistical Fill
```python
# Method 1: Fill all missing values in the DataFrame with a constant
df.fillna(0)

# Method 2: Fill a specific column with a constant string
df['category'] = df['category'].fillna('Unknown')

# Method 3: Fill with a statistical measure (e.g., column mean)
mean_units = df['units_sold'].mean()
df['units_sold'] = df['units_sold'].fillna(mean_units)

# Method 4: Per-column filling using a dictionary
fill_values = {
    'units_sold': df['units_sold'].median(),
    'discount': 0.0,
    'category': df['category'].mode()[0]  # mode() returns a Series
}
df.fillna(value=fill_values)
```

#### Forward and Backward Fill
Positional filling is ideal for time-series data or sorted records where an observation is likely identical to the one immediately preceding or following it. Starting in pandas 2.0, the direct `.ffill()` and `.bfill()` methods are preferred over `.fillna(method='ffill')`.

```python
# Method 1: Forward fill (propagate the last valid observation forward)
df.ffill() 

# Method 2: Backward fill (use the next valid observation to fill gaps)
df.bfill()

# Method 3: Limit propagation (only forward-fill up to 2 consecutive missing values)
df.ffill(limit=2)
```

#### Interpolation
Interpolation computes missing values based on the mathematical relationship between surrounding data points.

```python
# Method 1: Linear interpolation (assumes equal spacing between points)
df['units_sold'].interpolate(method='linear')

# Method 2: Time-based interpolation (adjusts for varying time gaps between rows)
df.set_index('date')['units_sold'].interpolate(method='time')

# Method 3: Polynomial interpolation
df['units_sold'].interpolate(method='polynomial', order=2)
```

**When to Use:**
- **Constants**: Use when missingness intrinsically means zero (e.g., missing discount means no discount).
- **Mean/Median**: Use for independent numeric observations. Median is more robust to outliers.
- **ffill / bfill**: Use for sorted time-series or state-based data where values persist until changed.
- **Interpolation**: Use for continuous time-series data where changes are gradual (e.g., temperature or gradual sales trends).

### Worked Examples

```python
# Minimal example
s_mini = pd.Series([1, np.nan, 3])
print(s_mini.fillna(2)) # Replaces NaN with 2

# Realistic scenario using the running dataset
clean_df = df.copy()

# 1. Fill missing discounts with 0.0
clean_df['discount'] = clean_df['discount'].fillna(0.0)

# 2. Fill missing category with the most frequent category (mode)
top_cat = clean_df['category'].mode()[0]
clean_df['category'] = clean_df['category'].fillna(top_cat)

# 3. For units sold, assume steady sales and forward-fill based on store
clean_df['units_sold'] = clean_df.groupby('store_id')['units_sold'].ffill()

# If the first record for a store was missing, ffill won't catch it. 
# Fallback to column median:
clean_df['units_sold'] = clean_df['units_sold'].fillna(clean_df['units_sold'].median())
```

### Common Errors and Gotchas

**Error:** `ValueError: time-weighted interpolation only works on Series or DataFrames with a DatetimeIndex`
- **Why it happens:** You attempted `method='time'` on a DataFrame with a default integer `RangeIndex`.
- **The fix:** Set the index to your datetime column first.
```python
# The fix:
df_time = df.set_index('date').interpolate(method='time')
```

**Gotcha:** Modifying a slice instead of the underlying DataFrame
- **Why it happens:** Doing `df[df['units_sold'].isna()]['units_sold'] = 0` triggers a `SettingWithCopyWarning` and usually doesn't modify the original `df`.
- **The fix:** Use `.fillna()` and reassign, or use `.loc`: 
```python
df.loc[df['units_sold'].isna(), 'units_sold'] = 0
```

## Replacing Specific Values

Sometimes data isn't missing (`NaN`), but instead contains placeholder values like `"??"`, `-999`, or `"N/A"` that need to be standardized or converted to `NaN` before proper imputation.

### Syntactic Variants

```python
# Method 1: Replace a specific scalar value
df.replace(-999, np.nan)

# Method 2: Replace multiple values with a single new value
df.replace(['??', 'Unknown', 'N/A'], np.nan)

# Method 3: Replace multiple values using a dictionary mapping
df.replace({
    '??': np.nan,
    0.0: 0.01  # e.g., applying a minimum discount
})

# Method 4: Column-specific replacement using a nested dictionary
df.replace({'category': {'Unknown': 'Other'}})

# Method 5: Replace using regex (e.g., removing non-numeric characters)
df['unit_price'].replace(r'[\$,]', '', regex=True)
```

**When to Use:**
Use `.replace()` to normalize messy input data into a clean set of recognized values or to explicitly convert placeholders to `np.nan` so `.fillna()` and `.dropna()` can process them correctly.

### Worked Examples

```python
# Minimal example
df_mini = pd.DataFrame({'val': [1, -999, 3]})
print(df_mini.replace(-999, np.nan))

# Realistic scenario using the running dataset
df_fixed = df.copy()

# 1. Correcting mislabeled categories
df_fixed['category'] = df_fixed['category'].replace('Food', 'Groceries')

# 2. Hypothetical cleaning of currency strings
# Imagine unit_price had been loaded as strings like "$299.99"
df_str = pd.DataFrame({'price': ['$299.99', '$49.99', '??']})

# Replace '??' with NaN, then strip the '$' symbol using regex
df_str['price'] = df_str['price'].replace('??', np.nan)
df_str['price'] = df_str['price'].replace(r'[\$]', '', regex=True).astype(float)
```

### Common Errors and Gotchas

**Gotcha:** Nothing changes when using `.replace()`
- **Why it happens:** `.replace()` looks for exact matches by default. If a cell contains `"?? "` (with a trailing space), `replace("??", np.nan)` won't match it.
- **The fix:** Use `regex=True` to match substrings, or strip whitespace from the column strings first.

```python
df_spaces = pd.DataFrame({'category': ['Electronics ', 'Clothing']})
# This does nothing
df_spaces['category'].replace('Electronics', 'Tech') 

# This works
df_spaces['category'] = df_spaces['category'].replace(r'Electronics.*', 'Tech', regex=True)
```

**Gotcha:** Replacing numbers vs. strings
- **Why it happens:** Replacing the string `'1'` will not replace the integer `1`. Data types must match exactly unless using regex.
- **The fix:** Ensure the type of the value you are replacing matches the column's dtype, or cast the column first.

## Applying to Multiple DataFrames

When working with a collection of DataFrames, you must apply missing data strategies consistently. A common pattern is applying a uniform fill strategy or using group-level statistics computed uniquely for each DataFrame.

### Uniform Fill Strategy

If you want to apply a constant or positional fill to all DataFrames simultaneously, you can use comprehensions.

```python
# The dict-based pattern: apply forward-fill to a dictionary of DataFrames
dfs_filled_dict = {month: data.ffill() for month, data in dfs.items()}

# The list-based pattern: apply forward-fill to a list of DataFrames
dfs_list = list(dfs.values())
dfs_filled_list = [data.ffill() for data in dfs_list]

# Dropping rows with missing target columns across all DataFrames
dfs_clean = {
    month: data.dropna(subset=['units_sold']) 
    for month, data in dfs.items()
}
```

### Imputation Using Per-DataFrame Statistics

Sometimes you want to impute using the localized mean of that specific DataFrame (e.g., filling January's missing sales with January's mean, not the annual mean).

```python
dfs_imputed = {}

for month, data in dfs.items():
    # Compute the mean units_sold for this specific month
    local_mean = data['units_sold'].mean()
    
    # Use .assign() to safely return a modified DataFrame
    dfs_imputed[month] = data.assign(
        units_sold=data['units_sold'].fillna(local_mean),
        discount=data['discount'].fillna(0) # constant fill for discount
    )
```

**Caveat:** When applying imputation across multiple DataFrames, ensure you are iterating over and modifying each piece of the collection, rather than attempting to compute a global mean before concatenation. Computing local means prevents data leakage (e.g., future months influencing past months).

## Related Chapters
- [[05-selecting-and-accessing-data]] — Filtering to inspect specific missing rows.
- [[07-data-types-and-conversion]] — Converting to nullable integer types to handle missing integers safely.
- [[09-aggregation-and-groupby]] — Using `transform` to impute missing values with group-specific means.
- [[12-multi-dataframe-workflows]] — More complex patterns for processing collections of DataFrames.
