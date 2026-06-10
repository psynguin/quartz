# Chapter 3: Selecting and Extracting Columns (Column Verbs)

In R's `dplyr` package, managing columns revolves largely around a few core functions: `select()` (for choosing or omitting columns), `pull()` (for extracting a single column as a vector), `rename()` (for renaming columns), and `relocate()` (for reordering columns). Tidy-select helpers like `starts_with()`, `ends_with()`, and `contains()` make choosing groups of columns straightforward.

In Python's `pandas`, you will use a mix of traditional bracket indexing `[]`, dot notation, and specific dataframe methods like `.filter()`, `.rename()`, and `.insert()`. 

Let's explore how to translate these R column operations into Python.

## 1. `select()`: Selecting Columns

In R, you select columns using `select(data, col1, col2)`.

### Traditional Bracket Notation in pandas
The most idiomatic way to select columns in pandas is using bracket notation. To select a single column, you pass the string name. To select multiple columns, you pass a *list* of string names.

```python
import pandas as pd
import numpy as np

# Sample data
df = pd.DataFrame({
    'id': [1, 2, 3],
    'first_name': ['Alice', 'Bob', 'Charlie'],
    'last_name': ['Smith', 'Jones', 'Brown'],
    'age': [25, 30, 35],
    'salary': [50000, 60000, 70000]
})

# R: select(df, first_name)
# Python: Select a single column as a Series
df['first_name']

# Python: Select a single column as a DataFrame (notice the double brackets)
df[['first_name']]

# R: select(df, first_name, last_name)
# Python: Select multiple columns (pass a list of columns)
df[['first_name', 'last_name']]
```

### The `.loc` accessor
You can also use `.loc`, which is technically for label-based selection of both rows and columns. When selecting only columns, use `:` for the row indexer.

```python
# Select multiple columns using .loc
df.loc[:, ['first_name', 'last_name']]

# R: select(df, first_name:age)
# Python: Select a range of columns using slice notation
df.loc[:, 'first_name':'age']

> **Note for R users:** In Python, standard slicing (like `0:3`) is typically *exclusive* of the end element. However, slicing with labels via `.loc` is an exception—it is **inclusive** at the end. Thus, `df.loc[:, 'first_name':'age']` will *include* the `age` column. This aligns perfectly with how R's `first_name:age` inclusive selection behaves!
```

### The `.iloc` accessor (by position)
If you need to select by column index (position), use `.iloc`.

```python
# R: select(df, 1, 3) (1-indexed)
# Python: 0-indexed column selection
df.iloc[:, [0, 2]]

# Select columns 1 through 3
df.iloc[:, 1:4]
```

### Dropping Columns (Negative Selection)

In R, you use a minus sign: `select(df, -salary)`.

In pandas, you use the `.drop()` method.

```python
# R: select(df, -salary)
# Python: Drop a single column
df.drop(columns=['salary'])
# alternatively: df.drop('salary', axis=1)

# R: select(df, -c(age, salary))
# Python: Drop multiple columns
df.drop(columns=['age', 'salary'])

### Alternative Methods for Negative Selection
You will also see these alternative methods for dropping or negatively selecting columns in Python code:

```python
# 1. The built-in `del` keyword drops a column entirely in-place (common in older Python code):
del df['salary']

# 2. Using Index difference:
df[df.columns.difference(['age', 'salary'])]

# 3. Boolean negation (~ acts as NOT) using .isin():
df.loc[:, ~df.columns.isin(['age', 'salary'])]
```
```

## 2. Advanced Tidy-Select Logic (`starts_with`, `matches`, `where`)

In R, you often use helpers like `starts_with()`, `ends_with()`, `matches()`, and `where()` to logically query and grab specific groups of columns. 

In pandas, you achieve this using a combination of the `.filter()` method, Pandas `.columns.str` accessors, and data type selection.

### Basic Pattern Matching: `.filter()`

For simple string containment or regex matching, `.filter()` is the most direct analogue to `starts_with`, `ends_with`, and `matches`.

```python
# R: select(df, starts_with("first"))
df.filter(regex='^first') # strict starts_with equivalent using regex

# R: select(df, ends_with("name"))
df.filter(regex='name$')

# R: select(df, contains("_"))
df.filter(like='_') # 'like' checks if the string is contained anywhere in the name

# R: select(df, matches("v[1-3]"))
df.filter(regex='v[1-3]')
```

### Complex Logical Selection (OR / AND logic)

If you need to combine conditions, such as columns that start with "A" *or* end with "B", R allows `select(df, starts_with("A") | ends_with("B"))`. 

In Python, you can use the Pandas `Index.str` methods to build boolean masks for the columns, which allows complex logic using `|` (OR), `&` (AND), and `~` (NOT).

```python
# R: select(df, starts_with("first") | ends_with("id"))
# Python: Build a boolean mask for columns
mask = df.columns.str.startswith('first') | df.columns.str.endswith('id')

# Apply mask using .loc (row indexer is :, column indexer is the mask)
df.loc[:, mask]

# R: select(df, starts_with("first") & !contains("name"))
mask2 = df.columns.str.startswith('first') & ~df.columns.str.contains('name')
df.loc[:, mask2]
```

