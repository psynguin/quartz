---
title: "Descriptive Statistics and Summary Tables"
tags: [python, data-handling, pandas, numpy]
chapter: 18
theme: "Descriptive Statistics and Analytical Output Tables"
---

Descriptive statistics synthesize large datasets into readable summary metrics, allowing you to gauge the central tendency, dispersion, and shape of your data's distribution without inspecting every row. Generating clean, readable summary tables is essential for exploratory data analysis (EDA) and for reporting findings to stakeholders.

## Getting a High-Level Overview: `.describe()`

The most direct way to generate a summary statistics table for a DataFrame is the `.describe()` method. By default, it computes metrics exclusively for numeric columns.

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

# Default behavior: numeric columns only
numeric_summary = df.describe()
print(numeric_summary)
```

### All Syntactic Variants

You can control exactly which data types `.describe()` targets:

```python
# Method 1: Default (numeric only)
df.describe()

# Method 2: Include all columns (numeric, object, categorical, datetime)
df.describe(include='all')

# Method 3: Include specific dtypes
df.describe(include=['object', 'category'])

# Method 4: Exclude specific dtypes
df.describe(exclude=['datetime64'])
```

**When to Use**: 
- Use the default `df.describe()` for a quick check of numerical distributions, minimums, maximums, and potential outliers.
- Use `df.describe(include='all')` when you also need to see the count of unique values (`unique`), the most frequent value (`top`), and its frequency (`freq`) for categorical/text columns.

> [!NOTE] Interpreting `.describe(include='all')` Output
> When mixing numeric and categorical columns, pandas computes all possible statistics. For metrics that don't make sense (e.g., the `mean` of a text column, or the `top` value of a float column), it fills the cells with `NaN`.

> [!WARNING] Common Gotcha: No Objects to Concatenate
> If you call `df.describe(include=[np.number])` on a DataFrame that contains *zero* numeric columns, pandas will raise a `ValueError: No objects to concatenate`. **Why it happens**: Pandas finds no columns matching the requested type, creating an empty list of objects to summarize. **The fix**: Ensure the DataFrame actually has numeric columns before restricting by dtype, or handle the error gracefully.

## Per-Column Descriptive Statistics

While `.describe()` is convenient, you often need to extract a specific metric or compute statistics that `.describe()` omits, such as skewness or kurtosis.

```python
# Central tendency
mean_val = df['units_sold'].mean()
median_val = df['units_sold'].median()
mode_val = df['category'].mode()[0]  # mode returns a Series, [0] gets the top one

# Dispersion
std_val = df['units_sold'].std()
var_val = df['units_sold'].var()
range_val = df['units_sold'].max() - df['units_sold'].min()

# Distribution shape
skewness = df['units_sold'].skew()  # Positive = right-tailed
kurtosis = df['units_sold'].kurt()  # High = heavy tails / outliers

print(f"Mean: {mean_val:.2f}, Skew: {skewness:.2f}")
```

> [!TIP] Vectorized Operations
> These methods can be applied to the entire DataFrame at once. Use `df.mean(numeric_only=True)` to get the mean for all numeric columns simultaneously as a Series.

## Correlation and Covariance

Understanding how numerical variables move together is crucial. Pandas provides `.corr()` for correlation and `.cov()` for covariance.

```python
# Compute correlation matrix
# numeric_only=True prevents errors from text columns
corr_matrix = df.corr(method='pearson', numeric_only=True)
print(corr_matrix)

# Compute covariance matrix
cov_matrix = df.cov(numeric_only=True)
```

### All Syntactic Variants & Methods

Pandas supports three correlation algorithms via the `method` parameter:

```python
# Method 1: Pearson (Default)
df.corr(method='pearson', numeric_only=True)

# Method 2: Spearman
df.corr(method='spearman', numeric_only=True)

# Method 3: Kendall
df.corr(method='kendall', numeric_only=True)

# Method 4: Covariance
df.cov(numeric_only=True)
```

**When to Use**:
- Use **Pearson** for standard correlation between continuous, normally distributed variables.
- Use **Spearman** when assessing monotonic relationships, particularly if your data contains heavy outliers or ordinal properties.
- Use **Kendall** for rank correlation, especially with small sample sizes or heavily tied ranks.
- Use **Covariance (`.cov()`)** when you need the unscaled measure of directional association, often used as an input matrix for PCA or portfolio optimization algorithms, rather than direct human interpretation.

> [!WARNING] Common Gotcha: String Conversion Errors
> Older versions of pandas silently ignored text columns when computing `.corr()` or `.cov()`. Newer versions will raise a `ValueError: could not convert string to float` if you pass a mixed-type DataFrame without explicit instruction. **Why it happens**: Mathematical routines cannot process text entries like "Electronics". **The Fix:** Always specify `numeric_only=True`.

## Frequency and Proportion Tables

To summarize categorical data, you need frequency counts and cross-tabulations.

### Univariate Frequencies: `.value_counts()`

```python
# Method 1: Basic frequency count
freq = df['category'].value_counts()

# Method 2: Proportions (percentages)
props = df['category'].value_counts(normalize=True)

# Method 3: Binning continuous data into discrete intervals
price_bins = df['unit_price'].value_counts(bins=3)

