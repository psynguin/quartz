---
title: "Python Native Data Structures"
tags: [python, data-handling, pandas, numpy]
chapter: 1
theme: "Python Native Data Structures as Data Containers"
---

## Introduction

Before diving into pandas and NumPy, you must master Python's native data structures. While pandas provides powerful specialized objects like Series and DataFrames, native Python containers—lists, tuples, dictionaries, and sets—are the glue that holds data science workflows together. You will use them constantly to select columns, map categorical values, deduplicate records, and store collections of DataFrames.

Run the following setup code to follow along with the examples in this chapter. This dataset will be our running example.

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

# Multi-DataFrame version (dict of monthly DataFrames)
dfs = {
    'january': df[df['date'].dt.month == 1].reset_index(drop=True),
    'february': df[df['date'].dt.month == 2].reset_index(drop=True),
    'march': df[df['date'].dt.month == 3].reset_index(drop=True),
}
```

## Lists

### Concept Explanation
A list is an ordered, mutable sequence of items. Think of it as a dynamic array or a resizable bookshelf where you can add, remove, and reorder items at will. 

**Why it matters:** In data science, lists are the default container for sequences of items. You will use them primarily to specify subsets of columns, build up rows of data before converting them into a DataFrame, or accumulate results in a loop.

### Syntactic Variants

Here are all the ways you can create and interact with lists:

```python
# Method 1: Literal creation (most common)
cols = ['store_id', 'category', 'units_sold']

# Method 2: The list() constructor
cols_from_tuple = list(('store_id', 'category', 'units_sold'))

# Method 3: List comprehension (powerful for transformation)
# E.g., getting all column names that start with 'unit'
unit_cols = [col for col in df.columns if col.startswith('unit')]

# Indexing and Slicing
first_col = cols[0]          # 'store_id'
last_col = cols[-1]          # 'units_sold'
subset = cols[1:3]           # ['category', 'units_sold'] (exclusive of index 3)

# Mutation (in-place)
cols.append('region')        # Adds single element to the end
cols.extend(['date'])        # Adds multiple elements from an iterable
cols.insert(0, 'date')       # Inserts element at specific index
removed = cols.pop()         # Removes and returns the last element
cols.remove('category')      # Removes the first occurrence of a value
del cols[0]                  # Removes an element by its index

# Iteration Patterns
for col in cols:
    pass
    
for i, col in enumerate(cols):
    pass  # Access to both index and item
    
for col, val in zip(cols, [1, 2, 3]):
    pass  # Iterate through multiple lists simultaneously

# Nested Lists (2D structures)
data_matrix = [
    ['S01', 'Electronics', 12],
    ['S02', 'Clothing', 35]
]
```

> [!TIP] When to Use
> Use lists when you need an ordered collection of items that you might need to modify (add/remove elements) later, or when passing a collection of column names to a pandas selection operation.

### Worked Example

A very common workflow is building a list of specific columns to extract from a DataFrame, perhaps filtering them dynamically via a list comprehension.

```python
# 1. Start with a minimal list of target columns
target_columns = ['store_id', 'region']

# 2. Dynamically add numeric columns to our list
numeric_cols = [col for col in df.columns if df[col].dtype in ['int64', 'float64']]
target_columns.extend(numeric_cols)

# 3. Use the list to slice the DataFrame
subset_df = df[target_columns]
print(subset_df.head(2))
```

### Common Errors and Gotchas

> [!WARNING] Common Error: `IndexError: list index out of range`
> **Why it happens:** You tried to access an index that doesn't exist.
> **The Fix:** Ensure your index is within `0` and `len(my_list) - 1`.

> [!WARNING] Common Error: `append()` vs `extend()` confusion
> **Why it happens:** Doing `my_list.append(['a', 'b'])` adds the *entire list* as a single element, creating a nested list. 
> **The Fix:** If you want to add individual elements from another list, use `my_list.extend(['a', 'b'])`.

## Tuples

### Concept Explanation
A tuple is an ordered, **immutable** sequence of items. Think of it as a read-only list. Once created, you cannot add, remove, or change its elements.

**Why it matters:** Immutability provides safety. In pandas workflows, tuples are often used to represent composite keys (like a multi-level index) or when passing a sequence of items that absolutely must not be altered by a downstream function. Tuples can also be used as dictionary keys (unlike lists), which is vital for complex aggregations.

### Syntactic Variants

```python
# Method 1: Literal creation (parentheses)
record = ('S01', 'Electronics', 12)

# Method 2: Literal creation (parentheses are optional but recommended)
record_alt = 'S01', 'Electronics', 12

