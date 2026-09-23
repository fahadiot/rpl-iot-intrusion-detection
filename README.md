# RPL IoT Intrusion Detection

Small ML project that classifies simulated IoT network traffic as Normal or one of four RPL routing attacks: Blackhole, Flooding, Rank, or Version.

RPL (Routing Protocol for Low-power and Lossy Networks) is what IoT devices use to build their routing tree, and it's a common target for these kinds of attacks.

## Dataset

Using the IoT-RPL 2021 dataset from Mendeley Data (Dhifallah, Tarhouni, Moulahi, Zidi), DOI: [10.17632/4rcbbry2sc.1](https://doi.org/10.17632/4rcbbry2sc.1).

The full dataset is split into 10 files (~10 million rows total). This project just uses one file (`0.csv`, ~1 million rows) to keep things fast to iterate on.

Class counts are pretty imbalanced:

| Label | Count |
|---|---|
| Flooding | 365,603 |
| Normal | 219,470 |
| Blackhole | 185,749 |
| Version | 147,224 |
| Rank | 130,529 |

## What's in the notebook

`model.ipynb` walks through the whole thing step by step:

- Load and clean the data (some columns are numbers, some are hex/text codes, some are comma separated lists like neighbor IDs)
- Encode the text labels into numbers
- Scale the features
- Train a Logistic Regression baseline, then Random Forest, then Gradient Boosting

One thing worth mentioning: the first run of this scored 100% accuracy, which was a red flag, not a good sign. Turned out one column (the simulated device ID) gave away the label directly, since each attacker device only ever ran one type of attack in the simulation. Dropped that column once I figured out why, and accuracy dropped to a more realistic ~72-76%.

## Results

| Model | Accuracy |
|---|---|
| Logistic Regression | 62.8% |
| Random Forest | 71.7% |
| Gradient Boosting | 73.1% |

Random Forest and Gradient Boosting only beat plain Logistic Regression by about 10 points, so most of the signal in this data is fairly simple.

Looking at the confusion matrix, both Random Forest and Gradient Boosting mix up Blackhole and Version attacks a lot more than any other pair, since those two look very similar in the available features. That seems to be a limit of the data itself rather than something a different model would fix (a CNN was tried too, but never did better than Random Forest, so it wasn't worth the extra complexity).

## Files

- `model.ipynb`: the main notebook
- `Dataset/RPL_Routing_Attacks.csv`: the data
- `images/`: class distribution chart, confusion matrix, classification report
- `Project_Report.pdf`: a write-up version of the same thing

## Running it

Open `model.ipynb` in Jupyter or VS Code and run all cells. Needs `pandas`, `numpy`, and `scikit-learn`.
