# FLOAT (Framework for Local Observation of Aquatic Tanks)

* MODULE: Edge-AI Anomaly Detection & Predictive Maintenance
* HARDWARE: ESP32-S3, INA219 (I2C), DC Pump, Micro Servo, Active Buzzer
* DESCRIPTION: This system implements a Statistical Anomaly Detection model to monitor the health of an aquarium pump. It uses a dynamic 3-Sigma threshold calculated during a learning phase and validates faults through temporal debouncing (3 consecutive samples) to eliminate false positives.


## Team Members
* Edoardo Zompanti, 1985499 - www.linkedin.com/in/edoardo-zompanti-a8678a3b4
* Michele Libriani, 1954541 - www.linkedin.com/in/michele-libriani-805985316
* Andrea Folino, 1986019 - www.linkedin.com/in/andrea-folino-981aa5322