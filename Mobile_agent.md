## Real Mobile Agent Experiment — Setup Notes

### Goal

Test whether system prompt-level humanization can change a GUI
agent's behavioral features enough to evade the paper's detector —
without using the paper's post-processing wrapper approach.

This directly addresses the gap identified in Section 7.1.1:
"humanization should be intrinsic to the model architecture itself."

---

### Device

**Huawei Nova 6**, Android API level 31

**Huawei P40 Pro 6**, Android API level 31

**Key limitation discovered:** ADB-injected touch events
(`adb shell input tap`) do NOT appear in `getevent` output —
they travel a different kernel path than real finger touches.
Collecting real MotionEvent data from ADB-controlled agents
requires root access to write directly to `/dev/input/eventX`.

Huawei Nova 6 has no root access (locked bootloader post-2018).

**Next step:** Switch to a rootable device (iQOO confirmed
as candidate) to enable MotionEvent collection.

---

### ADB Environment Setup

```bash
adb devices                          # verify connection
adb install ADBKeyboard.apk          # Chinese text input
adb shell ime set com.android.adbkeyboard/.AdbIME
adb shell -t -t getevent -lt | grep ABS_MT_POSITION
# Touch device: /dev/input/event1
```

---

### Vision LLM Selection

| API | Vision | GUI quality | Decision |
|---|---|---|---|
| DeepSeek | ❌ text only | — | rejected |
| SiliconFlow Qwen3-VL-8B | ✅ | too small, unstable | rejected |
| OpenAI GPT-4o | ✅ | reliable | **selected** |

---

### System Prompt Experiment Design

Four ablation conditions targeting specific behavioral features:

| Condition | Prompt change | Target feature |
|---|---|---|
| Naive | none | baseline |
| A1: Scroll | insert scroll before acting | action interval |
| A2: Wait | forced 2s pause | action interval |
| A3: Elderly | elderly user persona | tap duration, speed |
| A4: Imprecise | offset tap ±5-15px | startX/Y, endX/Y |

截止到7.9 **Status:** Prompts designed and tested. Full experiment pending rootable device for MotionEvent collection.

---

### Pending

1. Switch to a rootable device (iQOO)
2. Collect real MotionEvent logs via `getevent`
3. Extract 24 features, evaluate with trained SVM/XGBoost
4. Record naive vs. humanized comparison video via scrcpy















