# WISENS-AI: Wi-Fi Sensing Intelligence Platform

**Non-intrusive detection of presence, movement and activity variations from Wi-Fi Channel State Information (CSI). The project covers the full chain: ESP32 firmware, data pipeline, machine learning and real-time inference.**

Summer internship (July–August 2026, academic year 2025–2026) at **Tukhnanutha**, as part of the engineering curriculum at **SUP'COM, University of Carthage**. Built by two interns on one shared technical chain.

> **Status: validated research prototype, not a production sensor.** Limitations are documented below with the same care as the results.

---

## Overview

Cameras capture identifiable images, and PIR sensors lose a person who sits still. WISENS-AI instead reads how the **Wi-Fi radio channel is perturbed by people**, with no camera, no microphone and no wearable.

An ESP32 microcontroller exposes the per-subcarrier CSI of each Wi-Fi frame it receives. The project turns this signal into an occupancy and activity classifier:

```
Wi-Fi frames -> ESP32 CSI capture -> UART (CSV) -> cleaning -> windowing
             -> 51 features -> classifier -> real-time prediction
```

The system recognizes **10 scenarios**: empty room, stationary presence, light motion, ample motion, the same three levels with two occupants, object disturbance, zone entry and zone exit.

---

## Key results

All results use a **session-grouped** protocol: no test session ever appears in training. Chance level for 10 classes is 0.10.

| Evaluation (Random Forest, Zone 2) | Accuracy | Macro F1 |
|---|---|---|
| Held-out sessions (10 unseen sessions) | **0.646** | **0.597** |
| Grouped cross-validation (every session tested once) | 0.602 | 0.553 |
| *Invalid* random window split (kept only to measure leakage) | 0.806 | 0.800 |

**Findings**

1. **The evaluation protocol mattered more than the algorithm.** Windows overlap by 50%, so a random split leaks near-duplicate windows into the test set. It inflated accuracy by about **16 points** and macro F1 by about **20 points**, more than the whole spread between the four algorithms tested.
2. **Empty-room detection is strong** (F1 ≈ 0.99), so binary occupancy detection is usable inside a calibrated room.
3. **A motionless person is rarely recognized** (recall ≈ 0.04); most of those windows are classified as light motion. The boundary between "still" and "slow movement" is gradual in the physics.
4. **RSSI acts as an environment shortcut.** The eight RSSI features add +0.04 to +0.06 macro F1 within a room but cost up to −0.13 across rooms.
5. **Cross-room transfer fails** (cross-zone macro F1 0.03–0.20 vs 0.41–0.59 within a zone), so per-site calibration is required.

**Models compared** (held-out macro F1): Random Forest 0.597, Gradient Boosting 0.585, SVM (RBF) 0.515, k-NN 0.487. Random Forest was retained for its accuracy, low training cost (2.5 s) and inference time (0.028 ms per window), and because it is interpretable.

---

## System architecture

### Embedded firmware (ESP-IDF, C)

| Module | Role |
|---|---|
| `wifi_manager` | Station-mode association, bounded reconnection, gateway discovery |
| `traffic_generator` | Pings the gateway every 200 ms, because CSI only exists when frames are received (gives a deterministic 5 Hz rate) |
| `acquisition_manager` | Configures the CSI driver; minimal IRAM callback (no logging, no blocking) |
| `marker_button` | GPIO button with debounce and an `EMPTY → TRANSITION → STABLE` state machine, to label short events in real time |
| `csv_exporter` | Streams one CSV line per measurement over UART |
| `main.c` | Startup sequence and fixed-length CSI filter (384 bytes) |

Build: PlatformIO + ESP-IDF 5.5.2 on a TTGO LoRa32 v2.1 board, zero warnings, about 10.6% of RAM and 18.2% of Flash used.

### Host pipeline (Python)

| Stage | Script | What it does |
|---|---|---|
| Capture | `collect_csi.py` | Reads the serial stream into one CSV per session with experiment metadata |
| Merge | `fusion_dataset.py` | Merges sessions and detects each file's CSV schema from the data rather than trusting headers |
| Clean | `clean_dataset.py` | Validates length, removes the 4 constant driver placeholder values, flags RSSI anomalies, computes labels |
| Features | `extract_features.py` | Per-session calibration, windowing, 51 features in 6 families |
| Train / evaluate | `train_model.py`, `compare_models.py` | Session-grouped protocol, class weighting, model comparison |
| Real time | `realtime_core.py`, `realtime_predict.py`, `replay_realtime.py` | Streaming inference sharing the training feature code |

