---
title: "Window Functions: Rolling, Expanding, and Exponential"
tags: [python, data-handling, pandas, numpy]
chapter: 13
theme: "Rolling, Expanding, and Exponential Window Functions"
---

# Window Functions: Rolling, Expanding, and Exponential

Window functions allow you to perform calculations across a specific subset (or "window") of rows that are related to the current row. Unlike a standard aggregation (`.sum()` or `.mean()`) which groups rows into a single summary output, a window function returns a value for *every* row in the original dataset.

These functions are critical for time-series analysis and longitudinal data. They allow you to compute moving averages, running totals, and momentum signals while maintaining the exact shape and size of your original DataFrame.

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

## 13.1 Rolling Windows

### Concept Explanation
A rolling window takes a fixed number of periods (or a fixed time duration) and slides it down the DataFrame row by row. Imagine a 3-row window. For row 3, the window includes rows 1, 2, and 3. For row 4, the window includes rows 2, 3, and 4. You can then apply an aggregation like `.mean()` to the values currently inside the window. This is highly useful for smoothing out daily volatility to reveal an underlying trend (like a 7-day moving average of sales).

### All Syntactic Variants

```python
# Method 1: Basic row-based rolling window
df['rolling_sum'] = df['units_sold'].rolling(window=3).sum()
df['rolling_std'] = df['units_sold'].rolling(window=3).std()
df['rolling_min'] = df['units_sold'].rolling(window=3).min()

# Method 2: Minimum periods
df['rolling_mean'] = df['units_sold'].rolling(window=3, min_periods=1).mean()

# Method 3: Centered window (looks forward and backward)
df['centered_max'] = df['units_sold'].rolling(window=3, center=True).max()

# Method 4: Weighted window types (requires SciPy)
df['gaussian_mean'] = df['units_sold'].rolling(window=3, win_type='gaussian').mean(std=1)
df['triang_mean'] = df['units_sold'].rolling(window=3, win_type='triang').mean()

# Method 5: Time-based rolling (requires datetime index OR 'on' parameter)
df['14_day_sales'] = df.rolling('14D', on='date')['units_sold'].sum()

# Method 6: Applying custom functions
df['rolling_range'] = df['units_sold'].rolling(window=3).apply(lambda x: x.max() - x.min())
```

> [!TIP] When to Use What
> - Use **row-based rolling (`window=3`)** when your rows are evenly spaced (e.g., daily metrics with no missing days).
> - Use **time-based rolling (`window='14D'`)** when observations are irregularly spaced (like the sales dates in our dataset) to ensure the window covers exactly two weeks of calendar time.
> - Use `min_periods=1` when you want to see preliminary results for the early rows instead of getting `NaN` until the window is fully populated.

### Worked Examples

```python
# Minimal Example: 2-period moving average
df['sales_ma'] = df['units_sold'].rolling(window=2).mean()

# Progressive Build: A realistic smoothing scenario
# We want to smooth the units_sold over a 14-day calendar window.
# 1. Always ensure the data is sorted chronologically before rolling!
df_sorted = df.sort_values('date').copy()

# 2. Apply a 14-day rolling average, requiring at least 1 day of data
df_sorted['14d_sales_trend'] = (
    df_sorted.rolling('14D', on='date')['units_sold']
    .mean()
)
```

### Common Errors and Gotchas

**ValueError: window must be an integer**
- **The Error**: `ValueError: window must be an integer`
- **Why it happens**: You attempted to use a time string (like `'7D'`) in `.rolling()`, but your DataFrame index is not a DatetimeIndex, and you forgot to specify the `on=` parameter.
- **The Fix**:
  ```python
  # Correct (if 'date' is a column):
  df['rolling'] = df['units_sold'].rolling('7D', on='date').mean()
  ```

**Leading NaNs in Output**
- **The Error**: Your rolling columns start with `NaN` values.
- **Why it happens**: By default, a window of size `n` requires `n` rows of data to compute the first value.
- **The Fix**: Use `min_periods=1` so calculations begin immediately using whatever data is available.

### Applying to Multiple DataFrames

When applying rolling windows across multiple DataFrames, you must consider whether the window should reset at the boundary of each DataFrame, or whether it should roll smoothly across the entire collection. 

```python
# 1. Dict-based pattern (Resets rolling at the start of each df)
dfs_smoothed_dict = {
    k: df.assign(rolling_avg=df['units_sold'].rolling(window=2, min_periods=1).mean())
    for k, df in dfs.items()
}

# 2. List-based pattern (Resets rolling at the start of each df)
dfs_list = list(dfs.values())
dfs_smoothed_list = [
    df.assign(rolling_avg=df['units_sold'].rolling(window=2, min_periods=1).mean())
    for df in dfs_list
]
```

