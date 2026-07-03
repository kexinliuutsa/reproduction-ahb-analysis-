# Reproduction Notes
Turing Test on Screen: A Benchmark for Mobile GUI Agent Humanization
## Environment

- OS: macOS (Intel/x86_64)
- Python: 3.11 (Anaconda)
- Jupyter Notebook
- Key dependency fix: `xgboost==1.7.6` with `brew install libomp` (newest xgboost 3.x has broken OpenMP linkage on macOS Intel)


** Dataset Notes
*** Public release vs. paper version
The publicly released dataset on HuggingFace differs from the dataset used to generate the paper's figures in several important ways:
The public Formated_Data_Renamed.xlsx has 42 columns, whereas the author's notebook output shows 33 columns — the dataset was updated after the notebook was published, adding user4–7 and AutoGLM columns.
The public release contains data for all 7 human users, but the original notebook only used 3 (user1–3). This reproduction uses all 7.
The public gesture_recordings/ directory contains only RAW (non-humanized) recordings for UI-TARS, GPT-4o, and Claude-Sonnet. Humanized variants for these agents are not publicly released.
CPM-GUI-Agent: No data exists in the public release. Confirmed by inspecting the full file listing via HfApi.list_repo_files(). Paper Table 4 confirms CPM data was collected (2400 tap / 166 swipe), so this is a deliberate omission, not a download issue.
AutoGLM: RAW, rot+tap, and fake variants are publicly available and were used in this reproduction as a substitute for the missing CPM data.

*** Human participant demographics (confirmed from paper Table 4)

user1, user2, user3 — Young male
user4 — Young female
user5, user6 — Middle aged
user7 — Elderly

Agent coverage across humanization methods
UI-TARS, GPT-4o, Claude-Sonnet: RAW only (public release). Humanized variants not released.
AutoGLM: RAW ✅, rot+tap ✅, fake ✅
CPM-GUI-Agent: No data in public release.

User1–3 data is a subset of what the authors used
Local file counts: user1=26 files, user2=26 files, user3=26 files (78 total). The author's notebook shows higher swipe counts (user1=724, user2=1157, user3=305 = 2186 total) vs this reproduction (user1=680, user2=892, user3=247 = 1819 for the same 3 users). The public release contains approximately 73% of the author's data for user1–3.
Success rate data is only partially public

Online RM success rate: 35/116 rows available
RAW success rate: 36/116 rows available
Only Swipe success rate: 9/116 rows available
Fake Rot Tap success rate: 9/116 rows available

The paper's reported average success rates (0.68 RAW, 0.71 Online RM) were computed over all 116 tasks and cannot be reproduced from the public data. The 9 fully-complete rows yield: RAW=0.49, Online RM=0.54.

***Code Fixes
Fix 1: File search path (Critical)
Location: Section 1.0.2 and all cells using file_finder()
Original: file_finder(Path("logs/"), ...)
Fixed: file_finder(Path("."), ...)
Reason: The original hardcodes logs/ as the search root, but the dataset is split across logs/ (human data) and gesture_recordings/ (agent data). Without this fix, all agent gesture log files return FileNotFoundError.

Fix 2: Expanded human participant pool
Location: Section 1.0.3
Original: humans = list(itemgetter(2, 3, 4)(columns_list)) — 3 users only
Fixed: humans = list(itemgetter(2, 3, 4, 33, 34, 35, 36)(columns_list)) — all 7 users
Fix 3: CPM-GUI-Agent replaced with AutoGLM
Location: Section 1.0.3
Original: non_humanized_agents = list(itemgetter(12, 14, 16, 19)(columns_list)) — includes cpm_gui_agent raw
Fixed: non_humanized_agents = list(itemgetter(12, 14, 16, 37)(columns_list)) — replaces CPM with open_autoglm_agent_raw
Fix 4: Renamed import — cleanse_into_swipe
Location: Section 1.0.4
Original: from analysis.processing.extract_feature_of_swipes import cleanse_into_swipe
Fixed: from analysis.processing.extract_feature_of_swipes import deprecated_cleanse_into_swipe as cleanse_into_swipe
Reason: The function was renamed in the public release. Direct import raises ImportError.

