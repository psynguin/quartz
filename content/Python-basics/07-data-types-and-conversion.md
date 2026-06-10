---
title: "Data Types, Type Conversion, and Specialized Accessors"
tags: [python, data-handling, pandas, numpy]
chapter: 7
theme: "Data Types, Type Conversion, and Specialized Accessors"
---

Data types are the foundation of accurate data handling. A pandas column behaves differently depending on its data type: string columns offer text-processing methods, numeric columns offer mathematical operations, and datetime columns allow for time-based calculations. Understanding data types, converting between them efficiently, and using specialized accessors (`.str`, `.dt`, and `.cat`) are critical skills for cleaning datasets.

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

## Core Data Types and Inspection

### Concept Explanation
Pandas relies heavily on NumPy's data types but extends them. A DataFrame's columns can have distinct types, and knowing these types dictates which operations are available. 

- `int64`, `float64`: Standard 64-bit integers and floating-point numbers.
- `object`: Often represents strings or mixed types. Since it's an array of pointers to arbitrary Python objects, it is slow and memory-intensive.
- `bool`: True/False values.
- `datetime64[ns]`: Date and time values with nanosecond precision.
- `category`: A specialized pandas type for categorical variables with a limited, fixed number of possible values.
- **Nullable Integer Types** (`Int8`, `Int16`, `Int32`, `Int64`): Standard NumPy integers cannot represent missing values (`NaN`). Pandas introduced capitalized integer types that natively support missing values without converting the whole column to `float64`.
- **StringDtype** (`string`): A dedicated string type designed to behave more consistently than `object`.

> [!TIP] Mental Model
> Think of `object` as a safety net that pandas uses when it isn't completely sure what the data is. One of your first steps in data cleaning should be converting `object` columns to more specific types (`category`, `string`, `datetime64`, or numeric).

### Inspection Methods

Before applying any transformations, you must verify that your columns have the correct types.

```python
# Method 1: The .dtypes attribute (quickest way to check types programmatically)
print(df.dtypes)

# Method 2: The .info() method (interactive overview with memory usage and null counts)
df.info()

# Method 3: Selecting columns by type
numeric_df = df.select_dtypes(include=['int64', 'float64'])
text_df = df.select_dtypes(include=['object'])
```

**When to Use**:
*   Use `.dtypes` in a script or condition to programmatically check a column's type.
*   Use `.info()` interactively as the first step when loading a new dataset.
*   Use `.select_dtypes()` when you want to apply an operation only to specific types of columns.

### Common Errors and Gotchas

> [!WARNING] Mixed Types in Object Columns
> **Error/Gotcha**: You run `.dtypes` and see a column is `object`. You assume it's entirely strings, but operations fail randomly.
> **Why it happens**: `object` columns can contain a mix of strings, integers, and floats. Pandas falls back to `object` if even a single value is of a different type.
> **Fix**: Inspect mixed types using `df['col'].apply(type).value_counts()` and convert them explicitly using `.astype()` or `pd.to_numeric()`.

### Applying to Multiple DataFrames

Check for consistent types across collections of DataFrames:

```python
# Dict-based pattern: check if all dfs have exactly the same types
types_match = all(dfs['january'].dtypes.equals(d.dtypes) for d in dfs.values())

# List-based pattern: extract only numeric columns from each df
dfs_list = list(dfs.values())
numeric_dfs = [d.select_dtypes(include='number') for d in dfs_list]
```

> [!NOTE] Bulk Operations Caveat
> When checking types in bulk, be aware that `.equals()` checks for exact matches in column order as well as types. If one DataFrame has columns in a different order, `.equals()` will return `False` even if the schemas match.

## Type Conversion

### Concept Explanation
Sometimes pandas guesses the wrong data type when loading data, typically because a numeric column contains a stray string or missing value. Type conversion forces the data into the correct memory representation, unlocking specific methods and mathematical operations.

### Methods Comparison

| Method | Handles invalid strings | Memory Downcasting | Best For |
|--------|-------------------------|--------------------|----------|
| `.astype()` | Raises Error | Explicit via type name | Clean data, fast direct casting, dict-based multi-column |
| `pd.to_numeric()` | Yes (`errors='coerce'`) | Yes (`downcast=`) | Messy numeric strings, memory optimization |
| `pd.to_datetime()` | Yes (`errors='coerce'`) | N/A | Parsing dates with specific formats |

### Using `.astype()`

The simplest way to cast a column to a new type is using the `.astype()` method.

```python
# Convert a single column
df['units_sold'] = df['units_sold'].astype('float64')

# Convert multiple columns at once using a dictionary
df = df.astype({
    'units_sold': 'int64',
    'region': 'category'
})
```

### Safe Numeric Conversion and Downcasting

When `.astype()` fails due to messy data, use `pd.to_numeric()`. It safely converts data and can downcast numeric types to save memory.

