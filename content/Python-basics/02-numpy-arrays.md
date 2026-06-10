---
title: "NumPy Arrays and Vectorized Computation"
tags: [python, data-handling, pandas, numpy]
chapter: 2
theme: "NumPy Arrays and Vectorized Computation"
---

NumPy is the foundational library for numerical computing in Python. The core of NumPy is the `ndarray` (N-dimensional array), a fast, memory-efficient data structure designed for homogenous data. Vectorized computation allows you to apply mathematical operations to entire arrays without writing slow `for` loops, leveraging optimized C code under the hood.

### Mental Model: The Strict Grid

Think of a NumPy array as a strict, perfectly aligned grid. Unlike a Python list—which is like a flexible bag that can hold anything (a mix of integers, strings, and other lists)—a NumPy array forces all elements to be of the exact same data type. This strictness allows the computer to pack the data tightly in memory and execute mathematical operations on all elements simultaneously, completely bypassing the slow Python interpreter loop.

Let's initialize our standard dataset for this chapter. Although it originates as a pandas DataFrame, we will extract its columns as NumPy arrays to demonstrate these fundamental numerical concepts.

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

# Extracting columns as 1D NumPy arrays to demonstrate NumPy syntax
units_sold = df['units_sold'].values
unit_price = df['unit_price'].values
discount = df['discount'].values
```

## Array Creation and Shape Attributes

Before you can manipulate data in NumPy, you need to create arrays and understand their dimensional properties.

### Array Creation Methods

Depending on whether you have existing data or need to generate values from scratch, NumPy offers several factory functions.

```python
# Method 1: From an existing Python list
arr_from_list = np.array([12, 35, 8, 50, 22])

# Method 2: Initialization functions (pre-allocating memory)
zeros_arr = np.zeros(5)               # array([0., 0., 0., 0., 0.])
ones_arr = np.ones((2, 3))            # 2x3 matrix of 1s
full_arr = np.full((2, 2), 99.9)      # 2x2 matrix filled with 99.9
identity = np.eye(3)                  # 3x3 identity matrix

# Method 3: Generating sequences
range_arr = np.arange(0, 10, 2)       # array([0, 2, 4, 6, 8])
linear_arr = np.linspace(0, 1, 5)     # 5 evenly spaced numbers from 0 to 1
log_arr = np.logspace(1, 3, 3)        # array([10., 100., 1000.])

# Method 4: Random generation
np.random.seed(42)                    # Ensure reproducibility
rand_uniform = np.random.rand(3)      # 3 uniform random floats between 0 and 1
rand_normal = np.random.randn(2, 2)   # 2x2 matrix of standard normal distribution
rand_ints = np.random.randint(1, 100, size=5)  # 5 random integers between 1 and 99
rand_choices = np.random.choice(['A', 'B', 'C'], size=3) # e.g., ['C', 'A', 'A']

# Method 5: Explicit Type Coercion
# Specifying dtype at creation saves memory and prevents unexpected behavior later.
float_arr = np.array([1, 2, 3], dtype=np.float32)
```

> [!TIP] When to Use What
> - Use `np.array()` when converting a Python list, tuple, or pandas Series.
> - Use `np.zeros()` or `np.ones()` to pre-allocate an array of a known size before filling it (this is much faster than repeatedly appending to a list).
> - Use `np.arange()` or `np.linspace()` for numeric ranges, commonly used for plot axes or grid search parameters.

### Shape Attributes

Every NumPy array exposes attributes that describe its structural dimensions and memory footprint.

```python
sample_matrix = np.array([[1, 2, 3], [4, 5, 6]])

print(sample_matrix.shape)     # (2, 3) - Tuple of dimension sizes (rows, cols)
print(sample_matrix.ndim)      # 2 - Number of dimensions
print(sample_matrix.size)      # 6 - Total number of elements
print(sample_matrix.dtype)     # int64 - Data type of the elements
print(sample_matrix.itemsize)  # 8 - Memory size in bytes of each element
```

## Array Indexing and Selection

NumPy provides powerful and flexible indexing syntaxes to subset data efficiently.

### 1D and 2D Slicing

Standard slicing utilizes the `[start:stop:step]` syntax. In higher dimensions, separate the slices for each axis with a comma.

```python
# 1D Slicing
print(units_sold[0])          # 12 (first element)
print(units_sold[:3])         # [12 35  8] (first three elements)
print(units_sold[::-1])       # Reverses the array

