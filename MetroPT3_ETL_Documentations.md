# MetroPT-3 ETL — Full Walkthrough, Plain Language
*What every block of code does, and why we did it. Written so you can explain this to Pete or your team without needing to re-read the code.*

---

## 1. Connecting to your files

```python
from google.colab import drive
drive.mount('/content/drive')
```

**What it does:** Colab runs on Google's computers, not yours — so it can't see files sitting on your laptop or even in your Google Drive by default. This line builds a bridge between Colab and your Drive, so any code after this can reach files stored there.

**Why:** Without this, `pd.read_csv(...)` would fail — Colab simply wouldn't know your file exists.

---

## 2. Loading the raw file

```python
import pandas as pd
import numpy as np
df = pd.read_csv(".../MetroPT3(AirCompressor).csv")
print(df.shape)
print(df.head())
```

**What it does:**
- `import pandas as pd` — brings in the toolkit for working with spreadsheet-style data in Python, and gives it a short nickname (`pd`) so you don't retype "pandas" constantly.
- `import numpy as np` — a second toolkit, mainly for numbers and calculations we'll need later (like `np.nan`, which means "no value here").
- `pd.read_csv(...)` — opens the CSV file and loads it into a **DataFrame**, which is pandas' version of a spreadsheet you can control with code.
- `df.shape` — tells you (rows, columns) — you got `(1516948, 17)`.
- `df.head()` — shows just the first 5 rows, so you can eyeball that it loaded correctly without printing all 1.5 million rows.

**Why:** This is step one of any project — you can't do anything until the data is actually loaded into memory.

---

## 3. Cleaning up column names and dropping the leftover index

```python
df.columns = [c.strip() for c in df.columns]
df = df.rename(columns={df.columns[0]: 'idx'})
df = df.drop(columns=['idx'])
```

**What it does, line by line:**
- `[c.strip() for c in df.columns]` — this is called a **list comprehension**: a compact way of saying "for every column name (`c`) in the list of column names, remove any accidental extra spaces (`.strip()`), and give me back the cleaned-up list." We then assign that cleaned list back as `df.columns`.
- `df.rename(columns={df.columns[0]: 'idx'})` — the very first column in the file had no real name (it showed up as `Unnamed: 0`). This renames it to `idx` just so we can refer to it by a real name.
- `df.drop(columns=['idx'])` — then we delete that column entirely, since it was just a leftover row-counter from how the original file was built (it counts 0, 10, 20... because the original device logged every second, but this file only kept every 10th reading).

**Why:** Stray spaces in column names cause silent bugs later (e.g., `"TP2"` and `"TP2 "` look the same to a human but are different to code). The index column carries no useful information, so we remove clutter early.

---

## 4. Fixing the timestamp and sorting

```python
df['timestamp'] = pd.to_datetime(df['timestamp'])
df = df.sort_values('timestamp').reset_index(drop=True)
```

**What it does:**
- `pd.to_datetime(...)` — converts the timestamp column from plain text (which pandas treats as just letters and numbers, with no understanding of "time") into a real date/time type pandas can actually calculate with — compare, sort, subtract, etc.
- `sort_values('timestamp')` — puts every row in correct chronological order, oldest to newest.
- `.reset_index(drop=True)` — after sorting, pandas' internal row-numbering gets scrambled; this just renumbers rows cleanly from 0 again. Purely cosmetic, doesn't touch your actual data.

**Why:** Almost everything downstream — labeling based on "time before a failure," splitting by date, rolling averages — depends on the data being in true time order and on timestamps being usable as real dates, not just text.

---

## 5. Quality checks

```python
print(df.isna().sum())

digital_cols = ['COMP', 'DV_eletric', 'Towers', 'MPG', 'LPS', 'Pressure_switch', 'Oil_level', 'Caudal_impulses']
for c in digital_cols:
    print(c, "unique values:", sorted(df[c].unique()))
```

**What it does:**
- `df.isna().sum()` — for every column, counts how many blank/missing values exist. You got all zeros — nothing missing anywhere.
- The loop goes through each of the 8 on/off signal columns and prints every distinct value found in that column. You confirmed each one only ever contains `0.0` or `1.0` — never anything stray like `0.5` or `2`.

**Why:** Before building anything on top of this data, you need to know it's trustworthy. This is the equivalent of double-checking your ingredients are fresh before cooking — catching problems now is far cheaper than discovering them after building the whole pipeline on top of bad data.

---

## 6. Building the failure labels

