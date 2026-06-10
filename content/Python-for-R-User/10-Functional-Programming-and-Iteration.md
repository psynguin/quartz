# Chapter 10: Functional Programming and Iteration

In R, particularly within the Tidyverse, you are generally encouraged to avoid `for` loops. Instead, you rely on the `purrr` package (`map()`, `walk()`, `reduce()`) or base R's `apply` family to iterate over lists and dataframes. 

In Python, the philosophy is slightly different. While Python does have functional tools like `map()` and `reduce()`, the "Pythonic" way strongly prefers **list comprehensions** and standard `for` loops. When working with tabular data in Pandas, vectorized operations are the gold standard, backed by `.apply()` or `.map()` when custom functions are needed.

Let's look at how to translate your `purrr` skills into Python.

## 1. Mapping Functions over Lists (`purrr::map`)

### The R Concept
In R, if you have a list or vector and want to apply a function to each element, you use `map()` (which returns a list) or its typed variants like `map_dbl()` or `map_chr()`.
```R
# R: Mapping a function over a vector
numbers <- c(1, 2, 3, 4)
squared_list <- map(numbers, ~ .x^2)
squared_dbl <- map_dbl(numbers, ~ .x^2)
```

### The Python Focus
In Python, you have three main ways to accomplish this: list comprehensions, the built-in `map()` function, and standard `for` loops.

**1. List Comprehensions (Most Pythonic)**
List comprehensions are the direct, highly readable equivalent to `map()`. They return a list and are typically faster than `map()`.
```python
numbers = [1, 2, 3, 4]

# Equivalent to map_dbl (returns a list of numbers)
squared = [x**2 for x in numbers]
# [1, 4, 9, 16]

# You can also add conditional filtering (like purrr::keep) inside the comprehension
squared_evens = [x**2 for x in numbers if x % 2 == 0]
# [4, 16]
```

**2. The built-in `map()` function**
Python has a built-in `map()` function. Note that it returns a lazy map object (an iterator), so you must wrap it in `list()` to evaluate it immediately, similar to R's lazy/eager evaluation nuances.
```python
# Using a lambda function (Python's equivalent to the ~ .x syntax)
squared_map = list(map(lambda x: x**2, numbers))
# [1, 4, 9, 16]
```

**3. Dictionary Comprehensions**
If you need to iterate and return key-value pairs (similar to creating named lists in R), Python has dictionary comprehensions:
```python
square_dict = {x: x**2 for x in numbers}
# {1: 1, 2: 4, 3: 9, 4: 16}
```

## 2. Iterating inside DataFrames

### The R Concept
In R, you often use `mutate()` combined with `map()` or `rowwise()` to apply custom functions to columns or rows.
```R
# R: Column-wise mapping
df %>% mutate(new_col = map_dbl(old_col, custom_func))

# R: Row-wise mapping
df %>% 
  rowwise() %>% 
  mutate(sum_cols = sum(c_across(col1:col3)))
```

### The Python Focus
In Pandas, **vectorization is always your first choice**. If a vectorized function exists (like `df['A'] + df['B']`), use it. When you must apply a custom, non-vectorized function, use `.apply()` or `.map()`.

**1. Series `.apply()` (Element-wise on a single column)**
This is your direct equivalent to `mutate(new_col = map(old_col, func))`.
```python
import pandas as pd

df = pd.DataFrame({'A': [1, 2, 3], 'B': [10, 20, 30]})

def custom_math(x):
    return (x * 2) + 15

# Apply custom function to every element in column 'A'
df['A_custom'] = df['A'].apply(custom_math)

# With a lambda function (inline)
df['A_squared'] = df['A'].apply(lambda x: x**2)
```

