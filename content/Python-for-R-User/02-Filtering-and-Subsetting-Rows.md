# Chapter 2: Filtering and Subsetting Rows (Row Verbs)

When working with data, subsetting rows based on specific conditions, sorting them, or removing duplicates are foundational tasks. In R's `dplyr`, these operations are handled by "row verbs" like `filter()`, `arrange()`, `distinct()`, and `slice()`. 

In Python, Pandas provides multiple ways to achieve these tasks, ranging from standard boolean indexing to specialized methods. This chapter will walk you through translating these R concepts into Python.

## 1. `filter()` -> Boolean Indexing and `.query()`

In R, you use `filter()` to select rows that satisfy specific conditions.

**R:**
```R
# Single condition
filter(df, age > 30)

# Multiple conditions
filter(df, age > 30 & city == "New York")
filter(df, city %in% c("New York", "London"))
```

In Python, there are a few primary ways to filter rows: boolean indexing, the `.query()` method, and `.loc[]`.

**Python:**

### Method 1: Boolean Indexing (Standard Pandas)
The most common and traditional way to filter rows in Pandas is by passing a boolean mask inside the DataFrame brackets `[]`.

> [!WARNING]
> **Bitwise vs. Logical Operators:** R users will instinctively try to use `&` and `|` without parentheses (e.g., `df[df.a > 1 & df.b < 2]`), which throws a "truth value of a Series is ambiguous" error in Pandas due to operator precedence. **You must wrap each condition in parentheses** when using boolean indexing.

> [!NOTE]
> **NA Handling in Filtering:** In R, `filter(df, col == 1)` drops `NA` values automatically. In Pandas boolean indexing, conditions like `df['col'] == 1` evaluate to `False` for `NaN` values, effectively dropping them. However, using other string or math operations can sometimes result in `NaN`s propagating into the boolean mask itself, causing a `ValueError`. In those cases, chain `.fillna(False)` to your condition.

```python
import pandas as pd

# Single condition
df[df['age'] > 30]

# Multiple conditions (AND: &, OR: |, NOT: ~)
# Note: You MUST wrap each condition in parentheses
df[(df['age'] > 30) & (df['city'] == 'New York')]
df[(df['age'] > 30) | (df['city'] == 'London')]
df[~(df['city'] == 'Paris')] # Not Paris

# %in% equivalent: .isin()
df[df['city'].isin(['New York', 'London'])]

# dplyr::between() equivalent: .between()
df[df['age'].between(20, 30)]

# String matching (R's str_detect / startswith)
df[df['city'].str.startswith('New', na=False)]

# Missing data filtering (R's is.na() / !is.na())
df[df['age'].isna()]
df[df['age'].notna()]
```

### Method 2: The `.query()` Method (Great for method chaining)
`.query()` evaluates a string expression and is visually closer to R's `filter()`. It is highly readable and excellent when chaining methods (similar to `%>%` or `|>`).

```python
# Single condition
df.query("age > 30")

# Multiple conditions (can use 'and', 'or', 'not')
df.query("age > 30 and city == 'New York'")
df.query("age > 30 or city == 'London'")

# %in% equivalent within .query()
df.query("city in ['New York', 'London']")

# Referencing external variables using @
target_age = 30
df.query("age > @target_age")
```

### Method 3: Using `.loc[]`
`.loc[]` is primarily used for label-based indexing, but it also accepts boolean arrays. It is especially useful when you want to filter rows AND select specific columns at the same time.

```python
df.loc[df['age'] > 30]

# Filter rows AND select specific columns
df.loc[df['age'] > 30, ['name', 'city']]
```

### Method 4: `np.where()` (For conditional logic, returning indices)
While `np.where()` is more often used like R's `ifelse()`, you can use it to get the indices of matching rows, though it's less common for direct subsetting compared to the methods above.

```python
import numpy as np

indices = np.where(df['age'] > 30)[0]
df.iloc[indices]
```

## 2. `arrange()` -> `.sort_values()`

In R, `arrange()` is used to sort the rows of a data frame based on one or more columns.

**R:**
```R
# Ascending
arrange(df, age)

# Descending
arrange(df, desc(age))

# Multiple columns
arrange(df, city, desc(age))
```

In Python, the equivalent is `.sort_values()`.

**Python:**

```python
# Ascending (default)
df.sort_values(by='age')

# Descending
df.sort_values(by='age', ascending=False)

# Multiple columns
# Sort by city ascending, then age descending
df.sort_values(by=['city', 'age'], ascending=[True, False])

# Note on method chaining: 
# .sort_values() returns a new DataFrame, making it perfectly suited for chains
df.query("age > 30").sort_values(by='name')
```

## 3. `distinct()` -> `.drop_duplicates()`

In R, `distinct()` removes duplicate rows. You can apply it to the entire data frame or to specific columns.

**R:**
```R
# Entire data frame
distinct(df)

# Specific columns (keeps only the specified columns by default)
distinct(df, city)

# Specific columns, keeping all other columns
distinct(df, city, .keep_all = TRUE)
```

In Python, you use `.drop_duplicates()`.

**Python:**

```python
# Entire DataFrame (drops rows where ALL columns are identical)
df.drop_duplicates()

# Specific columns (keeps only the specified columns, like distinct(df, city))
df[['city']].drop_duplicates()

# Specific columns, keeping all other columns (equivalent to .keep_all = TRUE)
# Note: 'keep' argument can be 'first', 'last', or False (drop all duplicates)
df.drop_duplicates(subset=['city'])
df.drop_duplicates(subset=['city', 'age'], keep='first')
```

