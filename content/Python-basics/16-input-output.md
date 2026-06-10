---
title: "Reading and Writing Data — I/O Formats"
tags: [python, data-handling, pandas, numpy]
chapter: 16
theme: "Reading and Writing Data — I/O Formats"
---

Getting data into and out of pandas efficiently is fundamental to any data science workflow. Pandas supports an expansive array of file formats and data sources, each with specific strengths and trade-offs. 

Use this chapter to master the nuances of reading from and writing to CSVs, Excel files, JSON, columnar formats like Parquet, hierarchical stores like HDF5, and SQL databases.

```python
import pandas as pd
import numpy as np
import pathlib
import sqlite3
import json

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

## CSV (Comma-Separated Values)

CSV is the universal exchange format for tabular data. While widely supported and human-readable, it lacks data type preservation and can be slow to parse for large files. 

### Reading CSVs

The `pd.read_csv()` function is highly customizable. You will often need to tweak its parameters to handle messy real-world files correctly.

```python
# Setup: Create files used in the examples below
df.to_csv('sales.csv', index=False)
df.to_csv('sales.tsv', sep='\t', index=False)
df.to_csv('sales.txt', sep='|', index=False)
df.to_csv('sales_no_header.csv', header=False, index=False)

with open('sales_with_metadata.csv', 'w', encoding='utf-8') as f:
    f.write("Report: Q1 Sales\nGenerated: 2024\n")
df.to_csv('sales_with_metadata.csv', mode='a', index=False)

# Method 1: The default read (infers delimiter, header, and types)
csv_df = pd.read_csv('sales.csv')

# Method 2: Handling different delimiters (e.g., TSV, pipe-separated)
tsv_df = pd.read_csv('sales.tsv', sep='\t')
pipe_df = pd.read_csv('sales.txt', sep='|')

# Method 3: Parsing dates and enforcing specific data types
typed_df = pd.read_csv(
    'sales.csv',
    parse_dates=['date'],               # Parse specific columns as datetime64
    dtype={'store_id': 'category'},     # Enforce dtypes immediately to save memory
)

# Method 4: Skipping rows and selecting specific columns
subset_df = pd.read_csv(
    'sales_with_metadata.csv',
    skiprows=2,                         # Skip title/metadata rows at the top
    nrows=100,                          # Only read the first 100 rows (great for inspection)
    usecols=['date', 'units_sold']      # Only load these columns into memory
)

# Method 5: Handling missing values and encodings
clean_df = pd.read_csv(
    'sales.csv',
    na_values=['MISSING', 'N/A', '-99'], # Treat these custom strings as NaN
    encoding='latin1'                    # Fix "UnicodeDecodeError" when reading non-UTF8 files
)

# Method 6: Setting the index on load
indexed_df = pd.read_csv('sales.csv', index_col='date')

# Method 7: Specifying column names manually (overriding or providing headers)
no_header_df = pd.read_csv(
    'sales_no_header.csv', 
    header=None,                        # Explicitly state there is no header row
    names=['date', 'store', 'cat', 'units', 'price', 'disc', 'reg']
)
```

> [!TIP] `chunksize` for Large Files
> If a CSV is too large to fit in memory, read it in chunks. This returns an iterator of DataFrames.
> ```python
> chunk_iterator = pd.read_csv('sales.csv', chunksize=3)
> for chunk in chunk_iterator:
>     # process each chunk independently
>     pass
> ```

### Writing CSVs

```python
# Standard write. WARNING: this writes the index column by default!
df.to_csv('output.csv')

# Best practice: Drop the index if it's just a RangeIndex (0, 1, 2...)
df.to_csv('output.csv', index=False)

# Customizing the output format
df.to_csv(
    'output_custom.csv', 
    index=False, 
    sep=';', 
    na_rep='UNKNOWN' # Write NaNs as this string
)
```

**When to Use**: Use CSV for small-to-medium data sets, when sharing data with non-Python users (e.g., Excel or database users), or when human readability is essential. Do not use CSV for large datasets or data requiring exact preservation of complex data types.

### Common Errors and Gotchas

**The "Unnamed: 0" Column Error**
- **Error**: When you load a CSV, you see an extra column named `Unnamed: 0` containing row numbers.
- **Why**: You exported a DataFrame using `to_csv()` without `index=False`. Pandas wrote the index as the first column. When reading it back, pandas treats the index as a data column and creates a new index.
- **Fix**: Always write with `df.to_csv('file.csv', index=False)`. If reading an already-polluted file, use `pd.read_csv('file.csv', index_col=0)`.

---

## Excel Files

Excel files are common in business contexts but can be slow to parse. Pandas relies on underlying engines like `openpyxl` (for reading/writing `.xlsx`) and `xlrd` (for older `.xls`).

### Reading Excel

```python
# Setup: Create the Excel file used in the examples below
with pd.ExcelWriter('report.xlsx', engine='openpyxl') as writer:
    dfs['january'].to_excel(writer, sheet_name='January', index=False)
    dfs['february'].to_excel(writer, sheet_name='February', index=False)
    dfs['march'].to_excel(writer, sheet_name='March', index=False)

