# Chapter 1: The DataFrame Foundation

Welcome to Python! If you are coming from R, you are likely comfortable with `data.frame` or the tidyverse `tibble`. In Python, the equivalent foundation for tabular data is the `pandas.DataFrame`. 

## Creating DataFrames

**In R**, you construct tabular data using `tibble()` or `data.frame()`, and you convert objects using `as_tibble()`.
```R
library(tibble)

df <- tibble(
  id = c(1, 2, 3),
  name = c("Alice", "Bob", "Charlie"),
  age = c(25, 30, 35)
)
```

**In Python**, you use `pandas.DataFrame()`. You can construct a DataFrame from various Python data structures, most commonly dictionaries or lists of lists.

```python
import pandas as pd

# 1. Creating from a dictionary of lists (most similar to tibble())
df = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [25, 30, 35]
})
# Note: pd.DataFrame.from_dict() provides an explicit alternative

# 2. Creating from a list of dictionaries (row-wise creation)
df_from_rows = pd.DataFrame([
    {'id': 1, 'name': 'Alice', 'age': 25},
    {'id': 2, 'name': 'Bob', 'age': 30},
    {'id': 3, 'name': 'Charlie', 'age': 35}
])
# Note: pd.DataFrame.from_records() is another explicit way to build from row-wise data

# 3. Converting existing structures (similar to as_tibble())
# From a NumPy array
import numpy as np
matrix = np.array([[1, 2], [3, 4]])
df_from_matrix = pd.DataFrame(matrix, columns=['col1', 'col2'])
```

## Inspecting the Data

**In R**, you check the structure of your data using `glimpse()`, `str()`, `dim()`, or look at the first few rows with `head()`.
```R
glimpse(df)
head(df, n = 3)
```

**In Python**, data inspection relies on methods and attributes attached to the DataFrame object itself.

```python
# Equivalent to glimpse() or str() - shows rows, columns, data types, and memory usage
df.info()

# Equivalent to dim() - returns a tuple of (rows, columns)
df.shape

# Equivalent to typeof() or checking column classes - returns data types of all columns
df.dtypes

# Equivalent to summary() - provides summary statistics for continuous variables
df.describe()

# Equivalent to names() or colnames() - returns column names
df.columns

# Equivalent to rownames() - returns the row index labels
df.index

# Equivalent to head() - returns the first n rows (defaults to 5)
df.head()
df.head(2)  # specific number of rows

# Equivalent to tail() - returns the last n rows
df.tail(2)

# Equivalent to slice_sample() - returns a random sample of n rows
df.sample(n=2)
```

## The Index

**In R**, users are generally taught to ignore `rownames`, and the `tidyverse` actively discourages their use. 

**In Python**, the `Index` is a first-class citizen in pandas. Every DataFrame (and Series) has an index. By default, it is a sequence of integers starting from `0` (Python uses 0-based indexing, unlike R's 1-based indexing). The index is crucial because pandas uses it for **alignment** during operations. If your index becomes disorganized after filtering or joining, you can reset it to a clean 0-based sequence using `df.reset_index(drop=True)`.

## Extracting Columns: Series vs. DataFrame

**In R**, the way you extract a column determines what data structure you get back. `df$name` or `df[["name"]]` returns a vector, whereas `df["name"]` returns a 1-column data.frame/tibble.

**In Python**, a similar distinction exists. A 2-dimensional table is a `DataFrame`, but a single 1-dimensional column extracted from it is a `Series`. A Series is fundamentally different from a DataFrame (it has no column names, just a single name for the series, and an index).

```python
# --- Extracting a Series (Vector equivalent) ---

# 1. Standard bracket notation (Primary & Safest)
# Best practice, works with spaces or special characters in column names
names_series = df['name']
type(names_series)  # <class 'pandas.core.series.Series'>

# 2. Dot notation (Alternative)
# Quick to type, but fails if column name has spaces or shares a name with a DataFrame method (e.g., df.count)
names_series_dot = df.name

# 3. .loc extraction (Alternative)
names_series_loc = df.loc[:, 'name']

# 4. .iloc extraction by position (Alternative)
# Python uses 0-based indexing! The first column is index 0, the second is 1.
names_series_iloc = df.iloc[:, 1]


# --- Extracting a DataFrame (1-column Data Frame equivalent) ---

# Pass a list of columns inside the brackets (notice the double brackets)
names_dataframe = df[['name']]
type(names_dataframe)  # <class 'pandas.core.frame.DataFrame'>

# This is also how you select multiple columns
subset_df = df[['id', 'age']]
```

**Why this matters:** Many pandas methods behave differently (or only exist) on a `Series` versus a `DataFrame`. When chaining methods or performing calculations, always be mindful of whether your extraction resulted in a 1D Series or a 2D DataFrame.