**Caveat for Bulk Operations**: 
By applying `.rolling()` to each DataFrame individually, the first row of February has no access to the last row of January. If your rolling window needs to smoothly cross month boundaries, you **must** use a "combine-then-split" pattern: concatenate the DataFrames into one, sort by date, apply `.rolling()`, and then split them back up.

---

## 13.2 Expanding Windows

### Concept Explanation
An expanding window starts from the first row and grows with each subsequent row. If you want to track a metric "from the beginning of time up to the present moment," you use an expanding window. It is ideal for running totals, cumulative maximums, or tracking the all-time average of a KPI as new data continuously arrives.

### All Syntactic Variants

```python
# Method 1: Default expanding (cumulative mean of everything seen so far)
df['cumulative_mean_price'] = df['unit_price'].expanding().mean()

# Method 2: Setting min_periods
# Only outputs a value once 3 rows have been processed
df['expanding_sum'] = df['units_sold'].expanding(min_periods=3).sum()
```

> [!TIP] When to Use What
> - Use **`.expanding()`** over `.rolling()` when the total history matters for the current row's calculation (e.g., "What is the all-time average unit price?" vs "What is the average over the last 3 days?").
> - Choose **`min_periods`** carefully: use `min_periods=1` if you want a running metric from day 1; increase it if you need a statistically significant baseline (e.g. at least 30 observations) before reporting the first metric.

### Worked Examples

```python
# Minimal Example: All-time average units sold
df['all_time_avg_units'] = df['units_sold'].expanding().mean()

# Progressive Build: Tracking maximum price and dynamic volatility
df_sorted = df.sort_values('date').copy()

# Track the highest unit price seen up to the current date
df_sorted['max_price_seen'] = df_sorted['unit_price'].expanding().max()

# Compute the running standard deviation of units sold
df_sorted['historical_volatility'] = df_sorted['units_sold'].expanding().std()
```

### Common Errors and Gotchas

**Initial NaNs when Expanding Requires Multiple Periods**
- **The Error**: The first row of an `.expanding().std()` calculation is always `NaN`, even with `min_periods=1`.
- **Why it happens**: Certain statistical calculations, like sample standard deviation or variance, mathematically require at least two data points.
- **The Fix**: This is mathematically correct behavior. If your downstream pipeline breaks due to this `NaN`, chain a `.fillna(0)` at the end.

### Applying to Multiple DataFrames

```python
# 1. Dict-based pattern
dfs_expanding_dict = {
    k: df.assign(expanding_max=df['units_sold'].expanding().max())
    for k, df in dfs.items()
}

# 2. List-based pattern
dfs_expanding_list = [
    df.assign(expanding_max=df['units_sold'].expanding().max())
    for df in dfs_list
]
```

**Caveat for Bulk Operations**: 
Like `.rolling()`, an expanding window applied iteratively will restart its "all-time" tracking at row 0 of each DataFrame. To compute a true global expanding metric across a collection, you must first concatenate them.

---

## 13.3 Exponential Weighted Moving (EWM) Windows

### Concept Explanation
In a standard rolling mean, data points fall abruptly out of the window, causing sudden jumps in the output metric. In EWM, older data points gracefully "fade" in influence. This is heavily used in finance to create momentum signals and in data science to smooth noisy time series without introducing as much lag as a standard rolling window.

### All Syntactic Variants

You configure EWM by specifying how quickly the weights should decay. Pandas offers four mutually exclusive ways to set this:

```python
# Method 1: Span
df['ewm_span'] = df['units_sold'].ewm(span=3).mean()

# Method 2: Half-life
df['ewm_halflife'] = df['units_sold'].ewm(halflife=2).mean()

# Method 3: Alpha
df['ewm_alpha'] = df['units_sold'].ewm(alpha=0.5).mean()

# Method 4: Center of Mass (com)
df['ewm_com'] = df['units_sold'].ewm(com=2).mean()
```