# 2D Slicing
sales_matrix = np.array([
    [12, 35, 8],
    [50, 22, 18],
    [45, 30, 14]
])

print(sales_matrix[0, 1])     # 35 (Row 0, Col 1)
print(sales_matrix[:, 1])     # [35 22 30] (All rows, Col 1)
print(sales_matrix[1:, :2])   # Rows 1 to end, Cols 0 to 1
# [[50 22]
#  [45 30]]
```

### All Advanced Selection Methods

When slicing isn't enough, NumPy provides advanced indexing techniques to filter data based on positions or conditions.

```python
# Method 1: Fancy Indexing
# Pass a list or array of integer indices to extract specific elements.
indices = [0, 2, 4]
fancy_selection = units_sold[indices]  # array([12,  8, 22])

# Method 2: Boolean (Mask) Indexing
# Pass an array of booleans. Only elements corresponding to True are kept.
mask = units_sold > 20
bool_selection = units_sold[mask]      # array([35, 50, 22, 45, 30])

# Compound boolean conditions (MUST use &, |, ~ and wrap conditions in parentheses)
compound_mask = (units_sold > 20) & ~(unit_price > 100)
compound_selection = units_sold[compound_mask]

# Method 3: np.where (Vectorized if/else)
# Replace elements based on a condition: np.where(condition, if_true, if_false)
where_selection = np.where(units_sold < 20, 0, units_sold)

# Method 4: np.take
# Functional equivalent to fancy indexing. Useful when specifying a specific axis.
take_selection = np.take(units_sold, indices)

# Method 5: np.compress
# Functional equivalent to boolean indexing.
compress_selection = np.compress(mask, units_sold)
```

| Method | Syntax | Returns View or Copy? | Best For |
|--------|--------|-----------------------|----------|
| **Slicing** | `arr[0:5]` | View | Extracting contiguous blocks of data |
| **Fancy Indexing** | `arr[[0, 2, 4]]` | Copy | Extracting specific, non-contiguous positions |
| **Boolean Indexing** | `arr[arr > 10]` | Copy | Filtering data based on values (logic) |
| **`np.where`** | `np.where(cond, x, y)` | Copy (new array) | Replacing values conditionally |
| **`np.take`** | `np.take(arr, indices)` | Copy | Programmatic selection along an axis |

## Array Manipulation

Changing the structural format of your arrays is often required to prepare data for algorithms or visualizations.

### Reshape, Flatten, and Transpose

```python
matrix = np.arange(1, 7) # array([1, 2, 3, 4, 5, 6])

# Reshape: Change dimensions without changing data
reshaped = matrix.reshape((2, 3))
# Use -1 to let NumPy infer the dimension size automatically
auto_reshaped = matrix.reshape((2, -1)) # -1 evaluates to 3
# [[1, 2, 3]
#  [4, 5, 6]]

# Flattening: 2D down to 1D
flattened = reshaped.flatten()  # Always allocates new memory (COPY)
raveled = reshaped.ravel()      # Uses same memory if possible (VIEW)

# Transposing: Swap rows and columns
transposed = reshaped.T
```

### Combining and Splitting

```python
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])

# Stacking vertically (row-wise)
v_stacked = np.vstack((a, b))
# equivalent to: np.concatenate((a, b), axis=0)

# Stacking horizontally (column-wise)
h_stacked = np.hstack((a, b))
# equivalent to: np.concatenate((a, b), axis=1)

# Depth stacking (stacking along a third axis)
d_stacked = np.dstack((a, b))

# Splitting an array into multiple sub-arrays
v_split = np.vsplit(v_stacked, 2)    # Splits vertically back into [a, b]
h_split = np.hsplit(h_stacked, 2)    # Splits horizontally back into [a, b]
# np.split can do both depending on the axis argument
split_arrs = np.split(v_stacked, 2, axis=0)
```

### Adding Dimensions and Repeating

```python
# Add a new axis (turns 1D array of shape (9,) into 2D column vector of shape (9,1))
expanded = np.expand_dims(units_sold, axis=1)

# Remove axes of length 1
squeezed = np.squeeze(expanded)

# Repeat individual elements
rep = np.repeat([1, 2], 3)  # array([1, 1, 1, 2, 2, 2])