# Method 1: Read the first sheet (default behavior)
df_excel = pd.read_excel('report.xlsx')

# Method 2: Read a specific sheet by name or 0-indexed position
df_sheet2 = pd.read_excel('report.xlsx', sheet_name='February')
df_sheet3 = pd.read_excel('report.xlsx', sheet_name=2)

# Method 3: Read ALL sheets into a Dictionary of DataFrames
dfs_dict = pd.read_excel('report.xlsx', sheet_name=None)
# Returns: {'Sheet1': DataFrame, 'Sheet2': DataFrame}
```

> [!IMPORTANT] Use `sheet_name=None`
> Setting `sheet_name=None` is the gateway to multi-DataFrame workflows. It automatically ingests every worksheet into a single dictionary where the keys are the sheet names and the values are DataFrames.

### Writing Excel

To write a single DataFrame to a single sheet:
```python
df.to_excel('output.xlsx', sheet_name='Sales', index=False)
```

To write **multiple DataFrames** to different sheets in the same file, use the `pd.ExcelWriter` context manager:
```python
with pd.ExcelWriter('monthly_reports.xlsx', engine='openpyxl') as writer:
    dfs['january'].to_excel(writer, sheet_name='Jan', index=False)
    dfs['february'].to_excel(writer, sheet_name='Feb', index=False)
    dfs['march'].to_excel(writer, sheet_name='Mar', index=False)
```

**When to Use**: Use Excel I/O when generating reports for business stakeholders or ingesting human-edited spreadsheets. Avoid Excel for large analytical workloads due to extremely slow read/write times and file size limits.

### Common Errors and Gotchas

**Missing Engine Dependency**
- **Error**: `ImportError: Missing optional dependency 'openpyxl'. Use pip or conda to install openpyxl.`
- **Why**: Pandas relies on external libraries to read/write `.xlsx` files, and they are not installed with pandas by default.
- **Fix**: Run `pip install openpyxl` (or `pip install xlrd` for older `.xls` files) in your environment.

---

## JSON (JavaScript Object Notation)

JSON is the standard format for web APIs. JSON structures can vary significantly, which is why pandas supports different `orient` parameters.

### Reading and Writing JSON

```python
# Assume df is exported with orient='records' (a list of dictionaries)
df.to_json('data.json', orient='records', lines=True) # lines=True writes JSON Lines

# Read JSON back in
df_json = pd.read_json('data.json', orient='records', lines=True)
```

**Common Orientations:**
- `'records'`: `[{"col1": 1, "col2": "a"}, {"col1": 2, "col2": "b"}]` (Most common for APIs)
- `'split'`: `{"index": [0, 1], "columns": ["col1", "col2"], "data": [[1, "a"], [2, "b"]]}`
- `'index'`: `{"0": {"col1": 1, "col2": "a"}, "1": {"col1": 2, "col2": "b"}}`

### Flattening Nested JSON

APIs frequently return deeply nested JSON structures that `read_json` cannot flatten correctly. Use `pd.json_normalize()` to flatten nested dictionaries into columns with dot notation.

```python
nested_data = [
    {"id": 1, "user": {"name": "Alice", "location": {"city": "NY", "zip": "10001"}}},
    {"id": 2, "user": {"name": "Bob", "location": {"city": "SF", "zip": "94105"}}}
]

# Flatten the structure
flat_df = pd.json_normalize(nested_data)
# Columns will be: 'id', 'user.name', 'user.location.city', 'user.location.zip'
```

**When to Use**: Use JSON when communicating with web APIs, reading configuration files, or handling highly nested hierarchical payloads. Avoid JSON for simple tabular data due to verbose file sizes and slow parsing.

### Common Errors and Gotchas

**Incorrect Orientation**
- **Error**: `ValueError: Expected object or value` or `ValueError: Mixing dicts with non-Series may lead to ambiguous ordering.`
- **Why**: You are trying to read a JSON file using an `orient` parameter that does not match the actual data structure inside the file.
- **Fix**: Open the JSON file in a text editor to inspect its structure, then supply the matching `orient` parameter (e.g., `'records'` if it is a list of dictionaries).

---

## Parquet

Parquet is a highly compressed, columnar storage format. It is significantly faster than CSV, uses less disk space, and crucially, **preserves your pandas data types** (including datetimes and categoricals). It requires the `pyarrow` or `fastparquet` library.

### Reading and Writing Parquet

```python
# Write to Parquet (defaults to snappy compression)
df.to_parquet('data.parquet', engine='pyarrow', compression='snappy')

