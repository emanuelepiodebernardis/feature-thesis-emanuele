# IoT/IIoT Intrusion Detection System using Machine Learning

Author: Emanuele Pio De Bernardis  
Affiliation: Università eCampus  
License: MIT

---

## Overview

This repository implements a Machine Learning-based Intrusion Detection System (IDS) designed for IoT and IIoT network environments, with a focus on scalable deployment, adversarial robustness, and computational efficiency.

The system analyzes network flow telemetry derived from the TON_IoT dataset, simulating realistic IoT/IIoT smart infrastructure traffic under both benign and adversarial conditions.

The IDS supports two complementary detection paradigms:

- Binary classification: distinguishing normal vs malicious traffic
- Multiclass classification: identifying specific attack families (e.g., DoS, DDoS, MITM, ransomware, scanning, injection attacks)

The objective is to build a reproducible, benchmark-driven intrusion detection pipeline that jointly optimizes:

- predictive performance
- inference latency
- model memory footprint
- cross-domain generalization capability

---

## Research Context

Modern IoT/IIoT ecosystems introduce significant cybersecurity challenges due to:

- heterogeneous and resource-constrained devices
- large-scale distributed attack surfaces
- high variability in network traffic distributions
- difficulty in generalizing detection models across environments

This project addresses these challenges by focusing on:

- Detection of network-based cyberattacks, including:
  - DoS / DDoS attacks
  - Man-in-the-Middle (MITM) attacks
  - botnet activity
  - injection and reconnaissance-based attacks
  - ransomware-related traffic patterns
- evaluation under class imbalance conditions typical of real-world security datasets
- robustness assessment under cross-domain shift (TON_IoT → CIC-IoT-Dataset2023)
- benchmarking of computational efficiency (latency + model size) for deployable IDS scenarios
- integration of model interpretability via SHAP-based explanations

---

## Pipeline Overview

The proposed IDS follows a modular and reproducible pipeline:

1. **Data preprocessing and cleaning**
   - Handling missing values and invalid flows
   - Normalization of numerical features
   - Encoding of categorical variables

2. **Feature engineering**
   - Unified representation of network flow features
   - Alignment across heterogeneous datasets (TON_IoT and CIC-IoT-Dataset2023)

3. **Supervised model training**
   - Multiple ML classifiers trained under identical preprocessing conditions

4. **Evaluation framework**
   - Classification metrics (Accuracy, Precision, Recall, F1-score)
   - Threshold-independent metrics (ROC-AUC, PR-AUC)
   - Cross-validation (Stratified K-Fold)

5. **Efficiency analysis**
   - Inference latency (ms per 1000 samples)
   - Model size on disk (MB)
   - Trade-off analysis between accuracy and computational cost

6. **Explainability layer**
   - SHAP-based global and local feature attribution
   - Class-specific interpretability for attack categories (e.g., MITM detection analysis)

---

## Models and Results

The following supervised learning models are evaluated under identical experimental conditions on the held-out test set (38,095 samples, 20% stratified split).

### Binary Classification (normal vs attack)

| Model | F1 | ROC-AUC | Latency (ms/1k) | Size (MB) |
|-------|----|---------|-----------------|-----------|
| LightGBM | 0.9992 | >0.9999 | 77.6 | 1.375 |
| Random Forest | 0.9990 | >0.9999 | 303.2 | 21.908 |
| XGBoost | 0.9989 | >0.9999 | 87.1 | 0.765 |
| MLP (DL baseline) | 0.9959 | 0.9993 | — | 0.119 |
| Decision Tree (d=5) | 0.9943 | 0.9856 | 47.2 | 0.019 |
| Logistic Regression | 0.9900 | 0.9945 | 55.5 | 0.016 |

A 5-fold stratified cross-validation confirms stable performance across splits
(LightGBM: F1 = 0.9993 ± 0.0001, variance < 0.0006 for all ensemble models).

### Multiclass Classification (10 attack classes)

| Model | Accuracy | Macro F1 | Latency (ms/1k) |
|-------|----------|----------|-----------------|
| XGBoost | 0.9896 | 0.9694 | 129.4 |
| LightGBM | 0.9897 | 0.9693 | 550.9 |
| Random Forest | 0.9880 | 0.9678 | 317.1 |
| Logistic Regression | 0.8526 | 0.7903 | 43.8 |
| Decision Tree (d=5) | 0.8148 | 0.7145 | 42.8 |

The MITM class (208 test samples, 0.55% of test set) is the most challenging,
with XGBoost achieving F1 = 0.786 on this class.

Each model is assessed in terms of detection performance, robustness under domain shift,
and computational efficiency.

---

## Quantization and Embedded Export

Three quantization pipelines are implemented for MCU deployment:

