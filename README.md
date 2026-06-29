#reproduction-ahb-analysis-

##1. Paper Summary
We reproduce the Agent Humanization Benchmark, focusing on behavioral feature extraction and classification.
Paper title: TURING TEST ON SCREEN: A BENCHMARK FOR MOBILE GUI AGENT HUMANIZATION


## 2. Implementation

We reimplemented the pipeline into modular components:

- Data loader
- Gesture/sensor parser
- Feature extraction module
- Classification model

---

## 3. Experimental Setup

- Model: Random Forest
- Features: swipe kinematics, tap duration, interval statistics
- Train/Test split: 70/30

---

## 4. Results

AUC achieved: XX

Compared to paper: slightly lower / comparable

---

## 5. Key Challenges

- The original notebook was not modularized
- Feature extraction required refactoring
- Large logs caused performance bottlenecks

---

## 6. Conclusion