# Read from Parquet
df_pq = pd.read_parquet('data.parquet')

# Read ONLY specific columns (drastically reduces memory/I/O because Parquet is columnar)
df_pq_subset = pd.read_parquet('data.parquet', columns=['store_id', 'units_sold'])
```

### Reading Parquet Directories

Big data frameworks (like Spark or Dask) often save large datasets as a directory containing multiple partitioned `.parquet` files. Pandas can read the entire folder seamlessly:

```python
# Setup: Create a directory and some parquet files
pathlib.Path('sales_data').mkdir(exist_ok=True)
dfs['january'].to_parquet('sales_data/jan.parquet', engine='pyarrow')
dfs['february'].to_parquet('sales_data/feb.parquet', engine='pyarrow')

# Load all parquet files inside the "sales_data" directory as a single DataFrame
large_df = pd.read_parquet('sales_data/')
```

**When to Use**: Use Parquet as your default storage format for analytical workloads, big data pipelines, and when working with other big data tools (like Spark). It is the best choice for fast reads/writes and dtype preservation. Avoid when human readability via text editors is required.

### Common Errors and Gotchas

**Missing Engine Dependency**
- **Error**: `ImportError: Missing optional dependency 'pyarrow'.`
- **Why**: Pandas requires a fast backend engine to interact with the complex Parquet format.
- **Fix**: Run `pip install pyarrow` or `pip install fastparquet`.

---

## HDF5

HDF5 is a high-performance storage format designed for massive scientific data. It is excellent for storing multiple pandas DataFrames (or numpy arrays) inside a single file using a directory-like structure. It requires the `tables` library.

```python
# Writing a single dataset
df.to_hdf('store.h5', key='sales/2024', mode='w')

# Reading a single dataset
df_hdf = pd.read_hdf('store.h5', key='sales/2024')
```

**Storing Multiple Keys:**
Use the `pd.HDFStore` context manager to add multiple datasets dynamically.

```python
with pd.HDFStore('multi_store.h5', mode='w') as store:
    store['sales/jan'] = dfs['january']
    store['sales/feb'] = dfs['february']
    
    # You can access stored keys
    print(store.keys()) # Output: ['/sales/jan', '/sales/feb']
```

**When to Use**: Use HDF5 when you need to store multiple datasets or a massive dictionary of DataFrames in a single file with fast, hierarchical access. Avoid when cross-platform simplicity or compatibility with non-Python tools is paramount.

### Common Errors and Gotchas

**Missing Engine Dependency**
- **Error**: `ImportError: Missing optional dependency 'tables'.`
- **Why**: HDF5 functionality depends on the `PyTables` library.
- **Fix**: Run `pip install tables` in your environment.

---

## SQL Databases

Pandas interfaces with SQL databases via SQLAlchemy (for writing) or any DBAPI-compliant connection (like `sqlite3`). 

```python
# Setup an in-memory SQLite database for demonstration
conn = sqlite3.connect(':memory:')

# Write DataFrame to SQL
df.to_sql('sales_table', con=conn, index=False, if_exists='replace')
# if_exists options: 'fail' (default), 'replace', 'append'

# Read an entire table back into pandas
df_sql_full = pd.read_sql('SELECT * FROM sales_table', con=conn)

