---
name: data-analysis
description: Best practices and patterns for data analysis in OP App
---

# Data Analysis Patterns

Common workflows for analyzing data in OP App notebooks.

## Exploratory Analysis Workflow

### Step 1: Initial Inspection

```python
# Dataset overview
print(f"Dataset: {df.shape[0]:,} rows × {df.shape[1]} columns")
print("\nColumns:", ", ".join(df.columns))

# Data types
print("\nData types:")
for col, dtype in df.dtypes.items():
    print(f"  {col}: {dtype}")

# First few rows
print("\nSample data:")
print(df.head())
```

### Step 2: Data Quality

```python
# Missing values
missing = df.isnull().sum()
if missing.any():
    print("Missing values:")
    print(missing[missing > 0])

# Duplicates
dupes = df.duplicated().sum()
if dupes > 0:
    print(f"\nFound {dupes} duplicate rows")

# Unique values for categorical columns
cat_cols = df.select_dtypes(include=['object']).columns
for col in cat_cols:
    n_unique = df[col].nunique()
    print(f"\n{col}: {n_unique} unique values")
    if n_unique <= 10:
        print(df[col].value_counts())
```

### Step 3: Summary Statistics

```python
# Numeric summaries
numeric_summary = df.describe()
create_table(numeric_summary.T, 'Numeric Statistics', visible=True)

# Categorical distributions
for col in df.select_dtypes(include=['object']).columns:
    dist = df[col].value_counts().head(10)
    print(f"\n{col} distribution:")
    print(dist)
```

## Common Patterns

### Grouping and Aggregation

```python
# Group by single column
category_stats = df.groupby('category').agg({
    'sales': ['sum', 'mean', 'count'],
    'profit': 'sum'
}).round(2)

# Flatten column names
category_stats.columns = ['_'.join(col) for col in category_stats.columns]

# Visualize top performers
top_categories = category_stats.sort_values('sales_sum', ascending=False)
create_table(top_categories, 'Top Categories', visible=True)
```

### Time Series Analysis

```python
# Ensure datetime type
df['date'] = pd.to_datetime(df['date'])

# Monthly aggregation
monthly_sales = df.groupby(df['date'].dt.to_period('M'))['sales'].sum()
monthly_sales.index = monthly_sales.index.astype(str)

create_table(monthly_sales.to_frame(), 'Monthly Sales Trend', visible=True)
```

### Filtering and Subsetting

```python
# Filter by condition
high_value = df[df['sales'] > df['sales'].quantile(0.75)]
print(f"High-value transactions: {len(high_value)} ({len(high_value)/len(df):.1%})")

# Multiple conditions
recent_high_value = df[
    (df['date'] >= '2024-01-01') &
    (df['sales'] > 1000)
]

create_table(recent_high_value, 'Recent High-Value Orders', visible=False)
```

### Top N Analysis

```python
# Top customers
top_customers = (
    df.groupby('customer')['sales']
    .sum()
    .sort_values(ascending=False)
    .head(10)
)

create_table(top_customers.to_frame(), 'Top 10 Customers', visible=True)

# Calculate concentration
top_10_pct = top_customers.sum() / df['sales'].sum()
print(f"Top 10 customers represent {top_10_pct:.1%} of revenue")
```
