# FLOAT

**Framework for Local Observation of Aquatic Tanks**

| | |
|---|---|
| **Team** | Michele Libriani (1954541) · Andrea Folino (1986019) · Edoardo Zompanti (1985499) |
| **Course** | Internet of Things - Sapienza University of Rome |
| **Repository** | https://github.com/erdod/FLOAT |
| **Demo / video** | https://www.youtube.com/watch?v=z5SY-IZXE9s |

**FLOAT** is an edge-first IoT system that monitors and manages aquariums with low-power embedded devices. Its safety-critical loop detects pump motor anomalies and shuts the pump down in under 2 seconds, with no router and no Internet - while an optional cloud layer adds remote monitoring, control and alerts.

## ARCHITECTURE
The system is built around three devices:
*   **Target Node (ESP32-S3)** - at the tank: reads water temperature (DS18B20), drives the pump and the servo food dispenser, and reads water turbidity from a dedicated Arduino Uno. Deep-sleeps between cycles.
*   **Arduino Uno** - hosts the real analog turbidity sensor and answers the Target over a UART handshake, isolating the analog reading from radio noise.
*   **Observer Node (ESP32-S3)** - measures pump current and bus voltage with an INA219, runs the anomaly-detection and predictive-maintenance algorithms, and acts as the gateway to the dashboard and cloud. A second INA219 measures the auxiliary supply for total power consumption.

Communication between the two ESP32 nodes uses ESP-NOW: connectionless, low-power, and AES-CCM encrypted.

## TECHNICAL FEATURES
*   Edge-to-edge ESP-NOW link (encrypted, no router needed)
*   Anomaly detection: motor stall, dry-run, voltage drop, adaptive temperature limits, and low turbidity warnings
*   Robust calibration with a Hampel filter (Median Absolute Deviation) to exclude motor inrush current
*   CUSUM predictive maintenance: catches slow pump degradation before a hard failure
*   Real-time current and total-power monitoring (two INA219 sensors)
*   Emergency motor cutoff on a confirmed stall or dry-run
*   MQTT cloud bridge over TLS with a retained device shadow, plus a local web dashboard and email alerts
*   Deep-sleep duty cycling on the Target node for energy efficiency

## PERFORMANCE METRICS
*   Motor cutoff reaction time: **~1251 ms** analytically.
*   False-positive and false-negative rates: **2.4%** and **4.8%** over normal operation, evaluated with a built-in **3-class** confusion matrix scoring only the halt-relevant decisions (NORMAL, MOTOR_STALL, DRY_RUN).
*   Target node duty cycle: **~33-39%** (20 s deep-sleep vs active cycle).

## FUTURE WORK
*   **Fleet Management:** LoRaWAN telemetry integration for remote, multi-tank monitoring.
*   **Device-Level Security:** Implementing ESP32 Flash Encryption and Secure Boot V2 for hardcoded credentials.
*   **Advanced Control:** Active Heater Control and additional water-chemistry sensors (pH, dissolved oxygen, ammonia).

GitHub Repository: https://github.com/erdod/FLOAT