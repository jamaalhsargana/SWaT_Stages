# Attack Detection and Stage Localisation on the SWaT Testbed

This repo has the code from my thesis: *Development of an Intelligent Intrusion Detection System for Smart Water Treatment and Distribution Plants in IoT-enabled Critical Infrastructures*. My supervisor was Dr Qaiser Riaz.

The idea was to use the same sensor data to answer two questions:

1. Is the plant being attacked right now?
2. If yes, which stage of the plant is the attack on?

The notebook here is the original one I used for my thesis, uploaded as it is. The results below are exactly what the notebook printed. After my thesis I went back through the code again and found a few mistakes in how I evaluated the models, mostly in the stage detection part. I've written them down in [Known limitations](#known-limitations)

---

## The problem

SWaT (Secure Water Treatment) is a small but real working water treatment plant built by iTrust at the Singapore University of Technology and Design. It has six stages: raw water intake, chemical dosing, ultrafiltration, dechlorination, reverse osmosis and backwash. PLCs read the sensors (flow, level, pressure, conductivity, pH, ORP) and control the pumps and valves.

If an attacker gets into a plant like this, they can fake a sensor value or turn a pump on or off and push the plant into a dangerous state, like overflowing a tank, running a pump dry or stopping dechlorination. What I tried to do is detect these attacks using only the sensor and actuator readings (the physical side of the plant). I did not use network traffic in this project.

When I was reading papers on SWaT, most of them only looked at "attack or no attack". I thought it would also be really useful to know *where* the attack is happening, so I added a second task where the model predicts which stage an attacked record belongs to.

## Data

I used the December 2015 SWaT attack dataset from iTrust.

| | |
|---|---|
| Records | 449,919 rows, one every second, starting 28 December 2015 |
| Columns | Timestamp, 51 sensor and actuator readings, and a label (`Normal` / `Attack`) |
| Class balance | 395,298 normal and 54,621 attacked (about 7 to 1) |
| Attacks | 41 attacks listed by iTrust (36 with physical impact), each with a start time, end time and target |

I can't share the dataset so it's not in this repo. You can ask iTrust for access (search "iTrust Labs datasets SWaT") and download the December 2015 attack file.

**Stage labels.** The original data only tells you if a record is normal or attacked, so I made a `Stage` column myself using iTrust's attack list. The first digit of the component that was attacked tells you the stage (e.g. `LIT301` is in stage 3), so every record in that attack's time window got that stage number. When an attack hit components in more than one stage, I gave it just one stage. I still need to upload the script I used for this.

| Stage | Meaning | Records |
|---|---|---|
| 0 | Normal | 395,298 |
| 1 | Attack on stage 1 | 6,669 |
| 2 | Attack on stage 2 | 638 |
| 3 | Attack on stage 3 | 40,968 |
| 4 | Attack on stage 4 | 4,370 |
| 5 | Attack on stage 5 | 1,976 |

The notebook needs two files:

- `swat_dataset_2015-1.csv` – the original attack file
- `swat_dataset_2015-1-s.csv` – the same file but with my `Stage` column added

## What's in the repo

```
Classification-Attack_normal.ipynb   my research notebook (Google Colab), both tasks are in here
README.md                            this file
```

## How I did it

### Choosing features

I checked how much each column correlated with the label over the whole dataset and kept the ten with the highest correlation:

`AIT402, FIT401, LIT401, AIT502, FIT501, FIT502, FIT503, FIT504, PIT501, PIT503`

I only noticed later that all ten of these are from stages 4 and 5. So attacks on stages 1 to 3 can only be caught through the effects they have further down the plant.

### Task 1: attack or normal

1. I split each class based on time. The first 300,000 normal records and the first 40,000 attack records went to training, and the rest went to testing.
2. To help with the imbalance I added the training attack records in a second time. I did the same thing to the test set, which I now know was wrong (see limitations).
3. I shuffled the training data and divided it into five "buckets" of around 76,000 records each (about 79% normal and 21% attack).
4. I trained a model on each bucket and tested all of them on the same test set:
   - Random Forest (I used the regressor, 30 trees, depth 15, and counted anything above 0.90 as an attack)
   - Linear SVM
   - A small neural network (two hidden layers of 16 neurons, ReLU and sigmoid, Adam, 10 epochs, threshold 0.85)
5. For each bucket I recorded accuracy, F1 and mean absolute error, then took the average.

### Task 2: which stage

1. I made six buckets. Every bucket has all of the attack records (stage 3 once, and stages 1, 2, 4 and 5 copied five times) plus a different chunk of about 65,000 normal records.
2. I shuffled each bucket and split it randomly into train and test (70/30 for Random Forest, 80/20 for SVM).
3. Then I trained and tested on each bucket:
   - Random Forest classifier (30 trees, depth 15, entropy)
   - Linear SVM, with PCA and scaling first
   - A neural network, which unfortunately didn't work (explained below)

## Running it

I wrote this in Google Colab and while I was experimenting I ran the cells in different orders, so it won't run cleanly from top to bottom yet (sorry!). If you want to try it:

