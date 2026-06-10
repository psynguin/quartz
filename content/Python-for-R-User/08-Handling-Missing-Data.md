# Chapter 8: Handling Missing Data

Handling missing values is an inevitable part of data analysis. In R, missing data is typically represented by `NA`, and the `tidyr` package provides a rich set of functions like `drop_na()`, `replace_na()`, `fill()`, and `complete()` to manage them. 

In Python, pandas usually represents missing data with `NaN` (Not a Number) from NumPy, or `<NA>` in the newer nullable data types. The pandas library offers robust methods to drop, fill, and manipulate missing data: `.dropna()`, `.fillna()`, `.ffill()`, and `.bfill()`.

> [!WARNING] Data Type Coercion (`NaN` as Float)
> Traditional pandas uses `np.nan` to represent missing data, which is a floating-point value. If you have an integer column and introduce a missing value, pandas will silently convert the entire column's data type to `float`. This can be baffling for R users accustomed to type-specific missing values like `NA_integer_`. 
> 
> The modern solution in pandas is to use the newer nullable integer data type (e.g., specifying `dtype="Int64"` instead of `"int64"`), which uses `<NA>` and preserves the integer type even when missing values are present.

Let's look at how to translate `tidyr` missing data operations into `pandas`.

## Dropping Missing Values: `drop_na()` vs `.dropna()`

### The R Way
In R, you use `drop_na()` from `tidyr` to remove rows containing missing values.
```R
# Drop rows where ANY column has an NA
df %>% drop_na()

# Drop rows where specific columns have NA
df %>% drop_na(col1, col2)
```

### The Python Way
In pandas, the equivalent is `.dropna()`. By default, it drops any row containing at least one `NaN`.

```python
import pandas as pd
import numpy as np

# Sample Data
df = pd.DataFrame({
    'A': [1, 2, np.nan, 4],
    'B': [np.nan, 2, 3, 4],
    'C': ['a', 'b', 'c', np.nan]
})

# Drop rows where ANY column has a NaN (Equivalent to drop_na())
df_clean = df.dropna()

# Drop rows where SPECIFIC columns have a NaN (Equivalent to drop_na(A, B))
df_subset_clean = df.dropna(subset=['A', 'B'])

# Other useful alternatives in pandas:
# Drop columns instead of rows
df_cols_clean = df.dropna(axis=1)

# Drop rows where ALL columns are NaN
df_all_clean = df.dropna(how='all')

# Keep rows with at least 'thresh' non-NA values
df_thresh = df.dropna(thresh=2)

# Method chaining example
df_chained = (
    df
    .dropna(subset=['A'])
    .reset_index(drop=True)
)
```

## Replacing Missing Values: `replace_na()` vs `.fillna()`

### The R Way
In R, `replace_na()` is used to replace `NA` values with specified values, often done within a `mutate()` or for the whole dataframe.
```R
# Replace NA in specific columns
df %>% replace_na(list(A = 0, C = "Unknown"))
```

### The Python Way
In pandas, `.fillna()` is the primary tool for this. You can apply it to the entire DataFrame, to specific columns using a dictionary, or to individual Series.

```python
# 1. Using a dictionary to fill specific columns (Direct analogue to replace_na with list)
df_filled = df.fillna({'A': 0, 'C': 'Unknown'})

# 2. Filling a specific column (Series)
df['A'] = df['A'].fillna(0)

# 3. Filling the entire DataFrame with a single value
df_filled_all = df.fillna(0)

# 3b. Using .replace() as a frequent alternative to .fillna()
df_replaced = df.replace({np.nan: 0})

# 4. Using numpy.where() as an alternative (similar to if_else/replace_na combo)
df['A'] = np.where(df['A'].isna(), 0, df['A'])
```

## Forward and Backward Filling: `fill()` vs `.ffill()` / `.bfill()`

### The R Way
In R, `fill()` from `tidyr` handles forward-filling (carrying the last observation forward) or backward-filling (carrying the next observation backward).
```R
# Forward fill
df %>% fill(A, .direction = "down")

# Backward fill
df %>% fill(A, .direction = "up")
```

### The Python Way
Pandas provides dedicated methods `.ffill()` (forward fill) and `.bfill()` (backward fill). Previously, these were arguments within `.fillna(method='ffill')`, but the standalone methods are now preferred.

