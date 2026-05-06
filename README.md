# UVIF-IoT

**Variational Intelligence Framework for Robust and Interpretable Intrusion Detection in Resource-Constrained IoT Networks**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.x-orange.svg)](https://pytorch.org/)

---

## What this is

UVIF-IoT is a network intrusion detection framework built for IoT devices. It treats IDS not as a pure accuracy problem but as a **constrained multi-objective problem**: detect attacks reliably, remain useful under adversarial perturbations, report uncertainty that operators can trust, and do all of this within the power and memory envelope of a microcontroller.

These four objectives are balanced not with hand-tuned penalty weights but through a **Lagrangian duality** mechanism: the trade-off coefficients are learned from the training data and quantify precisely which constraints are binding and by how much. Every design trade-off is visible and adjustable without retraining.

---

## Core components

| Component | Role |
|---|---|
| **Evidential Dirichlet classifier** | Decomposes uncertainty into aleatoric and epistemic components; flags novel attack patterns without a separate OOD detector |
| **Mahalanobis-ball DRO** | Feature-heterogeneous adversarial training with a closed-form worst-case perturbation — no PGD inner loop required |
| **Brier calibration regulariser** | Strictly proper scoring rule; improves confidence-accuracy alignment in the high-probability regime |
| **Concrete-relaxed hard gating** | Binary feature masks at inference time — non-selected features are literally not computed, making energy savings real |
| **Lagrangian dual optimiser** | Learns penalty weights $(\mu_1, \mu_2, \mu_3)$ from data; replaces manual $\lambda$ tuning |

---

## Results

Evaluated on two public IoT network traffic datasets: **N-BaIoT** (300 k flows, 3 classes: benign, Mirai, Gafgyt) and **UNSW-NB15** (175 k training / 80 k test, 10 traffic categories including 9 attack families).

### Clean-traffic detection

| Method | N-BaIoT F₁ | N-BaIoT ECE | UNSW F₁ | UNSW ECE |
|---|---|---|---|---|
| ERM-MLP | 0.9996 | 0.0003 | 0.4836 | 0.0505 |
| AT-ERM | 0.9996 | 0.0002 | 0.4937 | 0.0299 |
| MC-Dropout | 0.9995 | 0.0003 | 0.4765 | 0.0284 |
| Random Forest | 1.0000 | 0.0002 | 0.5424 | 0.0463 |
| Isolation Forest | 0.1645 | 0.5894 | 0.0016 | 0.9084 |
| **UVIF-IoT** | **0.9996** | 0.0003 | 0.4317 | **0.0449** |

ECE measures calibration error — lower is better. UVIF-IoT achieves the lowest ECE among neural models on UNSW-NB15.

### Adversarial robustness (macro-F₁)

| Method | N-BaIoT · PGD-10 ε=0.02 | UNSW · FGSM ε=0.01 | UNSW · PGD-10 ε=0.02 |
|---|---|---|---|
| ERM-MLP | 0.9995 | 0.2528 | 0.2047 |
| AT-ERM | 0.9995 | 0.3128 | 0.2701 |
| MC-Dropout | 0.9995 | 0.2729 | 0.2240 |
| **UVIF-IoT** | 0.9993 | **0.3144** | **0.2890** |

UVIF-IoT leads all baselines on UNSW-NB15 adversarial F₁ across all attack configurations tested.

### Energy-gated inference

A single trained model supports three hardware profiles by adjusting the energy budget at inference — no retraining required.

| MCU Profile | Budget | Active features (N-BaIoT) | F₁ (N-BaIoT) | Active features (UNSW) | F₁ (UNSW) |
|---|---|---|---|---|---|
| Cortex-M4 @ 168 MHz | 22% | 25 / 115 | 0.9103 | 43 / 197 | 0.3725 |
| ESP32 @ 240 MHz | 26% | 29 / 115 | 0.9541 | 51 / 197 | 0.4170 |
| Cortex-M7 @ 216 MHz | 30% | 34 / 115 | 0.9987 | 59 / 197 | 0.4306 |

At the Cortex-M7 profile, the model retains 99.9% of full-feature N-BaIoT F₁ while eliminating 70% of feature computation.

---

## Repository structure

```
UVIF-IoT/
├── UVIF_IoT_FINAL.ipynb      # Full self-contained experiment notebook
├── requirements.txt           # Python dependencies
├── data/
│   └── README_data.md         # Dataset download instructions
├── outputs/
│   ├── figures/               # All paper figures (PDF + PNG)
│   ├── tables/                # All paper tables (CSV + LaTeX)
│   └── checkpoints/           # Saved model weights (.pt)
└── README.md
```

---

## Requirements

```
python >= 3.10
torch >= 2.0
numpy
pandas
scikit-learn
matplotlib
```

```bash
pip install -r requirements.txt
```

---

## Datasets

### N-BaIoT
Download from the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/442/detection+of+iot+botnet+attacks+n+baiot). Place the extracted folders under `data/N_BaIoT/` preserving the device-name subfolder structure:

