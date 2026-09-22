# RPL IoT Intrusion Detection (Routing Attack Classification)

A machine learning project that classifies IoT network traffic as **Normal** or one of four
**RPL routing-protocol attacks** Blackhole, Flooding, Rank, and Version and, more
importantly, documents the real engineering process behind it: a data leakage bug that was
found and fixed, an honest comparison across a statistical baseline and two stronger models,
and a diagnosis of *why* accuracy plateaus where it does.

This README is written to walk through that whole process, not just report a final number.

---

## 1. The Problem

Detect and classify **RPL** (Routing Protocol for Low-power and Lossy Networks) routing attacks in IoT networks using machine learning. The goal is to identify whether network traffic is Normal or one of four attacks: Blackhole, Flooding, Rank, and Version.

| Attack | What it does |
|---|---|
| **Blackhole** | A malicious node silently drops all traffic routed through it |
| **Flooding** | Overwhelms the network with excessive control messages |
| **Rank** | A node lies about its position in the routing tree to attract traffic |
| **Version** | Forces unnecessary, costly network-wide topology rebuilds |

**Goal:** given a single simulated RPL control message, predict whether it's Normal traffic
or one of these four attacks, A **5-class supervised classification** problem.

---

## 2. The Dataset



IoT-RPL 2021 dataset containing 1,048,575 network traffic records and five classes: Normal, Blackhole, Flooding, Rank, and Version.

- Class distribution (imbalanced — see below):

| Label | Count |
|---|---|
| Flooding | 365,603 |
| Normal | 219,470 |
| Blackhole | 185,749 |
| Version | 147,224 |
| Rank | 130,529 |

Columns include protocol layers (`frame_proto`, `protocol`, `control_type`), the RPL message
type (`type_cont_messg`: DIS/DIO/DAO), and protocol-specific fields (`DOAGID`, `DIO_info`,
`object_cont_pt`, etc.).


