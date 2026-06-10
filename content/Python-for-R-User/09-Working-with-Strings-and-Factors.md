# Chapter 9: Working with Strings and Factors

## String Manipulation

### R Concept: `stringr`
In R, text manipulation is streamlined by the `stringr` package. You use functions like `str_detect()` to find matches, `str_replace()` or `str_replace_all()` to substitute patterns, `str_extract()` to pull out substrings, and `str_to_lower()` / `str_to_upper()` for case conversions. These functions naturally vectorise over character vectors and play nicely with the `dplyr` pipe.

### Python Focus: Pandas `.str` Accessor
In Python, pandas provides a `.str` accessor that vectorizes string operations over a Series. You apply it immediately after selecting a text column.

#### Detecting Patterns (`str_detect`)

**R:**
```R
library(dplyr)
library(stringr)

df <- df %>% filter(str_detect(name, "John"))
```

**Python:**
Pandas `.str.contains()` is the direct analogue. Note that you often want to set `na=False` so that missing values evaluate to `False` rather than `NaN` (which would break boolean indexing).

```python
import pandas as pd

# Method 1: Boolean indexing (Standard)
df_filtered = df[df['name'].str.contains('John', na=False)]

# Method 2: query() method
df_filtered = df.query('name.str.contains("John")', engine='python')

# Method 3: Using .loc for explicit row filtering
df_filtered = df.loc[df['name'].str.contains('John', na=False)]

# Method 4: numpy where (if you need an array of booleans/conditions)
import numpy as np
is_john = np.where(df['name'].str.contains('John', na=False), True, False)

# Exact String Matching
# .str.fullmatch() checks if the entire string matches (useful instead of contains for exact matches)
df_exact = df[df['name'].str.fullmatch('John Doe', na=False)]
# .str.match() checks if the beginning of the string matches a regular expression
df_match = df[df['name'].str.match(r'^John', na=False)]
```

*Tip:* You can also use `.str.startswith('J')` or `.str.endswith('n')` for exact literal anchoring without writing regex.

#### Replacing Patterns (`str_replace` / `str_replace_all`)

**R:**
```R
df <- df %>% mutate(name = str_replace_all(name, "Dr\\.", ""))
```

**Python:**
Pandas `.str.replace()` serves as both `str_replace` and `str_replace_all`. In modern pandas, the default is literal replacement (`regex=False`), so you must specify `regex=True` when passing a regular expression.

```python
# Standard syntax
df['name'] = df['name'].str.replace(r'Dr\.', '', regex=True)

# Method chaining
df = df.assign(name=lambda x: x['name'].str.replace(r'Dr\.', '', regex=True))

# Note: Python's native string replace (for a single string, not a pandas Series)
"Dr. John".replace("Dr.", "")
```

#### Changing Case (`str_to_lower` / `str_to_upper`)

**R:**
```R
df <- df %>% mutate(name = str_to_lower(name))
```

**Python:**
```python
# Standard syntax
df['name'] = df['name'].str.lower()
df['name'] = df['name'].str.upper()

# Additional built-in case formatting
df['name'] = df['name'].str.title()      # Title Case
df['name'] = df['name'].str.capitalize() # Only first letter capitalized
```

#### Extracting Substrings (`str_extract` / `str_sub`)

**R:**
```R
df <- df %>% mutate(
  first_letter = str_sub(name, 1, 1),
  title = str_extract(name, "^[A-Za-z]+")
)
```

**Python:**
For character positional slicing (`str_sub`), Python uses standard indexing via `.str[]` or `.str.slice()`. For regex extraction (`str_extract`), use `.str.extract()`. **Note: Python's `.str.extract()` *requires* capture groups (parentheses `()`) around the pattern you want to extract, whereas `str_extract` in R does not.**

```python
# Slicing (str_sub) - Python is 0-indexed!
df['first_letter'] = df['name'].str[0]
# Alternatively:
df['first_letter'] = df['name'].str.slice(0, 1)

# Extracting with regex (str_extract)
# Notice the parentheses around the pattern: (^[A-Za-z]+)
df['title'] = df['name'].str.extract(r'(^[A-Za-z]+)')
```

#### Trimming and Padding Strings (`str_trim` / `str_pad`)

**R:**
```R
df <- df %>% mutate(
  clean_name = str_trim(name),
  padded_id = str_pad(id, width = 5, side = "left", pad = "0")
)
```

**Python:**
Pandas uses `.str.strip()` to remove whitespace (analogous to `str_trim`). For padding, use `.str.pad()` or the specialized `.str.zfill()` for zero-padding.

```python
# Trimming whitespace
df['clean_name'] = df['name'].str.strip()
# (Also available: .str.lstrip() and .str.rstrip() for left/right trimming)

# Padding strings
df['padded_id'] = df['id'].astype(str).str.pad(width=5, side='left', fillchar='0')

# Specialized zero-padding (equivalent to above)
df['padded_id'] = df['id'].astype(str).str.zfill(5)
```