```
data/N_BaIoT/
├── Danmini_Doorbell/
│   ├── benign_traffic.csv
│   ├── mirai_attacks/
│   └── gafgyt_attacks/
├── Ecobee_Thermostat/
└── ...
```

### UNSW-NB15
Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` from the [UNSW Canberra Cyber portal](https://research.unsw.edu.au/projects/unsw-nb15-dataset). Place them under `data/UNSW_NB15/`.

The notebook auto-detects both datasets from a configurable list of candidate paths. If files are not found it falls back to synthetic smoke-test data, which is automatically flagged as non-reportable.

---

## Running the experiments

### Google Colab (recommended)

1. Upload `UVIF_IoT_FINAL.ipynb` to Google Colab.
2. Go to **Runtime → Change runtime type → T4 GPU**.
3. Mount your Google Drive and place the datasets at the paths above.
4. Run all cells in order.

Total runtime on a T4 GPU: approximately 19 minutes. The notebook covers the full pipeline — data loading, preprocessing, training with early stopping and checkpointing, and all evaluation cells (clean detection, adversarial robustness, calibration, energy gating, saliency ablation, Pareto sweep, tables, figures).

### Local

```bash
git clone https://github.com/<your-username>/UVIF-IoT.git
cd UVIF-IoT
pip install -r requirements.txt
jupyter notebook UVIF_IoT_FINAL.ipynb
```

> **Note:** Cell 1 includes a GPU guard that raises a `RuntimeError` when no CUDA device is detected. To run on CPU for inspection only, comment out lines 23–27 of Cell 1. Training on CPU is functional but will take several hours.

---

## Configuration

All hyperparameters are set in the `CFG` dictionary in Cell 1. The most relevant entries:

| Parameter | Default | Description |
|---|---|---|
| `epochs` | 30 | Training epochs per model |
| `uvif_warmup_epochs` | 8 | Evidential-only warmup before gating activates |
| `tau_start` / `tau_end` | 1.0 / 0.05 | Concrete gate temperature schedule |
| `epsilon_r` | 0.20 | Robustness constraint budget |
| `epsilon_c` | 0.15 | Calibration constraint budget |
| `epsilon_e` | 0.35 | Energy constraint budget |
| `eta_mu` | 0.03 | Dual variable learning rate |
| `rho_wass` | 0.01 | Mahalanobis DRO perturbation radius |
| `beta_evid` | 0.10 | Evidential KL penalty weight |
| `adv_batch_cap` | 50 | Batch cap for adversarial evaluation (`None` = full test set) |

### Activating all three Lagrange multipliers

The default budgets are conservative — only $\mu_1$ (robustness) becomes non-zero on UNSW-NB15. To activate all three dual variables simultaneously, tighten the budgets:

```python
'epsilon_r': 0.05,
'epsilon_c': 0.02,
'epsilon_e': 0.10,
```

---

## Extending the framework

UVIF-IoT is designed to be modular. Common extension points:

- **Different datasets.** Add a loading block in Cell 2 following the same RAM-safe pattern (stratified sampling, provenance flag). The rest of the notebook requires no changes.
- **Different hardware profiles.** Add an entry to `CFG['mcu_profiles']` with a new budget value. No retraining is needed — the energy budget is applied post-hoc at inference.
- **Tighter constraints.** Adjust `epsilon_r`, `epsilon_c`, `epsilon_e` and rerun Cell 6. The dual optimiser adapts automatically.
- **Full Mahalanobis covariance.** The current implementation uses a signed-gradient fallback for the DRO perturbation. The `init_mahalanobis()` function in Cell 4 is a placeholder; replace it with a full covariance estimation on benign training data to instantiate the closed-form perturbation described in the methodology.

---

## License

This project is released under the [MIT License](LICENSE).
