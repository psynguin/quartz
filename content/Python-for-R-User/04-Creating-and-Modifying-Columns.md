# Chapter 4: Creating and Modifying Columns

## Introduction
In R's Tidyverse, adding or modifying columns is primarily handled by the `dplyr::mutate()` function, with variants like `transmute()` to keep only the new columns, and vectorised control flows like `if_else()` and `case_when()` for conditional logic. 

In pandas, creating and modifying columns can be done in-place using traditional bracket assignment, or out-of-place (useful for method chaining) using the `.assign()` method. Conditional logic relies heavily on NumPy functions like `np.where()` and `np.select()`.

## 1. Basic Column Creation and Modification (`mutate`)

### In R
```R
# Creating a new column and modifying an existing one
df <- df %>%
  mutate(
    new_col = col1 + col2,
    col1 = col1 * 10
  )
```

### In Python
**Method 1: Bracket Assignment (Traditional / In-place)**
This is the most common and idiomatic way in pandas to add or modify a single column. It modifies the DataFrame in place.
```python
import pandas as pd
import numpy as np

# Creating a new column
df['new_col'] = df['col1'] + df['col2']

# Modifying an existing column
df['col1'] = df['col1'] * 10
```

**Method 2: Using `.assign()` (Method Chaining / Out-of-place)**
Similar to `mutate()`, `.assign()` returns a new DataFrame with the new columns added. It does not modify the original DataFrame unless reassigned. This makes it perfect for method chaining.
```python
df = df.assign(
    new_col = lambda x: x['col1'] + x['col2'],
    col1_x10 = lambda x: x['col1'] * 10
)
```
*Note: We use `lambda x:` inside `.assign()` to refer to the DataFrame state at that exact point in the pipeline. This is the equivalent of using `.` or the implicit reference in a `%>%` pipeline.*

**Method 3: Using `df.eval()` (String Expressions)**
Highly valuable for R users, `.eval()` allows you to write calculations as string expressions, similar to base R's `transform()`.
```python
# In-place evaluation
df.eval("new_col = col1 + col2", inplace=True)

# Out-of-place evaluation
df_new = df.eval("new_col = col1 + col2")
```

### Inserting Columns at Specific Positions (`mutate(.after = "col")`)
In R, you can use `.before` or `.after` in `mutate()`. In pandas, you use `df.insert()` to create a column at a specific integer index. This happens in-place.
```python
# Insert 'new_col' at integer index 1 (the second position)
df.insert(loc=1, column='new_col', value=df['col1'] + df['col2'])
```

### The `SettingWithCopyWarning` (A Crucial Difference from R)
This is the most infamous warning new Python users face when coming from R. 
In R, this workflow is perfectly fine:
```R
df2 <- filter(df, x > 0)
df2$new_col <- 1  # Works perfectly
```
In pandas, if you subset a DataFrame and then try to add a column using bracket assignment, you will trigger a `SettingWithCopyWarning`:
```python
df2 = df[df['x'] > 0]
df2['new_col'] = 1  # Triggers SettingWithCopyWarning!
```
Pandas is warning you because it's not sure whether `df2` is a "view" of `df` or a separate copy, so it doesn't know if modifying `df2` will also modify `df`. To fix this, always use `.copy()` when assigning a filtered DataFrame to a new variable:
```python
df2 = df[df['x'] > 0].copy()
df2['new_col'] = 1  # Safe and warning-free
```

## 2. Creating Columns and Dropping Others (`transmute`)

### In R
`transmute()` creates new columns and automatically drops all existing ones.
```R
df_new <- df %>%
  transmute(new_col = col1 + col2)
```

### In Python
Pandas doesn't have a direct single-function equivalent to `transmute()`, but you can easily achieve this either by combining `.assign()` with column selection, or by constructing a new DataFrame.

**Method 1: Creating a new DataFrame**
This is often the cleanest approach if you are creating entirely new calculated columns.
```python
df_new = pd.DataFrame({
    'new_col': df['col1'] + df['col2']
})
```

**Method 2: `.assign()` followed by column selection**
Useful if you are in the middle of a method chain.
```python
df_new = (df
          .assign(new_col = df['col1'] + df['col2'])
          [['new_col']] # Select only the newly created column
         )
```

## 3. Conditional Mutation: Simple If-Else (`if_else`)

### In R
```R
df <- df %>%
  mutate(
    status = if_else(score >= 50, "Pass", "Fail")
  )
```

### In Python
The most direct and performant equivalent to `dplyr::if_else()` or `base::ifelse()` is NumPy's `np.where()`. It is vectorized and very fast.

**Method 1: `np.where()`**
```python
# np.where(condition, value_if_true, value_if_false)
df['status'] = np.where(df['score'] >= 50, 'Pass', 'Fail')
```