### Signal processing in brief

- Each measurement is 384 bytes; removing 4 placeholder bytes leaves 380 interleaved I/Q values, i.e. **190 subcarriers**.
- Amplitude `A_k = sqrt(I_k² + Q_k²)` is used. Phase is deliberately not used, because it is corrupted by uncompensated frequency and timing offsets on the ESP32.
- Windows of **10 samples (2 s at 5 Hz), stride 5**, never crossing a session or label boundary.
- Per-session scaling divides by the session standard deviation of the mean amplitude. No statistic is ever computed across sessions or on the test set.

### The 51 features (six families)

| Family | Count | Captures |
|---|---|---|
| A: Global amplitude | 10 | Level and spread of pooled amplitudes |
| B: Per-subcarrier statistics | 14 | Which subcarriers vary, and by how much |
| C: Temporal dynamics | 11 | Speed and acceleration of channel variation |
| D: Peaks | 4 | Isolated transient events |
| E: Subcarrier diversity | 4 | Concentration vs spread of variation |
| F: RSSI | 8 | Received power and its dynamics |

---

## Dataset

| | Zone 1 | Zone 2 (official) |
|---|---|---|
| Sessions | 35 | 50 |
| Scenarios | 8 | 10 |
| Measurements | 41,522 | 64,774 |
| Analysis windows | 8,240 | 12,877 |

- Two rooms, 85 independent sessions, 106,296 measurements and 21,117 windows in total, all at a measured **5.0 Hz** on fixed Wi-Fi channel 6.
- Each stationary scenario: 5 sessions of 310 s. Each transition scenario: 5 short sessions (one entry or exit cycle each).
- Raw captures are never modified by the pipeline, and an automated test enforces this.

> The raw data is not included in this repository.

---

## Engineering quality

- **30 automated tests** (`pytest`) protect the invariants the results depend on: the serial schema matches the firmware, exactly 4 placeholders are removed, there is a single feature-schema definition, windows never cross session boundaries, the leakage detector fires on a constructed overlap, and the pipeline never writes into raw data folders.
- **Reproducibility campaign:** all 51 features of all 21,117 windows were regenerated from the raw captures and matched **bit for bit**.
- **Pre-experiment audit** found and fixed nine silent inconsistencies, including mixed zone paths, a ten-class model served to a five-class real-time script, duplicated feature lists, and Wi-Fi credentials committed to the repository (moved to an untracked header).
- **Hard safeguards:** the evaluation aborts if any session appears in both train and test.

---

## Source code

The firmware and data pipeline were developed during an internship and are kept private. This repository documents the system, the method and the results.

## Limitations

- Only **two rooms**, **five sessions per class**, one operator, one board, one antenna, one channel.
- Cross-environment generalization **fails**; per-site calibration is currently required.
- A motionless occupant is almost invisible to the classifier on 2 s windows.
- Entry and exit are the same physical event in opposite directions, so a window-level model confuses them.
- Real-time replay covers 2 of the 10 scenarios, and offline and streaming calibration differ slightly on 6 features.
- Final on-board hardware validation was not repeated during the reproducibility campaign (a successful build is not a hardware test).

## Future work

More sessions, operators and rooms (the learning curve is still rising); domain adaptation and environment-invariant normalization; sequence models (e.g. HMM) to separate entry from exit; a reject class with confidence thresholding; multiple receivers; on-device inference (the firmware uses only about 10.6% of RAM and 18.2% of Flash).

---

## Tech stack

**Embedded:** C, ESP-IDF 5.5.2, FreeRTOS, PlatformIO, ESP32 (TTGO LoRa32 v2.1)
**Data and ML:** Python, NumPy, pandas, scikit-learn, matplotlib, pytest
**Concepts:** Wi-Fi CSI, OFDM, multipath propagation, feature engineering, grouped cross-validation, data leakage, permutation importance, domain generalization

## Privacy and ethics

The system captures **no image, no audio and no identity**, only radio-channel measurements. It must only be used in authorized spaces with the knowledge of the people present.

## Documentation

The full internship report (LaTeX) describes the physics, firmware design, protocol, results and negative findings in detail.

## Author

**Zoubair Garma**, SUP'COM, University of Carthage. Host company: Tukhnanutha.

[LinkedIn](https://www.linkedin.com/in/zoubair-garma-96a635337) · [GitHub](https://github.com/ZoubairGarma)