# Method 4: Cumulative counts (useful for Pareto analysis)
cumulative_props = df['category'].value_counts(normalize=True).cumsum()
```

**When to Use**: Use `.value_counts()` when you need the distribution of a *single* column or when assessing class imbalance. Use the `normalize=True` variant to report percentages instead of raw counts.

> [!WARNING] Common Gotcha: Missing Data in Counts
> By default, `.value_counts()` completely ignores `NaN` values. If you want to know how many missing values exist alongside the categories, you must pass `dropna=False`.

### Bivariate Frequencies: `pd.crosstab()`

When you need to cross-tabulate two or more variables, use `pd.crosstab()`.

```python
# Method 1: Basic frequency table
pd.crosstab(index=df['category'], columns=df['region'])

# Method 2: Normalize over the entire table (all values sum to 1)
pd.crosstab(df['category'], df['region'], normalize='all')

# Method 3: Normalize over rows (each row sums to 1)
pd.crosstab(df['category'], df['region'], normalize='index')

# Method 4: Normalize over columns (each column sums to 1)
pd.crosstab(df['category'], df['region'], normalize='columns')

# Method 5: Compute aggregated statistics inside the crosstab
pd.crosstab(
    index=df['category'], 
    columns=df['region'], 
    values=df['units_sold'], 
    aggfunc='mean'
).fillna(0)
```

**When to Use**:
- Use the **default** for raw counts to see absolute occurrences.
- Use `normalize='index'` to understand the conditional probability row-wise (e.g., "Given a category, what percentage is in each region?").
- Use `normalize='columns'` to understand the conditional probability column-wise (e.g., "Given a region, what percentage is in each category?").
- Use `values` and `aggfunc` to pivot summary metrics instead of frequencies.

> [!WARNING] Common Gotcha: Passing Column Names as Strings
> If you try to run `pd.crosstab('category', 'region')`, you will get a `ValueError: If using all scalar values, you must pass an index`. **Why it happens**: Unlike `.groupby()`, `pd.crosstab()` is a top-level pandas function, not a DataFrame method. It doesn't know what DataFrame your columns belong to. **The Fix**: You must pass the actual Series objects: `pd.crosstab(df['category'], df['region'])`.

## Formatting and Exporting Output

Before sharing summary tables, you usually want to format them and export them to standard file formats like CSV or Excel.

**Rounding numbers:**
```python
summary = df.describe().round(2)
```

**Styling for Jupyter Notebooks / HTML:**
Pandas includes a `.style` accessor that allows you to apply conditional formatting, though this formatting only renders in HTML contexts (like Jupyter or exported HTML files).
```python
# Highlights the maximum value in each column
# (Only visual in Jupyter Notebooks)
styled_summary = summary.style.highlight_max(color='lightgreen')
```

**Exporting a summary table to CSV:**
```python
# Saves the summary table to a CSV file. The index (statistics names) is kept.
summary.to_csv('numeric_summary.csv', index=True)
```

## Applying to Multiple DataFrames (Capstone Workflow)

When dealing with a collection of DataFrames (e.g., monthly sales data), a standard workflow is to generate a summary statistic for each DataFrame, combine them into a master comparison report, and export the result to an Excel workbook with multiple sheets.

Here is the complete capstone workflow for this chapter:

```python
# 1. Collect `.describe()` from each monthly df
summary_dfs = {month: month_df.describe() for month, month_df in dfs.items()}

# 2. Concat with keys to form a MultiIndex DataFrame
master_summary = pd.concat(summary_dfs, names=['month', 'statistic'])

# 3. Format as a comparison report by moving 'month' to the columns
comparison_report = master_summary.unstack(level='month')
# Round everything to 2 decimal places for readability
comparison_report = comparison_report.round(2)

print(comparison_report.head())

# 4. Export to Excel with multiple sheets
# We use a context manager to write multiple sheets to the same file
output_file = 'monthly_summaries.xlsx'

with pd.ExcelWriter(output_file) as writer:
    # Write the full, combined comparison report
    comparison_report.to_excel(writer, sheet_name='All_Stats_Combined')
    
    # Write individual statistics to their own sheets
    for stat in ['mean', 'std', 'max', 'min']:
        try:
            # Extract just this statistic using xs (cross-section)
            stat_df = master_summary.xs(stat, level='statistic')
            stat_df.to_excel(writer, sheet_name=f'{stat.title()}_Comparison')
        except KeyError:
            # Handle cases where the statistic might not exist
            pass
            
print(f"Summary reports exported to {output_file}")
```

This pattern—generate per-DataFrame statistics, concatenate with keys, unstack to format, and export via `ExcelWriter`—is one of the most powerful and common workflows for automated reporting in pandas.

## Related Chapters
- [[09-aggregation-and-groupby]] for creating custom aggregate tables using GroupBy.
- [[11-combining-dataframes]] to review the `pd.concat` mechanics used in the capstone.
- [[12-multi-dataframe-workflows]] for more patterns on iterating over dictionary collections.
- [[14-multiindex-hierarchical-indexing]] for a deeper dive into `.unstack()` and `.xs()`.
- [[16-input-output]] for detailed parameters on saving DataFrames to Excel and CSV.