# Method 3: Single-element tuple (comma is REQUIRED)
single = ('S01',) 

# Method 4: The tuple() constructor
record_from_list = tuple(['S01', 'Electronics', 12])

# Method 5: collections.namedtuple (structured records)
from collections import namedtuple
SalesRecord = namedtuple('SalesRecord', ['store', 'category', 'units'])
r1 = SalesRecord(store='S01', category='Electronics', units=12)
# Access via attribute: r1.store -> 'S01'

# Unpacking (very common in loops)
store, category, units = record
```

> [!TIP] When to Use
> Use tuples when the collection of items should conceptually never change throughout the lifecycle of your script, or when you need a sequence to serve as a dictionary key or a pandas MultiIndex key.

### Worked Example

Tuples appear naturally when iterating over pandas `GroupBy` objects. The iterator yields a tuple containing the group key and the DataFrame chunk for that group.

```python
# Grouping by multiple columns yields a tuple as the group key
grouped = df.groupby(['store_id', 'category'])

# Iterating over the groupby object unpacks the tuple
for group_key, group_df in grouped:
    # group_key is a tuple, e.g., ('S01', 'Electronics')
    store, category = group_key  # Unpacking the tuple key
    
    total_revenue = (group_df['units_sold'] * group_df['unit_price']).sum()
    print(f"{store} - {category}: ${total_revenue:.2f}")
    break # Just show the first one
```

### Common Errors and Gotchas

> [!WARNING] Common Error: `TypeError: 'tuple' object does not support item assignment`
> **Why it happens:** You tried to change an element, e.g., `my_tuple[0] = 'S09'`.
> **The Fix:** If you need to mutate the collection, use a list instead. If you must update a tuple, you have to create a brand new one: `my_tuple = ('S09',) + my_tuple[1:]`.

## Dictionaries

### Concept Explanation
A dictionary (`dict`) is a mutable, key-value mapping. Think of it as a physical lookup dictionary where every word (key) maps to a definition (value). In Python 3.7+, dictionaries maintain insertion order.

**Why it matters:** Dictionaries are the workhorses of data mapping. You will use them to rename columns, map categorical values to new labels, define aggregations, and—most importantly in advanced workflows—store collections of DataFrames.

### Syntactic Variants

```python
# Method 1: Literal creation
region_map = {'S01': 'North', 'S02': 'South', 'S03': 'East'}

# Method 2: The dict() constructor with keyword arguments
region_map_alt = dict(S01='North', S02='South', S03='East')

# Method 3: Dict comprehension (powerful for dynamic mapping)
# E.g., creating a mapping to lowercase column names
col_lower = {col: col.lower() for col in df.columns}

# Method 4: fromkeys() for initializing default values
status_dict = dict.fromkeys(['S01', 'S02', 'S03'], 'Active')

# Accessing Values
val = region_map['S01']           # Throws KeyError if 'S01' doesn't exist
safe_val = region_map.get('S04')  # Returns None if 'S04' doesn't exist
safe_val_def = region_map.get('S04', 'Unknown') # Returns 'Unknown'

# Mutation
region_map['S04'] = 'West'        # Adds or updates a key
region_map.setdefault('S05', 'Central') # Adds only if 'S05' is missing
removed_val = region_map.pop('S01')     # Removes key and returns value
region_map.update({'S06': 'West', 'S07': 'North'}) # Merges another dict
del region_map['S02']                   # Deletes a specific key

# Iteration Patterns
for k in region_map.keys(): pass
for v in region_map.values(): pass
for k, v in region_map.items(): pass

# Nested Dicts as Records (Easily converted to a DataFrame)
records_dict = {
    'record_1': {'store': 'S01', 'sales': 150},
    'record_2': {'store': 'S02', 'sales': 200}
}
df_from_dict = pd.DataFrame.from_dict(records_dict, orient='index')

# Specialized Dictionaries
from collections import defaultdict, OrderedDict

# defaultdict: automatically initializes missing keys with a default factory (e.g., list)
sales_by_store = defaultdict(list)
sales_by_store['S01'].append(12)  # No KeyError, 'S01' is initialized to []

# OrderedDict: historically used to remember insertion order (standard dicts do this natively since 3.7)
ordered_map = OrderedDict([('S01', 'North'), ('S02', 'South')])
```

> [!TIP] When to Use
> Use dictionaries whenever you have a 1-to-1 mapping relationship, such as `old_name -> new_name`, `category -> code`, or `month -> DataFrame`.

### Worked Example: Storing DataFrames in Dicts

While dictionaries are great for mapping values (e.g., using `.map()` on a pandas Series), their most powerful application in scalable workflows is holding collections of DataFrames. We will explore this deeply in [[12-multi-dataframe-workflows]], but here is a preview:

```python
# Preview: The `dict_of_dfs` pattern
# We already defined `dfs` in our setup block. It maps string keys to DataFrames.
# This allows us to apply operations to all DataFrames in a loop!