```python
FAILURES = [
    ("F1", pd.Timestamp("2020-04-18 00:00:00"), pd.Timestamp("2020-04-18 23:59:00")),
    ("F2", pd.Timestamp("2020-05-29 23:30:00"), pd.Timestamp("2020-05-30 06:00:00")),
    ("F3", pd.Timestamp("2020-06-05 10:00:00"), pd.Timestamp("2020-06-07 14:30:00")),
    ("F4", pd.Timestamp("2020-07-15 14:30:00"), pd.Timestamp("2020-07-15 19:00:00")),
]

df['in_failure_window'] = False
df['failure_id'] = None
for fid, start, end in FAILURES:
    mask = (df['timestamp'] >= start) & (df['timestamp'] <= end)
    df.loc[mask, 'in_failure_window'] = True
    df.loc[mask, 'failure_id'] = fid
```

**What it does:**
- `FAILURES` is just a list holding the 4 real, documented failure events — each one's ID, start time, and end time, taken directly from the dataset's own official report.
- The loop goes through each failure one at a time. For each one, `mask` becomes a **true/false filter**: "is this row's timestamp between the start and end of this failure?" Then `df.loc[mask, ...]` uses that filter to update only the matching rows — marking `in_failure_window = True` and recording which failure (`failure_id`) it belongs to.

**Why:** This turns the plain list of 4 failure events into something usable on every single row of your dataset — a permanent record of "was this exact moment part of a real failure, and if so, which one."

```python
failure_starts = [f[1] for f in FAILURES]
def time_to_next(ts):
    future = [s for s in failure_starts if s > ts]
    return (min(future) - ts).total_seconds() / 60.0 if future else np.nan

df['time_to_next_failure_min'] = df['timestamp'].apply(time_to_next)
```

