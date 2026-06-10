# Chapter 5: Grouping and Summarizing

Grouping and summarizing data is a cornerstone of data analysis. In the Tidyverse, you typically pair `group_by()` with `summarise()` to compute aggregate statistics, `mutate()` to add group-level statistics without collapsing rows, and `filter()` to drop entire groups. You use `count()` or `table()` for quick frequency tables. In Pandas, the `.groupby()` method is used for grouping, followed by `.agg()` for summarization, `.transform()` for group-wise mutations, `.filter()` for group-wise filtering, and `.value_counts()` or `pd.crosstab()` handle frequency counts.

## Grouping and Summarizing Multiple Columns

### In R
In R, you define the grouping variables with `group_by()` and then calculate the summary statistics using `summarise()`.

```R
# R: Grouping and summarizing
summary_df <- df %>%
  group_by(category, sub_category) %>%
  summarise(
    mean_sales = mean(sales, na.rm = TRUE),
    total_profit = sum(profit, na.rm = TRUE),
    n_obs = n()
  ) %>%
  ungroup()
```

### In Python
In Pandas, you use `.groupby()` passing a column name or a list of column names. You then apply the `.agg()` method using a dictionary to specify which operations apply to which columns. Notice that Pandas often sets the grouping columns as the index of the resulting DataFrame. To bring them back as regular columns (similar to `ungroup()` in R), you use `.reset_index()`.

```python
# Python: Grouping and summarizing (Standard approach)
summary_df = df.groupby(['category', 'sub_category']).agg({
    'sales': 'mean',
    'profit': 'sum',
    'category': 'size' # 'size' counts the number of rows per group
}).reset_index()

# Renaming columns after aggregation
summary_df = summary_df.rename(columns={
    'sales': 'mean_sales',
    'profit': 'total_profit',
    'category': 'n_obs'
})
```

#### Alternative Python Approaches for Aggregation

**1. Named Aggregation (Pandas 0.25+)**
This approach closely mirrors `dplyr::summarise()`, allowing you to specify the output column name, the source column, and the aggregation function in one step.

```python
# Python: Named aggregation (Very similar to R)
summary_df = df.groupby(['category', 'sub_category']).agg(
    mean_sales=('sales', 'mean'),
    total_profit=('profit', 'sum'),
    n_obs=('sales', 'size')
).reset_index()
```

**2. Applying multiple functions to the same column**
If you need both the mean and the sum of `sales`, you can pass a list of functions in the `.agg()` dictionary.

```python
# Python: Multiple functions per column
summary_df = df.groupby('category').agg({
    'sales': ['mean', 'sum', 'min', 'max']
}).reset_index()

# This creates a MultiIndex for columns (e.g., ('sales', 'mean')). 
# To flatten the MultiIndex columns:
summary_df.columns = ['_'.join(col).strip('_') for col in summary_df.columns]
```

**3. Direct aggregation without `.agg()`**
If you want to apply the same summary function to all numerical columns, you can call the method directly on the `GroupBy` object.

```python
# Python: Direct method call
mean_df = df.groupby('category').mean(numeric_only=True).reset_index()
```

**Note on Default NA Handling:** A major quality-of-life difference between R and Python involves missing values during aggregation. In R, `mean(x)` returns `NA` if there are any missing values, requiring you to explicitly set `na.rm = TRUE`. Pandas aggregation functions (like `.mean()`, `.sum()`) automatically ignore `NaN`s by default.

## Group-wise Mutations (`transform`)

### In R
A very common operation in R is to add a grouped statistic as a new column without collapsing the dataframe into a summary table. This is done with `group_by()` followed by `mutate()`.

```R
# R: Group-wise mutation
df <- df %>%
  group_by(category) %>%
  mutate(mean_sales = mean(sales, na.rm = TRUE)) %>%
  ungroup()
```

### In Python
This is a massive conceptual leap for many R users. If you try to use `.agg()` or `.mean()`, Pandas will collapse the dataframe. To add an aggregated value back to the original rows, you must use `.transform()`. This function computes the grouped statistic and broadcasts it back to match the original index.

