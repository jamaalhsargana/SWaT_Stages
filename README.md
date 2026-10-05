# Attack Detection and Stage Localisation on the SWaT Water Treatment Testbed

Machine-learning intrusion detection for an industrial control system (ICS), developed for my MS thesis at NUST (2023):
*Development of an Intelligent Intrusion Detection System for Smart Water Treatment and Distribution Plants in IoT-enabled Critical Infrastructures* (supervisor: Dr Qaiser Riaz).

The project asks two questions of the same sensor data:

1. **Detection:** is the plant under attack at this second?
2. **Localisation:** if so, which of the plant's process stages is being attacked?

> **Status.** This repository holds the original research notebook, published as it was run. Its recorded outputs are reported below exactly as they appear in the notebook, together with the evaluation problems I have since found in it (see [Known limitations](#known-limitations)). A corrected re-evaluation is in progress.

---

## Problem

The Secure Water Treatment (SWaT) testbed at iTrust, Singapore University of Technology and Design, is a scaled-down six-stage water treatment plant: raw water intake (P1), chemical dosing (P2), ultrafiltration (P3), dechlorination (P4), reverse osmosis (P5) and backwash (P6). Programmable logic controllers read sensors (flow, level, pressure, conductivity, pH, ORP) and drive actuators (pumps and motorised valves).

Attacks on such plants manipulate sensor readings or actuator states to push the physical process into an unsafe condition: overflowing a tank, running a pump dry, or stopping dechlorination. This project detects those attacks from the **process-level sensor and actuator readings** (the physical layer), not from network traffic.

Most published work on SWaT stops at binary detection. This project also attempts **stage localisation**: labelling each attacked record with the plant stage that was targeted, so an operator knows where to look.

## Data

| Item | Detail |
|---|---|
| Source | iTrust SWaT dataset, December 2015 collection (attack/test file) |
| Records | 449,919 rows, one per second, starting 28 December 2015 |
| Columns | Timestamp, 51 sensor and actuator readings, label (`Normal` / `Attack`) |
| Class balance | 395,298 normal, 54,621 attacked (about 7.2 : 1) |
| Attacks | 41 attacks documented by iTrust (36 with physical impact), each with start/end time and attack point |

**Access.** The SWaT dataset is not redistributable. Request access from iTrust (search "iTrust Labs datasets SWaT") and download the December 2015 attack file. No data is included in this repository.

**Stage labels (added for this project).** The notebook expects a second copy of the file with an extra `Stage` column:

| Stage value | Meaning | Records |
|---|---|---|
| 0 | Normal operation | 395,298 |
| 1 | Attack targeting stage 1 | 6,669 |
| 2 | Attack targeting stage 2 | 638 |
| 3 | Attack targeting stage 3 | 40,968 |
| 4 | Attack targeting stage 4 | 4,370 |
| 5 | Attack targeting stage 5 | 1,976 |

Stage labels were derived from iTrust's attack list. Each attack's target component (for example `LIT301`) encodes its stage in its first digit, and every record inside that attack's time window was labelled with that stage. Attacks hitting components in more than one stage were assigned to a single stage. The script that applied this mapping is not yet in the repository.

**Expected files** (names used by the notebook):

- `swat_dataset_2015-1.csv`: the iTrust attack file
- `swat_dataset_2015-1-s.csv`: the same file with the `Stage` column

## Repository contents

```
Classification-Attack_normal.ipynb   Research notebook (Google Colab), both pipelines
README.md                             This file
```

## Method

### Feature selection

Pearson correlation of every column with the label was computed on the full dataset. The 10 features with the strongest correlation were kept for both pipelines:

`AIT402, FIT401, LIT401, AIT502, FIT501, FIT502, FIT503, FIT504, PIT501, PIT503`

All 10 belong to stages 4 and 5. Attacks on stages 1–3 are therefore detected only through their downstream effects.

### Pipeline A: attack vs normal (binary)

1. Split each class by position in the time series: the first 300,000 normal and first 40,000 attack records go to training, the rest to testing.
2. Append the training attack records a second time to reduce imbalance (the test set is built the same way; see limitations).
3. Shuffle the training set and cut it into 5 "buckets" of about 76,000 records (about 79% normal, 21% attack).
4. Train one model per bucket and evaluate each against the same held-out test set:
   - Random Forest (regressor, 30 trees, depth 15, output thresholded at 0.90)
   - Linear SVM
   - Neural network (16-16-1, ReLU/sigmoid, Adam, 10 epochs, output thresholded at 0.85)
5. Report accuracy, F1 and mean absolute error per bucket and averaged across buckets.

### Pipeline B: stage localisation (6 classes)

1. Build 6 buckets. Each bucket contains **all** attack records: stage 3 once, and stages 1, 2, 4 and 5 copied 5 times. Each bucket also contains a different slice of about 65,000 normal records.
2. Shuffle each bucket and split it randomly 70/30 (Random Forest) or 80/20 (SVM) into train and test.
3. Train and evaluate per bucket:
   - Random Forest classifier (30 trees, depth 15, entropy)
   - Linear SVM after PCA and standard scaling
   - Neural network (attempted; see results)

## How to run

The notebook was written and run in **Google Colab** and is not yet runnable top-to-bottom; cells were executed out of order during development. To reproduce it:

1. Obtain the two CSV files described in [Data](#data).
2. Open the notebook in Colab and replace the PyDrive download cell (cell 4) with a direct read from your own storage, for example:
   ```python
   df  = pd.read_csv('/content/drive/MyDrive/swat/swat_dataset_2015-1.csv')
   dfs = pd.read_csv('/content/drive/MyDrive/swat/swat_dataset_2015-1-s.csv')
   ```
3. Use `pandas<2.0`, because the notebook calls `DataFrame.append`, which pandas 2.0 removed. Alternatively, replace each `a.append(b)` with `pd.concat([a, b])`.
4. Skip the cells that are known not to run in a fresh session:
   - The Logistic Regression and Gaussian Naive Bayes cells, which reference an undefined `x_train`
   - The stage-detection neural network cells, which reference an undefined `dfs_corr_1_nn`
5. Import `RandomForestClassifier` before the first Random Forest cell in Pipeline B; it is currently imported later.

Main dependencies: `pandas<2.0`, `numpy`, `scikit-learn`, `tensorflow`, `matplotlib`, `seaborn`.

## Results

All figures below are copied from the outputs saved in the notebook.

### Pipeline A: attack vs normal

| Model | Buckets with saved output | Test accuracy, mean (range) | Test F1 (attack class), mean (range) |
|---|---|---|---|
| Random Forest | 4 of 5 (buckets 2–5) | 0.834 (0.833–0.835) | 0.455 (0.452–0.461) |
| Linear SVM | 4 of 5 (buckets 1, 2, 4, 5) | 0.899 (0.884–0.904) | 0.727 (0.675–0.744) |
| Neural network | 5 of 5 | 0.884 (0.833–0.906) | 0.625 (0.317–0.722) |

The linear SVM gave the best and most stable attack detection. The neural network was strong on four buckets but collapsed on bucket 2 (F1 0.32), so its average hides high variance. The Random Forest regressor, thresholded at 0.90, missed most attacks.

### Pipeline B: stage localisation

| Model | Accuracy | Macro F1 | Weakest class |
|---|---|---|---|
| Random Forest | 0.999 (all 5 saved buckets) | 1.00 | none below 0.99 |
| Linear SVM + PCA | 0.931–0.939 (6 buckets) | 0.82–0.83 | Stage 2: F1 0.37 |
| Neural network | 0.72 | 0.16 | Stages 2–5: F1 0.00 |

**These stage results should not be read as real localisation performance.** The Random Forest's near-perfect score is explained by the leakage described below. The SVM's per-class scores (stage 2 F1 0.37) are a better guide to how hard the task is. The neural network cell trained a single sigmoid output with binary cross-entropy on 6-class labels; its loss diverged to large negative values and it predicted almost only the normal class. **A working multi-class neural network for stage localisation is not part of this notebook.**

## Known limitations

These are the issues I found when reviewing the notebook. I list them because they determine how far the numbers above can be trusted.

**Stage localisation (Pipeline B): results are inflated by leakage**

1. **The binary label is used as an input feature.** `X = dfs_corr.iloc[:, 0:11]` takes the 10 sensors *plus* the `Attacked` column. Since `Attacked = 0` exactly when `Stage = 0`, the model is told whether a record is an attack and only has to choose among stages 1–5.
2. **Copies of the same record land in both train and test.** Minority stages are copied 5 times *before* the random train/test split, so identical rows appear on both sides.
3. **Random splits of a per-second time series.** Neighbouring seconds of the same attack are nearly identical; random splitting puts them in both train and test, so the model can recognise an attack it has effectively already seen.
4. **Buckets are not independent.** Every bucket contains the same attack records, so the 5–6 buckets are near-repeats of one experiment, not independent evaluations.
5. **PCA does not reduce dimensionality here.** `PCA(n_components=11)` on 11 inputs is a rotation, not a reduction. It is also fitted on the whole bucket, test rows included.

**Attack detection (Pipeline A): smaller issues**

6. **Test attacks are counted twice.** The held-out test set appends its 14,619 attack records a second time (29,238 attack rows), which changes accuracy and F1.
7. **Feature selection used the full dataset**, test period included.
8. **Decision thresholds (0.90, 0.85) were chosen by hand** without a separate validation set.
9. **The Random Forest is a regressor** thresholded into a classifier; a classifier with class weights is the standard choice.
10. **Point-wise metrics only.** The SWaT literature also reports event-based detection (was each attack caught, and how fast), which point-wise accuracy does not show.

**General**

11. **Mean absolute error is used as a classification metric.** It adds nothing beyond accuracy for 0/1 labels.
12. **Outputs are missing for some buckets** (Random Forest bucket 1, SVM bucket 3). SVM buckets 2 and 4 report identical scores, which suggests one output was overwritten by a re-run.
13. **The 10 features all come from stages 4 and 5**, which limits localisation of upstream attacks.
14. **Only the December 2015 attack file is used.** There is no separate normal-operation training run and no test on the 2019 SWaT collections.

## Planned revisions

- Remove the `Attacked` column from stage-localisation inputs.
- Split train and test **chronologically by attack event**, so no attack appears in both.
- Resample (or use class weights) **only inside the training set**, after splitting.
- Select features on training data only, and choose thresholds on a validation set.
- Add event-based metrics, a proper multi-class neural network, and a time-series model (LSTM or autoencoder).
- Release the stage-labelling script and convert the notebook into runnable scripts with a pinned `requirements.txt`.

## Reference

Sargana, J. H. (2023). *Development of an Intelligent Intrusion Detection System for Smart Water Treatment and Distribution Plants in IoT-enabled Critical Infrastructures.* MS thesis, School of Electrical Engineering and Computer Science, National University of Sciences and Technology (NUST), Islamabad.

Dataset: Goh, J., Adepu, S., Junejo, K. N., and Mathur, A. (2017). A dataset to support research in the design of secure water treatment systems. In *Critical Information Infrastructures Security (CRITIS 2016)*, Springer, pp. 88–99.

## Acknowledgements

Data provided by iTrust, Centre for Research in Cyber Security, Singapore University of Technology and Design.
