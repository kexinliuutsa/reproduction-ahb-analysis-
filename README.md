## Reproduction: Passing the Turing Test on Screen — Agent Humanization Benchmark

This repository documents a reproduction of the core analysis pipeline from the paper **"Turing Test on Screen: A Benchmark for Mobile GUI Agent Humanization"**. The paper is currently an arXiv preprint version.

The paper introduces the Agent Humanization Benchmark (AHB) and frames GUI agent detection as an adversarial game between a detector and an agent. Then, this paper tests several humanization strategies (B-spline noise, history matching, fake action injection, long-press correction) to reduce the detectability of agent touch behavior relative to human behavior.

My reproduction is not a from-scratch reimplementation. It reuses the original authors' code and dataset, with the work here focused on getting the pipeline running, fixing a handful of bugs along the way, and reproducing the paper's main figures and tables.

Original code: [Gebro13/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark](https://github.com/Gebro13/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark)
Original dataset: [Hugging Face Dataset](https://huggingface.co/datasets/lyyang2766/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark)

## How to reproduce
My Laptop: macOS (Intel/x86_64)
### 1. Environment setup

Tested on macOS (Anaconda, Python 3.11). The commands below reflect what was actually needed to get the original repo running in this environment — if you're on Linux, the xgboost/OpenMP step likely isn't necessary.

```bash
git clone https://github.com/Gebro13/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark.git
cd Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark
pip install -r requirements.txt
pip install torch tqdm huggingface_hub
```

xgboost on macOS requires the OpenMP runtime, which isn't installed by default:

```bash
brew install libomp
pip uninstall xgboost -y
pip install xgboost==1.7.6
```

(The newest xgboost, 3.x, has a broken OpenMP link on macOS as of this writing — 1.7.6 is the version actually used in this reproduction.)

### 2. Download metadata

```bash
PWD_TEMP=$(pwd)
cd ..
git clone --depth 1 --branch main https://huggingface.co/datasets/lyyang2766/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark --filter=blob:none --sparse
cd Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark
git sparse-checkout set metadata
cp -r ./metadata/* $PWD_TEMP/
cd $PWD_TEMP
```

If `git clone` fails (like me) with a connection error (it did in this reproduction's network environment), use the Python download method in step 3 instead, and manually grab `swipe_data.pkl`, `tasks.csv`, and `Formated_Data_Renamed.xlsx` from the `metadata/` folder on the [Hugging Face dataset page](https://huggingface.co/datasets/lyyang2766/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark/tree/main) via the web UI instead.

### 3. Download gesture log data

The full dataset is tens of GB across many separate archives. This reproduction only downloaded human data and three agent categories (UI-TARS, GPT-4o, Claude-3.5-Sonnet) — see the Dataset section below for why.

If `git clone` doesn't work in your network environment, use `huggingface_hub` instead, which downloads over plain HTTPS:

```bash
pip install huggingface_hub
```

```python
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="lyyang2766/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark",
    repo_type="dataset",
    allow_patterns=[
        "gesture_recordings/ui_tars_no_humanity_gesture_recording/*",
        "gesture_recordings/mobile_agent_e_gpt_4o_no_humanity_gesture_recording/*",
        "gesture_recordings/mobile_agent_e_claude_sonnet_37_no_humanity_gesture_recording/*",
    ],
    local_dir="."
)
```

For human data, download these zip archives manually from the dataset page and extract them into a `logs/` folder at the project root (any subfolder name works — the code searches the whole project directory):

- `gesture_recording_o_2025_10_2026_01.zip`
- `gesture_recording_2025111x.zip`
- `logs_user123_1011_till_1015_gesture_recording.zip`
- `logs_user4567_gesture_recording.zip`

After downloading, verify completeness by comparing local file counts against the Hugging Face API's file listing — see [docs/reproduction_notes.md](docs/reproduction_notes.md) for the exact code used, since partial downloads can fail silently.

### 4. Apply the code fixes

A handful of bugs in the original codebase needed fixing before the notebook would run — outdated function/parameter names, a hardcoded search path, a missing function, a threshold-direction issue. The fixed versions are in this repository's `analysis/` folder; if you're working from a fresh clone of the original repo instead, see [docs/reproduction_notes.md](docs/reproduction_notes.md) for the exact diffs.

### 5. Run the notebook

```bash
jupyter notebook
```

Open `analysis_playground.ipynb` and run cells from the top.

## Dataset

Raw touch-event logs (.log files) per session, plus a metadata Excel file (Formated_Data_Renamed.xlsx) that acts as the benchmark's evaluation table — one row per task, with columns linking to session timestamps for 7 human users and several agents/humanization variants, plus per-session success outcomes.

This reproduction uses the complete human dataset and three of the five agent categories: UI-TARS, Mobile-Agent-E (GPT-4o), and Mobile-Agent-E (Claude-3.5-Sonnet). CPM-GUI-Agent and AutoGLM were not obtained — CPM-GUI-Agent in particular doesn't appear to exist in the public `gesture_recordings/` directory at all, based on inspecting the full file listing. This subset was enough to reproduce the paper's core claims (see below), though analyses that explicitly loop over all agent categories show artifacts from the missing ones — details in [docs/reproduction_notes.md](docs/reproduction_notes.md).

Downloading was its own challenge: the full dataset is tens of GB split across many separate archives, and `git clone` over HTTPS (for both the GitHub repo and the HF dataset) failed repeatedly with connection errors in this reproduction's network environment. Switching to the `huggingface_hub` Python library worked around this. Partial downloads can also fail silently, so file counts were cross-checked against `HfApi.list_repo_files()` before trusting any download as complete.

## What's reproduced

**Fully reproduced, matching the paper:**

- Figure 3a — action interval distribution by type
- Figure 3b — tap duration distribution (overlaid KDE)
- Figure 4 — per-feature AUC/accuracy decrease from RAW to best humanization method, across all 5 task clusters (reproduced via several chart styles: scatter, paired bars, slope charts, quiver arrows)
- Figure 8 — gesture length vs. duration scatter
- Figure 10 — success ratio by task cluster, including the paper's specifically cited statistic (Trip Planning success drops from 0.75 to 0.15 under Fake Rot Tap)
- Figure 11 — action interval distribution after fake-action injection
- Figure 12 — maxDev distribution, humans vs. non-humanized agents
- Table 1 — full structure: per-cluster, per-method (RAW / rotation-and-match / rotation-and-match-disturbance / b-spline-disturbance) feature AUC and SVM/XGBoost accuracy, landing in the same 0.92–1.00 range that the paper reports

**Additional findings along the way:**

- ML classifiers (SVM/XGBoost) resist humanization more than individual features do — single geometric features drop from ~1.0 to ~0.5 AUC under optimal humanization, but svm/xgb accuracy and TPR stay around 0.85–0.97 under the same conditions. This lines up with the paper's own forward-looking point (Section 7.2) that detection may eventually shift from single-feature kinematics to intent-level, multi-feature patterns.
- Removing UI-TARS from the agent pool changes which features rank as most discriminative — acceleration features (a20/a50) stay stable, but deviation features (dev80/dev50/dev20) drop out of the top 5, suggesting UI-TARS has an especially strong trajectory-deviation signature relative to GPT-4o/Claude-based agents.

**Not attempted:** sensor event analysis (needs data that wasn't downloaded), the mutual-information/Information-Gain calculation behind Table 6, Figure 9's feature correlation matrix, and an experimental "UltimateClassifier" referenced in the notebook that doesn't appear to map to anything in the published paper.

## On result reliability

Not everything matched the paper exactly, and rather than only showing the parts that did, the reasons for each mismatch are tracked in [docs/reproduction_notes.md](docs/reproduction_notes.md). Briefly:

- Some mismatches were genuine bugs hit while setting this up — outdated function names, a changed function signature, a missing function that had to be reimplemented, missing output directories — all fixed and logged.
- One mismatch was traced to an actual issue in the original codebase: a thresholding function used a comparison direction that produced near-inverted accuracy values for tap-duration detection. Reversing the direction (in a separate copy, without touching the original function) gave results consistent with the paper across all 5 clusters. The original function's unmodified output is kept as the primary result; the reversed version is documented as a diagnostic, not a "fix."
- Some mismatches came from comparing against the wrong scope — an early feature-ranking check mixed all 5 task clusters while comparing against the paper's single-cluster table, which was corrected once spotted.
- Some mismatches trace back to the missing CPM-GUI-Agent/AutoGLM data — most visibly, an average success-rate calculation that only had complete data for 9 of 116 task rows because it required all four columns to be non-empty simultaneously.

## Notebook

[analysis_playground.ipynb](analysis_playground.ipynb) is the full working session, kept as-is rather than cleaned up — including dead ends and a couple of duplicated cells — so the actual process is visible.

## Repository structure

``` text analysis_playground.ipynb    main reproduction notebook, with outputs
analysis/                    modified copy of the original authors' code
  lib/
  processing/
results0125/                 tap-duration and interval-based accuracy CSVs
results1202/                 feature extraction CSVs (full / Social Media cluster / no-UI-TARS variants)
results1220/                 final per-cluster, per-method AUC/learner accuracy table
docs/
  reproduction_notes.md      full log of issues hit and how they were resolved
requirements.txt
Formated_Data_Renamed.xlsx   benchmark evaluation metadata table
tasks.csv
README.md
```

Raw log data (logs/, gesture_recordings/) isn't included here due to size (tens of GB) — it's excluded via .gitignore. See the Dataset section above for how to obtain it.
