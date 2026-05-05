# Crypto Protocol Risk Scoring (CRS)

![Python](https://img.shields.io/badge/python-3.12-blue?logo=python&logoColor=white)
![Licence](https://img.shields.io/badge/licence-MIT-green)
![Statut](https://img.shields.io/badge/statut-v1.0-blue)

**Outil de ranking ML qui attribue un score de vigilance à des protocoles crypto (DeFi, CEX, bridges) pour prioriser les revues de sécurité humaines.**

> [English version](README.md)

---

## Vue d'ensemble

CRS est un projet de machine learning conçu pour classer des protocoles crypto par score de risque afin de prioriser les revues de sécurité humaines. Face à ~500 protocoles actifs et une capacité d'analyse limitée, l'objectif est de faire remonter les 30 candidats les plus à risque chaque trimestre, non pas pour prédire des hacks, mais pour appuyer un triage basé sur le risque.

Le projet couvre la collecte de données, le feature engineering, la comparaison de modèles, une évaluation orientée ranking et l'interprétabilité.

---

## Problématique

Une équipe sécurité à capacité limitée (un analyste à mi-temps) ne peut pas revoir intégralement chaque protocole chaque trimestre. La question est : **quels 30 protocoles parmi ~500 doit-on examiner en premier ?**

Une approche naïve (plus grand TVL, lancement le plus récent) n'est pas systématique et passe à côté de signaux structurels importants. CRS remplace la sélection ad hoc par un ranking reproductible, piloté par le modèle.

---

## Ce que le modèle fait, et ne fait pas

**Fait :**
- Apprendre le profil structurel des protocoles historiquement ciblés dans la base DefiLlama.
- Attribuer un `risk_score` à chaque protocole pour trier ~500 candidats.
- Orienter la capacité d'analyse limitée vers les cas les plus prioritaires.

**Ne fait pas :**
- Prédire si un protocole *va* être hacké.
- Utiliser des features pré-hack. La version actuelle repose sur des snapshots ; l'alignement temporel est documenté comme la limite principale et constitue la première piste d'amélioration.
- Remplacer le jugement humain. Chaque alerte requiert une validation par l'analyste.

---

## Données et sources

**Sources retenues :**

| Source | Rôle |
|--------|------|
| DefiLlama `/protocols` | Métadonnées : TVL, date de lancement, chaînes actives, nombre d'audits (4 800+ protocoles) |
| DefiLlama `/hacks` | Labels : 472 hacks documentés, 15,85 Md$ de pertes, période 2016–2026 |

**Sources explorées et écartées :**

- **Rekt News** : parsé intégralement (286 entrées) → 0 feature ML exploitable. Les tags encodent des blockchains et des techniques d'attaque, pas un statut d'audit structuré.
- **Dune Analytics** : couverture 0,8 % des protocoles DefiLlama après join. Biais de sélection non contrôlable, matching fragile.
- **CertiK Skynet / DeFiSafety** : pas d'API publique stable, couverture incomplète, scraping non reproductible.

**Dataset final :** 4 883 protocoles actifs (TVL > 0, date de lancement valide), dont 164 hackés (3,4 %) et 4 719 sains (96,6 %). Un protocole multi-incidents apparaît plusieurs fois, géré par split groupé (voir Méthodologie).

> **Note :** Les endpoints publics de l'API DefiLlama ont migré vers une offre payante en avril 2026. Les données utilisées ici correspondent à un snapshot de mars 2026.

---

## Méthodologie

### Notebook 1 : Collecte et assemblage
`01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb`

- Appel DefiLlama `/hacks` → labels
- Appel DefiLlama `/protocols` → features brutes
- Exploration et rejet de Rekt News, Dune, CertiK
- Feature engineering initial
- Documentation du biais temporel (snapshots 2026 vs hacks 2016–2025)
- Export Parquet → `data/`

### Notebook 2 : Modélisation et évaluation
`02_Crypto_Protocol_Risk_Scoring.ipynb`

- EDA : distribution des features, déséquilibre de classes (baseline naïve = 95,9 % d'accuracy sans rien apprendre)
- Pipeline sklearn avec `RobustScaler` + `OneHotEncoder` dans un `ColumnTransformer`
- Split train/val/test **groupé par protocole** (`GroupShuffleSplit`) ; aucun protocole commun entre les sets
- Comparaison Logistic Regression, Random Forest, XGBoost
- Gestion du déséquilibre : `class_weight='balanced'`, `scale_pos_weight=28.9` (XGBoost)
- Threshold tuning (seuil 0.3 pour mode max-recall)
- Évaluation finale sur test set : ranking protocol-level par `risk_score`
- Interprétabilité : feature importances, SHAP

---

## Feature engineering

| Feature | Justification |
|---------|---------------|
| `log_tvl` | Compresse la distribution asymétrique du TVL (10 k$ à 3 Md$) |
| `age_days` | Âge du protocole depuis la date de lancement |
| `tvl_per_day` | TVL / âge ; capture le profil "honey pot récent" (TVL élevé, peu audité) |
| `is_multichain` | Quantifie la surface d'attaque cross-chain |
| `is_bridge` | Les bridges ont une exposition au risque spécifique |
| `is_dex`, `is_lending` | Flags de catégorie de protocole |
| `audit_status` | Présence ou absence d'audits documentés |

Dans le notebook de modélisation, des variables dérivées supplémentaires sont testées, notamment `audit_per_year`, `lending_audit_score` (is_lending x audit_count) et `category` (encodée en one-hot).

---

## Modèles comparés

| Modèle | Notes |
|--------|-------|
| Logistic Regression | Baseline, interprétable, frontière de décision linéaire |
| Random Forest | Ensemble, gère les non-linéarités |
| XGBoost | Gradient boosting, `scale_pos_weight` pour le déséquilibre |

Modèle final : **Random Forest refitté sur train + validation** (meilleur ROC-AUC sur le set de validation, confirmé par le Recall@k sur le test set).

---

## Approche d'évaluation

L'accuracy standard est inutilisable sur un déséquilibre 3,4 % / 96,6 %. Les métriques retenues sont :

- **Recall@k** : parmi les k % de protocoles avec le score le plus élevé, quelle proportion des hacks réels couvre-t-on ?
- **Lift@k** : gain par rapport à une sélection aléatoire au rang k.
- **ROC-AUC** : capacité globale de discrimination.

---

## Résultats

Évaluation sur le **test set** (976 protocoles uniques, 25 hackés, jamais vus pendant l'entraînement) :

| Top k% | Alertes | Recall@k | Lift@k |
|--------|---------|----------|--------|
| 5 % | 48 | 40 % | 8,1x |
| 10 % | 97 | **44 %** | **4,4x** |
| 15 % | 146 | 48 % | 3,2x |
| 20 % | 195 | 60 % | 3,0x |

**ROC-AUC (test) : 0,7836**

Lecture : en inspectant les 97 protocoles avec le score le plus élevé (10 % de la population), on couvre 44 % des hacks documentés du test set, soit 4,4 fois mieux qu'une sélection aléatoire.

---

## Interprétabilité

Les feature importances et les valeurs SHAP sont calculées dans le Notebook 2. Points clés :

- Les feature importances du modèle arboré font ressortir `chain_count`, `age_days`, `is_lending`, `tvl_per_day` et `log_tvl` comme signaux principaux.
- `audit_count` reste présent mais doit être interprété avec prudence : il reflète en partie un effet de taille, les gros protocoles étant à la fois plus audités *et* plus ciblés.
- Les SHAP summary plots montrent la contribution individuelle de chaque protocole à chaque prédiction.

---

## Structure du repository

```
CRS/
├── 01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb  # Collecte de données
├── 02_Crypto_Protocol_Risk_Scoring.ipynb                       # Modélisation & évaluation
├── data/
│   └── df_defi_risk.parquet                                    # Dataset traité
├── exports/
│   ├── 01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.pdf
│   └── 02_Crypto_Protocol_Risk_Scoring.pdf
├── crypto_protocol_risk_scoring.pkl                            # Modèle sauvegardé
├── pyproject.toml
├── README.md                                                   # Version anglaise
└── README.fr.md                                               # Ce fichier (français)
```

---

## Installation et exécution

```bash
# Cloner le repo
git clone https://github.com/V-Vaal/CRS.git
cd CRS

# Créer le venv Python 3.12 avec uv et installer les dépendances
uv venv --python 3.12 .venv
source .venv/bin/activate
uv sync

# Lancer JupyterLab
jupyter lab
```

Exécuter les notebooks dans l'ordre :
1. `01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb` → génère `data/df_defi_risk.parquet`
2. `02_Crypto_Protocol_Risk_Scoring.ipynb` → entraîne le modèle et évalue sur le test set

**Aucune clé API requise.** Les données proviennent des API publiques DefiLlama (snapshot mars 2026). Note : les endpoints DefiLlama ont migré vers une offre payante depuis avril 2026.

---

## Limites

**Biais temporel.** TVL, `audit_count` et `chain_count` sont des snapshots de mars 2026 appliqués à des hacks survenus entre 2016 et 2025. Un protocole hacké peut afficher un TVL effondré post-hack ; le modèle observe une réalité post-incident pour les exemples positifs.

**Corrélation ≠ causalité.** `audit_count` élevé est corrélé au risque parce que les gros protocoles sont à la fois plus audités *et* plus ciblés. C'est un proxy de taille, pas un signal causal exploitable en production.

**Biais CEX.** `audit_count` est un indicateur natif DeFi (audits de smart contracts). Les CEX remontent avec des scores élevés parce qu'ils ont 0 audit smart contract, pas parce qu'ils sont intrinsèquement plus risqués dans le sens DeFi du terme.

**Dataset incident-level.** Un protocole multi-incidents pèse proportionnellement plus en entraînement. Piste de correction : déduplication ou pondération par protocole avant le fit.

**`hacked=0` ≠ sûr.** La classe négative regroupe protocoles genuinement sains, protocoles trop petits pour être ciblés et protocoles hackés non documentés (bruit de label).

---

## Pistes d'amélioration

**Alignement temporel des features (priorité haute).** TVL pré-hack via `/api/protocol/{slug}` et `/api/inflows/{protocol}/{timestamp}` → features temporellement alignées (TVL à T-30, volatilité sur 90 j). Résoudrait partiellement le biais temporel.

**Graph Neural Networks.** Les modèles tabulaires ne capturent pas les interactions entre protocoles et wallets. PyTorch Geometric permettrait d'exploiter les patterns de transactions : flash loans, interactions inter-protocoles, contamination de liquidité.

**Anomaly detection.** Isolation Forest ou autoencoders pour identifier les protocoles structurellement atypiques, utile en l'absence de labels fiables sur de nouveaux protocoles.

**Survival analysis.** Modéliser le temps jusqu'au hack plutôt qu'un classifieur binaire. Plus adapté aux données censurées (protocoles actifs dont on ne connaît pas encore le futur).

**Features on-chain.** Dune Analytics avec un matching robuste (par adresse de contrat, pas par nom) donnerait accès à des métriques d'activité, de concentration des wallets et de flux de liquidité.

---

## Stack technique

| Composant | Version |
|-----------|---------|
| Python | 3.12 |
| pandas | ≥ 2.0 |
| numpy | ≥ 1.24 |
| scikit-learn | ≥ 1.3 |
| XGBoost | ≥ 2.0 |
| SHAP | ≥ 0.44 |
| matplotlib / seaborn | ≥ 3.7 / 0.12 |
| pyarrow | ≥ 14.0 |
| uv | gestion des dépendances |

---

## Licence

MIT - Voir [LICENSE](LICENSE).

---

## Auteur

**Valentin Valluet**

- GitHub : [github.com/V-Vaal](https://github.com/V-Vaal)
- LinkedIn : [linkedin.com/in/valentin-valluet](https://linkedin.com/in/valentin-valluet)
- X : [@val2_x](https://x.com/val2_x)