**Method 2: Boolean Masking / `.loc`**
You can also use `.loc` to conditionally assign values based on a boolean mask. This is very common in older pandas codebases, though slightly more verbose than `np.where()`.
```python
df['status'] = 'Fail' # Set default value first
df.loc[df['score'] >= 50, 'status'] = 'Pass' # Overwrite where condition is met
```

## 4. Multiple Conditions (`case_when`)

### In R
```R
df <- df %>%
  mutate(
    grade = case_when(
      score >= 90 ~ "A",
      score >= 80 ~ "B",
      score >= 70 ~ "C",
      TRUE ~ "F"
    )
  )
```

### In Python
For multiple conditions, `numpy.select()` is the exact analogue to `case_when()`. You provide a list of conditions and a corresponding list of choices.

**Method 1: `np.select()`**
```python
conditions = [
    df['score'] >= 90,
    df['score'] >= 80,
    df['score'] >= 70
]

choices = ['A', 'B', 'C']

# The 'default' argument acts as the `TRUE ~ "F"` equivalent
df['grade'] = np.select(conditions, choices, default='F')
```

**Method 2: Dictionary Mapping (for exact matches)**
If your `case_when` logic is checking for exact equality (e.g., `category == "X" ~ 1`), pandas `.map()` or `.replace()` is much simpler and more efficient than writing out full conditions.
```python
# R equivalent: case_when(group == "A" ~ 1, group == "B" ~ 2)
mapping = {'A': 1, 'B': 2}
df['group_num'] = df['group'].map(mapping) 
```

**Method 3: `.apply()` with a custom function**
If your logic is too complex for vectorization (e.g., involving strings, lists, or external API calls), you can write a standard Python function and apply it row-by-row. 
*Warning: `.apply(axis=1)` might look like a fast, vectorized `rowwise()`, but it is effectively a slow for-loop under the hood and should only be used as a last resort.*
```python
def assign_grade(row):
    if row['score'] >= 90:
        return 'A'
    elif row['score'] >= 80:
        return 'B'
    elif row['score'] >= 70:
        return 'C'
    else:
        return 'F'

# axis=1 tells pandas to pass rows to the function instead of columns
df['grade'] = df.apply(assign_grade, axis=1)
```

**Method 4: List Comprehensions (Faster than `.apply()`)**
For complex logic, a list comprehension is usually far more performant than `.apply()`.
```python
def assign_grade_scalar(score):
    if score >= 90: return 'A'
    elif score >= 80: return 'B'
    elif score >= 70: return 'C'
    else: return 'F'

df['grade'] = [assign_grade_scalar(x) for x in df['score']]
```

**Method 5: `zip()` for Multi-Column Row-wise Iteration**
If your custom logic requires multiple columns, `zip()` is an extremely Pythonic and fast way to do multi-column row-wise operations.
```python
# Example logic using two columns, assuming f(x, y) is defined
# df['new_col'] = [f(x, y) for x, y in zip(df['col1'], df['col2'])]
```

## 5. Applying Functions Across Multiple Columns (`across`)

### In R
```R
# Round all numeric columns
df <- df %>%
  mutate(across(where(is.numeric), round, 2))

# Multiply specific columns by 10
df <- df %>%
  mutate(across(c(col1, col2), ~ .x * 10))
```

### In Python
Pandas has several ways to apply operations across multiple columns simultaneously, often without needing a special functional wrapper like `across()`. You simply combine column selection techniques with vectorized operations.

**Method 1: Direct Operations on DataFrame Subsets**
For vectorized operations (like math or string methods), you can operate directly on a subset of the DataFrame. This is the fastest and most pythonic way.

```python
# Multiply specific columns by 10
cols = ['col1', 'col2']
df[cols] = df[cols] * 10

# Round all numeric columns (equivalent to: across(where(is.numeric), round, 2))
numeric_cols = df.select_dtypes(include='number').columns
df[numeric_cols] = df[numeric_cols].round(2)

# Apply logic to columns matching a pattern (equivalent to: across(starts_with("test"), ...))
# 1. Select the columns using .filter()
test_cols = df.filter(regex='^test').columns
# 2. Apply the operation
df[test_cols] = df[test_cols] * 100

# Apply logic using complex combinations (equivalent to: across(starts_with("A") | ends_with("B"), ...))
# 1. Build a boolean mask for columns
mask = df.columns.str.startswith('A') | df.columns.str.endswith('B')
complex_cols = df.columns[mask]
# 2. Apply the operation
df[complex_cols] = df[complex_cols].fillna(0)
```

**Method 2: `.apply()` across columns**
For applying functions that take a Series (an entire column) as input.
```python
# Apply a function to specific columns
df[cols] = df[cols].apply(lambda x: x * 10)
```

**Method 3: `.map()` for element-wise application**
If you need to apply a function to every single *element* in a subset of columns. (Note: In pandas versions prior to 2.1.0, this method was called `applymap()`).
```python
# Convert all string columns to uppercase
str_cols = df.select_dtypes(include='object').columns
df[str_cols] = df[str_cols].map(str.upper)
```