## 4. `slice()` -> `.iloc[]`, `.head()`, `.tail()`, and `.nlargest()`

In R, `slice()` lets you select, remove, or duplicate rows by their integer position. Functions like `slice_head()`, `slice_tail()`, `slice_max()`, and `slice_sample()` provide specialized behaviors.

**R:**
```R
# Select rows by position
slice(df, 1:5)

# Head / Tail
slice_head(df, n = 5)
slice_tail(df, n = 5)

# Top N by a column
slice_max(df, order_by = age, n = 5)

# Sample random rows
slice_sample(df, n = 5)
```

In Python, integer-position indexing is done using `.iloc[]`, and there are specific methods for the other `slice_*()` variants.

**Python:**

### Slicing by Position: `.iloc[]` and `.take()`
`.iloc[]` is strictly for integer-location based indexing. Remember that Python uses 0-based indexing, and the end index in a slice is exclusive.

```python
# Select first 5 rows (indices 0, 1, 2, 3, 4)
# Equivalent to R's slice(df, 1:5)
df.iloc[0:5]

# Slicing from row index 10 up to (but not including) 20
# Equivalent to R's slice(df, 10:20) (keeping in mind R is 1-indexed and inclusive)
df.iloc[10:20]

# Select specific rows by index position
df.iloc[[0, 2, 4]]

# .take() is a fast alternative to .iloc for selecting rows by position
df.take([0, 2, 4])
```

### Head and Tail
Pandas has dedicated `.head()` and `.tail()` methods.

```python
# First 5 rows (default is 5)
df.head(5)

# Last 5 rows
df.tail(5)
```

### Top / Bottom N: `.nlargest()` and `.nsmallest()`
To get the rows with the largest or smallest values in a specific column without sorting the entire DataFrame first, use `.nlargest()` or `.nsmallest()`.

```python
# Top 5 rows ordered by age (largest first)
# Equivalent to slice_max()
df.nlargest(5, 'age')

# Bottom 5 rows ordered by age (smallest first)
# Equivalent to slice_min()
df.nsmallest(5, 'age')
```

### Random Sampling: `.sample()`
To select a random sample of rows.

```python
# Sample 5 random rows
# Equivalent to slice_sample(df, n = 5)
df.sample(n=5)

# Sample a percentage (fraction) of rows
# Equivalent to slice_sample(df, prop = 0.2)
df.sample(frac=0.2)

# Set seed for reproducibility
df.sample(n=5, random_state=42)
```

## 5. Complex Column-Based Filtering (`if_any()` and `if_all()`)

In R's `dplyr`, `if_any()` and `if_all()` allow you to filter rows based on conditions applied across multiple columns simultaneously, often combined with tidy-select helpers.

**R:**
```R
# Keep rows where ANY column starting with "score" is greater than 50
filter(df, if_any(starts_with("score"), ~ .x > 50))

# Keep rows where ALL numeric columns are greater than 0
filter(df, if_all(where(is.numeric), ~ .x > 0))
```

In Python, this logic is achieved by selecting the subset of columns (e.g., using `.filter()`), applying the boolean condition, and then reducing it row-wise using `.any(axis=1)` or `.all(axis=1)`.

**Python:**

### Method 1: Using `.any()` and `.all()` with `.filter()`

```python
# if_any(): Keep rows where ANY column starting with "score" is > 50
# 1. Filter columns: df.filter(regex='^score')
# 2. Apply condition: .gt(50)  (which means > 50)
# 3. Reduce row-wise: .any(axis=1)
mask_any = df.filter(regex='^score').gt(50).any(axis=1)
df[mask_any]

# if_all(): Keep rows where ALL numeric columns are > 0
numeric_cols = df.select_dtypes(include='number')
mask_all = numeric_cols.gt(0).all(axis=1)
df[mask_all]
```

### Method 2: Method Chaining Alternative
You can embed this logic directly inside a `.loc[]` call to maintain a clean method chain.

```python
# Chaining the if_any logic
df.loc[lambda x: x.filter(regex='^score').gt(50).any(axis=1)]
```

## Summary Comparison Table

| R (dplyr) | Python (pandas) |
| :--- | :--- |
| `filter(df, col == val)` | `df[df['col'] == val]` <br> `df.query("col == 'val'")` |
| `filter(df, col %in% vals)`| `df[df['col'].isin(vals)]` <br> `df.query("col in @vals")` |
| `filter(df, between(col, a, b))` | `df[df['col'].between(a, b)]` |
| `filter(df, is.na(col))` | `df[df['col'].isna()]` |
| `arrange(df, col)` | `df.sort_values(by='col')` |
| `arrange(df, desc(col))` | `df.sort_values(by='col', ascending=False)` |
| `distinct(df)` | `df.drop_duplicates()` |
| `distinct(df, col, .keep_all=TRUE)` | `df.drop_duplicates(subset=['col'])` |
| `slice(df, 1:5)` | `df.iloc[0:5]` |
| `slice(df, 10:20)` | `df.iloc[10:20]` <br> `df.take(range(10, 20))` |
| `slice_head(df, n=5)` | `df.head(5)` |
| `slice_max(df, order_by=col, n=5)` | `df.nlargest(5, 'col')` |
| `slice_sample(df, n=5)` | `df.sample(n=5)` |
