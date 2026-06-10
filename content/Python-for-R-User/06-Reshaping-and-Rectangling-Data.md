# Chapter 6: Reshaping and Rectangling Data

Reshaping data (often called "pivoting" or "melting") and manipulating strings within columns are common data wrangling tasks. In R's `tidyr` package, you typically use `pivot_longer()`, `pivot_wider()`, `separate()`, and `unite()`. In pandas, these concepts translate to `.melt()`, `.pivot()`, `.pivot_table()`, and a suite of `.str` accessor methods. 

## 1. Pivoting Longer (Wide to Long)

### The R Logic
In R, when you have data spread across multiple columns that should be collapsed into key-value pairs, you use `pivot_longer()`.

```r
# R: tidyr
df %>%
  pivot_longer(
    cols = starts_with("Q"),
    names_to = "quarter",
    values_to = "revenue"
  )
```

### The Python Approach

In pandas, the equivalent operation is called "melting" and you use the `.melt()` method. You specify the columns to keep as identifiers (`id_vars`), and the columns to melt into rows (`value_vars`). 

#### 1. Using `.melt()`

```python
import pandas as pd
import numpy as np

# Sample Data
df = pd.DataFrame({
    'store': ['A', 'B'],
    'Q1': [100, 150],
    'Q2': [110, 160],
    'Q3': [105, 155]
})

# Pandas melt
df_long = df.melt(
    id_vars=['store'],             # Columns to keep as identifiers
    value_vars=['Q1', 'Q2', 'Q3'], # Columns to unpivot (if omitted, uses all non-id_vars)
    var_name='quarter',            # Equivalent to names_to
    value_name='revenue'           # Equivalent to values_to
)
print(df_long)
```

#### 2. Using `pd.wide_to_long()`

If your column names have a common prefix followed by an identifier (e.g., `revenue_1`, `revenue_2`), pandas has a specialized function that is closer in spirit to advanced `pivot_longer()` features.

```python
df_stub = pd.DataFrame({
    'store': ['A', 'B'],
    'revenue_Q1': [100, 150],
    'revenue_Q2': [110, 160]
})

# Wide to long with a stubname
pd.wide_to_long(
    df_stub,
    stubnames='revenue',    # The common prefix
    i='store',              # The id column
    j='quarter',            # The new variable name
    sep='_',                # Separator between stubname and suffix
    suffix=r'\w+'           # Regex matching the suffix
).reset_index()
```

## 2. Pivoting Wider (Long to Wide)

### The R Logic
In R, you use `pivot_wider()` to spread a key-value pair across multiple columns.

```r
# R: tidyr
df_long %>%
  pivot_wider(
    names_from = quarter,
    values_from = revenue
  )
```

### The Python Approach

In pandas, there are two primary methods for this: `.pivot()` and `.pivot_table()`. 

#### 1. Using `.pivot()`
> **Important Note on Duplicate Handling:** When transitioning from `pivot_wider`, it is crucial to understand that `.pivot()` requires unique index-column pairs. If your data contains duplicate entries for an index-column combination, `.pivot()` will fail with a `ValueError`. To gracefully handle duplicates, you must use `.pivot_table()` instead, which processes them via the `aggfunc`.

```python
# Pandas pivot
df_wide = df_long.pivot(
    index='store',     # Column(s) to use as new index
    columns='quarter', # Column whose values will become new columns (names_from)
    values='revenue'   # Column containing the values to populate (values_from)
).reset_index()        # Flatten the index back to a standard column

# Note: After pivoting, the columns will have a "name" (quarter in this case). 
# You can remove it for cleaner output:
df_wide.columns.name = None
```

#### 2. Using `.pivot_table()`
If you have duplicate index-column pairs and need to aggregate them (e.g., taking the mean or sum), use `.pivot_table()`. This is analogous to `values_fn` in `pivot_wider()`.

```python
# Create data with duplicate store-quarter pairs
df_long_dup = pd.DataFrame({
    'store': ['A', 'A', 'B'],
    'quarter': ['Q1', 'Q1', 'Q1'],
    'revenue': [50, 50, 150]
})

# Pandas pivot_table with aggregation
df_wide_agg = df_long_dup.pivot_table(
    index='store',
    columns='quarter',
    values='revenue',
    aggfunc='sum'      # Aggregation function for duplicates
).reset_index()
```

#### 3. Method Chaining Alternative using `.unstack()`
Pandas also supports `.stack()` and `.unstack()` for reshaping using a MultiIndex.

```python
# Convert store and quarter into a MultiIndex, then unstack the 'quarter' level
df_wide_unstack = (
    df_long
    .set_index(['store', 'quarter'])
    .unstack('quarter')
)
# Flatten MultiIndex columns
df_wide_unstack.columns = [col[1] for col in df_wide_unstack.columns]
df_wide_unstack = df_wide_unstack.reset_index()
```

## 3. Separating Columns

### The R Logic
In R, `separate()` splits a single column into multiple columns based on a delimiter or regex.

```r
# R: tidyr
df %>%
  separate(col = name_age, into = c("name", "age"), sep = "_")
```

### The Python Approach