| Model | Format | Orig. (KB) | Quant. (KB) | CR | F1 | Target |
|-------|--------|-----------|------------|-----|-----|--------|
| Logistic Regression | C via m2cgen | 4.44 | 3.20 | 1.38x | 0.9900 | Arduino Mega / ESP32-C3 |
| Decision Tree (d=5) | C via m2cgen | 8.07 | 4.76 | 1.69x | 0.9943 | Arduino Mega / ESP32-C3 |
| MLP | TFLite Micro INT8 | 121.69 | 13.03 | 9.34x | 0.9959 | ESP32-C3 |
| XGBoost | INT8 binary (.bin) | 771.12 | 369.52 | 2.09x | 0.9989 | ESP32-C3 |
| LightGBM | INT8 binary (.bin) | 1396.24 | 73.85 | 18.91x | 0.9992 | ESP32-C3 |

Quantized models are in `quant_outputs/`. The serialization/deserialization
code for the custom INT8 binary format is in `embedded_model_io.py`.

---

## Physical Hardware Benchmarks

All five quantized models were deployed and benchmarked on real hardware
using PlatformIO firmware (see `ids_hw/` folder).

| Model | Board | Mean latency (µs) | SRAM used (B) | SRAM limit (B) |
|-------|-------|------------------|--------------|----------------|
| Logistic Regression | Arduino Mega 2560 | 1,270 | 7,180 | 8,192 |
| Decision Tree d=5 | Arduino Mega 2560 | 35 | 7,182 | 8,192 |
| Logistic Regression | ESP32-C3 SuperMini | 144 | — | 409,600 |
| Decision Tree d=5 | ESP32-C3 SuperMini | 3 | — | 409,600 |
| MLP TFLite INT8 | ESP32-C3 SuperMini | 805 | 1,676 | 409,600 |
| LightGBM INT8 | ESP32-C3 SuperMini | 5,564 | — | 409,600 |
| XGBoost INT8 | ESP32-C3 SuperMini | 8,240 | — | 409,600 |

The ESP32-C3 SuperMini (RISC-V 160 MHz) is **8.8–11.7× faster** than the
Arduino Mega 2560 (AVR 16 MHz) on identical compiled C code.

---

## Embedded Hardware Firmware

The `ids_hw/` folder contains the PlatformIO firmware, Arduino/ESP32 sketches,
and automation scripts used for physical benchmarking on real hardware.
See [`ids_hw/README.md`](ids_hw/README.md) for full setup and usage instructions.

Quick start:

```powershell
cd ids_hw
.\flash.ps1 -Device mega    -Model logreg  -Collect 20
.\flash.ps1 -Device esp32c3 -Model mlp     -Collect 20
```

---

## Cross-Domain Evaluation

Models trained on TON_IoT are evaluated against CIC-IoT-Dataset2023 using a
10-feature harmonised representation. Normalised distribution shift δ is computed
per feature; five of ten features exceed the δ > 1 threshold, with packet
asymmetry reaching δ = 5.95 — nearly six source standard deviations.

---

## Dataset: TON_IoT Network Dataset

Name: TON_IoT Network Dataset — IoT/IIoT network traffic for intrusion detection  
Provider: Cyber Range & IoT Labs, UNSW Canberra (SEIT)  
Official page: <https://research.unsw.edu.au/projects/toniot-datasets>  
License: Creative Commons Attribution 4.0 International (CC BY 4.0)

This repository uses the train/test network flows subset often distributed as
`train_test_network.csv` (~29.9 MB; 44 columns). The flows were captured in
realistic IoT/IIoT smart-environment scenarios using tools such as Argus and
Zeek. The dataset contains benign and malicious traffic and is suitable for
intrusion detection, anomaly detection, and ML benchmarking.

Download it from:
<https://www.kaggle.com/datasets/arnobbhowmik/ton-iot-network-dataset>

---

## Dataset: CIC-IoT Dataset 2023

Name: CIC IoT Dataset 2023  
Provider: Canadian Institute for Cybersecurity (CIC), University of New Brunswick  
Official page: <https://www.unb.ca/cic/datasets/iotdataset-2023.html>  
License: Research/academic use (as defined by CIC dataset policy)

This dataset contains modern IoT network traffic capturing:

- real-world IoT communications
- diverse attack scenarios
- updated attack patterns compared to older CIC datasets

It is used in this project for:

- external validation (cross-domain testing)
- robustness evaluation of trained models
- domain shift analysis between TON_IoT and CIC-IoT environments

---

## Related Publication

This repository accompanies the following paper currently under submission:

> De Bernardis, E. P., & Kuznetsov, O. (2026).  
> *Lightweight Machine Learning Intrusion Detection for IoT/IIoT Networks:  
> Quantization Strategies and Physical Deployment on Resource-Constrained  
> Microcontrollers*. Submitted to MDPI.

All code, model export scripts, PlatformIO firmware, and benchmark results
are publicly available in this repository under the MIT licence.


## How to contribute
See [CONTRIBUTING.md](CONTRIBUTING.md). Please run lint before commits.

## Citation
See [CITATION.cff](CITATION.cff).

## License
[MIT](LICENSE)
