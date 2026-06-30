#reproduction-ahb-analysis-

# Reproduction: Passing the Turing Test on Screen — Agent Humanization Benchmark

## 1. Overview

This repository documents my reproduction of the analysis pipeline from the paper:

**"Turing Test on Screen: A Benchmark for Mobile GUI Agent Humanization"**
(Zhu et al., 2026)

The paper introduces the Agent Humanization Benchmark (AHB), which evaluates whether GUI agents' touch behavior (swipes, taps, timing) can be distinguished from human behavior, and proposes humanization strategies to reduce this gap.

This is **not** a from-scratch reimplementation. I reused the authors' official codebase and dataset, and focused on:
- Setting up the environment and dependencies
- Resolving data path / package compatibility issues to get the original analysis notebook running
- Reproducing select figures and statistics from the paper (touch dynamics distributions, detector accuracy, etc.)

Original code: [Gebro13/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark](https://github.com/Gebro13/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark)
Original dataset: [HuggingFace Dataset](https://huggingface.co/datasets/lyyang2766/Passing-the-Turing-Test-on-Screen-Agent-Humanization-Benchmark)

## 2. Environment Setup

- OS: macOS
- Python: 3.11 (via Anaconda)
- Key dependencies: `numpy`, `pandas`, `matplotlib`, `scipy`, `scikit-learn`, `seaborn`, `xgboost==1.7.6`, `torch`
- Note: `xgboost` on macOS requires the OpenMP runtime (`brew install libomp`) to load correctly; this was not listed in the original `requirements.txt`.

## 3. Dataset

The dataset consists of raw touch-event logs (`.log` files) collected from human users and GUI agents (UI-TARS, Mobile-Agent-E with GPT-4o / Claude-3.5-Sonnet, CPM-GUI-Agent, AutoGLM), plus a metadata Excel file (`Formated_Data_Renamed.xlsx`) that serves as the benchmark evaluation table.

Each row in the metadata table corresponds to one task, recording:
- Task identity (app, task description)
- Session timestamps for multiple human users and multiple agents/humanization variants
- Per-session success outcome (which human/agent succeeded at which task)

This structure allows every recorded session to be traced back to its task, participant, and success outcome.

### Download challenges

- Full dataset spans tens of GB, split across many separate zip archives rather than a single bundle
- `git clone` over HTTPS repeatedly failed due to network connectivity issues
- Resolved by using the `huggingface_hub` Python library (`snapshot_download` with `allow_patterns`) to selectively download only the needed subfolders
- Required manual verification (comparing local file counts against the remote file listing via `HfApi`) since partial downloads did not raise errors

## 4. Reproduction Status

✅ Environment and dependencies set up
✅ Data downloaded and reorganized into the expected `logs/` directory structure
✅ Fixed a hardcoded path bug in `gesture_log_reader_utils.py` (file search scope)
✅ Successfully extracted touch-event traces (swipes, taps) from raw logs for human users and UI-TARS / GPT-4o / Claude agents
🔄 In progress: reproducing tap duration / swipe distribution plots (Figure 3 in the paper)
⬜ Not yet attempted: sensor event analysis (requires `sensor_recordings/`, not downloaded)
⬜ Not yet attempted: full detector training (SVM / XGBoost) across all agent categories

## 5. Notes / Issues Encountered

See [docs/reproduction_notes.md](docs/reproduction_notes.md) for a detailed log of issues encountered and how they were resolved (path bugs, missing dependencies, partial data downloads, etc.)
