## Reproduction Notes

**Paper:** Turing Test on Screen: A Benchmark for Mobile GUI Agent Humanization  
**arXiv:** 2604.09574  


### Environment

- **OS:** macOS (Intel/x86_64)
- **Python:** 3.11 (Anaconda, Jupyter Notebook)
- **Key dependency fix:** `xgboost==1.7.6` + `brew install libomp`
  - The newest xgboost (3.x) has broken OpenMP linkage on macOS Intel
  - Neither `torch` nor `libomp` were documented in the original `requirements.txt`


### Dataset

#### Public release vs. paper version

- Public `Formated_Data_Renamed.xlsx` has 42 columns vs. the author's 33 — I thought the dataset was updated after notebook publication, adding user4–7 and AutoGLM
- Original notebook used only **3 human users** (user1–3); this reproduction uses all **7**
- Public `gesture_recordings/` contains only **RAW** recordings for UI-TARS, GPT-4o, Claude — humanized variants not released
- **CPM-GUI-Agent:** No gesture recording data was found in the public release (confirmed via `HfApi.list_repo_files()`), despite the paper's Table 4 reporting that CPM data was collected (2400 taps / 166 swipes). The reason for this omission is unclear.
- **AutoGLM:** RAW, rot+tap, and fake variants are all available in the public release. Substituted for CPM as the fourth non-humanized agent in this reproduction.


#### Dataset availability in public release

| Agent | RAW gesture recordings |
|---|---|
| UI-TARS | ✅ available |
| GPT-4o | ✅ available |
| Claude-Sonnet | ✅ available |
| AutoGLM | ✅ available (RAW + rot_tap + fake) |
| CPM-GUI-Agent | ❌ not in public release |

Humanization strategies (Online RM, Only Swipe, Fake Rot Tap) for UI-TARS/GPT-4o/Claude are generated on-the-fly by the analysis pipeline, not pre-recorded files. Their absence from the download does not prevent reproducing the corresponding analyses.

#### Data completeness

**Human gesture data:** Users 1–3 each have 26 log files in the public release (78 total). The author's notebook reports swipe counts of user1=724, user2=1157, user3=305 (2186 total), compared to user1=680, user2=892, user3=247 (1819 total) in this reproduction — the public release contains approximately 73% of the author's data for the same 3 users.

**Success rate data:** The paper's reported average success rates (0.68 RAW, 0.71 Online RM) cannot be reproduced from the public release. The success rate columns in the Excel file contain free-text annotations in mixed Chinese and English rather than structured numeric data. The structured success rate CSV referenced in `plot_spa_evaluation_results.py` was not included in the public release. Figure 10 (success ratio by cluster) was partially reproduced using the 9 rows that contain structured "n/m" ratio data in the `only swipe success rate` and `rot_tap_fake success rate` columns.

### Code Fixes

#### Fix 1 — File search path *(Critical)*

**Location:** Section 1.0.2 and all cells using `file_finder()`

```python
# Original
file_finder(Path("logs/"), "gesture_recording_" + stripped_timestamp + ".log")

# Fixed
file_finder(Path("."), "gesture_recording_" + stripped_timestamp + ".log")
```

Human data lives in `logs/`, agent data in `gesture_recordings/`. The hardcoded `logs/` root caused all agent files to return FileNotFoundError.


#### Fix 2 — Expanded human participant pool

**Location:** Section 1.0.3

```python
# Original (3 users)
humans = list(itemgetter(2, 3, 4)(columns_list))

# Fixed (all 7 users)
humans = list(itemgetter(2, 3, 4, 33, 34, 35, 36)(columns_list))
```
Figure 3b was correspondingly updated to display all 7 users


#### Fix 3 — CPM replaced with AutoGLM
Reason: CPM data is not in the public release
**Location:** Section 1.0.3

```python
# Original (includes CPM at index 19)
non_humanized_agents = list(itemgetter(12, 14, 16, 19)(columns_list))

# Fixed (AutoGLM at index 37)
non_humanized_agents = list(itemgetter(12, 14, 16, 37)(columns_list))
```


#### Fix 4 — Renamed import: cleanse_into_swipe

**Location:** Section 1.0.4

```python
# Original
from analysis. processing.extract_feature_of_swipes import cleanse_into_swipe

# Fixed
from analysis. processing.extract_feature_of_swipes import deprecated_cleanse_into_swipe as cleanse_into_swipe
```


#### Fix 5 — Missing parameter in raw_faker

**Location:** Section 1.0.4

```python
# Original
bot_line_fit(..., duration_us=500*1000, neighbor_time_delta_us=11000)

# Fixed
bot_line_fit(..., duration_us=500*1000, neighbor_time_delta_us=11000, end_upfinger_time_us=600*1000)
```

`end_upfinger_time_us` was added as a required parameter in the public release.


#### Fix 6 — Tap duration threshold direction *(Critical finding)*

