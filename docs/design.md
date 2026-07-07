# FLOAT - Design Document 

## 1. Hardware Components

| Component | Node | Role |
|---|---|---|
| ESP32-S3 (Heltec WiFi LoRa 32 V3) | Target | Sensing/actuation; ESP-NOW; deep sleep |
| ESP32-S3 (Heltec WiFi LoRa 32 V3) | Observer | Detection; WiFi STA; dashboard + cloud gateway; INA219 host |
| Arduino Uno | Turbidity | Hosts the analog turbidity sensor; serves readings over UART |
| INA219 #1 (I2C) | Observer | Pump supply current |
| INA219 #2 (I2C, separate bus) | Observer | Arduino Uno supply current |
| DS18B20 waterproof probe (1-Wire) | Target | Water temperature |
| Analog turbidity sensor | Arduino Uno | Water clarity (raw ADC → NTU) |
| DC pump (3-5 V) | Target | Filtration / circulation |
| SG90 micro-servo | Target | Food-dispenser gate |
| Active buzzer | Observer | Local audible alarm |
| LiPo 3.7 V | Target | Power |
| Regulated 5 V source | Arduino Uno | Power|


## 2. Network Architecture

The Target↔Observer link uses **ESP-NOW** (IEEE 802.11 connectionless frames between two MAC addresses): no router, no DHCP, no IP. Both nodes are pinned to the **same radio channel**: the Observer joins the local WiFi as a station and reads back its channel; the Target scans for the same SSID and locks ESP-NOW to that channel (fallback to channel 13 if the network is not found). ESP-NOW frames are **AES-CCM encrypted**.

The Observer simultaneously acts as the gateway to the outside world over its WiFi station interface: it serves the dashboard on the local network, bridges telemetry/alerts to an MQTT cloud broker over TLS, and sends e-mail notifications.

```
  Arduino Uno --UART--► Target (ESP32-S3) --ESP-NOW (AES-CCM encrypted, --------► Observer (ESP32-S3)
   • turbidity          • temp, pump, servo          ch = WiFi ch)                • detection + CUSUM
                                                                                  • 2× INA219
                                                                                  • WiFi STA gateway
                                                                                          │
                            ┌------------------------------┌------------------------------┤
                            ▼                              ▼                              ▼
                   Local dashboard (LAN)        MQTT broker (HiveMQ, TLS)        E-mail (Apps Script)
                                                float/aq1/{telemetry,state,      HTTPS webhook on alert
                                                alert,cmd}
```

## 3. Communication Protocols

### 3.1 Target ↔ Observer (ESP-NOW)

| Direction | Message | Meaning |
|---|---|---|
| T → O | `LOG:<text>` | Human-readable status line |
| T → O | `DATA:SENSOR:<ntu>,<temp_c>` | Turbidity + temperature |
| T → O | `HB:<pump><halt>` | Idle heartbeat (used for OFF→ON reconciliation) |
| T → O | `CMD:<name>\|ID:<n>` | Reliable command; Observer replies `ACK:<n>` |
| O → T | `CMD:START_LEARN` / `START_MONITOR` / `STOP_MEASURE` | Drive the detection phase per cycle |
| O → T | `HALT` | Emergency pump stop (sent ×10 in a burst against RF loss) |
| O → T | `CMD:CLEAR_HALT`, `CMD:AUTO_ON_RESET`, `CMD:AUTO_OFF`, `CMD:PUMP_*`, `CMD:FEED*`, `CMD:CALIBRATE` | Mode / manual control |
| O → T | `CAL:<μ,σ,th_stall,th_dry,th_hi,th_lo>` | Calibration result (logged on the Target) |
| O → T | `ACK:<n>` | Handshake reply |

### 3.2 Target ↔ Arduino Uno (UART)

A request/echo handshake on `Serial1`: the Target sends `'T'`, the Uno replies with the raw ADC value terminated by newline, and the Target confirms with `'E'<value>`. The Uno sleeps on its watchdog (~4 s) and listens briefly on each wake; the Target repeats `'T'` long enough to always catch a wake window. On timeout the Target falls back to a safe default reading. The acquisition runs **before** the ESP32 radio is enabled, to keep RF noise off the analog line.

### 3.3 Dashboard → Observer (HTTP, polling)

| Endpoint | Action |
|---|---|
| `GET /` | Serve the dashboard |
| `GET /data` | Live JSON (current, EWMA, voltage, temperature, turbidity, mode, thresholds, auxiliary supply) |
| `GET /events?last=<id>` | Incremental event feed |
| `GET /mode?set=on\|off` | Switch autonomous / manual mode |
| `GET /cmd?a=<action>&sec=<n>` | Manual control (`clear_halt` always; pump/feed/calibrate only in OFF mode) |
| `GET /eval?...` | Evaluation harness (set ground-truth class, reset, read confusion matrix) |