**What it does:**
- `failure_starts` pulls out just the 4 start times from `FAILURES` (the `f[1]` grabs the second item in each tuple — the start time).
- `def time_to_next(ts):` defines a small custom function — given any single timestamp (`ts`), it finds every failure start time that's still ahead of it (`future`), and returns how many minutes away the *closest* one is. If there's no failure left ahead (we're past the last one), it returns `np.nan` — meaning "not a number," pandas' way of marking "no answer here."
- `.apply(time_to_next)` runs that function on every single row's timestamp, building a full column — essentially a countdown timer ticking down toward whichever failure is coming up next.

**Why:** This is the raw ingredient the actual warning label is built from — you can't say "is this within 30 minutes of a failure" without first calculating exactly how many minutes away the next failure is, for every row.

```python
df['horizon_label_30min'] = (
    (df['time_to_next_failure_min'] <= 30) &
    (df['time_to_next_failure_min'] > 0) &
    (~df['in_failure_window'])
).astype(int)
```

**What it does:** builds the actual target label using three conditions, all joined with `&` (meaning "and — all of these must be true"):
1. `time_to_next_failure_min <= 30` — the countdown timer shows 30 minutes or less
2. `time_to_next_failure_min > 0` — but the failure hasn't started yet (still in the future, not zero or negative)
3. `~df['in_failure_window']` — the `~` means "not" — so this excludes rows that are already inside a failure itself

`.astype(int)` converts the true/false result into `1`/`0`, which is the standard format machine learning models expect for labels.

**Why:** This is your actual prediction target — the exact question you're asking the model to learn to answer: "is this moment sitting in the danger zone right before a failure, and not already during one?"

---

## 7. Building the second, smarter warning label

```python
window = 720
roll_mean = df['Oil_temperature'].rolling(window=window, min_periods=100).mean().shift(1)
roll_std = df['Oil_temperature'].rolling(window=window, min_periods=100).std().shift(1)
df['oil_temp_zscore'] = (df['Oil_temperature'] - roll_mean) / roll_std
df['oil_temp_anomaly'] = (df['oil_temp_zscore'].abs() > 3).astype(int)
```

**What it does:**
- `window = 720` — since each reading is 10 seconds apart, 720 readings covers roughly 2 hours.
- `.rolling(window=720, ...)` — creates a "sliding lookback" over the last 720 readings at every point in the data, like a scope constantly looking backward over the past 2 hours as it moves forward through time.
- `.mean()` and `.std()` — calculate the average and how spread-out (standard deviation) the oil temperature was over that lookback window.
- `.shift(1)` — this is important: it shifts the calculation back by one row, so that when we compare *today's* reading, we're only using information from *before* this exact moment — never peeking at the future. This is what keeps the check honest.
- `oil_temp_zscore` — a standard statistics measure: how many "standard deviations" away from the recent average is the current reading. A z-score of 0 means "exactly average," a z-score of 5 means "way outside the normal range."
- `oil_temp_anomaly` — flags `1` if that z-score is bigger than 3 in either direction (very unusual), else `0`.

**Why:** Rather than a fixed rule like the 30-minute label, this teaches the code what "normal" oil temperature looks like *recently*, and flags moments that break from that pattern — a data-driven definition of "something's off," not a fixed clock rule.

```python
df['is_off'] = df['Motor_current'] < 0.1
df['stretch_id'] = (df['is_off'] != df['is_off'].shift()).cumsum()
```

**What it does:**
- `is_off` — marks every row `True` if the motor current is basically zero (meaning the compressor is resting, not working).
- `df['is_off'] != df['is_off'].shift()` — compares each row's `is_off` value to the *previous* row's value; this is `True` only at the exact moments the state flips (off→on or on→off).
- `.cumsum()` — a running total that increases by 1 every time that flip happens. The effect: every continuous "on" or "off" stretch gets its own unique ID number, like numbering each uninterrupted block of red or green on a timeline.

**Why:** We need to treat each rest period as one single event (not 500 separate 10-second readings) so we can measure "how long did this one resting stretch actually last."

```python
stretches = df.groupby('stretch_id').agg(
    is_off=('is_off', 'first'), start=('timestamp', 'min'), end=('timestamp', 'max')
)
stretches['duration_min'] = (stretches['end'] - stretches['start']).dt.total_seconds() / 60 + (10/60)
```

**What it does:**
- `.groupby('stretch_id')` — groups all rows that share the same stretch ID together, then `.agg(...)` collapses each group down into a single summary row, recording whether it was an "off" or "on" stretch, and its start and end time.
- `duration_min` — calculates how long each stretch lasted, in minutes (the small `+ 10/60` just accounts for the fact that the last reading itself also covers 10 more seconds).

**Why:** This turns your raw 1.5-million-row timeline into a much smaller table of "here are all the distinct resting/working stretches, and how long each one lasted" — the actual thing we want to check for abnormal behavior.

```python
off_stretches = stretches[stretches['is_off']].copy()
off_stretches['baseline_mean'] = off_stretches['duration_min'].rolling(20, min_periods=5).mean().shift(1)
off_stretches['baseline_std'] = off_stretches['duration_min'].rolling(20, min_periods=5).std().shift(1)
off_stretches['duration_zscore'] = (off_stretches['duration_min'] - off_stretches['baseline_mean']) / off_stretches['baseline_std']
off_stretches['duration_anomaly'] = (off_stretches['duration_zscore'] > 3).astype(int)
```

**What it does:** same z-score idea as the oil temperature check, but applied to rest-stretch *durations* instead of temperature readings — looking at the last 20 rest periods, calculating what's "normal" for their length, and flagging any single rest period that's abnormally long compared to that recent pattern. Again, `.shift(1)` keeps it honest — only ever comparing against past stretches, never future ones.

```python
anomaly_stretch_ids = set(off_stretches[off_stretches['duration_anomaly'] == 1].index)
df['off_duration_anomaly'] = df['stretch_id'].isin(anomaly_stretch_ids).astype(int)
```

**What it does:** collects the ID numbers of every rest stretch flagged as abnormal, then goes back to the *original* full-size dataset and marks every single row belonging to one of those flagged stretches as `1`.

**Why:** The anomaly was detected at the "whole stretch" level, but we need the flag to exist on every individual 10-second row again, so it lines up with everything else in your table.

```python
df['adaptive_statistical_label'] = (
    (df['oil_temp_anomaly'] == 1) | (df['off_duration_anomaly'] == 1)
).astype(int)
```

**What it does:** the `|` symbol means "or." This creates the final combined label — `1` if *either* the oil temperature check *or* the rest-duration check flagged something unusual, `0` if neither did.

**Why:** This is your second, "adaptive statistical" threshold — the one your proposal committed to alongside the fixed 30-minute rule. It caught real warning signs 21–44 hours before each of the 4 failures, much earlier than the fixed label alone ever could.

---

## 8. Splitting into training and testing

```python
split_boundary = pd.Timestamp("2020-07-14 00:00:00")
df['split'] = df['timestamp'].apply(lambda t: 'train' if t < split_boundary else 'test')
```

**What it does:**
- `split_boundary` — a single cutoff date we chose deliberately, positioned to cleanly separate F1/F2/F3 from F4.
- `.apply(lambda t: ...)` — a `lambda` is just a quick, throwaway function written in one line, without needing a full `def`. This one says: "for every timestamp, if it's before the boundary, label it 'train'; otherwise, label it 'test'."

**Why:** Since this is time-ordered data, splitting randomly would let the model sneak a peek at moments very close to what it's later tested on — like studying with the answer key mixed into your notes. Splitting by a clean date, with a failure entirely held out for test, is the honest way to check whether the model generalizes to something it's never seen.

---

## 9. Saving the final files

```python
train_df = df[df['split'] == 'train'].drop(columns=['split'])
test_df = df[df['split'] == 'test'].drop(columns=['split'])

train_df.to_csv(".../metropt3_train.csv", index=False)
test_df.to_csv(".../metropt3_test.csv", index=False)
```

**What it does:**
- `df[df['split'] == 'train']` — filters down to just the rows marked "train," and does the same for "test," creating two separate tables.
- `.drop(columns=['split'])` — removes the now-unnecessary `split` column from each, since once they're in separate files, you don't need a column telling you which is which anymore.
- `.to_csv(..., index=False)` — writes each table out as its own CSV file. `index=False` stops pandas from adding an extra, unwanted row-number column into the saved file.

**Why:** This is the actual handoff. Two clean, ready-to-use files — one for training the model, one held back purely for honest testing — sitting in the shared Drive folder, ready for Jigar to build on.

---

## The big picture, one paragraph

You took a raw sensor file, made sure it was clean and trustworthy, then built two different kinds of "warning" labels — one simple and clock-based (30 minutes before a failure), one smarter and pattern-based (catches things drifting from normal hours in advance) — using only information available *at the time*, never peeking into the future. Then you split the whole thing by date, holding one real failure completely back for honest testing, and saved the result as two clean files ready for the next person on your team to build on.

---
---

# Reference & Handoff Notes

*Everything below is reference material — for you to check details, and for whoever picks up the modeling and feature-engineering stages next.*

---

## A. What the two output files actually contain

Both `metropt3_train.csv` and `metropt3_test.csv` have the same columns. Here's every one, and where it came from:

### Original sensor readings (kept as-is from the raw file)

| Column | Meaning |
|---|---|
| `timestamp` | When the reading was taken (every 10 seconds) |
| `TP2`, `TP3`, `H1`, `DV_pressure`, `Reservoirs` | Five different pressure readings (bar) |
| `Oil_temperature` | Oil temperature (°C) |
| `Motor_current` | How much electricity the motor is drawing (amps) |
| `COMP`, `DV_eletric`, `Towers`, `MPG`, `LPS`, `Pressure_switch`, `Oil_level`, `Caudal_impulses` | Eight on/off state signals (each is 0 or 1) |

### Columns we built during ETL

| Column | Meaning | How to use it |
|---|---|---|
| `in_failure_window` | True/False — is this moment inside one of the 4 actual failures | Usually **excluded** from training features (it's "during" a failure, not "before" one) |
| `failure_id` | Which failure (F1–F4), blank otherwise | Useful for analysis and grouping, not a model input |
| `time_to_next_failure_min` | Countdown in minutes to the next failure | Mostly a building block; could be a target for a regression model |
| `horizon_label_30min` | **1 = within 30 min before a failure**, else 0 | **Main prediction target (fixed threshold)** |
| `oil_temp_zscore` | How far oil temp is from its recent normal | Diagnostic; can also be a feature |
| `oil_temp_anomaly` | 1 = oil temp statistically unusual | Component of the adaptive label |
| `off_duration_anomaly` | 1 = machine rested abnormally long | Component of the adaptive label |
| `adaptive_statistical_label` | **1 = either statistical signal fired**, else 0 | **Second prediction target (adaptive threshold)** |

---

## B. Key decisions we made, and why (so nobody re-litigates them)

**Why we split by date instead of shuffling randomly.**
This is time-series data. If you shuffle rows randomly into train and test, moments that are seconds apart (and nearly identical) end up on both sides — the model effectively sees the test answers during training, and scores look amazing but are fake. Splitting by a clean date prevents this. This exact mistake is what produced a previous team's suspiciously perfect scores, so we deliberately avoided it.

**Why we held out F4 specifically.**
F4 is the most recent failure and the cleanest single event. Keeping the first three (F1–F3) for training gives the model three real failures to learn from, while F4 stays completely untouched — a fair test of "can it recognize a failure it has never seen, later in time."

**Why we used a "look-back only" approach for the statistical labels (the `.shift(1)`).**
Every rolling calculation only ever uses data from *before* the current moment, never after. This matters enormously: if the statistical label accidentally used future data to decide "is now unusual," it would be cheating, and any model trained on it would fall apart in real use. The shift guarantees honesty.

**Why the raw sensor values were NOT scaled/normalized here.**
Different sensors are on very different number scales (some read single digits, others in the hundreds). Models usually need these brought onto a common scale — BUT that scaling must be calculated from the training data only and then applied to test, to avoid leakage. That's a modeling-stage step, deliberately left for Jackie, not done in ETL. (The previous team made the same choice, so this is consistent with precedent.)

**Why there are two labels, not one.**
The fixed 30-minute label is exactly what the client asked for and is easy to measure and explain, but it's extremely rare (hard to learn from) and only warns at the last minute. The adaptive statistical label catches earlier, broader warning signs (21–44 hours ahead) with far more examples. Together they form the "dual-threshold approach" the project proposal committed to. See Section D for how to actually use them in modeling.

---

## C. Things to be careful about (real caveats, not hidden)

**1. The labels are our construction, not ground truth from the data.**
The dataset itself is unlabeled. The 4 failure *times* are real and documented, but the 30-minute window and the statistical thresholds are choices we made. They're reasonable and defensible, but they're design decisions — if a reviewer asks "why 30 minutes, why 3 standard deviations," the honest answer is "client requirement" and "a standard statistical convention," not "the data told us so."

**2. Only 4 failures exist, all the same type (air leak).**
This is the single biggest limitation. The model will only ever learn one kind of failure signature, and with just one failure in the test set, results can swing a lot. Report performance with this caveat front and center — don't let strong numbers get read as "this generalizes to all pump failures."

**3. This is a compressor, not a pump.**
Model-development dataset, chosen as a defensible proxy (same sensor types, same pressure-decay/rising-current failure signature). Fine for building and validating the method — but any claim about real pump performance needs the eventual client (Zenodo) dataset, not this one.

**4. The 30-minute label is severely imbalanced (0.046% positive).**
Plain accuracy is meaningless here — a model that always says "normal" scores 99.95%. Evaluation must use precision, recall, F1, and PR-AUC, never raw accuracy. (This is already in the proposal, but it's worth repeating because it's the easiest way for results to look good while being useless.)

**5. One anomaly we noticed but did not fully resolve.**
Right before F1, the compressor sat idle for ~15 hours straight (far longer than its usual ~10-minute rest cycles). It's likely a genuine precursor, but we haven't confirmed *why*. Worth a closer look during modeling/analysis rather than assuming it's noise.

---

## D. Handoff notes for the modeling stage (Jackie) and features (Jigar)

**For Jigar (feature engineering, comes first):**
- Build sliding-window features (mean, std, min, max, trend) over the raw sensor columns, using the 60-minute and 120-minute window lengths from the proposal. At 10-second sampling that's 360 and 720 rows per window.
- **Critical:** compute windows *within* each split separately. Never build a window that straddles the train/test boundary, and never build windows across the whole dataset before splitting — that reintroduces the exact leakage the date-split was designed to prevent.
- The `split` information is preserved by the fact that train and test are already in separate files — just process each file independently.

**For Jackie (modeling):**
- **Two targets, two separate models.** Train one model to predict `horizon_label_30min` (precise, near-term) and, if useful, a second for `adaptive_statistical_label` (earlier, broader). They answer different questions and should be evaluated separately, each with its own precision/recall/F1 and its own "how many minutes early did it warn" measure. Don't merge them into one averaged score.
- **Resample the training data only.** Apply SMOTE (or similar) to the training split alone; keep the test split at its natural, realistic imbalance — this is an explicit client requirement. The two labels have very different rarity (0.046% vs 2.56%), so they'll likely need different resampling strength.
- **Idea worth considering (not a decided requirement):** the `adaptive_statistical_label` (or the raw `oil_temp_zscore` / anomaly flags) could be used as an *input feature* when predicting the 30-minute label — "is an early warning already active" is genuinely useful context for the near-term prediction. This is a design choice for you to test, not something ETL locked in.
- **Drop leakage-prone columns from the feature set.** `failure_id`, `in_failure_window`, and `time_to_next_failure_min` all directly encode the answer and must not be fed to the model as inputs — they're for labeling and analysis only.

**How the two alerts are meant to work together (context, for the dashboard/backend team too):**
- Think of it as two escalating tiers, not one signal: the statistical label is the early "elevated risk, keep watching" heads-up (hours out), and the 30-minute label is the "act now, switch to backup" trigger (minutes out).
- Exactly how these display and escalate is Chalaka's (backend) and Ishan's (dashboard) call — ETL just provides the two clean label columns that make it possible.