**2. DataFrame `.apply()` (Row-wise or Column-wise)**
When you call `.apply()` on an entire DataFrame, you can pass `axis=1` to apply the function row-by-row (like R's `rowwise()`), or `axis=0` to apply it column-by-column.
```python
# Row-wise iteration (axis=1). The lambda function receives an entire row as a Series.
df['Row_Sum'] = df.apply(lambda row: row['A'] + row['B'], axis=1)

# Column-wise iteration (axis=0). The lambda function receives an entire column.
col_sums = df.apply(lambda col: col.sum(), axis=0)
```

> [!WARNING]
> R users might mistake `.apply(axis=1)` for a fast, vectorized `rowwise()`. In reality, `.apply(axis=1)` is effectively a slow `for` loop under the hood and should only be used as a last resort when true vectorization isn't possible.

If you absolutely *must* loop over a DataFrame's rows and `.apply()` is too slow, `.itertuples()` is the most performant way to do it:
```python
for row in df.itertuples():
    # Access elements by attribute: row.A
    pass
```

**3. `np.vectorize()` (Speeding up custom functions)**
If you have a custom Python function that works on single values and `.apply()` is too slow, `np.vectorize()` can convert it into a vectorized function that acts quickly on Pandas series. This bridges the gap between `.apply()` and true vectorization.
```python
import numpy as np

# Convert custom function to a vectorized one
vectorized_math = np.vectorize(custom_math)

# Apply it quickly to a column
df['A_custom'] = vectorized_math(df['A'])
```

**4. Series `.map()` (Value Substitution)**
For mapping a dictionary or a function to substitute values, `.map()` is often used on Series.
```python
mapping_dict = {1: 'One', 2: 'Two', 3: 'Three'}
df['A_mapped'] = df['A'].map(mapping_dict)
```

## 3. Side Effects (`purrr::walk`)

### The R Concept
In R, `walk()` is used when you want to call a function for its side effects (like printing, writing to disk, or plotting) without returning a transformed list.
```R
# R: Printing each element
walk(list_of_files, print)
```

### The Python Focus
Python does not have a strict `walk()` equivalent because **standard `for` loops** are the idiomatic way to handle side effects. Python loops are not discouraged or heavily penalized in performance like they historically were in R.

```python
files = ['file1.csv', 'file2.csv', 'file3.csv']

# The Pythonic equivalent to walk()
for f in files:
    print(f"Processing {f}...")
    # write to disk, generate plot, etc.
```

## 4. Accumulating / Reducing (`purrr::reduce`)

### The R Concept
`reduce()` iteratively combines elements of a list into a single value or object. A common use case in R is iteratively joining a list of dataframes.
```R
# R: Reducing a list of dataframes into one by joining
list_of_dfs %>% reduce(full_join, by = "id")
```

### The Python Focus
Python has `functools.reduce()`, which operates exactly the same way.

**1. Using `functools.reduce`**
```python
from functools import reduce

numbers = [1, 2, 3, 4]

# Multiply all numbers together sequentially: (((1*2)*3)*4)
product = reduce(lambda x, y: x * y, numbers)
# 24
```

**2. Reducing (Merging) Multiple DataFrames**
Just like in R, `reduce` is brilliant for merging a list of Pandas DataFrames on a common key.
```python
df1 = pd.DataFrame({'id': [1, 2], 'val1': ['A', 'B']})
df2 = pd.DataFrame({'id': [1, 2], 'val2': ['C', 'D']})
df3 = pd.DataFrame({'id': [1, 2], 'val3': ['E', 'F']})

dfs = [df1, df2, df3]

# Equivalent to reduce(full_join, dfs, by = "id")
merged_df = reduce(lambda left, right: pd.merge(left, right, on='id', how='outer'), dfs)
```
*(Note: If you are simply concatenating rows or columns, `pd.concat(dfs)` is much faster and preferred over `reduce()`.)*

## 5. Returning and Binding DataFrames (`purrr::map_dfr`)

### The R Concept
R users frequently use `map_dfr()` to iterate over a list of items (like file paths), apply a function that returns a dataframe, and bind them all into a single dataframe.
```R
# R: Read and bind multiple CSVs
combined_df <- map_dfr(files, read_csv)
```

### The Python Focus
The most Pythonic and efficient way to accomplish this is to combine a list comprehension with `pd.concat()`.
```python
# Python: Read and bind multiple CSVs
files = ['data1.csv', 'data2.csv', 'data3.csv']
combined_df = pd.concat([pd.read_csv(f) for f in files], ignore_index=True)
```

## 6. Grid Expansion and Chains (`tidyr::expand_grid` / `purrr`)

### The R Concept
In R, you might use `expand_grid()` or `purrr::cross()` to generate combinations of elements.

### The Python Focus
In Python, the `itertools` module from the standard library handles these operations efficiently.

```python
import itertools

# Equivalent to tidyr::expand_grid or purrr::cross2
combinations = list(itertools.product(['A', 'B'], [1, 2]))
# [('A', 1), ('A', 2), ('B', 1), ('B', 2)]

# Equivalent to unlisting a list of lists (flattening)
nested_list = [[1, 2], [3, 4]]
flattened = list(itertools.chain.from_iterable(nested_list))
# [1, 2, 3, 4]
```

## Summary of Translations

| R (`purrr` / `dplyr`) | Python Equivalent |
| :--- | :--- |
| `map(list, func)` | `[func(x) for x in lst]` or `list(map(func, lst))` |
| `map_dbl(list, func)` | `[func(x) for x in lst]` (Python lists can be mixed types) |
| `map_dfr(list, func)` | `pd.concat([func(x) for x in lst])` |
| `keep(list, condition)` | `[x for x in lst if condition(x)]` |
| `walk(list, func)` | `for x in lst: func(x)` |
| `reduce(list, func)` | `functools.reduce(func, lst)` |
| `expand_grid(x, y)` | `list(itertools.product(x, y))` |
| `mutate(A = map_dbl(B, func))`| `df['A'] = df['B'].apply(func)` |
| `rowwise() %>% mutate(...)` | `df.apply(lambda row: func(row), axis=1)` |

While `map()`, `.apply()`, and `reduce()` are available in Python, remember the Zen of Python: readability counts. List comprehensions and `for` loops are often the most highly regarded solutions for iteration outside of Pandas vectorized operations.