**Location:** Section 1.3

`leave_taps` uses `pixel_length(gesture) <= 5` (pixel displacement), **not** FingerEvent count as the paper implies. After this filter, UI-TARS taps have a duration of ~1755–2928 µs (below 5000 µs); humans have ~84000 µs (above 5000 µs).

```python
# Original — inverted: classifies duration≥5000 as agent → raw_accuracy ≈ 0.002
y_pred = (raw_scores >= 5000).astype(int)

# Fixed — correct direction → raw_accuracy ≈ 0.997
y_pred = (raw_scores < 5000).astype(int)
```

The author's reported 0.997 was likely generated with a count-based `is_tap` (`len(gesture) < 5`), which would retain longer UI-TARS taps with duration >5000µs, making the original direction correct for their version. Both versions are preserved in the notebook.


#### Fix 7 — Parameter name change in session generator

**Location:** Sections 1.4.1, 1.4.2, 1.4.3, and subsequent cells

```python
# Original
filtering_and_modification_function=None

# Fixed
batched_filtering_and_modification_function=None
```


#### Fix 8 — ThresholdPosterior attribute access

**Location:** Sections 1.4.3, judge_library.py

```python
# Original
threshold_info['threshold']

# Fixed
threshold_info.threshold
```

`ThresholdPosterior` is a dataclass, not a dictionary.


#### Fix 9 — draw_motion_event2 not in public release

**Location:** Sections 1.14, 3.8, 3.10.2

```python
# Removed
from draw_motion_event2 import plot_gestures_2
```

Private module not included in the public release. `plot_gestures_2` is imported but never called in any of these cells.


#### Fix 10 — _gesture_start_end_us not in public release

**Location:** Section 1.6

Original notebook imports `_gesture_start_end_us` from `tap_duration_extract`, but this function does not exist in the public release of that module. Reimplemented locally using `FingerEvent.timestamp_us`.


#### Fix 11 — judge_library import path

**Location:** Section 1.15/3.9

```python
# Original
import judge_library

# Fixed
import analysis.processing.judge_library as judge_library
```


#### Fix 12 — result_csv_acc variable naming

**Location:** Section 1.14/3.8

Used `result_csv_acc` as a separate variable to preserve the AUC version in `result_csv`. Also replaced a read from the non-existent `final_acc_and_learner_results_0104.csv` with `result_csv_read = result_csv_acc`.



#### Fix 13 — Figure 3a: two plots overlapping

**Location:** Section 1.4.1

Original code draws two histplots without `plt.figure()` between them — the zoom-in plot overwrites the full-range plot. Added `plt.figure()` and `plt.show()` as separators. Paper Figure 3a corresponds to the first plot (ylim=100, xlim=120), which was invisible in the original notebook output.



### Key Observations

**Notebook code and paper figures do not fully correspond.** Running the published notebook does not reproduce the exact paper figures. Figure 3a (only zoom-in visible), Figure 4 (different color and y-axis), and Figure 11 (histogram vs KDE) all differ. The codebase was likely not finalized at the time of public release.

**`is_tap` uses pixel displacement, not FingerEvent count.** The paper states `|f| < 5` as the tap definition, which readers naturally interpret as FingerEvent count. The public codebase uses `pixel_length(gesture) <= 5` (pixel displacement). This is an undocumented change that significantly affects tap duration distributions and threshold-based accuracy results.

**Feature set changed between versions.** The public `build_features_dataframe` produces 24 features with standardized names (a20, maxDevSigned, dev20...) vs the author's 46+ features with different names (50%_vel_x, max_devs...). Feature rankings cannot be directly compared.

**ML classifiers resist humanization.** While individual geometric features drop from ~1.0 to ~0.5 AUC under optimal humanization, `svm_accuracy`/`xgb_accuracy` and `svm_tpr`/`xgb_tpr` remain at ~0.85–0.97. Supports paper Section 7.2's claim about intent-level detection.



### Sections Not Reproduced

**Figure 5 (Feature Selection Impact):** Requires ~1200 SVM/XGBoost runs (24 feature counts × 5 random samples × 10 cluster-method combinations). Computationally prohibitive.

**Section 1.1 (Sensor event counts):** Requires `sensor_recordings/` (10.8 GB, not downloaded). Corresponds to the paper Table 5 only.


### Warning Messages

**`Multitouch event found... Skipping this file`**  
Some log files in `logs_user4567_gesture_recording/` contain two-finger gestures. The codebase only supports single-touch and skips these files automatically. Does not affect core results.

**`Incomplete sample detected... Keeping the traces with complete properties`**  
Some log files have a trailing incomplete record. The parser discards only the incomplete record. Impact negligible (typically 1 record lost per file).

These warnings appear more frequently in this reproduction than in the author's notebook because user4–7 data contains more multitouch events.                                                                       
                                                                                                              
