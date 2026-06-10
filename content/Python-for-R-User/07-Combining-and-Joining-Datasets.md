# Chapter 7: Combining and Joining Datasets

When working with multiple datasets, you frequently need to combine them either by appending them vertically (adding rows), horizontally (adding columns), or by merging them based on common key columns. In R's `dplyr`, you typically use `bind_rows()`, `bind_cols()`, and the `*_join()` family of functions. In pandas, these operations are primarily handled by `pd.concat()` and `pd.merge()`.

## Binding Rows (Appending Datasets)

**The R Concept: `bind_rows()`**

In R, if you have two datasets with the same (or similar) columns and you want to stack them vertically, you use `bind_rows()`.

```R
# R - dplyr
combined_df <- bind_rows(df1, df2)
```

**The Python Approach: `pd.concat()`**

In Python, the equivalent is `pd.concat()`, which takes a list of DataFrames. By default, `pd.concat()` concatenates along the row axis (`axis=0`). 

```python
import pandas as pd

# Python - pandas
combined_df = pd.concat([df1, df2], ignore_index=True)
```

*Note: `ignore_index=True` is often used to reset the row indices so they run continuously from 0 to n-1, rather than keeping the original indices from `df1` and `df2`.*

**Tracking Source Data (`.id` equivalent):**
In R, you often use `bind_rows(df1, df2, .id = "source")` to create a column identifying the origin of each row. In Pandas, you use the `keys` argument, which creates a MultiIndex. You can then reset the index to turn it into a regular column:

```python
# R - dplyr
combined_df <- bind_rows(df1 = df1, df2 = df2, .id = "source")

# Python - pandas
combined_df = pd.concat([df1, df2], keys=["df1", "df2"])
# Convert the resulting MultiIndex into a standard column
combined_df = combined_df.reset_index(level=0).rename(columns={'level_0': 'source'})
```

**Alternatives:**
Older versions of pandas had a `.append()` method (e.g., `df1.append(df2)`), but this has been deprecated and removed in recent pandas versions. `pd.concat()` is the standard and most efficient way.

## Binding Columns

**The R Concept: `bind_cols()`**

In R, if you have two datasets with the exact same number of rows and you want to staple them together side-by-side without a key, you use `bind_cols()`.

```R
# R - dplyr
wide_df <- bind_cols(df1, df2)
```

**The Python Approach: `pd.concat()` with `axis=1`**

To bind columns in pandas, you use the exact same `pd.concat()` function but specify `axis=1` (the column axis). 

```python
# Python - pandas
wide_df = pd.concat([df1, df2], axis=1)
```