```python
df_fill = pd.DataFrame({
    'Time': [1, 2, 3, 4, 5],
    'Value': [10, np.nan, np.nan, 40, np.nan]
})

# Forward fill for specific column (Equivalent to fill(.direction = "down"))
df_fill['Value'] = df_fill['Value'].ffill()

# Backward fill for specific column (Equivalent to fill(.direction = "up"))
df_fill['Value'] = df_fill['Value'].bfill()

# Apply to the entire DataFrame
df_ffilled = df_fill.ffill()

# Limit the number of consecutive NaNs to fill
# Useful if you don't want to carry an observation too far forward
df_limited_fill = df_fill.ffill(limit=1)
```

## Interpolating Missing Data

### The R Way
In R, you might use packages like `zoo` (e.g., `na.approx()`) or `imputeTS` to interpolate missing values in time series or continuous data.

### The Python Way
Pandas has robust built-in interpolation capabilities using `.interpolate()`, which is very commonly used for time series data to fill `NaN`s.

```python
# Linearly interpolate missing values
df_fill['Value_interp'] = df_fill['Value'].interpolate(method='linear')

# Many other methods are available, such as polynomial, spline, or time-based 
# interpolation if the dataframe has a datetime index.
# df.interpolate(method='time')
```

## Making Implicit Missing Values Explicit: `complete()`

### The R Way
In R, `complete()` takes a set of columns, finds all unique combinations of their values, and ensures all those combinations exist in the dataframe, filling in `NA` for other columns.
```R
df %>% complete(Group, Time)
```

### The Python Way
Pandas doesn't have a single direct equivalent to `complete()`, but you can achieve the same result by creating a MultiIndex from the unique combinations and reindexing, or by merging with a cross-join.

**Method 1: Reindexing with a MultiIndex (More pandas-idiomatic)**

```python
df_incomplete = pd.DataFrame({
    'Group': ['A', 'A', 'B'],
    'Time': [1, 2, 1],
    'Value': [10, 20, 30]
})

# Create all combinations of Group and Time
idx = pd.MultiIndex.from_product(
    [df_incomplete['Group'].unique(), df_incomplete['Time'].unique()], 
    names=['Group', 'Time']
)

# Set the current index to the columns we want to complete, then reindex and reset
df_complete = (
    df_incomplete
    .set_index(['Group', 'Time'])
    .reindex(idx)
    .reset_index()
)
```

**Method 2: Cross Merge / Cartesian Product**

```python
# Create unique dataframes for each column
groups = pd.DataFrame({'Group': df_incomplete['Group'].unique()})
times = pd.DataFrame({'Time': df_incomplete['Time'].unique()})

# Cross merge creates all combinations
all_combinations = groups.merge(times, how='cross')

# Left merge back the original data
df_complete_merge = all_combinations.merge(df_incomplete, on=['Group', 'Time'], how='left')
```

## Checking for Missing Values

### The R Way
```R
# Check if values are NA
is.na(df)

# Count NAs per column
colSums(is.na(df))
```

### The Python Way
Pandas uses `.isna()` (or its alias `.isnull()`).

```python
# Returns a boolean mask of the DataFrame
mask = df.isna()

# Check if values are NOT missing (Equivalent to !is.na())
not_missing_mask = df.notna() # .notnull() is an identical alias

# Top-level functional equivalent
mask_functional = pd.isna(df)

# Count missing values per column
na_counts = df.isna().sum()

# Count total missing values in the entire DataFrame
total_na = df.isna().sum().sum()

# Check if there are ANY missing values in the DataFrame
has_na = df.isna().values.any()
```

## Summary Comparison

| R (tidyr/base) | Python (pandas) | Note |
| :--- | :--- | :--- |
| `is.na(df)` | `df.isna()` or `pd.isna(df)` | Returns boolean mask |
| `!is.na(df)` | `df.notna()` or `df.notnull()` | Returns boolean mask of non-missing values |
| `drop_na()` | `df.dropna()` | Drops rows with any NaN |
| `drop_na(col)` | `df.dropna(subset=['col'])` | Drops rows with NaN in specific columns |
| `replace_na(list(A=0))` | `df.fillna({'A': 0})` or `df.replace({np.nan: 0})` | Replace NaNs with specific values |
| `fill(col, .direction="down")`| `df['col'].ffill()` | Forward fill |
| `fill(col, .direction="up")` | `df['col'].bfill()` | Backward fill |
| `zoo::na.approx()` | `df['col'].interpolate()` | Interpolate missing values |
| `complete(col1, col2)` | `.reindex()` or `.merge(how='cross')` | Makes implicit missing values explicit |