# Tile entire array sequences
til = np.tile([1, 2], 3)    # array([1, 2, 1, 2, 1, 2])
```

## Mathematical Operations

NumPy eliminates the need for manual iteration over arrays for mathematical calculations.

### Element-wise Arithmetic and Ufuncs

```python
# Direct arithmetic applies element-by-element
revenue = units_sold * unit_price
discounted_revenue = revenue * (1 - discount)

# Universal Functions (ufuncs) are fast, C-optimized mathematical functions
log_revenue = np.log(revenue + 1)
sqrt_sales = np.sqrt(units_sold)
exp_discount = np.exp(discount)
abs_diff = np.abs(units_sold - 20)
```

### Reduction Operations and the `axis` Parameter

A reduction operation aggregates an array down to a smaller size, summarizing its contents. The `axis` parameter defines the direction of the operation.

```python
matrix2d = np.array([[1, 2, 3],
                     [4, 5, 6]])

print(np.sum(matrix2d))          # 21 (Global sum)
print(np.sum(matrix2d, axis=0))  # [5 7 9] (Sum across rows, collapsing vertically)
print(np.sum(matrix2d, axis=1))  # [6 15] (Sum across columns, collapsing horizontally)

# Other core reductions
print(np.mean(units_sold))       # Arithmetic mean
print(np.std(unit_price))        # Standard deviation
print(np.max(discount))          # Maximum value

# Cumulative operations
print(np.cumsum(units_sold))     # Cumulative sum
print(np.cumprod(unit_price))    # Cumulative product

# Additional Statistical functions
print(np.median(units_sold))     # Median
print(np.percentile(units_sold, 75)) # 75th percentile
print(np.unique(discount))       # Unique values sorted
print(np.bincount(np.array([0, 1, 1, 2]))) # Count occurrences of non-negative ints

# Matrix Multiplication
# Use np.dot or np.matmul for dot products and matrix multiplication
vector_a = np.array([1, 2, 3])
vector_b = np.array([4, 5, 6])
dot_product = np.dot(vector_a, vector_b)
matrix_product = np.matmul(matrix2d, vector_a)  # (2x3) * (3x1)
```

> [!NOTE] Understanding Axes
> In a 2D array, `axis=0` refers to the rows (pointing downwards), and `axis=1` refers to the columns (pointing rightwards). Running an operation over `axis=0` crushes the rows down, leaving you with column-wise results.

## Broadcasting

Broadcasting is a set of rules that allows NumPy to perform mathematical operations between arrays of different shapes. NumPy virtually "stretches" the smaller array to match the shape of the larger one without allocating new memory for duplicated values.

### The Broadcasting Rules
1. If the arrays have a different number of dimensions, the shape of the smaller-dimensional array is padded with ones on its left side.
2. Two arrays are compatible in a dimension if they have the exact same size, or if one of the sizes is exactly `1`.

### Common Patterns

```python
features = np.array([
    [10, 200, 3],
    [20, 250, 4],
    [30, 300, 5]
])

# Pattern 1: Subtracting column means (Standardization)
col_means = np.mean(features, axis=0) # Shape: (3,)
# features shape is (3,3). col_means shape is (3,). 
# Broadcasting allows this direct subtraction.
centered_features = features - col_means

# Pattern 2: Scaling rows
# We want to scale each ROW by a different weight.
weights = np.array([0.5, 1.0, 2.0]) # Shape: (3,)

# `features * weights` fails here because shapes (3,3) and (3,) align on the rightmost dimension.
# We must reshape `weights` to (3,1) so it broadcasts across the columns.
scaled_rows = features * weights.reshape(3, 1)
```

## Boolean Operations

Boolean logic is essential for data cleaning and pipeline validations.

```python
has_discount = discount > 0

print(np.any(has_discount))  # True (Is at least one item discounted?)
print(np.all(has_discount))  # False (Are all items discounted?)

# Dealing with NaNs and Infs
data_with_nan = np.array([1.0, 2.0, np.nan, 4.0, np.inf])

print(np.isnan(data_with_nan))  # [False False  True False False]
print(np.isinf(data_with_nan))  # [False False False False  True]

# Clean the data by substituting problematic values using np.where
clean_data = np.where(np.isnan(data_with_nan), 0, data_with_nan)

# Filtering and assigning values directly in-place using boolean masks
mutable_data = np.array([1.0, -5.0, 3.0, -2.0, 10.0])
mutable_data[mutable_data < 0] = 0  # Zero-out negative values directly
```

## Sorting and Searching

```python
arr_to_sort = np.array([45, 12, 50, 8, 30])

