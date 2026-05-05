# Crypto Protocol Risk Scoring (CRS)

![Python](https://img.shields.io/badge/python-3.12-blue?logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-v1.0-blue)

**A machine learning ranking tool that assigns a vigilance score to crypto protocols (DeFi, CEX, bridges) to prioritize human security reviews.**

> [Version française](README.fr.md)

---

## Overview

CRS is a machine learning project designed to rank crypto protocols by vigilance score in order to prioritize human security reviews. Given ~500 active protocols and limited analyst capacity, the goal is to surface the 30 highest-risk candidates each quarter, not to predict hacks, but to support risk-based triage.

The project covers data collection, feature engineering, model comparison, ranking-oriented evaluation, and interpretability.

> The notebooks are primarily written in French as they document the project methodology in detail. Each notebook includes an English summary at the top to make the workflow understandable for non-French readers.

---

## Problem Statement

A security team with limited capacity (one part-time analyst) cannot review every protocol from scratch each quarter. The question is: **which 30 protocols out of ~500 should be reviewed first?**

A naive approach (biggest TVL, most recent launch) is not systematic and misses important structural signals. CRS replaces ad hoc selection with a reproducible, model-driven ranking.

---

## What the Model Does, and Does Not Do

**Does:**
- Learn the structural profile of protocols historically targeted in the DefiLlama hack dataset.
- Assign a `risk_score` to each protocol to sort ~500 candidates.
- Direct limited analyst capacity toward the highest-priority cases.

**Does not:**
- Predict whether a protocol *will* be hacked.
- Use pre-hack features. Current version uses snapshot-based inputs; temporal alignment is documented as the main limitation and the primary improvement path.
- Replace human judgment. Every alert requires analyst validation.

---

## Dataset and Sources

**Sources used:**

| Source | Role |
|--------|------|
| DefiLlama `/protocols` | Metadata: TVL, launch date, active chains, audit count (4,800+ protocols) |
| DefiLlama `/hacks` | Labels: 472 documented hacks, $15.85B in losses, 2016–2026 |

**Sources explored and discarded:**

- **Rekt News**: fully parsed (286 entries) → 0 usable ML features. Tags encode blockchains and attack techniques, not structured audit status.
- **Dune Analytics**: 0.8% coverage of DefiLlama protocols after join. Uncontrollable selection bias, fragile matching.
- **CertiK Skynet / DeFiSafety**: no stable public API, incomplete coverage, non-reproducible scraping.

**Final dataset:** 4,883 active protocols (TVL > 0, valid launch date), of which 164 hacked (3.4%) and 4,719 clean (96.6%). Multi-incident protocols appear multiple times, handled by grouped split (see Methodology).

> **Note:** DefiLlama public API endpoints were migrated to a paid plan in April 2026. Data used here reflects a March 2026 snapshot.

---

## Methodology

### Notebook 1: Data Collection
`01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb`

- DefiLlama `/hacks` call → labels
- DefiLlama `/protocols` call → raw features
- Exploration and rejection of Rekt News, Dune, CertiK
- Initial feature engineering
- Documentation of temporal bias (2026 snapshots vs. 2016–2025 hacks)
- Parquet export → `data/`

### Notebook 2: Modeling and Evaluation
`02_Crypto_Protocol_Risk_Scoring.ipynb`

- EDA: feature distributions, class imbalance (naive baseline = 95.9% accuracy with zero learning)
- sklearn pipeline: `RobustScaler` + `OneHotEncoder` in a `ColumnTransformer`
- Train/val/test split **grouped by protocol** (`GroupShuffleSplit`); no protocol shared across sets
- Model comparison: Logistic Regression, Random Forest, XGBoost
- Imbalance handling: `class_weight='balanced'`, `scale_pos_weight=28.9` (XGBoost)
- Threshold tuning (0.3 for max-recall mode)
- Final evaluation on test set: protocol-level ranking by `risk_score`
- Interpretability: feature importances, SHAP

---

## Feature Engineering

| Feature | Rationale |
|---------|-----------|
| `log_tvl` | Compresses asymmetric TVL distribution ($10K to $3B) |
| `age_days` | Protocol age from launch date |
| `tvl_per_day` | TVL / age; captures "recent honey pot" profile (high TVL, low audit exposure) |
| `is_multichain` | Quantifies cross-chain attack surface |
| `is_bridge` | Bridge protocols have specific risk exposure |
| `is_dex`, `is_lending` | Protocol category flags |
| `audit_status` | Presence/absence of documented audits |

In the modeling notebook, additional derived features are tested, including `audit_per_year`, `lending_audit_score` (is_lending x audit_count), and `category` (one-hot encoded).

---

## Models Compared

| Model | Notes |
|-------|-------|
| Logistic Regression | Baseline, interpretable, linear decision boundary |
| Random Forest | Ensemble, handles non-linearities |
| XGBoost | Gradient boosting, `scale_pos_weight` for imbalance |

Final model: **Random Forest refitted on train + validation** (best ROC-AUC on validation set, confirmed by Recall@k on test set).

---

## Evaluation Approach

Standard accuracy is meaningless on a 3.4% / 96.6% imbalance. The primary metrics are:

- **Recall@k**: among the top k% of protocols by predicted score, what fraction of true hacks are captured?
- **Lift@k**: how much better than random selection is the model at rank k?
- **ROC-AUC**: global discrimination ability.

---

## Key Results

Evaluation on the **test set** (976 unique protocols, 25 hacked, never seen during training):

| Top k% | Alerts | Recall@k | Lift@k |
|--------|--------|----------|--------|
| 5% | 48 | 40% | 8.1x |
| 10% | 97 | **44%** | **4.4x** |
| 15% | 146 | 48% | 3.2x |
| 20% | 195 | 60% | 3.0x |

**ROC-AUC (test): 0.7836**

Interpretation: by reviewing the 97 highest-scored protocols (top 10%), we cover 44% of documented hacks in the test set: 4.4x better than random selection.

---

## Interpretability

Feature importances and SHAP values are computed in Notebook 2. Key findings:

- Tree-based feature importances highlight `chain_count`, `age_days`, `is_lending`, `tvl_per_day`, and `log_tvl` as the main signals.
- `audit_count` remains present but must be interpreted cautiously: it partly reflects a size effect, because large protocols are both more audited and more targeted.
- SHAP summary plots show individual protocol contributions to each prediction.

---

## Repository Structure

```
CRS/
├── 01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb  # Data collection
├── 02_Crypto_Protocol_Risk_Scoring.ipynb                       # Modeling & evaluation
├── data/
│   └── df_defi_risk.parquet                                    # Processed dataset
├── exports/
│   ├── 01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.pdf
│   └── 02_Crypto_Protocol_Risk_Scoring.pdf
├── crypto_protocol_risk_scoring.pkl                            # Saved model
├── pyproject.toml
├── README.md                                                   # This file (English)
└── README.fr.md                                               # French version
```

---

## How to Run

```bash
# Clone the repository
git clone https://github.com/V-Vaal/CRS.git
cd CRS

# Create a Python 3.12 virtual environment with uv and install dependencies
uv venv --python 3.12 .venv
source .venv/bin/activate
uv sync

# Launch JupyterLab
jupyter lab
```

Run notebooks in order:
1. `01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb` → generates `data/df_defi_risk.parquet`
2. `02_Crypto_Protocol_Risk_Scoring.ipynb` → trains the model and evaluates on the test set

**No API key required.** Data comes from DefiLlama public APIs (March 2026 snapshot). Note: DefiLlama endpoints have moved to a paid plan since April 2026.

---

## Limitations

**Temporal bias.** TVL, `audit_count`, and `chain_count` are March 2026 snapshots applied to hacks from 2016–2025. A hacked protocol may show collapsed TVL post-incident; the model observes a post-hack reality for positive examples.

**Correlation ≠ causation.** High `audit_count` correlates with risk because large protocols are both more audited *and* more targeted. This is a size proxy, not a causal signal usable in production.

**CEX bias.** `audit_count` is a native DeFi indicator (smart contract audits). CEX protocols surface with high scores because they have 0 smart contract audits, not because they are intrinsically riskier in a DeFi sense.

**Incident-level dataset.** A multi-incident protocol carries proportionally more weight in training. Correction path: deduplication or per-protocol weighting before fit.

**`hacked=0` ≠ safe.** The negative class mixes genuinely safe protocols, protocols too small to be targeted, and undocumented hacks (label noise).

---

## Future Improvements

**Temporal feature alignment (high priority).** Pull pre-hack TVL via `/api/protocol/{slug}` and `/api/inflows/{protocol}/{timestamp}` → time-aligned features (TVL at T-30, 90-day volatility). Partially resolves the temporal bias.

**Graph Neural Networks.** Tabular models cannot capture inter-protocol interactions. PyTorch Geometric would allow modeling transaction patterns: flash loans, cross-protocol interactions, liquidity contagion.

**Anomaly detection.** Isolation Forest or autoencoders to identify structurally atypical protocols, useful when reliable labels are absent for new protocols.

**Survival analysis.** Model time-to-hack rather than a binary classifier. Better suited to censored data (active protocols with unknown futures).

**On-chain features.** Dune Analytics with robust matching (by contract address, not name) would add wallet concentration, activity metrics, and liquidity flow data.

---

## Tech Stack

| Component | Version |
|-----------|---------|
| Python | 3.12 |
| pandas | ≥ 2.0 |
| numpy | ≥ 1.24 |
| scikit-learn | ≥ 1.3 |
| XGBoost | ≥ 2.0 |
| SHAP | ≥ 0.44 |
| matplotlib / seaborn | ≥ 3.7 / 0.12 |
| pyarrow | ≥ 14.0 |
| uv | dependency management |

---

## License

MIT - See [LICENSE](LICENSE)

---

## Author

**Valentin Valluet**

- GitHub: [github.com/V-Vaal](https://github.com/V-Vaal)
- LinkedIn: [linkedin.com/in/valentin-valluet](https://linkedin.com/in/valentin-valluet)
- X: [@val2_x](https://x.com/val2_x)