```python
# Python: Group-wise mutation using transform
df['mean_sales'] = df.groupby('category')['sales'].transform('mean')

# Using transform with a custom function
df['sales_centered'] = df.groupby('category')['sales'].transform(lambda x: x - x.mean())
```

## Filtering Entire Groups (`filter`)

### In R
You can drop entire groups based on an aggregate condition using `group_by()` followed by `filter()`.

```R
# R: Group-wise filtering
large_categories <- df %>%
  group_by(category) %>%
  filter(sum(sales, na.rm = TRUE) > 1000) %>%
  ungroup()
```

### In Python
Pandas has a `.filter()` method specifically for `GroupBy` objects that does exactly this. Note that it takes a function (often a lambda) that evaluates to a single boolean value for each group DataFrame.

```python
# Python: Group-wise filtering
large_categories = df.groupby('category').filter(lambda x: x['sales'].sum() > 1000)
```

## Counting Frequencies (`count` / `table`)

### In R
To quickly count the number of occurrences of each unique value in a column, R uses `count()` (which groups, counts, and ungroups) or `tally()` (which requires prior grouping).

```R
# R: Counting frequencies
counts <- df %>% count(category, sort = TRUE)
```

### In Python
In Pandas, `.value_counts()` is the most direct equivalent to R's `count()`. For a single column, it returns a Pandas Series.

```python
# Python: Value counts on a single column
counts_series = df['category'].value_counts()

# To get a DataFrame back with the index reset:
counts_df = df['category'].value_counts().reset_index()
# In older pandas versions, columns might be 'index' and 'category'.
# In newer pandas (1.5+), columns are 'category' and 'count'.

# To include NaN values in the count (equivalent to count(..., na.rm=FALSE)):
counts_df = df['category'].value_counts(dropna=False).reset_index()

# To get proportions instead of raw counts:
prop_df = df['category'].value_counts(normalize=True).reset_index()
```

**Counting across multiple columns:**
You can also use `.value_counts()` on multiple columns by calling it on the DataFrame itself (Pandas 1.1+).

```python
# Python: Value counts on multiple columns
counts_df = df[['category', 'sub_category']].value_counts().reset_index()
```

**Alternative: Using `.groupby().size()`**
This is functionally identical to `.value_counts()` but doesn't sort the results in descending order by default.

```python
# Python: Counting using groupby and size
counts_df = df.groupby(['category', 'sub_category']).size().reset_index(name='count')
counts_df = counts_df.sort_values('count', ascending=False)
```

**Alternative: Using `pd.crosstab()`**
For R users familiar with `table()`, `pd.crosstab()` is an excellent alternative for creating 2D (or multi-dimensional) frequency tables across categorical columns.

```python
# Python: Frequency table (similar to R's table())
cross_tab = pd.crosstab(df['category'], df['sub_category'])
```

## The Importance of `.reset_index()` (`ungroup` equivalent)

In R, `group_by()` adds grouping metadata to the tibble. `summarise()` strips one level of grouping by default, but it's best practice to use `ungroup()` to remove all grouping metadata so subsequent operations aren't accidentally applied per group.

In Pandas, grouping operations usually move the grouping keys into the DataFrame's **Index**. An Index in Pandas is special and functions differently than regular columns. To return the Index back to standard columns (the closest equivalent to `ungroup()`), append `.reset_index()` to your aggregation chain.

```python
# Python: Grouping without reset_index (returns a DataFrame with an Index)
grouped_index_df = df.groupby('category').sum(numeric_only=True)
# Here, 'category' is the index. You cannot access df['category'] directly.

# Python: Grouping with reset_index (returns a standard DataFrame)
grouped_standard_df = df.groupby('category').sum(numeric_only=True).reset_index()
# Here, 'category' is a regular column.

# Alternative: as_index=False in groupby
grouped_standard_df = df.groupby('category', as_index=False).sum(numeric_only=True)
```
Using `as_index=False` inside `.groupby()` is often cleaner than `.reset_index()`, but it doesn't always work perfectly with `.agg()` if you are using dictionary-based aggregation that changes column names or shapes unexpectedly. `.reset_index()` is the most robust and universally applicable method.
