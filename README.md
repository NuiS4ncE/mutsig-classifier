# Predicting Cancer Types from Mutational Profiles & Signature Activities

> A small course project repo. We train ML models to guess **cancer types** from
> 96‑channel mutational **catalogues** (SBS trinucleotide profiles) and optionally
> predicted **signature activities** (SBS/DBS/ID exposures). It’s not perfect, but it works
> good enough for our analysis and figures.

---

## TL;DR

- Input catalog files are in **PCAWG-wide format** (rows = channels; columns = samples like `Type::SampleID`).
- We **transpose** those, **normalize** per-sample, and parse the **label** from the column name left side (before `::`).
- We (optionally) merge in **activities** if they exist *and overlap* by sample IDs.
- Models: Logistic Regression, RandomForest, HistGB, Linear SVM, and a small MLP.
- We report **accuracy**, **macro-F1**, confusion matrices, and per‑class F1 bars.
- Results get saved to `model_results_summary.csv`.

There’s a couple rough edges and a few hacks—pls read the “Troubleshooting” section if something looks weird. :D

---

## Repo Layout (kinda)

```
.
├─ project_data/
│  ├─ catalogs/
│  │  ├─ WGS/
│  │  │  ├─ WGS_PCAWG.96.csv
│  │  │  └─ WGS_Other.96.csv
│  │  └─ WES/
│  │     ├─ WES_TCGA.96.csv
│  │     └─ WES_Other.96.csv
│  └─ activities/
│     ├─ WGS/
│     │  ├─ WGS_PCAWG.activities.csv
│     │  └─ WGS_Other.activities.csv
│     └─ WES/
│        ├─ WES_TCGA.activities.csv
│        └─ WES_Other.activities.csv
├─ notebook.ipynb   # the main analysis (run this one)
└─ model_results_summary.csv  # written after training (auto)
```

> If you don’t have activities files, **no problem**. You’ll still get solid results from the 96‑channel profiles alone.

---

## Data Format (important-ish)

**Catalogs (96 SBS):**

- Must have two meta columns: `Mutation type` and `Trinucleotide`.
- All other columns are **samples**, named like `Liver-HCC::SP97749`.
- We parse the label from the header **left of `::`** → `Liver‑HCC` in the example.
- We convert (Trinucleotide, Mutation type) into channel names like `A[C>A]A`.

**Activities (optional):** either
- **Case A**: a column `Signature`, with the rest of columns being samples (same `Type::SampleID` style). We transpose.
- **Case B**: rows are samples; signature columns named like `SBS*`, `DBS*`, `ID*` plus some ID column.

If activities don’t **overlap** with catalog samples, we stick to catalogs (to avoid unlabelled rows).

---

## Requirements

- Python 3.9+ (3.10 also fine).
- Packages: `numpy`, `pandas`, `scikit-learn`, `matplotlib`, `seaborn`.

You can install like so (there’s no `requirements.txt` yet, sorry):

```bash
pip install numpy pandas scikit-learn matplotlib seaborn
```

> Note: Some very old scikit‑learn builds don’t like `multi_class='multinomial'` or `l1_ratio` in `LogisticRegression`. We handle that by using a compat config (so it should not crash, hopefully).

---

## How to Run (quickstart)

1. Put the CSV files under `project_data/` as shown above.
2. Open the Jupyter notebook (`notebook.ipynb`) and run from top to bottom (or use “Run All”).
3. Watch the console prints for sanity info:
   - number of samples & features
   - whether the index is unique
   - label counts
4. At the end you’ll get:
   - **Result table**: a Pandas DF of metrics (`results_df`)
   - **Plots**: confusion matrices & per‑class F1 bars
   - Saved CSV: `model_results_summary.csv`

If you’re missing activities, you’ll see something like:

```
{'profiles_only': 96, 'combined': 96}
```

This just means *combined == profiles_only*, which is fine.

---

## Configuration & Knobs

Inside the notebook:

- `INCLUDE_WGS = True` / `INCLUDE_WES = True`
  Toggle data sources (depends what you have).
- `MIN_SAMPLES_PER_CLASS = 15`
  Classes with fewer than this many samples are dropped (stabilizes stratified split).
- `RUN_MODE = 'quick'`
  Switch to `'thorough'` to increase permutation repeats for feature importance (slower).
- `MERGE_STRATEGY = 'intersection'`
  Keeps only samples present in **both** catalogs and activities (recommended). Set to `'catalog_only'` to ignore activities.

---

## What You’ll See

- **Metrics table** per (feature‑set × model):
  - `accuracy`, `f1_macro`, `f1_weighted`, etc.
- **Best models per feature set**
- **Confusion matrix heatmap** (row‑normalized)
- **Per‑cancer type F1** bars (sorted)
- **Top features** for the best models:
  - RF: impurity importances
  - LogReg: mean |coef| across classes
  - Others: permutation importance

Some biologically sensible signals often pop up:
- UV‑related channels / signatures for melanomas,
- Smoking‑related signatures for lung & head/neck,
- Mismatch repair-ish stuff for certain GI tumors, etc. (don’t overclaim tho).

---

## Troubleshooting (stuff I tripped over)

- **Huge number of `y` nulls**
  - Usually means you pulled in **activities‑only** samples via an outer join (no labels).
  - We now use **intersection** to avoid that. If it still happens, check activities headers match `Type::SampleID`.

- **Duplicate sample index**
  - Can happen if the same sample appears in multiple files.
  - We **collapse** duplicates (mean for features, mode for labels). Should print a notice.

- **`LogisticRegression` TypeError about `multi_class`**
  - Old scikit‑learn. We removed the arg and fall back to a safe setup. If it still complains, switch to `solver='lbfgs', penalty='l2'`.

- **Activities missing**
  - Totally ok. Profiles alone (`96` features) gives you a baseline. You’ll just see `activities_only` absent and `combined` == `profiles_only`.

- **Weird channel names**
  - Catalogs must have `Mutation type` and `Trinucleotide`. If not, the loader bails out.

If you’re stuck, print some heads/tails:

```python
print(X.shape, y.shape)
print(y.value_counts().head(15))
list(FSETS.keys()), {k: len(v) for k, v in FSETS.items()}
```

---

## Reproducibility note

We set `RANDOM_STATE = 42` and use stratified splits. For more reliable estimates add **5‑fold StratifiedKFold** and average the scores (left out to keep the run time reasonable-ish).

---




## TODOs (we might do later, or not)

- Proper `requirements.txt` and/or `environment.yml`
- Add cross‑validation mode (`RUN_MODE='thorough'`)
- A proper label harmonization map (merge some subtypes more cleanly)
- Better documentation for activities formats (with examples)