### Type-based Selection: `where()`

In R, `select(df, where(is.numeric))` chooses columns based on their data type.

In Python, you use the `.select_dtypes()` method.

```python
# R: select(df, where(is.numeric))
df.select_dtypes(include='number') # or include=['int64', 'float64']

# R: select(df, where(is.character) | where(is.factor))
df.select_dtypes(include=['object', 'category'])

# R: select(df, !where(is.numeric))
df.select_dtypes(exclude='number')
```

### List Comprehensions (A Pythonic Alternative)
Sometimes, python users prefer generating a list of columns using list comprehensions and passing that to the bracket notation.

```python
# Select columns that start with 'first'
cols = [c for c in df.columns if c.startswith('first')]
df[cols]

# Select columns of a specific data type
# R: select_if(df, is.numeric)
# Python:
df.select_dtypes(include=['number']) # or 'int64', 'float64', 'object'
```

## 3. `pull()`: Extracting a Single Column as a Vector/Series

In R, `pull(df, age)` extracts the column as a vector rather than a 1-column dataframe.

In pandas, extracting a column as a pandas `Series` is the default behavior when you use single brackets or dot notation. 

```python
# R: pull(df, age)
# Python: Single brackets return a Series
df['age']

# Python: Dot notation returns a Series (only works if column name has no spaces and doesn't conflict with DataFrame methods)
df.age
```

If you literally need a NumPy array (the closest equivalent to an R atomic vector):

```python
# Get the underlying numpy array
df['age'].values
# or
df['age'].to_numpy()
```

If you want to extract a column *and remove it* from the dataframe simultaneously, use `.pop()`:

```python
# Extracts 'salary' as a Series and modifies df in-place to remove 'salary'
salary_series = df.pop('salary')
```

### Safe Extraction: `df.get()`
If you want to extract a column that might not exist without causing an error, use `df.get()`. This is analogous to `purrr::pluck()` or using `dplyr::any_of()`:

```python
# Returns None if the column does not exist
df.get('missing_column')

# Or specify a default value to return
df.get('missing_column', default=pd.Series([]))
```

## 4. `rename()`: Renaming Columns

In R, you use `rename(df, new_name = old_name)`.

In pandas, you use the `.rename()` method with a dictionary mapping `{old_name: new_name}`.

```python
# R: rename(df, emp_id = id, given_name = first_name)
# Python:
df.rename(columns={'id': 'emp_id', 'first_name': 'given_name'})

# Alternatively, you can use axis=1
df.rename({'id': 'emp_id'}, axis=1)
```

### Renaming all columns at once

In R, you might use `names(df) <- c(...)` or `setNames()`.

```python
# R: names(df) <- c("col1", "col2", ...)
# Python: Assign a list directly to df.columns
df.columns = ['ID', 'First', 'Last', 'Age', 'Salary']

# Convert all columns to uppercase
df.columns = df.columns.str.upper()

# Replace spaces with underscores
df.columns = df.columns.str.replace(' ', '_')
```

## 5. `relocate()`: Reordering Columns

In R, `relocate(df, age, .before = first_name)` moves columns around easily.

Pandas does not have a direct, single-function equivalent to `relocate()` with `.before` and `.after` arguments. Instead, you typically reorder columns by passing a complete list of columns in the desired order.

```python
# Suppose our columns are ['id', 'first_name', 'last_name', 'age', 'salary']

# R: select(df, age, everything()) OR relocate(df, age)
# Python: Reorder by providing a list
new_order = ['age', 'id', 'first_name', 'last_name', 'salary']
df[new_order]

# Alternatively, you can use .reindex() to select and arrange columns:
df.reindex(columns=new_order)
```

### A programmatic approach to `relocate()`
If you have many columns and just want to move one to the front without typing them all out, you can manipulate the list of column names:

```python
# Move 'age' to the front
cols = list(df.columns)
cols.remove('age')
cols.insert(0, 'age')
df[cols]
```

### Inserting a new column at a specific location
If you are creating a new column and want to place it at a specific position, you can use `.insert()`. This operates *in-place*.

```python
# R: mutate(df, initial = substr(first_name, 1, 1), .after = first_name)
# Python: Insert 'initial' at index 2 (0-indexed)
df.insert(2, 'initial', df['first_name'].str[0])
```

## Summary of Method Chaining

You can chain these methods together similarly to the `%>%` or `|>` pipe in R.

```python
# R:
# df %>%
#   rename(emp_id = id) %>%
#   select(-salary)
#   # (Row filtering comes in the next chapter)

# Python:
(df
 .rename(columns={'id': 'emp_id'})
 .drop(columns=['salary'])
 .filter(regex='^(?!last).*') # Keep columns that do NOT start with 'last'
)
```

With pandas, column selection is very straightforward with `[]` brackets, while methods like `.rename()` and `.drop()` provide the functional equivalents to R's `dplyr` verbs.
