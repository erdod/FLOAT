# FLOAT - Evaluation Document (Final)

## 1. Requirements Overview

| # | Requirement | Metric | Target | Result |
|---|---|---|---|---|
| R1 | Motor anomaly cutoff (stall / dry-run) | Reaction time | < 2000 ms |  ≈1.25 s |
| R2 | Environmental warning (temperature / turbidity out of band) | Values out of range | warn on a sustained out-of-band excursion |  warning on every induced excursion; advisory (pump keeps running) |
| R3 | Detection performance (accuracy & false alarms) | Labelled confusion matrix | accuracy > 90 % & macro FPR/FNR < 5 % |  95.3 % accuracy, 2.4 % FPR, 4.8% FNR (43 cycles) |
| R4 | Connectionless safety communication | No router / Internet on the safety path | Pump stops with no WiFi infrastructure |  Verified |

## 2. Evaluation Methodology - Labelled Confusion Matrix

Anomaly detection is a **classification** problem, so it is evaluated with a confusion matrix rather than with anecdotal pass/fail observations. The Observer carries a built-in harness so the evaluation runs on the real hardware, on the real current signal, with no separate test rig.

### 2.1 Classes

The confusion matrix scores only the classes that lead to a **HALT** - the safety-critical decisions the system actually acts on:

| Index | Class |
|---|---|
| 0 | NORMAL |
| 1 | MOTOR_STALL |
| 2 | DRY_RUN |

The advisory conditions (`VOLTAGE_DROP`, `TEMP_TOO_HIGH`, `TEMP_TOO_LOW`, `TURBIDITY_LOW`) are still **detected and warned** on the live system, but they are deliberately **excluded from the matrix**: they never stop the pump, so scoring them against the halt decision would conflate two different jobs. They are validated separately (chapter 4). This keeps the matrix a clean measure of the one question that matters for pump safety: *given a cycle, does the system correctly decide halt / no-halt and for the right reason?*

### 2.2 How the harness works

- The operator sets the **ground-truth** class for the upcoming cycle via `GET /eval?truth=<k>` (or from the developer dashboard panel, reachable with `?dev=1`).
- During the cycle, the operator physically induces the corresponding condition.
- At the end of each monitored cycle the Observer records exactly **one** outcome into `confmat[truth][detected]`. A clean monitored cycle is recorded as `detected = NORMAL`; a confirmed **halt-class** anomaly (stall / dry-run) is recorded as that class. Warning-only anomalies (temperature, voltage, turbidity) do **not** write to the matrix, so an advisory event during a labelled cycle never pollutes the result. A flag (`eval_cycle_recorded`) guarantees one outcome per cycle, so a late second anomaly never double-counts.
- `GET /eval` returns the full 3×3 matrix and the cycle count as JSON; `?reset=1` clears it and `?off=1` stops labelling.

The unit of evaluation is the **pump cycle**, which is the real unit of decision in the system (one HALT decision per cycle).

### 2.3 Metrics

For a 3x3 Confusion Matrix with classes $k$, total cycles $N$, and $TP_k, FN_k, FP_k, TN_k$ representing True Positives, False Negatives, False Positives, and True Negatives for class $k$:

$$Accuracy = \frac{\sum TP_k}{N}$$

$$FNR_k = \frac{FN_k}{TP_k + FN_k}$$
$$FPR_k = \frac{FP_k}{FP_k + TN_k}$$

$$Macro\ FNR = \frac{\sum FNR_k}{3}$$
$$Macro\ FPR = \frac{\sum FPR_k}{3}$$

## 3. R1 - Motor Anomaly Cutoff < 2000 ms

When the pump stalls or runs dry, the system must cut power to the pump within **2000 ms** of the onset.

### Why this target
A brushed DC pump under stall draws several times its rated current, and the windings overheat in the order of seconds to tens of seconds depending on thermal mass. A 2-second cutoff keeps the reaction well inside that window while remaining comfortably achievable given the sampling and communication budget.

### Reaction time analysis

| Stage | Duration |
|---|---|
| Sample period | 400 ms |
| Confirmation window (3 × 400) | 1200 ms |
| HALT burst | 50 ms |
| Target ISR + GPIO write | < 1 ms |
| **Analytical reaction time** | **≈ 1251 ms** |