# Method 1: np.sort (returns a sorted COPY, original is unchanged)
sorted_arr = np.sort(arr_to_sort)      # array([8, 12, 30, 45, 50])

# Method 2: arr.sort (sorts the array IN-PLACE)
arr_copy = arr_to_sort.copy()
arr_copy.sort(axis=0)

# Method 3: np.argsort (returns the INDICES that would sort the array)
sort_indices = np.argsort(arr_to_sort) # array([3, 1, 4, 0, 2])
sorted_via_args = arr_to_sort[sort_indices]

# Find indices of non-zero elements (or where condition is True)
nonzero_indices = np.nonzero(arr_to_sort > 20)

# Finding indices of minimum or maximum values
min_idx = np.argmin(arr_to_sort)       # 3
max_idx = np.argmax(arr_to_sort)       # 2

# Binary search for insertion points to maintain a sorted order
sorted_base = np.array([10, 20, 30, 40])
insert_idx = np.searchsorted(sorted_base, 25) # Returns 2
```

## NumPy I/O

While you will primarily use pandas for I/O with tabular formats like CSV and Excel, NumPy has specialized binary formats for saving arrays efficiently.

```python
# 1. Binary .npy format (Fastest, preserves shape and strict dtype)
np.save('units.npy', units_sold)
loaded_units = np.load('units.npy')

# 2. Compressed .npz format (Stores multiple arrays in an archive)
np.savez('sales_data.npz', u=units_sold, p=unit_price)
loaded_npz = np.load('sales_data.npz')
u_array = loaded_npz['u']

# 3. Text format (TXT/CSV)
np.savetxt('units.csv', units_sold, delimiter=',')
txt_loaded = np.loadtxt('units.csv', delimiter=',')

# 4. genfromtxt (Advanced text loading, handles missing values gracefully)
data_from_txt = np.genfromtxt('units.csv', delimiter=',', filling_values=0)
```

## Common Errors and Gotchas

### 1. The View vs. Copy Mutation Bug

When you slice an array, you create a **view**. If you mutate the slice, the original array changes. If you assume you created a copy, you will silently corrupt your source data.

```python
# The Error (Logical)
arr = np.array([1, 2, 3, 4, 5])
first_three = arr[:3]
first_three[0] = 999  

print(arr) 
# [999,   2,   3,   4,   5]  # The original data was mutated!
```

**Why it happens:** To stay computationally fast, NumPy avoids copying data in memory. Slicing merely returns a new "window" looking at the exact same memory buffer.
**The fix:** Explicitly append `.copy()` when you intend to modify a subset of data independently from its source.

```python
arr = np.array([1, 2, 3, 4, 5])
first_three = arr[:3].copy()
first_three[0] = 999
# arr remains untouched: [1, 2, 3, 4, 5]
```

### 2. Unexpected Type Coercion

Because NumPy arrays require uniform types, inserting a value of a different type will trigger an implicit upcast or result in an error.

```python
arr = np.array([1, 2, 3])  # dtype is int64

try:
    arr[1] = "text"
except ValueError as e:
    print(e)
# ValueError: invalid literal for int() with base 10: 'text'
```

**Why it happens:** The array was initialized as integers. NumPy cannot force the string `"text"` into an integer space, causing a crash. If you initialize an array with mixed types (`[1, 2, "3"]`), NumPy preemptively coerces everything into strings, making math operations fail later.
**The fix:** Be explicit about your array's `dtype`, and use `.astype()` if you need to convert types safely before insertion.

```python
arr_str = arr.astype(str)
arr_str[1] = "text"
```

### 3. Shape Mismatch in Broadcasting

When performing arithmetic between arrays of different dimensions, you will encounter shape errors if the broadcasting rules are violated.

```python
arr1 = np.array([1, 2, 3])
arr2 = np.array([1, 2, 3, 4])

try:
    arr1 + arr2
except ValueError as e:
    print(e)
# ValueError: operands could not be broadcast together with shapes (3,) (4,) 
```

**Why it happens:** NumPy tries to align the arrays element-wise. Because they have different lengths, and neither has a dimension of size `1` that can be stretched, the operation is invalid.
**The fix:** Ensure compatible dimensions by slicing the arrays to match, padding them, or utilizing `np.newaxis` (`reshape`) if you intend to broadcast across a completely new dimension.

## Related Chapters
- [[01-python-native-data-structures]]
- [[03-pandas-series]]
- [[17-performance-and-optimization]]