**[IoT-RPL 2021: Cyber Attack Dataset Based on RPL Routing for IoT](https://data.mendeley.com/datasets/4rcbbry2sc/1)**
— Walid Dhifallah, Mounira Tarhouni, Tarek Moulahi, Salah Zidi. Mendeley Data, DOI:
[10.17632/4rcbbry2sc.1](https://doi.org/10.17632/4rcbbry2sc.1), licensed CC BY 4.0.

---

## 3. Preprocessing Pipeline

Real-world data isn't clean — several columns mix numbers, hex codes, and comma-separated
lists depending on the RPL message type. For every feature column, the pipeline:

1. **Detects list-type columns** (e.g. `to`, a comma-separated neighbor list like `"4,10,11"`)
   → converts to a **neighbor count** instead.
2. **Tries numeric conversion** (`pd.to_numeric`) — if 90%+ of a column converts cleanly, it's
   treated as a real number (gaps filled with 0).
3. **Otherwise, treats it as categorical** and applies **Label Encoding** (text → integer ID).
4. The target label (`Normal`/`Blackhole`/.../`Version`) is also **Label Encoded** into a plain
   integer — no further transformation needed, since every model used here (Logistic
   Regression, Random Forest, Gradient Boosting) works directly with integer labels.
5. **Scaling**: `StandardScaler` normalizes every feature to a comparable range (protocol
   fields like `DOAG_info` reach into the billions; others are 0/1 flags).
6. **Train/test split**: 80/20, **stratified** to preserve class proportions given the imbalance.

Full step-by-step version, with explanations: [`model.ipynb`](model.ipynb).

---

## 4. The Data Leakage Bug — and Why It Mattered

The first training run scored a **suspicious 100% accuracy** across all 5 classes. Instead of
reporting that as a win, we investigated *why*.

**Finding:** the `from` column (the simulated device's node ID) **perfectly predicted the
label** — every value mapped to exactly one class:

| `from` value | Always labeled |
|---|---|
| 1–11 | Normal |
| 12 | Blackhole |
| 13, 16 | Flooding |
| 14 | Rank |
| 15 | Version |

Each attacker device in the simulation only ever ran **one** type of attack for the whole run.
The model wasn't learning attack *behavior* — it was memorizing "device #12 = Blackhole,"
a shortcut that would be **completely useless in a real network** where you don't already
know which device is malicious.

**Fix:** dropped `from` from the feature set entirely, forcing the model to learn from actual
traffic/message content. Accuracy dropped to a realistic **~72–76%** — not a regression, but
the removal of a lie.

---

## 5. Baseline: Logistic Regression

Before reaching for a more complex model, Logistic Regression (a basic linear statistical
classifier) was trained first (Section 7 of the notebook) to check how much of the signal in
the data is simple versus how much needs a stronger model to capture.

## 6. Model Comparison

Three models total were trained and evaluated on the same leak-free data:

| Model | Test Accuracy | Notes |
|---|---|---|
| **Logistic Regression** (basic statistical baseline) | 62.82% | Most of the signal is fairly simple/near-linear |
| **Random Forest** (100 trees, balanced class weights) | 71.71% | Built specifically for tabular data like ours |
| **Gradient Boosting** (`HistGradientBoostingClassifier`) | **73.06%** | Highest raw accuracy — but doesn't fix the real bottleneck (see next section) |

Random Forest and Gradient Boosting only beat Logistic Regression by about 8–10 points,
meaning the extra model complexity buys a real but modest gain over a simple statistical model.

**Why try Gradient Boosting after Random Forest?** To test whether a fundamentally different,
stronger algorithm could break through the accuracy plateau. Gradient Boosting builds trees
sequentially, each one correcting the previous trees' errors, rather than Random Forest's
independent trees averaged together — a real track record of beating Random Forest on tabular
data. If the plateau were a model-capability problem, this is the model that should have
broken through it. It didn't fix the underlying confusion (Section 7) — which is itself the
finding: the bottleneck is the data, not the model.

---

## 7. Diagnosis: Where the Remaining Errors Actually Come From

Overall accuracy hides detail. The confusion matrix (Random Forest) shows exactly where:

```
           Blackhole  Flooding  Normal   Rank  Version
Blackhole      12023      2083    3215   3673    16156   ← 37,150 real Blackhole rows
Flooding           0     53011   13514   6132      463
Normal            10       132   42339    763      650
Rank             331       366    3088  21854      467
Version          174      1904    2826   3373    21168
```

**Of 37,150 real Blackhole attacks, 16,156 were misclassified as Version.** That's the single
biggest error source. Investigating *why*: a column-by-column comparison of Blackhole vs.
Version rows shows their real protocol-content fields (`DIO_info`, `DOAGID`, message type
proportions, etc.) are **nearly identical** between the two classes. The one column that
*does* differ (`to`, the neighbor list) turns out to encode **which simulated nodes were used
per scenario** — the same kind of leakage as `from`, not genuine attack behavior.

**Conclusion:** Blackhole and Version attacks look almost the same at the level of a single
message. This is a **data limitation, not a model limitation** — confirmed by testing Random
Forest and Gradient Boosting, two very differently-built models, and finding the exact same
confusion pattern in both (Gradient Boosting didn't fix it, it just shifted which of the two
classes gets favored: Blackhole recall improved from 32% → 69%, while Version recall dropped
from 72% → 29%).

A genuine fix would likely require **sequence/time-based features** (e.g. how a node's
claimed rank changes across its last several messages) rather than single-message snapshots
— which this dataset can't cleanly support (no timestamp column, and the only per-node
grouping key available is `from`, which would reintroduce the original leakage).

---

## 8. Key Takeaways

- **A perfect score is a red flag, not a win** — always ask *why* before trusting a great result.
- **Switching models isn't a substitute for understanding the data** — Random Forest and
  Gradient Boosting, two very differently-built models, hit the identical confusion pattern,
  proving the bottleneck was the data, not the model.
- **Class imbalance matters** — handled via stratified splitting and class weighting, not ignored.
- **Not every "improvement" is real** — a raw accuracy increase (Gradient Boosting) can hide a
  trade-off (worse recall on a different class) that only shows up in a per-class breakdown.
- **Simpler is often good enough** — Logistic Regression got within about 8–10 points of the
  more complex models, and a CNN tried early on never beat plain Random Forest despite far more
  code and setup, so it was dropped in favor of these simpler, better-suited models.

---

## 9. Limitations & Honest Caveats

- Final accuracy (~71–76%) is capped by genuine overlap between Blackhole and Version classes
  in the available features, not by model choice (see Section 7).
- No production/deployment layer — this is a modeling/analysis project (an earlier Flask web
  UI was removed since it wasn't wired to the model and added complexity without value).

---

## Tech Stack

`pandas`, `numpy`, `scikit-learn` (LabelEncoder, StandardScaler, LogisticRegression,
RandomForestClassifier, HistGradientBoostingClassifier).

## Project Structure

```
Dataset/RPL_Routing_Attacks.csv       # dataset
model.ipynb                            # full pipeline, step by step, with explanations
images/                                 # confusion matrix, class distribution, classification report
README.md
```

## How to Run

Open [`model.ipynb`](model.ipynb) in Jupyter or VS Code and run all cells top to bottom.
Requires `pandas`, `numpy`, `scikit-learn`.

## Dataset Citation

> Dhifallah, W., Tarhouni, M., Moulahi, T., Zidi, S. (2024). *IoT-RPL 2021: Cyber Attack
> Dataset Based on RPL Routing for IoT.* Mendeley Data, V1. DOI:
> [10.17632/4rcbbry2sc.1](https://doi.org/10.17632/4rcbbry2sc.1)

Note: the dataset's official label names are **Blackhole Attack**, **Flooding Attack**,
**DODAG Version Number Attack**, and **Decreased Rank Attack** — shortened to `Blackhole`,
`Flooding`, `Version`, and `Rank` respectively in the CSV and throughout this project.