cleaned_dfs = {}
for month, month_df in dfs.items():
    # Apply identical logic to each month's data
    month_df_clean = month_df.dropna()
    
    # Store it back in a new dictionary
    cleaned_dfs[month] = month_df_clean

# Alternatively, use a dict comprehension for cleaner code:
cleaned_dfs_functional = {month: mdf.dropna() for month, mdf in dfs.items()}
```

### Common Errors and Gotchas

> [!WARNING] Common Error: `KeyError: 'S09'`
> **Why it happens:** You used bracket notation `my_dict['S09']` but the key isn't in the dictionary.
> **The Fix:** Use `.get('S09', 'default_value')` if you anticipate missing keys.

> [!WARNING] Common Error: `RuntimeError: dictionary changed size during iteration`
> **Why it happens:** You tried to add or delete keys while looping over the dictionary.
> **The Fix:** Iterate over a copy of the keys instead: `for k in list(my_dict.keys()):`

## Sets

### Concept Explanation
A set is an unordered collection of **unique** elements. Think of it as a mathematical set or a bag of distinct items.

**Why it matters:** Sets are optimized for fast membership testing (checking if an item exists) and deduplication. When you need to find the unique categories in a dataset, or find the overlapping column names between two DataFrames, sets are the most efficient tool.

### Syntactic Variants

```python
# Method 1: Literal creation
valid_categories = {'Electronics', 'Clothing', 'Food'}

# Method 2: The set() constructor (often used to deduplicate a list)
unique_stores = set(df['store_id'].tolist())

# Method 3: Set comprehension
high_discount_stores = {row['store_id'] for _, row in df.iterrows() if row['discount'] > 0.10}

# Set Operations
set_A = {'Electronics', 'Clothing'}
set_B = {'Clothing', 'Food'}

union = set_A | set_B             # {'Electronics', 'Clothing', 'Food'}
intersection = set_A & set_B      # {'Clothing'}
difference = set_A - set_B        # {'Electronics'} (In A but not in B)
sym_diff = set_A ^ set_B          # {'Electronics', 'Food'} (In either, but not both)

# Membership Testing (Very Fast O(1))
is_valid = 'Electronics' in valid_categories
```

> [!TIP] When to Use
> Use sets when you need to deduplicate data, test if a value exists within a large collection of items, or compare groups of items (e.g., finding the missing columns between two datasets).

### Worked Example

Imagine you receive a new dataset and want to ensure it only contains expected categories before processing it.

```python
expected_categories = {'Electronics', 'Clothing', 'Food', 'Home'}
actual_categories = set(df['category'].unique())

# Find categories in our data that are NOT in the expected list
unexpected = actual_categories - expected_categories

if unexpected:
    print(f"Warning: Found unexpected categories: {unexpected}")
else:
    print("All categories are valid.")
```

### Common Errors and Gotchas

> [!WARNING] Common Error: Initializing an empty set with `{}`
> **Why it happens:** In Python, `{}` creates an empty dictionary, not an empty set.
> **The Fix:** Always use `set()` to create an empty set.

## Choosing Among Structures

To summarize, here is a decision matrix to help you choose the right container for your intermediate data operations:

| Structure | Ordered? | Mutable? | Allows Duplicates? | Best Used For... | Big-O Access | Big-O Insert | Big-O Search |
|-----------|----------|----------|--------------------|------------------|--------------|--------------|--------------|
| **List**  | Yes | Yes | Yes | Collecting ordered results, column subsets | $O(1)$ | $O(1)$ (append) | $O(N)$ |
| **Tuple** | Yes | No | Yes | Composite keys, grouping keys, read-only data | $O(1)$ | N/A | $O(N)$ |
| **Dict**  | Yes (3.7+) | Yes | No (Keys) | Mapping values, storing multiple DataFrames | $O(1)$ | $O(1)$ | $O(1)$ (by key) |
| **Set**   | No | Yes | No | Deduplication, fast membership tests, intersections | N/A | $O(1)$ | $O(1)$ |

Understanding these underlying structures will make your pandas code cleaner, more efficient, and easier to debug as you transition into vectorized operations.

## Related Chapters
- [[02-numpy-arrays]]
- [[12-multi-dataframe-workflows]]