> [!TIP] When to Use What
> When choosing how to specify EWM decay:
> - Use **`span`** when you want an EWM that roughly corresponds to an $N$-period rolling moving average (common in technical analysis, like a 10-day EMA).
> - Use **`halflife`** when you want to think in terms of time (e.g., "a 7-day halflife" means a week-old observation has exactly half the weight of today's).
> - Use **`alpha`** when you want direct control over the smoothing factor ($0 < \alpha \leq 1$). Higher alpha discounts older observations faster.
> - Use **`com`** (Center of Mass) when aligning with physical or statistical formulas that require this specific parameterization.

### Worked Examples

```python
# Minimal Example: Simple exponential moving average
df['ewm_sales'] = df['units_sold'].ewm(span=3).mean()

# Progressive Build: Creating a momentum signal
df_sorted = df.sort_values('date').copy()

# We use a halflife of 2 rows to strongly favor recent pricing/sales trends
df_sorted['short_term_trend'] = df_sorted['units_sold'].ewm(halflife=2).mean()
df_sorted['long_term_trend'] = df_sorted['units_sold'].ewm(halflife=7).mean()

# A crossover signal (when short term exceeds long term)
df_sorted['momentum_signal'] = df_sorted['short_term_trend'] > df_sorted['long_term_trend']
```

### Common Errors and Gotchas

**ValueError: Comutually exclusive arguments**
- **The Error**: `ValueError: com, span, halflife, and alpha are mutually exclusive`
- **Why it happens**: You attempted to specify more than one decay parameter in the `.ewm()` call, such as `.ewm(span=3, alpha=0.5)`.
- **The Fix**: Choose exactly one decay parameter. 

### Applying to Multiple DataFrames

```python
# 1. Dict-based pattern
dfs_ewm_dict = {
    k: df.assign(ewm_trend=df['units_sold'].ewm(span=3).mean())
    for k, df in dfs.items()
}

# 2. List-based pattern
dfs_ewm_list = [
    df.assign(ewm_trend=df['units_sold'].ewm(span=3).mean())
    for df in dfs_list
]
```

**Caveat for Bulk Operations**: 
EWM intrinsically relies on a continuous history to build up the exponentially weighted sum. If you process data in chunks (like separate months), the beginning of each chunk will have no history to draw from, leading to "cold starts" where the initial values are heavily skewed towards the very first observation of that file.

---

## 13.4 Rank and Cumulative Methods

### Concept Explanation
Pandas provides optimized, direct methods for simple cumulative operations and ranking. These are row-by-row functions that compute values incrementally from the start of the data or assign relative standings across the dataset. 
- **Real-world use cases**: Creating a running bank account balance (`.cumsum()`), calculating compounded portfolio returns (`.cumprod()`), or finding the top 3 selling stores in a given quarter (`.rank()`).

### All Syntactic Variants

```python
# Cumulative Operations
df['running_total'] = df['units_sold'].cumsum()
df['max_seen_so_far'] = df['units_sold'].cummax()
df['min_seen_so_far'] = df['units_sold'].cummin()
df['discount_multiplier'] = (1 - df['discount']).cumprod()
```

> [!TIP] When to Use What (Cumulative Methods)
> - Use **`.cumsum()`** for running totals like account balances or year-to-date sales.
> - Use **`.cumprod()`** for compounding effects, such as cumulative returns on an investment over time.
> - Use **`.cummax()`** or **`.cummin()`** to track high-water or low-water marks, like the all-time high price of a stock or the lowest inventory level reached.

```python
# Ranking Data
# Default rank (averages the rank for ties)
df['sales_rank_avg'] = df['units_sold'].rank()

# Dense rank (no gaps in rank values when ties occur)
df['sales_rank_dense'] = df['units_sold'].rank(method='dense')

# Min rank (both tied items get the lowest rank, e.g., 1, 2, 2, 4)
df['sales_rank_min'] = df['units_sold'].rank(method='min', ascending=False)
```

> [!TIP] When to Use What (Ranking Methods)
> - Use **`method='average'`** (the default) when statistical properties matter, as the sum of ranks remains constant.
> - Use **`method='dense'`** when you want no gaps between ranks (e.g., 1st, 2nd, 2nd, 3rd). Great for awarding medals or discrete tiers.
> - Use **`method='min'`** when tied items should all receive the highest possible numerical rank (e.g., 1st, 2nd, 2nd, 4th). Typical in sports ranking.

### Worked Examples

```python
# Minimal Example: Running total of sales
df['running_sales'] = df['units_sold'].cumsum()

# Progressive Build: Calculating cumulative revenue and ranking transactions
df_sorted = df.sort_values('date').copy()

# 1. Compute per-transaction revenue
df_sorted['revenue'] = df_sorted['units_sold'] * df_sorted['unit_price']

# 2. Cumulative sum to see total revenue grow over time
df_sorted['cumulative_revenue'] = df_sorted['revenue'].cumsum()

# 3. Rank the rows by the revenue they generated (dense ranking, descending)
df_sorted['revenue_rank'] = df_sorted['revenue'].rank(method='dense', ascending=False)
```

### Common Errors and Gotchas

**TypeError in Cumulative Methods on Strings**
- **The Error**: `TypeError: cannot perform cumsum with type String` (or object).
- **Why it happens**: Calling `.cumsum()` on a mixed-type or string column (like `store_id`) will concatenate the strings row-by-row, which is extremely memory intensive and often crashes.
- **The Fix**: Always explicitly select numeric columns before applying cumulative methods.
  ```python
  # Correct: 
  df[['units_sold', 'unit_price']].cumsum()
  ```

### Applying to Multiple DataFrames

```python
# 1. Dict-based pattern
dfs_ranked_dict = {
    k: df.assign(rank=df['units_sold'].rank(method='dense'))
    for k, df in dfs.items()
}

# 2. List-based pattern
dfs_ranked_list = [
    df.assign(rank=df['units_sold'].rank(method='dense'))
    for df in dfs_list
]
```

**Caveat for Bulk Operations**: 
Rankings and cumulative sums will be constrained to the specific DataFrame. A rank computed inside the 'january' DataFrame will rank items against January sales only, not against the global dataset.

---

## 13.5 GroupBy Rolling

### Concept Explanation
When dealing with panel data or data containing multiple entities (like multiple stores, users, or products), applying a window function blindly across the entire DataFrame will mix data from different entities. GroupBy Rolling solves this by splitting the data by a category, applying the rolling window to each group independently, and then combining the results back.

### All Syntactic Variants

```python
# Assume df is sorted chronologically
df_sorted = df.sort_values('date').copy()

# Method 1: Row-based GroupBy rolling
df_sorted.groupby('store_id')['units_sold'].rolling(window=3).mean()

# Method 2: Time-based GroupBy rolling
df_sorted.groupby('store_id').rolling('7D', on='date')['units_sold'].sum()
```

> [!TIP] When to Use What
> - Use **row-based GroupBy rolling** if every group has exactly the same frequency of observations with no missing rows.
> - Use **time-based GroupBy rolling (`on='date'`)** when observations are irregularly spaced or if groups have missing dates, ensuring the rolling window respects calendar time per group.

### Worked Examples

```python
# Minimal Example: GroupBy rolling mean
df_sorted = df.sort_values(['store_id', 'date']).copy()
df_sorted['store_avg'] = df_sorted.groupby('store_id')['units_sold'].rolling(2).mean().values

# Progressive Build: GroupBy rolling sum with proper MultiIndex handling
df_sorted['store_rolling_sales'] = (
    df_sorted.groupby('store_id')['units_sold']
    .rolling(window=2, min_periods=1)
    .sum()
    .reset_index(level=0, drop=True) 
)
```

### Common Errors and Gotchas

**MultiIndex Alignment Failure**
- **The Error**: `TypeError: incompatible index of inserted column with frame index` or the column fills entirely with `NaN`.
- **Why it happens**: By default, `df.groupby('group_col')['val'].rolling(n).mean()` returns a Series with a **MultiIndex** (Level 0: the group key, Level 1: the original DataFrame index). If you try to assign this directly back to the DataFrame as a new column, pandas fails to align the MultiIndex with the DataFrame's single index.
- **The Fix**: Drop the grouping level from the index using `.reset_index(level=0, drop=True)` before assignment.
  ```python
  df['rolling_avg'] = (
      df.groupby('store_id')['units_sold']
      .rolling(2).mean()
      .reset_index(level=0, drop=True) # Essential fix!
  )
  ```

### Applying to Multiple DataFrames

```python
# 1. Dict-based pattern (groupby rolling within each df)
dfs_gb_dict = {
    k: df.sort_values('date').assign(
        store_avg=lambda x: x.groupby('store_id')['units_sold']
                            .rolling(2).mean().reset_index(0, drop=True)
    )
    for k, df in dfs.items()
}

# 2. List-based pattern (groupby rolling within each df)
dfs_gb_list = [
    df.sort_values('date').assign(
        store_avg=lambda x: x.groupby('store_id')['units_sold']
                            .rolling(2).mean().reset_index(0, drop=True)
    )
    for df in dfs_list
]
```

**Caveat for Bulk Operations**: 
If your data is sharded by month, applying GroupBy rolling iteratively means Store 1's history from January is lost when calculating its moving average in February. If you need groups to cross data chunks, you must concatenate them into a master DataFrame before performing the groupby rolling operation.

---

## Related Chapters
- [[09-aggregation-and-groupby]] — For understanding the `.groupby()` mechanics used before `.rolling()`.
- [[12-multi-dataframe-workflows]] — For advanced patterns of applying operations over collections of DataFrames.
- [[14-multiindex-hierarchical-indexing]] — To handle the MultiIndex results output by grouped window operations.