**CRITICAL WARNING FOR R USERS:** `pd.concat(axis=1)` aligns data based on the **row index**. R's `bind_cols()` pastes columns blindly without checking alignment. Pandas aligns them by the Index. If the indices don't match exactly, the result will have misaligned rows and injected `NaN`s. This is a very common trap for R users! If you truly want a blind paste (like R's `bind_cols`), you must reset or drop the indices first:

```python
df1.reset_index(drop=True, inplace=True)
df2.reset_index(drop=True, inplace=True)
wide_df = pd.concat([df1, df2], axis=1)
```

## Joins (Merging Datasets)

In `dplyr`, joining datasets relies on specialized functions for each type of join: `left_join()`, `inner_join()`, `right_join()`, and `full_join()`.
In pandas, almost all joining is done using a single function: `pd.merge()`, where the `how` argument dictates the type of join.

### Left Join

**The R Concept: `left_join()`**

```R
# R - dplyr
# Joins by common column name 'id'
joined_df <- left_join(df1, df2, by = "id")

# Joins by differently named columns
joined_df <- left_join(df1, df2, by = c("id1" = "id2"))
```

**The Python Approach: `pd.merge()`**

To perform a left join in pandas, use `pd.merge()` with `how="left"`.

```python
# Python - pandas
# Joins by common column name 'id'
joined_df = pd.merge(df1, df2, on="id", how="left")

# Joins by differently named columns
joined_df = pd.merge(df1, df2, left_on="id1", right_on="id2", how="left")
```

**Using the DataFrame Method:**
You can also call `.merge()` directly on a DataFrame object. This is very useful for method chaining.

```python
# Python - DataFrame method
joined_df = df1.merge(df2, on="id", how="left")

# Method chaining example
result = (
    df1.query("status == 'active'")
    .merge(df2, on="id", how="left")
    .fillna(0)
)
```

### Inner Join

**The R Concept: `inner_join()`**

```R
# R - dplyr
joined_df <- inner_join(df1, df2, by = "id")
```

**The Python Approach: `pd.merge()`**

For an inner join, change the `how` argument. Actually, `how="inner"` is the default behavior of `pd.merge()`, but it is best practice to specify it explicitly for readability.

```python
# Python - pandas
joined_df = pd.merge(df1, df2, on="id", how="inner")
# or
joined_df = df1.merge(df2, on="id", how="inner")
```

### Right Join and Full Outer Join

**The R Concept: `right_join()` and `full_join()`**

```R
# R - dplyr
right_df <- right_join(df1, df2, by = "id")
full_df <- full_join(df1, df2, by = "id")
```

**The Python Approach: `pd.merge()`**

Simply pass `"right"` or `"outer"` to the `how` argument.

```python
# Python - pandas
right_df = pd.merge(df1, df2, on="id", how="right")
full_df = pd.merge(df1, df2, on="id", how="outer")
```

## Joining on Multiple Columns

**The R Concept**
```R
# R - dplyr
joined_df <- left_join(df1, df2, by = c("first_name", "last_name"))
```

**The Python Approach**
Pass a list of column names to the `on` parameter.

```python
# Python - pandas
joined_df = pd.merge(df1, df2, on=["first_name", "last_name"], how="left")
```

## Dealing with Overlapping Column Names

When you join two datasets that share column names not used in the key, both R and Python append a suffix to the column names to distinguish them.

**The R Concept**
By default, `dplyr` adds `.x` and `.y`.
```R
# R - dplyr
left_join(df1, df2, by = "id", suffix = c("_left", "_right"))
```

**The Python Approach**
By default, pandas adds `_x` and `_y`. You can customize this using the `suffixes` argument.

```python
# Python - pandas
joined_df = pd.merge(df1, df2, on="id", how="left", suffixes=("_left", "_right"))
```

## Joining on the Index

In R, row names (indices) are generally discouraged in the `tidyverse`, and joins always happen on columns. In pandas, the row index is a first-class citizen, and you may sometimes need to join on the index itself.

If `df1` has an index you want to join on, and `df2` has a matching index:
```python
# Join on the index of both DataFrames
joined_df = pd.merge(df1, df2, left_index=True, right_index=True, how="inner")
```

Alternatively, pandas provides the `.join()` method, which is specifically designed for joining on indices. It performs a left join by default.

```python
# Alternative: .join() method (defaults to how='left')
joined_df = df1.join(df2, how="inner")
```
Generally, `pd.merge()` is more robust and explicitly clear for most column-based operations similar to `dplyr`.

## Advanced Operations and Alternatives

### Coalescing Joins (Patching Missing Data)

**The R Concept: `coalesce()` after a join**
In R, you might join two datasets and then use `coalesce(x, y)` to fill missing values in one dataset with values from another.

**The Python Approach: `combine_first()` or `update()`**
Pandas provides `.combine_first()` to patch missing values in one DataFrame with non-null values from another, aligning on their indices.

```python
# Python - pandas
# df1 is patched with values from df2 where df1 has NaNs
patched_df = df1.combine_first(df2)
```

Alternatively, `df1.update(df2)` modifies `df1` in-place, overwriting existing values with non-null values from `df2` (rather than just filling `NaN`s).

### Rolling or Fuzzy Joins

**The R Concept: `fuzzyjoin` or `data.table` rolling joins**
Sometimes you need to match on the nearest key (like timestamps) rather than an exact match.

**The Python Approach: `pd.merge_asof()`**
Pandas has a dedicated function for this, which is extremely performant for time-series data. Note that DataFrames must be sorted by the join key beforehand.

```python
# Python - pandas
# Matches each row in df1 to the closest row in df2 where df2.time <= df1.time
joined_df = pd.merge_asof(df1, df2, on="time", direction="backward")
```

### Comparing DataFrames

**The R Concept: `waldo::compare()` or `all.equal()`**
When you need to see exactly what changed between two versions of a dataset.

**The Python Approach: `df.compare()`**
Pandas has a built-in method to show differences between two identically shaped DataFrames side-by-side.

```python
# Python - pandas
differences = df1.compare(df2)
```