---

## Factors / Categorical Data

### R Concept: `forcats` and Factors
In R, categorical data is handled via the `factor` class. The `forcats` package provides specialized functions like `fct_reorder()` to sort factor levels based on another variable, `fct_lump()` to group rare levels together, and `fct_relevel()` to manually set the order.

### Python Focus: Pandas `Categorical` Data Type
In pandas, categorical data is managed using the `category` dtype. Similar to factors, it optimizes memory usage and specifies a logical ordering for sorting and plotting. Pandas categorical operations are accessed via the `.cat` accessor.

#### Creating Categorical Data

**R:**
```R
df$category <- factor(df$category, levels = c("Low", "Medium", "High"), ordered = TRUE)
```

**Python:**
```python
# Method 1: Using CategoricalDtype (Recommended for ordered categories)
from pandas.api.types import CategoricalDtype
cat_type = CategoricalDtype(categories=["Low", "Medium", "High"], ordered=True)
df['category'] = df['category'].astype(cat_type)

# Method 2: pd.Categorical directly
df['category'] = pd.Categorical(df['category'], categories=["Low", "Medium", "High"], ordered=True)

# Method 3: Unordered simple conversion
df['category'] = df['category'].astype('category')
```

#### Reordering Levels Manually (`fct_relevel`)

**R:**
```R
df$category <- fct_relevel(df$category, "High", "Medium", "Low")
```

**Python:**
Use the `.cat.reorder_categories()` method. You must pass exactly the same categories that already exist, just in the new order.

```python
# Changes the logical order of the categories
df['category'] = df['category'].cat.reorder_categories(["High", "Medium", "Low"])

# If you want to add/remove categories while reordering, use set_categories
df['category'] = df['category'].cat.set_categories(["High", "Medium", "Low", "Critical"])
```

#### Lumping Rare Levels (`fct_lump`)

**R:**
```R
df <- df %>% mutate(category = fct_lump(category, n = 3))
```

**Python:**
Pandas doesn't have a direct single function for `fct_lump`. You generally combine `.value_counts()` to find the top categories and `np.where()` or `.where()` to collapse the rest.

```python
# Step 1: Identify the top 3 categories
top_3 = df['category'].value_counts().nlargest(3).index

# Method 1: using np.where
df['category_lumped'] = np.where(df['category'].isin(top_3), df['category'], 'Other')

# Method 2: using pandas .where() (keeps True, replaces False)
df['category_lumped'] = df['category'].where(df['category'].isin(top_3), 'Other')

# Note: The result of where/np.where is usually an 'object' (string) dtype.
# Cast back to category if required:
df['category_lumped'] = df['category_lumped'].astype('category')
```

#### Reordering by Another Variable (`fct_reorder`)

**R:**
```R
# Commonly used for plotting: reorder category by the median of 'value'
df$category <- fct_reorder(df$category, df$value, .fun = median)
```

**Python:**
To reorder a categorical column based on the values of another column, group by the category, calculate the aggregation metric, and extract the sorted index. Then apply it via `.cat.reorder_categories()`.

```python
# Calculate the median of 'value' for each 'category' and extract the sorted index
ordered_cats = df.groupby('category')['value'].median().sort_values().index

# Apply the new order
df['category'] = df['category'].cat.reorder_categories(ordered_cats, ordered=True)

# Method chaining equivalent
df = df.assign(
    category=lambda x: x['category'].cat.reorder_categories(
        x.groupby('category')['value'].median().sort_values().index, 
        ordered=True
    )
)
```

#### Ordering by Frequency (`fct_infreq`)

**R:**
```R
df$category <- fct_infreq(df$category)
```

**Python:**
To order categories by their frequency (most frequent to least frequent), extract the index from `.value_counts()` and use `.cat.reorder_categories()`.

```python
# .value_counts() automatically sorts by frequency descending
freq_ordered_cats = df['category'].value_counts().index

df['category'] = df['category'].cat.reorder_categories(freq_ordered_cats, ordered=True)
```

#### Renaming and Modifying Levels (`fct_recode` / `fct_expand` / `fct_drop`)

**R:**
```R
df <- df %>% mutate(
  category = fct_recode(category, "Critical" = "High"),
  category = fct_expand(category, "Unknown"),
  category = fct_drop(category)
)
```

**Python:**
Pandas provides specific `.cat` methods for renaming, adding, and dropping categories.

```python
# Renaming categories (fct_recode) - uses a dictionary of {old: new}
df['category'] = df['category'].cat.rename_categories({"High": "Critical"})

# Adding new categories (fct_expand)
df['category'] = df['category'].cat.add_categories(["Unknown"])

# Dropping unused categories (fct_drop)
df['category'] = df['category'].cat.remove_unused_categories()

# Removing specific categories explicitly
df['category'] = df['category'].cat.remove_categories(["Low"])
```
