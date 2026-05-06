---
title: scikit-learn
---

![Train/test split flowing through fit and predict](/assets/images/topics/scikit-learn.svg)
<!-- .element: class="title-illustration" -->

# scikit-learn

The classical-ML toolkit for Python.

---

## What scikit-learn is

- **Models** — linear regression, decision trees, random forests, SVMs, k-means, ...
- **Pre-processing** — scaling, encoding categoricals, imputing missing
- **Pipelines** — chain pre-processing + model into one object
- **Model selection** — train/test split, cross-validation, grid search
- **Metrics** — accuracy, precision/recall, RMSE, ROC, ...

It's the right tool for tabular ML. For images / text / audio at scale, reach for **PyTorch** or **TensorFlow**.

---

## Install

```bash
uv add scikit-learn pandas matplotlib
```

Imports you'll see everywhere:

```python
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, classification_report
```

---

## The fit / predict pattern

Every estimator has the same shape:

```python
model = SomeEstimator(**hyperparams)
model.fit(X_train, y_train)        # learn from data
preds = model.predict(X_test)      # apply
score = model.score(X_test, y_test)
```

- `X` — features (2D: rows × columns)
- `y` — target (1D for regression / binary; 1D or 2D for multi-class)
- `fit` — train; mutates the model in place
- `predict` — apply; pure
- `score` — convenience metric (model-dependent)

Once you know this shape, every sklearn model "works the same".

---

## A complete classification example

```python
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report

X, y = load_iris(as_frame=True, return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y,
)

model = LogisticRegression(max_iter=1000)
model.fit(X_train, y_train)

print(classification_report(y_test, model.predict(X_test)))
#               precision    recall  f1-score   support
#            0       1.00      1.00      1.00        10
#            1       1.00      1.00      1.00        10
#            2       1.00      1.00      1.00        10
```

35 lines including imports. That's a working classifier.

---

## Train/test split

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,        # 20% held out for evaluation
    random_state=42,      # reproducible shuffling
    stratify=y,           # preserve class proportions in both splits
)
```

**Never train on data you'll evaluate on.** Bad numbers from bad splits is the most common ML mistake.

---

## Pre-processing

Most models expect numeric, scaled features. sklearn provides transformers:

```python
from sklearn.preprocessing import StandardScaler, OneHotEncoder

scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)
# fit_transform on train, transform on test (don't refit!)

X_test_scaled = scaler.transform(X_test)
```

Same shape: `fit`, `transform`, `fit_transform`. Always `fit` on train, `transform` on test — fitting on test leaks.

---

## Pipelines — bundle everything

```python
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("clf",    LogisticRegression(max_iter=1000)),
])

pipe.fit(X_train, y_train)
pipe.score(X_test, y_test)            # 0.97
```

The pipeline is itself an estimator — calls to `.fit` / `.predict` flow through every step. Saves you from re-fitting the scaler on test data by accident.

---

## Mixed feature types

```python
from sklearn.compose import ColumnTransformer

preprocess = ColumnTransformer([
    ("num", StandardScaler(),  ["age", "income"]),
    ("cat", OneHotEncoder(),   ["city", "role"]),
])

pipe = Pipeline([
    ("prep", preprocess),
    ("clf",  LogisticRegression()),
])

pipe.fit(X_train, y_train)
```

`ColumnTransformer` applies different transformers to different columns. The cleanest way to handle real-world tables.

---

## Cross-validation

```python
from sklearn.model_selection import cross_val_score

scores = cross_val_score(pipe, X, y, cv=5, scoring="accuracy")
scores.mean(), scores.std()        # (0.965, 0.022)
```

Splits the data into 5 folds, trains 5 times, returns 5 scores. Better than a single train/test split for small datasets.

---

## Hyperparameter tuning

```python
from sklearn.model_selection import GridSearchCV

grid = GridSearchCV(
    pipe,
    param_grid={
        "clf__C": [0.1, 1, 10],
        "clf__penalty": ["l1", "l2"],
    },
    cv=5,
    scoring="accuracy",
    n_jobs=-1,                  # all cores
)
grid.fit(X, y)

grid.best_params_              # {'clf__C': 1, 'clf__penalty': 'l2'}
grid.best_score_               # 0.971
```

`pipeline_step__hyperparameter` is the prefix syntax. `RandomizedSearchCV` is faster for large grids.

---

## Common models — picking one

| Task | Start with | If that's not enough |
| --- | --- | --- |
| Binary / multi-class | `LogisticRegression` | `RandomForestClassifier`, `XGBoost` |
| Regression | `LinearRegression`, `Ridge` | `GradientBoostingRegressor`, `XGBoost` |
| Clustering | `KMeans` | `DBSCAN`, `AgglomerativeClustering` |
| Dimensionality reduction | `PCA` | `t-SNE`, `UMAP` |
| Anomaly detection | `IsolationForest` | one-class SVM |

Tree ensembles (Random Forest, XGBoost, LightGBM) win most tabular competitions. Always have one in your shortlist.

---

## Metrics — match to the problem

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, confusion_matrix, classification_report,
    mean_squared_error, r2_score,
)
```

- **Imbalanced classes?** Don't use accuracy. Use precision/recall/F1.
- **Probability matters?** ROC-AUC, log loss.
- **Regression?** RMSE, MAE, R². RMSE punishes large errors more than MAE.

The metric should match the cost of being wrong.

---

## Saving and loading models

```python
import joblib

joblib.dump(pipe, "model.joblib")
# In another process / day / machine:
loaded = joblib.load("model.joblib")
loaded.predict(new_X)
```

`joblib` is faster than `pickle` for large arrays. **Pin scikit-learn version** with the model — versions can change internal structures and break loading.

---

## When to stop and reach for deep learning

| Problem | sklearn or DL? |
| --- | --- |
| Tabular, < 1M rows | sklearn |
| Tabular, 10M+ rows | sklearn (XGBoost / LightGBM) |
| Images | DL (PyTorch + a pretrained model) |
| Text classification | DL (transformers via `transformers` lib) |
| Time series forecasting | sklearn or `statsmodels`; DL for very long series |
| LLM-shaped problems | not sklearn; see the LLM agents deck |

Tabular is sklearn's home turf. Outside it, the deep-learning ecosystem usually wins.

---

## What's next

- **LLM agents** — when the "model" is an external API
- **Async & concurrency** — for embedding training/inference in a web app