```python
# Create a messy series
messy_series = pd.Series(['10', '20', 'Error', '40'])

# errors='coerce' forces invalid strings to NaN
clean_numeric = pd.to_numeric(messy_series, errors='coerce')

# Memory downcasting: float64 to float32
df['unit_price'] = pd.to_numeric(df['unit_price'], downcast='float')

# Memory downcasting: int64 to the smallest possible integer type (e.g., int8)
df['units_sold'] = pd.to_numeric(df['units_sold'], downcast='integer')
```

### Common Errors and Gotchas

> [!WARNING] ValueError with `.astype()`
> **Error/Gotcha**: You try to convert a column containing non-numeric characters (like `"N/A"`) to an integer using `.astype()`, and it raises `ValueError: could not convert string to float: 'N/A'`.
> **Why it happens**: `.astype()` is strict and does not know how to handle invalid strings.
> **Fix**: Use `pd.to_numeric(df['col'], errors='coerce')` to safely force invalid values to `NaN`.

### Date and Time Conversion

Converting strings to datetimes requires `pd.to_datetime()`.

```python
# Method 1: Let pandas infer the format (convenient but slower)
pd.to_datetime(['2024-01-05', '2024-01-12'])

# Method 2: Explicit format (much faster and safer)
pd.to_datetime(['05/01/2024', '12/01/2024'], format='%d/%m/%Y')

# Converting to timedelta
pd.to_timedelta(['1 days', '2 days 05:00:00'])
```

**When to Use**:
*   Use `pd.to_datetime()` without `format` (Method 1) when exploring data interactively and you trust pandas to infer correctly, or when formats are highly mixed.
*   Use `pd.to_datetime()` with `format` (Method 2) for production pipelines and large datasets. Explicit formatting is significantly faster and prevents ambiguous parsing errors (e.g., confusing mm/dd/yyyy with dd/mm/yyyy).

### Specifying Types on Load

The most efficient way to handle types is to specify them when reading the data, preventing pandas from allocating unnecessary memory.

```python
# Specify dtypes directly during I/O
# df_loaded = pd.read_csv('data.csv', dtype={'store_id': 'category', 'units_sold': 'Int32'})
```

### Applying to Multiple DataFrames

When preparing a collection of DataFrames for concatenation, you must ensure their types align perfectly.

```python
# Dict-based pattern: cast types consistently across all DataFrames
cast_dict = {'units_sold': 'Int64', 'discount': 'float32'}
dfs_consistent = {k: d.astype(cast_dict) for k, d in dfs.items()}

# List-based pattern: safely convert datetime column in all DataFrames
dfs_list_dates = [
    d.assign(date=pd.to_datetime(d['date'], errors='coerce')) 
    for d in list(dfs.values())
]
```

## Categorical Data

The `category` dtype is one of pandas' most powerful features for memory optimization and logical ordering.

### Concept Explanation
Use `category` when a column contains a limited number of unique string values (e.g., 'North', 'South', 'East'). Under the hood, pandas stores each unique string only once, and uses small integers (codes) to represent the data in the array. This drastically reduces memory usage and speeds up grouping and sorting operations.

### Creating Categoricals

```python
# Method 1: Using astype (quick and simple)
df['region'] = df['region'].astype('category')

# Method 2: Using pd.Categorical
df['region'] = pd.Categorical(df['region'])

# Method 3: Using pd.CategoricalDtype to enforce specific categories and order
from pandas.api.types import CategoricalDtype

size_type = CategoricalDtype(categories=['Small', 'Medium', 'Large'], ordered=True)
# df['size'] = df['size'].astype(size_type)
```

**When to Use**:
*   Use `.astype('category')` for quick memory savings on simple, unordered string columns.
*   Use `pd.Categorical()` when you need to create a categorical array directly from a list or NumPy array outside of a DataFrame.
*   Use `CategoricalDtype` when you need strict control over the allowed categories and their logical ordering, especially when preparing data for machine learning or when concatenating multiple DataFrames later.

### The `.cat` Accessor

Once a column is categorical, you unlock the `.cat` accessor to manipulate the categories themselves.

```python
# Ensure categorical
df['store_id'] = df['store_id'].astype('category')

# View the categories and underlying integer codes
print(df['store_id'].cat.categories)
print(df['store_id'].cat.codes)

# Add a new category (required before assigning a new unobserved value)
df['store_id'] = df['store_id'].cat.add_categories(['S04'])

# Rename existing categories
df['store_id'] = df['store_id'].cat.rename_categories({'S01': 'Store_1', 'S02': 'Store_2'})

# Set categories entirely (drops any categories not in the new list)
df['store_id'] = df['store_id'].cat.set_categories(['Store_1', 'Store_2', 'S03', 'S04'])

# Remove a specific category (values belonging to it become NaN)
df['store_id'] = df['store_id'].cat.remove_categories(['S03'])

# Remove all categories that have no instances in the data
df['store_id'] = df['store_id'].cat.remove_unused_categories()
```