### 3.4 Observer ↔ Cloud (MQTT over TLS, base `float/aq1`)

| Topic | Direction | Purpose |
|---|---|---|
| `…/telemetry` | publish (~5 s) | Live current/EWMA/voltage/temperature/turbidity |
| `…/state` | publish, **retained** | Device shadow: mode, auto, halted, calibrated, μ, thresholds, drift |
| `…/alert` | publish on event | Anomaly reason + severity + context |
| `…/cmd` | subscribe | Remote commands (mode, clear-halt, and OFF-mode manual actions) |

## 4. Software Architecture

### 4.1 Target Node

```
            boot (RTC + NVS state restored)
                          │
                 reset-storm filter 
                          │
      read DS18B20 temperature and read turbidity 
                          │
               init encrypted ESP-NOW
                          │
                      send data
                          │
        ┌-----------------┼-------------------------------┐
        │ system halted?  │ default-mode OFF?             │ else: AUTO CYCLE
        ▼                 ▼                               ▼
  stay awake,       stay awake, run loop()        set NVS safety latch
  show HALTED       as a manual bench             (halted=true, pumping=true)
  on dashboard                                            │
                                        ┌-----------------┼-------------------┐
                                        │ bootCount == 0? │ turbidity ≤ 100?  │ else
                                        ▼                 ▼                   ▼
                                  LEARN (pump 10 s,   MONITOR (pump 10 s,  feed (servo) +
                                  START_LEARN)        START_MONITOR)       short pump 5 s
                                        └-----------------┼-------------------┘
                                                          │
                                                   deep sleep 20 s
```

### 4.2 Observer Node

```
                 loop()
                   │
              read INA219 #1 
              read INA219 #2 
                   │
   ┌---------------┼-----------------------┐
   │system locked? │ mode == LEARNING?     │ mode == MONITORING & calibrated?
   ▼               ▼                       ▼
 buzz + HALT   collect samples      grace period:
                                    STALL  : EWMA > μ + 3σ
                                    DRY_RUN: EWMA < μ - 3σ 
                                    VOLT   : V < max(3.30, 0.90 V_cal)
                                    TEMP   : T outside learned band
                                    TURB   : turbidity < 100 NTU
                                           │
                                           │ confirm 3× consecutive
                                           ▼
                               classify by priority → act
                                           │
                                           ▼
                                      else → IDLE
```

### 4.3 Operating Modes

- **Autonomous (ON):** the Target runs the sense → sleep → act cycle on its own.
- **Manual (OFF):** the Target stays awake; the operator can run the pump (timed), dispense food (once or on an interval), or force a calibration, from the dashboard or via MQTT. 

A mode change always discards the calibration and forces a fresh relearn, so thresholds are never applied to a stale baseline.

## 5. Anomaly Detection Algorithm

### 5.1 Calibration (Hampel filter)

Pump current baseline computation:

```
MAD   = median( |x_i - median(x)| )
σ_H   = 1.4826 × MAD
keep x_i  if  |x_i - median(x)| ≤ 3 × σ_H
μ, σ  = mean and std of the kept (steady-state) samples only
```

Because the test is anchored on the **median**, an inrush spike captured during learning is rejected as an outlier instead of inflating the thresholds.

Derived thresholds:

```
th_stall    = μ + 3σ                       (floored at μ + 15 mA)
th_dry_run  = μ - 3σ                       (floored at μ - 15 mA)
th_volt_min = max(3.30 V, 0.90 × V_cal)
temperature band = [ μ_T - Δ , μ_T + Δ ],  Δ = max(5σ_T, 1.5 °C),  clamped to [16, 32] °C
```

Stall and dry-run are **symmetric Shewhart control limits** around the learned mean - an upper `μ + 3σ` and a lower `μ - 3σ`, each kept at least 15 mA from the mean. The voltage floor is a **physical battery cutoff** (`max(3.30 V, 0.90·V_cal)`). The temperature band is **learned per tank** from the temperature samples collected during the same learning window: a tropical tank calibrated at 26 °C gets different limits than a cold-water tank at 20 °C, with no manual configuration.

### 5.2 Monitoring (EWMA + confirmation + grace)

```
ewma_t = α · I_raw + (1 - α) · ewma_{t-1}        α = 0.2
```