Fix 5: Missing parameter in raw_faker
Location: Section 1.0.4, raw_faker function
Original: bot_line_fit(..., duration_us=500*1000, neighbor_time_delta_us=11000)
Fixed: bot_line_fit(..., duration_us=500*1000, neighbor_time_delta_us=11000, end_upfinger_time_us=600*1000)
Reason: The public version of bot_line_fit added end_upfinger_time_us as a required positional argument.
Fix 6: Figure 3b — expanded to all 7 human users
Location: Section 1.2
Original: generator_list = [humans_list[0], humans_list[1], humans_list[2], ...]
Fixed: generator_list = [*humans_list, ...]
Also fixed: Type label replacement changed to dict comprehension: {humans[i]: f"Human {i+1}" for i in range(len(humans))}

Fix 6: Figure 3b — expanded to all 7 human users
Location: Section 1.2
Original: generator_list = [humans_list[0], humans_list[1], humans_list[2], ...]
Fixed: generator_list = [*humans_list, ...]
Also fixed: Type label replacement changed to dict comprehension: {humans[i]: f"Human {i+1}" for i in range(len(humans))}

Fix 7: Tap duration threshold direction (Critical finding)
Location: Section 1.3
Root cause: leave_taps uses is_tap which checks pixel_length(gesture) <= 5 (pixel displacement ≤ 5px), NOT FingerEvent count as the paper states. After this filter, UI-TARS taps have duration ~1755–2928µs across all 5 clusters (well below the 5000µs threshold), while humans have ~84000µs.
Original code: y_pred = (raw_scores >= 5000) — classifies duration≥5000 as agent. Since agents have SHORT duration and humans have LONG duration, this is inverted → raw_accuracy ≈ 0.002.
Fixed version: y_pred = (raw_scores < 5000) → raw_accuracy ≈ 0.997, consistent with paper's reported values.
Author's 0.997 was likely generated with a count-based is_tap (len(gesture) < 5) rather than the pixel-displacement-based version in the public release, making the original threshold direction correct for their version.
Both versions preserved in notebook. The _fixed version is correct for the public codebase.

Fix 8: Parameter name change in session generator
Location: Sections 1.4.1, 1.4.2, 1.4.3, and multiple subsequent cells
Original: filtering_and_modification_function=None
Fixed: batched_filtering_and_modification_function=None
Reason: The public version renamed this parameter in ranged_batched_modified_generator_with_session_timestamp.

Fix 9: ThresholdPosterior attribute access
Location: Sections 1.4.3, judge_library.py
Original: threshold_info['threshold'] / online_humanity_classifier["threshold"] *= 1e6
Fixed: threshold_info.threshold / online_humanity_classifier.threshold *= 1e6
Reason: ThresholdPosterior is a dataclass object, not a dictionary.

Fix 10: draw_motion_event2 module not in public release
Location: Sections 1.14, 3.8, 3.10.2
Original: from draw_motion_event2 import plot_gestures_2
Fixed: Import line removed.
Reason: Private module not included in the public release. plot_gestures_2 is imported but never actually called in any of these cells.

Fix 11: _gesture_start_end_us reimplemented
Location: Section 1.6
Original: from analysis.processing.tap_duration_extract import _gesture_start_end_us
Fixed — reimplemented locally:
def _gesture_start_end_us(gesture):
    start_us = gesture[0].timestamp_us
    end_us = gesture[-1].timestamp_us
    return start_us, end_us
Also fixed: Duration converted to seconds (÷1e6), xlim expanded to 50, ylim to 1.0 to match paper's Figure 8.

Fix 12: judge_library import path
Location: Section 1.15/3.9
Original: import judge_library
Fixed: import analysis.processing.judge_library as judge_library
Reason: judge_library.py exists at analysis/processing/judge_library.py but is not on the Python path as a top-level module.

Fix 13: result_csv_acc variable naming
Location: Section 1.14/3.8
Fixed: Use result_csv_acc as separate variable to preserve AUC version in result_csv. Also replaced read from non-existent final_acc_and_learner_results_0104.csv with result_csv_read = result_csv_acc.