### Ordered Categoricals and Sorting

Ordered categories allow you to sort strings logically rather than alphabetically.

```python
months = ['January', 'February', 'March']
month_type = CategoricalDtype(categories=months, ordered=True)

s = pd.Series(['March', 'January', 'February']).astype(month_type)

# Sorting respects the logical categorical order
print(s.sort_values())
```

### Common Errors and Gotchas

> [!WARNING] Modifying Categorical Values
> You cannot assign a value to a categorical column if that value is not in the `.categories` list.
> ```python
> # df['region'] = df['region'].astype('category')
> # df.loc[0, 'region'] = 'West' # ValueError: Cannot setitem on a Categorical with a new category
> ```
> **Fix**: Add the category first using `.cat.add_categories(['West'])`.

> [!WARNING] Merging Categoricals
> If you merge or concatenate two DataFrames with categorical columns, but their categories or orders don't match exactly, pandas will silently cast the resulting column back to `object` type, destroying your memory optimizations. Mathematical arithmetic on categoricals is also generally unsupported.

### Applying to Multiple DataFrames

To avoid the concatenation pitfall mentioned above, always enforce the exact same `CategoricalDtype` across all DataFrames.

```python
# Define the global categorical schema
cat_type = CategoricalDtype(categories=['Electronics', 'Clothing', 'Food'], ordered=False)

# Dict-based pattern: Convert to category safely across a dict of dfs
dfs_cat = {k: d.assign(category=d['category'].astype(cat_type)) 
           for k, d in dfs.items()}

# List-based pattern: Convert to category across a list of dfs
dfs_list_cat = [d.assign(category=d['category'].astype(cat_type)) for d in list(dfs.values())]
```

> [!NOTE] Bulk Operations Caveat
> When converting columns to categorical in bulk, any value present in the data that is not defined in the global `CategoricalDtype` categories will be silently converted to `NaN`. Always validate your categories exhaustively before applying them across multiple DataFrames.

## String Operations via `.str` Accessor

### Concept Explanation
Pandas provides the `.str` accessor to apply string manipulation methods element-wise to an entire column. This avoids the need for slow `for` loops or `.apply()`. It brings standard Python string methods natively to DataFrame columns.

### Basic String Transformations

```python
# Lower, upper, and title case
df['category'].str.lower()
df['category'].str.upper()
df['category'].str.title()

# Stripping whitespace (useful for messy inputs)
noisy = pd.Series(['  Apple  ', 'Banana\n', '\tCherry'])
noisy.str.strip()   # Removes from both ends
noisy.str.lstrip()  # Removes from left only
noisy.str.rstrip()  # Removes from right only
```

### Searching and Replacing

```python
# Check if string contains a substring (supports regex)
df['category'].str.contains('tron')

# Exact start/end matching
df['category'].str.startswith('Elec')
df['category'].str.endswith('ing')

# Replace substring (use regex=False for literal replacement to avoid warnings)
df['category'].str.replace('Electronics', 'Tech', regex=False)

# Replace with regex (e.g., remove any digits)
df['category'].str.replace(r'\d+', '', regex=True)
```

### Splitting and Extracting

```python
names = pd.Series(['Smith, John', 'Doe, Jane'])

# Split into a list of strings
names.str.split(', ')

# Split and expand into separate DataFrame columns
split_df = names.str.split(', ', expand=True)
split_df.columns = ['Last Name', 'First Name']

# Extracting via Regex Capture Groups
product_codes = pd.Series(['PROD-123-A', 'PROD-456-B'])
product_codes.str.extract(r'PROD-(\d+)-([A-Z])')
```

### String Properties and Concatenation

```python
# Length of each string
df['store_id'].str.len()

# Count occurrences of a specific character/pattern
df['category'].str.count('o')

# Find position of substring (-1 if not found)
df['category'].str.find('thing')

# Get character at specific index
df['store_id'].str.get(0)

# Concatenate strings across rows
df['store_id'].str.cat(df['category'], sep=' - ')
```

**When to Use**:
*   Use standard `.str` methods (like `.str.lower()` or `.str.replace(regex=False)`) for simple, literal string manipulation.
*   Use regex-enabled methods (`.str.extract()`, `.str.contains()`, `.str.replace(regex=True)`) when the data contains complex, variable patterns (e.g., extracting ID numbers from free-text).

### Common Errors and Gotchas

