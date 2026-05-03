---
title: NumPy and pandas
---

![DataFrame table with three rows](/assets/images/topics/numpy-pandas.svg)
<!-- .element: class="title-illustration" -->

# NumPy & pandas

Array math and dataframes — Python for data.

---

## Why NumPy?

Python lists are slow for numeric work — every element is a full PyObject. NumPy gives you:

- **`ndarray`** — contiguous, typed, vectorized arrays
- **Broadcasting** — operations apply element-wise without loops
- **C-speed** — internals are C/Fortran; Python is just orchestrating
- **Foundation** — pandas, scikit-learn, PyTorch all build on it

---

## Install

```bash
uv add numpy pandas
uv run python
```

```python
>>> import numpy as np
>>> import pandas as pd
```

`uv add jupyterlab` if you want notebooks.

---

## Arrays

```python
import numpy as np

a = np.array([1, 2, 3, 4])
a.shape                       # (4,)
a.dtype                       # dtype('int64')

b = np.zeros((3, 4))          # 3×4 of zeros
c = np.arange(0, 1, 0.1)      # [0.0, 0.1, ..., 0.9]
d = np.linspace(0, 1, 5)      # [0.0, 0.25, 0.5, 0.75, 1.0]
e = np.random.default_rng(42).standard_normal((3, 3))
```

---

## Vectorized math

```python
a = np.array([1, 2, 3, 4])
a + 10                        # array([11, 12, 13, 14])
a * 2                         # array([2, 4, 6, 8])
a ** 2                        # array([ 1,  4,  9, 16])
np.sqrt(a)                    # array([1.   , 1.41, 1.73, 2.   ])
a.sum(), a.mean(), a.std()    # (10, 2.5, 1.118...)
```

No `for` loop. The op runs in C across the whole array.

---

## Slicing & indexing

```python
m = np.arange(12).reshape(3, 4)
# array([[ 0,  1,  2,  3],
#        [ 4,  5,  6,  7],
#        [ 8,  9, 10, 11]])

m[0]                          # row 0:    [0, 1, 2, 3]
m[:, 1]                       # col 1:    [1, 5, 9]
m[1:, 1:3]                    # array([[5, 6], [9, 10]])

m[m > 5]                      # boolean mask: [ 6,  7,  8,  9, 10, 11]
m[m > 5] = -1                 # in-place
```

Boolean masks are how you "filter" without `for`.

---

## pandas — DataFrame

A 2D labeled table. Excel + SQL + numpy combined.

```python
import pandas as pd

df = pd.DataFrame({
    "name": ["Alice", "Bob", "Carol"],
    "age":  [30, 25, 35],
    "city": ["NYC", "LA", "NYC"],
})
df
#     name  age city
# 0  Alice   30  NYC
# 1    Bob   25   LA
# 2  Carol   35  NYC

df.shape                      # (3, 3)
df.dtypes
# name    object
# age      int64
# city    object
```

---

## Reading data

```python
df = pd.read_csv("data.csv")
df = pd.read_csv("https://example.com/data.csv")
df = pd.read_excel("data.xlsx", sheet_name="Q1")
df = pd.read_json("data.json")
df = pd.read_parquet("data.parquet")
df = pd.read_sql("SELECT * FROM users", connection)

df.head()                     # first 5 rows
df.tail(3)
df.info()                     # column types + memory
df.describe()                 # summary stats for numeric cols
```

`read_csv` alone has 50+ options for date parsing, type hints, encoding, ...

---

## Selecting

```python
df["name"]                            # one column → Series
df[["name", "age"]]                   # multiple → DataFrame
df.iloc[0]                            # row 0 by position
df.loc[df["name"] == "Alice"]         # row(s) by label / mask

df.loc[df["age"] >= 30, ["name"]]
#     name
# 0  Alice
# 2  Carol
```

`iloc` is positional; `loc` is label/mask-based. Don't mix them.

---

## Filtering and combining masks

```python
df[df["age"] > 28]
df[(df["age"] > 25) & (df["city"] == "NYC")]   # AND
df[(df["age"] < 26) | (df["age"] > 32)]        # OR
df[df["city"].isin(["NYC", "LA"])]
df[df["name"].str.startswith("A")]

df.query("age > 28 and city == 'NYC'")         # SQL-like alternative
```

Use `&` / `|` (not `and`/`or`) on Series. Always parenthesize.

---

## Adding and transforming columns

```python
df["seniority"] = df["age"] - 21
df["adult"] = df["age"] >= 18
df["greeting"] = "Hello, " + df["name"]
df["age_bucket"] = pd.cut(df["age"], bins=[0, 26, 31, 100],
                                      labels=["young", "mid", "senior"])

# Apply a function row-wise (slower; reach for vectorized first)
df["initial"] = df["name"].str[0]
df["upper_name"] = df["name"].str.upper()
```

Vectorized string methods (`df.col.str.<method>`) are usually fastest.

---

## Group-by

```python
df.groupby("city")["age"].mean()
# city
# LA     25.0
# NYC    32.5
# Name: age, dtype: float64

df.groupby("city").agg(
    count=("name", "size"),
    avg_age=("age", "mean"),
    youngest=("age", "min"),
)
#       count  avg_age  youngest
# city
# LA        1     25.0        25
# NYC       2     32.5        30
```

`groupby` is the bread-and-butter analytic primitive. Master it.

---

## Joining

```python
orders = pd.DataFrame({"user_id": [1, 1, 2], "amount": [10, 20, 5]})
users  = pd.DataFrame({"id": [1, 2], "name": ["Alice", "Bob"]})

orders.merge(users, left_on="user_id", right_on="id")
#    user_id  amount  id   name
# 0        1      10   1  Alice
# 1        1      20   1  Alice
# 2        2       5   2    Bob
```

`how="inner" | "left" | "right" | "outer"` — same semantics as SQL.

---

## Missing data

```python
df.isna().sum()              # count NaNs per column
df.dropna()                  # drop rows with any NaN
df.fillna(0)                 # replace NaN with 0
df["age"].fillna(df["age"].median())

# Per-column
df.fillna({"age": 0, "city": "Unknown"})
```

`pd.NA` is the modern, type-preserving missing value (replaces `NaN` for non-float types).

---

## Time series

```python
df["created"] = pd.to_datetime(df["created"])

df.set_index("created").resample("D").size()
# Daily count of events

df["created"].dt.year
df["created"].dt.weekday          # 0=Mon, 6=Sun
df["created"].dt.tz_localize("UTC").dt.tz_convert("America/New_York")
```

Use UTC internally; convert at the boundary.

---

## Plotting (sketch)

```python
df["age"].hist()
df.groupby("city")["age"].mean().plot(kind="bar")
```

For anything serious, reach for `matplotlib`, `plotly`, or `seaborn`. pandas's plot methods are fine for quick exploration.

---

## Performance — when pandas hurts

For 100M+ rows, pandas slows down. Alternatives:

| Tool | When |
| --- | --- |
| **`polars`** | Drop-in for most pandas — multi-core, lazy, much faster |
| **`duckdb`** | SQL on dataframes / Parquet, embedded |
| **PyArrow** | Columnar I/O; pandas can use Arrow-backed arrays now |
| **Dask** | Out-of-core / distributed pandas |
| **Spark** | Distributed at scale |

Profile before reaching for these — vectorized pandas covers more than you'd think.

---

## What's next

- **scikit-learn** — fit/predict on pandas DataFrames
- **LLM agents** — Python's other "data" frontier