Fix 14: make_feature_table_but_mutual_information_binary return value
Location: Section 3.10.1
Original: unhumanized_result_csv, _, _ = make_feature_table_but_mutual_information_binary(...) — attempts to unpack 3 values
Fixed: unhumanized_result_csv = make_feature_table_but_mutual_information_binary(...) — function returns single DataFrame

Fix 15: Figure 3a — two plots overlapping
Location: Section 1.4.1
The original code draws two histplots sequentially without plt.figure() between them, causing the zoom-in plot to overwrite the full-range plot. Added plt.figure() and plt.show() between the two plots. The paper's Figure 3a corresponds to the first plot (ylim=100, xlim=120), which was not visible in the original notebook output.

Fix 16: Figure 4 visualization updated to match paper
Location: Section 3.5/3.8

Color scheme changed to green (#c8e6c9 light / #2e7d32 dark)
Switched from grouped barplot to overlaid barplot
ylim set to (0.5, 1.0)
ylabel changed from "AUC" to "ACC" (paper Figure 4 uses ACC)

Fix 17: Figure 11 visualization updated to match paper
Location: Section 1.4.2
Original: sns.histplot
Fixed: sns.kdeplot with fill=True, alpha=0.3, bw_adjust=0.5, clip=(0, 8)
Reason: Paper's Figure 11 shows smooth KDE curves, not histogram bars.
                                                                                                              
** Key Observations
** Notebook code and paper figures do not fully correspond
Running the published notebook does not reproduce the exact paper figures. Notable discrepancies: Figure 3a (only zoom-in visible), Figure 4 (different color scheme and y-axis), Figure 11 (histogram vs. KDE). The codebase was likely not finalized at the time of public release.

*** is_tap uses pixel displacement, not FingerEvent count
The paper states |f| < 5 as the tap definition, implying FingerEvent count. However, is_tap in the public codebase uses pixel_length(gesture) <= 5 (pixel displacement ≤ 5px). This is a significant undocumented change that affects tap duration distributions and threshold-based accuracy calculations.

*** Feature set changed between author's version and public release
Section 1.9 comparison: the public build_features_dataframe produces 24 features (a20, maxDevSigned, dev20...) vs the author's version which had 46+ features with different names (50%_vel_x, max_devs...). Feature rankings cannot be directly compared.

*** ML classifiers resist humanization
While individual geometric features drop from ~1.0 to ~0.5 AUC under optimal humanization, svm_accuracy/xgb_accuracy and svm_tpr/xgb_tpr remain at ~0.85–0.97 under the same conditions. Supports the paper's Section 7.2 claim about intent-level detection.

*** Non-deterministic results in Section 1.3
execute_tap_duration_of_participants_per_index uses random.normalvariate without fixed seed. Added random.seed(42) for reproducibility.

*** make_feature_and_learner_table duplicated in notebook
Appears in both Sections 1.11 and 3.5. Copy-paste artifact from iterative development. Second run overwrites result_csv with identical results.

** Sections Not Reproduced
Figure 5 (Feature Selection Impact): Requires ~1200 SVM/XGBoost runs. Function plot_acc_increase_as_more_feature_used exists in codebase but computationally prohibitive within this reproduction's time constraints.
Section 1.1 (Sensor event counts): Requires sensor_recordings/ data (10.8 GB, not downloaded). Corresponds to paper Table 5 only — no analytical figures.
Section 1.15 / 3.9 (UltimateClassifier): Exploratory code combining swipe/tap/interval classifiers. No direct correspondence to any paper figure or table. Not included in the paper.

** Warning Messages Explanation
Multitouch event found... Skipping this file
Some log files in logs_user4567_gesture_recording/ contain two-finger gesture events. The codebase only supports single-touch analysis and skips these files automatically. Does not affect core results.
Incomplete sample detected... Keeping the traces with complete properties
Some log files contain a trailing incomplete gesture record. The parser discards only the incomplete record. Impact negligible (typically 1 record lost per file).
These warnings appear more frequently in this reproduction than in the author's notebook because this reproduction uses 7 human users (including user4–7 whose data contains more multitouch events).                                                                                                              
                                                                                                              
                                                                                                              
                                                                                                              
                                                                                                              