1. Get the two CSV files from the [Data](#data) section.
2. Open the notebook in Colab. Cell 4 downloads my files from my Google Drive using PyDrive, which won't work for anyone else, so replace it with something like this:
   ```python
   df  = pd.read_csv('/content/drive/MyDrive/swat/swat_dataset_2015-1.csv')
   dfs = pd.read_csv('/content/drive/MyDrive/swat/swat_dataset_2015-1-s.csv')
   ```
3. Use `pandas<2.0`. I used `DataFrame.append`, which was removed in pandas 2.0. Or you can change each `a.append(b)` to `pd.concat([a, b])`.
4. Skip these cells because they won't run in a fresh session:
   - the Logistic Regression and Gaussian Naive Bayes cells (they use an `x_train` variable that doesn't exist anymore)
   - the stage neural network cells (they use `dfs_corr_1_nn`, which never gets created)
5. Move the `RandomForestClassifier` import above the first Random Forest cell in Task 2. Right now it gets imported further down.

Libraries: `pandas<2.0`, `numpy`, `scikit-learn`, `tensorflow`, `matplotlib`, `seaborn`.

## Results

All of these numbers come directly from the outputs saved in the notebook.

### Task 1: attack or normal

| Model | Buckets with saved output | Test accuracy, mean (range) | Test F1 for attacks, mean (range) |
|---|---|---|---|
| Random Forest | 4 of 5 (buckets 2–5) | 0.834 (0.833–0.835) | 0.455 (0.452–0.461) |
| Linear SVM | 4 of 5 (buckets 1, 2, 4, 5) | 0.899 (0.884–0.904) | 0.727 (0.675–0.744) |
| Neural network | 5 of 5 | 0.884 (0.833–0.906) | 0.625 (0.317–0.722) |

The linear SVM worked best and was also the most consistent. The neural network did well on four buckets but really dropped on bucket 2 (F1 of 0.32), so the average hides quite a lot of variation. The Random Forest with my 0.90 threshold missed most of the attacks.

### Task 2: which stage

| Model | Accuracy | Macro F1 | Weakest class |
|---|---|---|---|
| Random Forest | 0.999 on all 5 saved buckets | 1.00 | nothing below 0.99 |
| Linear SVM + PCA | 0.931–0.939 across 6 buckets | 0.82–0.83 | stage 2, F1 0.37 |
| Neural network | 0.72 | 0.16 | stages 2–5, F1 0.00 |

Please don't read these as real stage detection results. The Random Forest score looks almost perfect, but that's mostly because of the leakage problems I explain below. I think the SVM's per-class scores show more honestly how hard this task is. It had a lot of trouble with stage 2, which only has 638 records.

The neural network for this task just didn't work. I set it up with one sigmoid output and binary cross-entropy, which is only meant for two classes, and then trained it on six classes. The loss went to huge negative numbers and the model predicted "normal" for almost everything. So there is no working neural network for stage detection in this notebook.

## Known limitations

When I went back through my notebook I found these problems. I'm listing all of them here because they affect how much the numbers above can be trusted, and because I learned a lot from finding them.

**Stage detection – the results are inflated**

1. **The model was basically given the answer.** For stage detection, `X = dfs_corr.iloc[:, 0:11]` takes the ten sensors *and* the `Attacked` column. A record is normal exactly when its stage is 0, so the model already knew which records were attacks and only had to choose between stages 1 to 5.
2. **The same rows were in both train and test.** I copied the smaller stages five times before splitting, so the exact same rows ended up on both sides.
3. **I split time-series data randomly.** Readings that are one second apart during the same attack are almost the same. With a random split, neighbouring seconds go into both train and test, so the model gets tested on attacks it has more or less already seen.
4. **The buckets are not independent.** Every bucket has the same attack records, so six buckets is closer to running one experiment six times than doing six separate tests.
5. **PCA didn't actually reduce anything.** I asked for 11 components from 11 inputs, which just rotates the data. I also fitted it on the whole bucket including the test rows.

**Attack detection – smaller issues**

6. **The test attacks are counted twice.** I added the 14,619 test attack records a second time (29,238 in total), which changes both accuracy and F1.
7. **I picked the features using the whole dataset**, including the test period.
8. **I chose the thresholds (0.90 and 0.85) by hand**, without a separate validation set.
9. **I used a Random Forest regressor with a cut-off.** A classifier with class weights would have been the better choice.
10. **I only measured performance record by record.** SWaT papers usually also report event-based results (was each attack caught, and how fast), which accuracy on its own doesn't show.

**General**

11. **Mean absolute error doesn't add anything here.** With 0/1 labels it's just the error rate.
12. **Some outputs are missing.** Random Forest bucket 1 and SVM bucket 3 don't have saved results, and SVM buckets 2 and 4 show exactly the same scores, so I think one of them got overwritten when I re-ran a cell.
13. **All ten features are from stages 4 and 5**, so it's hard to find attacks that happen earlier in the plant.
14. **I only used the December 2015 attack file.** I didn't train on a separate normal-operation run or test on the 2019 SWaT data.

## What I'm working on next

- Removing the `Attacked` column from the stage detection inputs
- Splitting train and test by attack and in time order, so the same attack never appears on both sides
- Only resampling (or using class weights) on the training data, after the split
- Choosing features using only the training data, and thresholds using a validation set
- Adding event-based metrics, a proper multi-class neural network, and a time-series model like an LSTM or autoencoder
- Uploading my stage labelling script, and turning the notebook into proper scripts with a `requirements.txt`

## Reference

Sargana, J. H. (2023). *Development of an Intelligent Intrusion Detection System for Smart Water Treatment and Distribution Plants in IoT-enabled Critical Infrastructures.* MS thesis, School of Electrical Engineering and Computer Science, National University of Sciences and Technology (NUST), Islamabad.

Dataset: Goh, J., Adepu, S., Junejo, K. N. and Mathur, A. (2017). A dataset to support research in the design of secure water treatment systems. In *Critical Information Infrastructures Security (CRITIS 2016)*, Springer, pp. 88–99.

## Thanks

Big thanks to iTrust, Centre for Research in Cyber Security at the Singapore University of Technology and Design, for sharing the SWaT data with researchers.