> [!WARNING] `.str` Accessor on Missing Values
> **Error/Gotcha**: String methods like `.str.contains()` return `NaN` for missing values, not `False`. This breaks boolean indexing (`ValueError: Cannot mask with non-boolean array containing NA / NaN values`).
> **Why it happens**: The result of `NaN` contains 'x' is undefined (`NaN`), which is not a valid boolean.
> **Fix**: Use the `na=False` parameter inside the string method: `df[df['category'].str.contains('Tech', na=False)]`.

### Applying to Multiple DataFrames

```python
# Dict-based pattern: apply a string replacement across all DataFrames
dfs_clean_strings = {
    k: d.assign(store_id=d['store_id'].str.replace('S', 'Store_', regex=False)) 
    for k, d in dfs.items()
}

# List-based pattern: apply a string replacement across a list of DataFrames
dfs_list_strings = [
    d.assign(store_id=d['store_id'].str.replace('S', 'Store_', regex=False)) 
    for d in list(dfs.values())
]
```

> [!NOTE] Bulk Operations Caveat
> When applying string operations in bulk, ensure that the target columns are actually of type `object` or `string` in every DataFrame. If pandas parsed an entirely null column as `float64` in one file, calling `.str` on it will raise an `AttributeError`.

## Datetime Operations via `.dt` Accessor

### Concept Explanation
The `.dt` accessor exposes datetime properties and methods for `datetime64` columns, allowing you to extract components or shift time efficiently. This enables complex time-series analysis without needing manual parsing.

### Extracting Date Components

```python
# Ensure the column is a datetime
df['date'] = pd.to_datetime(df['date'])

# Extracting components
df['date'].dt.year
df['date'].dt.month
df['date'].dt.day
df['date'].dt.hour
df['date'].dt.minute
df['date'].dt.second

# Useful derived properties
df['date'].dt.dayofweek   # 0 = Monday, 6 = Sunday
df['date'].dt.day_name()  # 'Monday', 'Tuesday', etc.
df['date'].dt.is_month_end

# Normalize removes the time component, setting it to midnight
df['date'].dt.normalize()

# Extract just the date or time part (returns object type, not datetime64)
df['date'].dt.date
df['date'].dt.time
```

### Timedelta Arithmetic

You can add or subtract datetimes to create `timedelta64` objects, which have their own `.dt` accessor.

```python
# Create a shipping date 3 days after the order date
df['ship_date'] = df['date'] + pd.to_timedelta('3 days')

# Calculate the difference between dates
df['processing_time'] = df['ship_date'] - df['date']

# Extract the integer number of days from the timedelta
df['processing_days'] = df['processing_time'].dt.days
```

### Timezone Handling

By default, pandas datetimes are "timezone-naive". You can localize them and convert between zones.

```python
# Localize to UTC
utc_dates = df['date'].dt.tz_localize('UTC')

# Convert to a different timezone
ny_dates = utc_dates.dt.tz_convert('America/New_York')
```

**When to Use**:
*   Use basic `.dt` property extraction (`.dt.year`, `.dt.month`) for feature engineering before grouping or machine learning.
*   Use `.dt.normalize()` or `.dt.date` when you need to match dates from different sources that may have inconsistent time components.
*   Use timedelta arithmetic when you need to measure duration (e.g., time to resolution, delivery times).

### Common Errors and Gotchas

> [!WARNING] Timezone Naive vs. Aware Comparisons
> **Error/Gotcha**: `TypeError: Cannot compare tz-naive and tz-aware datetime-like objects`.
> **Why it happens**: You attempt to subtract or compare a datetime column that has a timezone with one that does not (or with `pd.Timestamp.now()`, which is naive).
> **Fix**: Make both timezone-aware using `.dt.tz_localize()` or make both naive by converting and then removing the timezone with `.dt.tz_localize(None)`.

### Applying to Multiple DataFrames

Extracting date features across a collection of DataFrames is a standard feature engineering step.

```python
# Dict-based pattern: Add year and month name to all DataFrames
def add_date_features(dataframe):
    return dataframe.assign(
        year=dataframe['date'].dt.year,
        month_name=dataframe['date'].dt.month_name()
    )

dfs_featured = {k: add_date_features(d) for k, d in dfs.items()}

# List-based pattern: shift dates in all DataFrames by 1 day
dfs_shifted = [
    d.assign(date=d['date'] + pd.to_timedelta('1 days')) 
    for d in list(dfs.values())
]
```

> [!NOTE] Bulk Operations Caveat
> When converting or localizing timezones across multiple DataFrames, watch out for files collected during Daylight Saving Time (DST) transitions. If you use `.dt.tz_localize()` blindly, you may hit an `AmbiguousTimeError`. Use `ambiguous='NaT'` or `ambiguous='infer'` to handle bulk timezone localization safely.

## Related Chapters
* [[05-selecting-and-accessing-data]]
* [[06-modifying-dataframes]]
* [[08-missing-data-handling]]
* [[12-multi-dataframe-workflows]]