In pandas, string operations are accessed via the `.str` accessor. You can use `.str.split()` with `expand=True` to create a DataFrame from the split lists, which you then assign back to the original DataFrame.

#### 1. Standard Assignment (Direct extraction)

```python
df_sep = pd.DataFrame({'name_age': ['Alice_30', 'Bob_25', 'Charlie_35']})

# .str.split() with expand=True returns a DataFrame with separate columns
df_sep[['name', 'age']] = df_sep['name_age'].str.split('_', expand=True)

# You can drop the original column if desired
df_sep = df_sep.drop(columns=['name_age'])
```

#### 2. Method Chaining (Using `.assign()`)

```python
df_sep = pd.DataFrame({'name_age': ['Alice_30', 'Bob_25', 'Charlie_35']})

df_sep_chained = (
    df_sep
    .assign(
        name=lambda x: x['name_age'].str.split('_').str[0],
        age=lambda x: x['name_age'].str.split('_').str[1]
    )
    .drop(columns=['name_age'])
)
```

#### 3. Using `.str.extract()` for Complex Regex
If the split is more complex, you can extract named capture groups using regular expressions. This is the pandas equivalent of `extract()` in `tidyr`.

```python
df_complex = pd.DataFrame({'info': ['A-100', 'B-200']})

# Extract directly into new columns using regex named groups (?P<name>...)
extracted = df_complex['info'].str.extract(r'(?P<category>[A-Z])-(?P<id>\d+)')
df_complex = pd.concat([df_complex, extracted], axis=1)
```

## 4. Uniting Columns

### The R Logic
In R, `unite()` concatenates multiple columns into one.

```r
# R: tidyr
df %>%
  unite(col = "name_age", name, age, sep = "_")
```

### The Python Approach

In pandas, string concatenation is straightforward. You simply add the string representations of the columns together.

#### 1. Standard Assignment

```python
df_unite = pd.DataFrame({'name': ['Alice', 'Bob'], 'age': [30, 25]})

# Convert numerical columns to string using .astype(str) before concatenating
df_unite['name_age'] = df_unite['name'] + '_' + df_unite['age'].astype(str)

df_unite = df_unite.drop(columns=['name', 'age'])
```

#### 2. Using `.agg()` for Many Columns
If you need to unite many columns, you can apply a string join across the row axis (`axis=1`).

```python
df_many = pd.DataFrame({'part1': ['A', 'B'], 'part2': ['1', '2'], 'part3': ['X', 'Y']})

# Join all elements in the row with a hyphen
df_many['united'] = df_many[['part1', 'part2', 'part3']].agg('-'.join, axis=1)
```

#### 3. Method Chaining

```python
df_unite = pd.DataFrame({'name': ['Alice', 'Bob'], 'age': [30, 25]})

df_unite_chained = (
    df_unite
    .assign(name_age=lambda x: x['name'] + '_' + x['age'].astype(str))
    .drop(columns=['name', 'age'])
)
```

## 5. Unnesting Lists into Rows

### The R Logic
In R, `unnest()` is used when a single cell contains multiple values (often stored as a list) that need to be spread out into separate rows.

```r
# R: tidyr
df %>%
  unnest(items)
```

### The Python Approach

In pandas, the essential equivalent is `.explode()`. It transforms each element of a list-like cell to a row, replicating the index values.

```python
df_nest = pd.DataFrame({
    'user': ['Alice', 'Bob'],
    'items': [['apple', 'banana'], ['orange']]
})

# Explode the 'items' column
df_exploded = df_nest.explode('items')
print(df_exploded)
```

## 6. Rectangling Categorical Data to Binary Flags

### The R Logic
In R, transforming categorical data into binary flags is often done using `pivot_wider()` with `values_fill = 0` or using specialized packages like `recipes`.

```r
# R: tidyr
df %>%
  mutate(value = 1) %>%
  pivot_wider(names_from = category, values_from = value, values_fill = list(value = 0))
```

### The Python Approach

Pandas offers `pd.get_dummies()`, an essential data science step used for one-hot encoding. It effortlessly widens categorical columns into binary (0/1) indicator variables, which is conceptually similar to `pivot_wider` with a binary fill.

```python
df_cat = pd.DataFrame({'id': [1, 2, 3], 'category': ['A', 'B', 'A']})

# Create dummy variables
df_dummies = pd.get_dummies(df_cat, columns=['category'], dtype=int)
print(df_dummies)
```

## 7. Cross-Tabulation for Counts

### The R Logic
In R, generating a cross-tabulated frequency matrix often involves `table()` or `xtabs()`, which output frequency distributions in a wide format.

```r
# R: base
table(df$store, df$quarter)
```

### The Python Approach

For "rectangling" frequency counts into wide-format matrices, `pd.crosstab()` is extremely useful. It builds a frequency table out of two or more columns quickly.

```python
df_freq = pd.DataFrame({
    'store': ['A', 'A', 'B', 'B', 'A'],
    'quarter': ['Q1', 'Q2', 'Q1', 'Q1', 'Q2']
})

# Create a frequency matrix
freq_matrix = pd.crosstab(
    index=df_freq['store'], 
    columns=df_freq['quarter']
)
print(freq_matrix)
```