- **EWMA** damps single-sample spikes (one spike moves the average by ≤20 % of its magnitude) while tracking a sustained shift.
- **Grace period:** the first 4 samples after the pump starts are skipped to let the motor reach steady state; the EWMA is then re-seeded with the settled value so inrush never leaks into the monitoring window.
- **Confirmation gate:** an anomaly flag must hold for `CONFIRM_NEEDED = 3` consecutive samples before it is acted upon.

### 5.3 Anomaly Types, Actions and Priority

| Anomaly | Condition (on EWMA / reading) | Physical cause | Action |
|---|---|---|---|
| `MOTOR_STALL` | `ewma > μ + 3σ` | Impeller blocked / mechanical fault | **HALT** (10× burst), lock, buzzer |
| `DRY_RUN` | `ewma < μ - 3σ` (and `ewma > 20 mA`, no voltage fault) | No water, pump running dry | **HALT** (10× burst), lock, buzzer |
| `VOLTAGE_DROP` | `V < max(3.30 V, 0.90 V_cal)` | Supply sag / brown-out | **Warning** only (pump keeps running) |
| `TEMP_TOO_HIGH` | `T > th_temp_high` | Heater fault / overheating | **Warning** only |
| `TEMP_TOO_LOW` | `T < th_temp_low` | Cold draft / heater failure | **Warning** only |
| `TURBIDITY_LOW` | `turbidity < 100 NTU` | Cloudy / dirty water | **Warning** only |
| `DEGRADATION` | CUSUM exceeds limit (§6) | Slow wear / partial fouling | **Warning** (predictive maintenance) |

Priority when several flags are active at once: `MOTOR_STALL` > `DRY_RUN` > `VOLTAGE_DROP` > `TEMP` > `TURBIDITY`. Only the two safety-critical anomalies stop the pump.

## 6. Predictive Maintenance - CUSUM

At the end of each healthy monitoring cycle, the mean current of that cycle is compared to the learned baseline and accumulated in a one-sided CUSUM:

```
dev      = (cycle_mean - μ) - K·σ            K = 1.0   (tolerance band)
cusum   += dev,   clamped at 0
fire if   cusum > H·σ                        H = 3.0   (decision threshold)
```

A small but **persistent** upward drift in the pump's healthy current accumulates until it crosses the limit and raises a `DEGRADATION` service warning, while ordinary cycle-to-cycle noise washes out. This catches gradual wear or partial fouling before it becomes a hard stall - the predictive-maintenance goal.

## 7. Multi-Sensor Fusion

Optical turbidity readings depend on water viscosity, which falls as temperature rises (particles settle faster), so the same suspended load reads differently at 20 °C and 30 °C. The Target applies a linear correction before transmitting:

```
NTU_corrected = NTU_raw / (1 + 0.005 × (T - 25))
```

Without it, a warming tank would appear to "self-clean", masking a real water-quality problem. 

## 8. Dashboard, Cloud and Remote Control

- **Local dashboard:** a single-page interface served by the Observer over the local network, with a live consumption chart, system state, event feed, manual controls (in OFF mode), and a developer-only evaluation panel.
- **MQTT cloud bridge:** publishes telemetry, a retained device-shadow state, and alerts to HiveMQ Cloud, and subscribes to a command topic for remote control (a phone MQTT client can switch mode, clear a halt, or drive manual actions).
- **E-mail alerts:** on a confirmed anomaly the Observer triggers an HTTPS webhook (a Google Apps Script web app) that sends a notification e-mail.

## 9. IoT Security

| Layer | Measure | Rationale |
|---|---|---|
| ESP-NOW (Target↔Observer) | **AES-CCM encryption** (PMK + per-peer LMK) | Confidentiality + authenticity of the safety link |
| MQTT (Observer↔cloud) | **TLS** with **broker-certificate verification** and username/password auth | Encrypted, server-authenticated, access-controlled cloud channel |
| E-mail webhook | HTTPS | Encrypted transport to the notification endpoint |

## 10. Deep Sleep and Energy

The node remains in deep sleep for 20 s between operating cycles to minimize power consumption. Duty Cycle: The active time is 33.3% in the Pump + Feed scenario and 39.4% in the Pump-only scenario, resulting in different average current consumptions. Battery Life: The estimated battery life is approximately 51 hours for Pump + Feed and 40 hours for Pump-only.

## 11. Evaluation Instrumentation

The built-in evaluation harness records the operator's induced ground-truth against the device's actual prediction to build a 3x3 confusion matrix focusing strictly on the safety-critical pump actions (NORMAL, MOTOR_STALL, DRY_RUN). From this data, it computes Accuracy as the total number of correct predictions divided by the total tested cycles. Furthermore, it calculates the Macro FPR and Macro FNR by extracting the individual False Positive and False Negative rates for each specific class (one-vs-rest) and then mathematically averaging them.