## 4. R2 - Environmental Warning

If the water temperature leaves the safe band and the reading stays out of band across the confirmation gate (**3 consecutive monitoring samples, ≈1.2 s**), the system raises a **warning**. The pump is **not** stopped. Low turbidity is the twin environmental advisory and is gated the same way. 

### Why advisory, not a halt
A temperature excursion develops slowly and is rarely caused by the pump. Cutting circulation would remove oxygenation and make the situation worse, so the correct response is to alert the owner while keeping the water moving. 

### Adaptive temperature band
The safe band is **learned per tank** during calibration:
```
band = [ μ_T − Δ , μ_T + Δ ],  Δ = max(5σ_T, 1.5 °C),  clamped to [16, 32] °C
```
so a tropical tank calibrated at 26 °C and a cold-water tank at 20 °C get appropriate limits automatically, with no manual configuration.

### Turbidity threshold
Before evaluating the threshold, the Arduino Uno converts the raw analog sensor reading into **Nephelometric Turbidity Units (NTU)** by linearly mapping the ADC value between a clean-water calibration point and a maximum dirtiness cap.

TURB_CLEAN_ADC = 750      
TURB_DIRTY_ADC = 10       
TURB_MAX_NTU   = 800.0

Unlike the temperature band, turbidity uses a **fixed threshold** rather than an adaptive baseline. 
The Observer node triggers an environmental advisory when: Turbidity < 100.0 NTU

## 5. R3 - Detection Performance (Accuracy > 90 %, macro FPR/FNR < 5%)

Over labelled monitored cycles the detector must reach **> 90 % accuracy** while keeping the **macro FPR/FNR < 5%** - it must both classify cycles correctly and, above all, never halt a healthy pump.

### Why it matters
A false halt disrupts circulation and alarms the user for no reason; repeated false alerts erode trust and push the operator to disable the safety features - defeating the purpose of the system. 

### Results
Across the 43 labelled cycles the detector reached **95.3 % accuracy** with a **2.4 % FPR, 4.8% FNR**.

## 6. R4 - Connectionless Safety Communication

The safety-critical Target↔Observer link must work without a WiFi router or Internet.

### Why ESP-NOW
ESP-NOW is a connectionless Layer-2 protocol: it exchanges raw 802.11 frames between two MAC addresses with no DHCP, router or IP stack. The dashboard and cloud are informational only; the safety loop is fully independent of them.

| Protocol | Router required | Association latency | TX current peak |
|---|---|---|---|
| MQTT (WiFi) | Yes | 200–500 ms | ~180 mA |
| HTTP REST | Yes | 300–600 ms | ~180 mA |
| BLE GATT | No | 20–50 ms | ~20 mA |
| **ESP-NOW** | **No** | **< 5 ms** | **~80 mA** |

### Results
The safety loop operates purely at the MAC level; the pump stops on a stall/dry-run with no router and no Internet present. The frames are additionally AES-CCM encrypted.

## 7. Confusion-Matrix Results

| truth ＼ detected | NORMAL | STALL | DRY | 
|---|:---:|:---:|:---:|
| **NORMAL** | **15** | 0 | 0 |
| **MOTOR_STALL** | 1 | **14** | 0 | 
| **DRY_RUN** | 1 | 0 | **12** | 

**Aggregate metrics** (43 labelled cycles: 15 NORMAL, 15 MOTOR_STALL, 13 DRY_RUN):

| Metric | Value | Meaning |
|---|---|---|
| Accuracy | 95.3 % (41/43) | cycles classified into the exact correct class |
| False-positive rate | **0 %** (0/15) | NORMAL cycles never flagged as a halt |
| False-negative rate | 7.1 % (2/28) | fault cycles reported as NORMAL |
| Macro FPR | 2.4 % | per-class one-vs-rest FPR, averaged over the three classes |
| Macro FNR | 4.8 % | per-class one-vs-rest FNR, averaged over the three classes |

### Why the two misses happen
Both errors are the same kind: a fault-labelled cycle