# Execute a complex SQL query directly into a DataFrame
query = """
    SELECT region, SUM(units_sold) as total_units
    FROM sales_table
    GROUP BY region
"""
df_sql_agg = pd.read_sql_query(query, con=conn)
```

> [!NOTE] SQLAlchemy for Production
> For connecting to PostgreSQL, MySQL, or SQL Server, install and use `SQLAlchemy` to create an engine, then pass that engine to the `con` parameter.

**When to Use**: Use SQL when data is stored in relational databases. It allows you to push computation (like filtering and aggregation) to the database before loading into pandas memory.

### Common Errors and Gotchas

**Invalid Queries or Missing Tables**
- **Error**: `sqlite3.OperationalError: no such table: sales_table` or `DatabaseError: Execution failed on sql...`
- **Why**: You are querying a table that does not exist or your SQL syntax is incorrect for the database dialect.
- **Fix**: Verify the table name exists in the database schema and ensure your SQL query string is valid.

---

## Pickle

Pickle is Python's native binary serialization format. It is fast and guarantees 100% preservation of all pandas and Python object types, including complex MultiIndexes.

```python
# Write and read
df.to_pickle('data.pkl')
df_pickled = pd.read_pickle('data.pkl')
```

**When to Use**: Use Pickle for short-term, local caching of Python objects, or when you need to serialize complex Python objects that don't easily map to columns. Avoid for long-term storage or sharing data externally due to security risks and version incompatibility.

> [!CAUTION] Pickle Security Warning
> Never use `pd.read_pickle()` on files obtained from untrusted sources. Pickles can execute arbitrary malicious code upon unpickling. Use Parquet instead for safe, cross-language binary storage.

### Common Errors and Gotchas

**Unpickling Errors**
- **Error**: `ModuleNotFoundError: No module named '...'` or `EOFError: Ran out of input`.
- **Why**: You are unpickling an object that relies on a library or class structure that isn't present in your current environment, or the pickle file is corrupted/from an incompatible Python version.
- **Fix**: Ensure the environment reading the pickle exactly matches the one that wrote it. For better long-term resilience, use Parquet.

---

## Other Notable Formats

- **Clipboard**: Use `pd.read_clipboard()` to parse tabular data copied from an Excel file, webpage, or text editor directly into a DataFrame.
- **HTML**: `pd.read_html('https://example.com/table.html')` scrapes all `<table>` elements on a webpage and returns a list of DataFrames.
- **XML**: `pd.read_xml('data.xml')` parses XML paths.
- **Feather**: `pd.read_feather()` and `df.to_feather()`. Fast, lightweight columnar format based on Apache Arrow, optimized for data movement rather than long-term storage.
- **Fixed-Width Format (FWF)**: `pd.read_fwf('data.txt')` parses legacy text files where columns have strict character widths instead of delimiters.

**When to Use**: 
- Use **Clipboard** for quick, ad-hoc data imports during interactive analysis.
- Use **HTML** for scraping tables from web pages.
- Use **XML** for parsing enterprise data feeds.
- Use **Feather** for fast, temporary data transfer between R and Python or caching.
- Use **FWF** when dealing with legacy mainframe text extracts.

---

## Format Comparison Table

When deciding which format to use, reference this comparison:

| Format | Speed | dtype preservation | Human readable | Best for |
|--------|-------|-------------------|----------------|----------|
| **CSV** | Medium | No | Yes | Exchange, small data, sharing with non-technical users |
| **Excel** | Slow | Partial | Yes | Business reporting, human editing |
| **Parquet** | Fast | Yes | No | Analytics, big data pipelines, compressed storage |
| **HDF5** | Fast | Yes | No | Python/scientific workflows, multiple DataFrame collections |
| **Pickle** | Fast | Yes | No | Python-only persistence, local temporary caching |
| **JSON** | Medium | Partial | Yes | APIs, semi-structured/nested data payloads |

---

## Applying to Multiple DataFrames

I/O operations frequently serve as the entry and exit points for multi-DataFrame workflows. You will often need to read multiple files into a dictionary, or write a dictionary of processed DataFrames to disk.

### 1. Loading a Folder of CSVs into a Dict
Use Python's `pathlib` to scan a directory and load all CSVs into a dictionary mapping filenames to DataFrames.

```python
path = pathlib.Path('sales_data_folder')
# Simulate creating the folder and files for the example
path.mkdir(exist_ok=True)
for k, v in dfs.items(): v.to_csv(path / f"{k}.csv", index=False)

# Read pattern
dfs_loaded = {
    filepath.stem: pd.read_csv(filepath) 
    for filepath in path.glob('*.csv')
}
# Output dict keys: 'january', 'february', 'march'
```

### 2. Saving a Dict of DataFrames to an Excel Multi-Sheet
```python
with pd.ExcelWriter('yearly_compilation.xlsx') as writer:
    for month_name, month_df in dfs.items():
        month_df.to_excel(writer, sheet_name=month_name.capitalize(), index=False)
```

### 3. Saving a Dict of DataFrames to HDF5
```python
with pd.HDFStore('yearly_compilation.h5') as store:
    for month_name, month_df in dfs.items():
        # Store using the month name as the key
        store[f'data/{month_name}'] = month_df
```

### 4. Saving a Dict of DataFrames to a Parquet Directory
This saves one highly compressed file per dictionary entry into an organized directory structure.

```python
out_dir = pathlib.Path('parquet_output')
out_dir.mkdir(exist_ok=True)

for month_name, month_df in dfs.items():
    month_df.to_parquet(out_dir / f"{month_name}.parquet")
```

---

## Related Chapters
- [[12-multi-dataframe-workflows]]
- [[15-method-chaining-and-pipelines]]
- [[17-performance-and-optimization]]
