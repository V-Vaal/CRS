# Crypto Protocol Risk Scoring

![Python](https://img.shields.io/badge/python-3.12-blue?logo=python&logoColor=white)
![License](https://img.shields.io/badge/licence-MIT-green)
![Status](https://img.shields.io/badge/statut-v1.0-blue)

**Outil de ranking ML qui attribue un score de vigilance à des protocoles crypto (DeFi, CEX, bridges) pour prioriser les revues de sécurité humaines.**

---

## Ce que le modèle fait — et ne fait pas

**Fait** : le modèle apprend le profil structurel des protocoles *historiquement ciblés* dans la base DefiLlama et attribue un `risk_score` à chaque protocole. Ce score permet de trier ~500 protocoles et d'orienter les 30 revues humaines trimestrielles vers les cas les plus critiques.

**Ne fait pas** : il ne prédit pas si un protocole *va* être hacké. Les features sont des snapshots actuels, pas des données pré-hack. Les corrélations captées (`audit_count` élevé = plus ciblé) reflètent un effet de taille, pas un mécanisme causal. Toute alerte doit être validée par un analyste.

---

## Contexte

**Client fictif** : DeFiGuard Capital — fonds d'allocation de rendement DeFi, 12 personnes, 1 analyste sécurité à mi-temps. Le `risk_score` s'insère en amont de la revue humaine : les 30 protocoles retenus chaque trimestre sont ceux avec le score le plus élevé parmi les ~500 candidats, pas ceux que l'analyste connaît déjà.

---

## Données

**Sources retenues :**

| Source | Rôle |
|--------|------|
| DefiLlama `/protocols` | Métadonnées : TVL, date de lancement, chaînes actives, audits (4 800+ protocoles) |
| DefiLlama `/hacks` | Labels : 472 hacks documentés, $15,85 milliards de pertes, période 2016–2026 |

**Sources explorées et écartées :**

- **Rekt News** : parsé intégralement (286 entrées) → 0 feature ML exploitable. Les tags encodent des blockchains et des techniques d'attaque, pas un statut d'audit structuré.
- **Dune Analytics** : couverture 0,8% des protocoles DefiLlama après join. Biais de sélection non contrôlable, matching fragile.
- **CertiK Skynet / DeFiSafety** : scraping non pérenne, couverture incomplète, pas d'API publique stable.

**Dataset final** : 4 883 protocoles actifs (TVL > 0, date de lancement valide), dont 164 hackés (3,4%) et 4 719 sains (96,6%). Un protocole multi-incidents apparaît plusieurs fois — géré par split groupé en NB2.

---

## Pipeline

### Notebook 1 — Collecte et assemblage
`01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb`

- Appel DefiLlama `/hacks` → labels
- Appel DefiLlama `/protocols` → features brutes
- Exploration et rejet de Rekt News, Dune, CertiK
- Feature engineering initial : `log_tvl`, `age_days`, `tvl_per_day`, `is_multichain`, `is_bridge`, `is_dex`, `is_lending`, `audit_status`
- Documentation du biais temporel (snapshots 2026 vs hacks 2016–2025)
- Export Parquet → `data/`

### Notebook 2 — Modélisation et évaluation
`02_Crypto_Protocol_Risk_Scoring.ipynb`

- EDA : distribution des features, déséquilibre de classes (baseline naïve = 95,9% accuracy sans rien apprendre)
- Pipeline sklearn avec `RobustScaler` + `OneHotEncoder` dans un `ColumnTransformer`
- Split train/val/test **groupé par protocole** (`GroupShuffleSplit`) — aucun protocole commun entre les sets
- Comparaison Logistic Regression, Random Forest, XGBoost
- Gestion du déséquilibre : `class_weight='balanced'`, `scale_pos_weight=28.9` (XGB)
- Threshold tuning (seuil 0.3 pour mode max-recall)
- Évaluation finale sur test set : ranking protocol-level par `risk_score`
- Interprétabilité : feature importances, SHAP

---

## Choix méthodologiques

**Split groupé par protocole.** Un protocole impliqué dans plusieurs incidents (multi-incidents) ne peut pas apparaître à la fois en train et en test. Sans ce grouping, la fuite d'identité implicite surestime les performances.

**Recall@k comme métrique métier.** L'accuracy est inutilisable sur un déséquilibre 3,4% / 96,6%. La métrique qui compte est : parmi les 10% de protocoles avec le score le plus élevé, quelle proportion des hacks réels couvre-t-on ? C'est le Recall@k, complété par le Lift@k (gain vs sélection aléatoire).

**Feature engineering motivé par le domaine.** `tvl_per_day` (TVL / âge) capture le profil "honey pot récent" — protocole jeune avec gros TVL, peu audité, peu testé. `is_multichain` quantifie la surface d'attaque. `log_tvl` compresse la distribution asymétrique (10k$ à 3Md$).

**Biais assumé, pas caché.** Les features sont des snapshots 2026 pour des hacks survenus entre 2016 et 2025. Une correction asymétrique (TVL pré-hack pour les hackés seulement) aggraverait le biais. La limite est documentée en section Limites.

---

## Résultats

Évaluation sur le **test set** (974 protocoles uniques, 25 hackés, jamais vus pendant l'entraînement) :

| Top k% | Alertes | Recall@k | Lift@k |
|--------|---------|----------|--------|
| 5%     | 48      | 32%      | 6,5x   |
| 10%    | 97      | **44%**  | **4,4x** |
| 15%    | 146     | 52%      | 3,5x   |
| 20%    | 194     | 56%      | 2,8x   |

**ROC-AUC test : 0,7925** (modèle final : Logistic Regression refitté sur train+val).

Lecture : en inspectant les 97 protocoles avec le score le plus élevé (10% de la population), on couvre 44% des hacks documentés du test set — soit 4,4 fois mieux qu'une sélection aléatoire.

---

## Limites

**Biais temporel.** TVL, audit_count et chain_count sont des snapshots de mars 2026 appliqués à des hacks survenus entre 2016 et 2025. Un protocole hacké peut afficher un TVL effondré post-hack — le modèle voit une réalité post-incident pour les positifs.

**Corrélation ≠ causalité.** `audit_count` élevé est corrélé au risque parce que les gros protocoles sont à la fois plus audités *et* plus ciblés. Ce n'est pas un signal causal exploitable en production.

**Biais CEX.** `audit_count` est un indicateur natif DeFi (audits de smart contracts). Les CEX remontent avec des scores élevés parce qu'ils ont 0 audit smart contract, pas parce qu'ils sont intrinsèquement plus risqués dans le sens DeFi du terme.

**Dataset incident-level.** Un protocole multi-incidents pèse proportionnellement plus en entraînement. Piste de correction : déduplication ou pondération par protocole avant le fit.

**`hacked=0` ≠ sûr.** La classe négative regroupe protocoles genuinement sains, protocoles trop petits pour être ciblés, et protocoles hackés non documentés (bruit de label).

---

## Pistes d'amélioration

**Enrichissement temporel (priorité haute).** TVL pré-hack via `/api/protocol/{slug}` et `/api/inflows/{protocol}/{timestamp}` — features temporellement alignées (TVL à T-30, volatilité sur 90j). Résoudrait partiellement le biais temporel.

**Graph Neural Networks.** Les modèles tabulaires ne capturent pas les interactions entre protocoles et wallets. PyTorch Geometric permettrait d'exploiter les patterns de transactions : flash loans, interactions inter-protocoles, contamination de liquidité.

**Anomaly detection.** Isolation Forest ou Autoencoders pour identifier les protocoles structurellement atypiques — utile en l'absence de labels fiables sur de nouveaux protocoles.

**Survival analysis.** Modéliser le temps jusqu'au hack plutôt qu'un classifieur binaire. Plus adapté aux données censurées (protocoles actifs dont on ne connaît pas encore le futur).

**Features on-chain.** Dune Analytics avec un matching robuste (par adresse de contrat, pas par nom) donnerait accès à des métriques d'activité, de concentration des wallets, et de flux de liquidité.

---

## Stack

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
1. `01_Crypto_Protocol_Risk_Scoring_Collecte_des_données.ipynb` — génère `data/df_defi_risk.parquet`
2. `02_Crypto_Protocol_Risk_Scoring.ipynb` — entraîne le modèle et évalue sur le test set

Aucune clé API requise. Les données sont issues d'API publiques DefiLlama.

---

## Licence

MIT — voir [LICENSE](LICENSE).

---

## Auteur

**Valentin Valluet**

- GitHub : [github.com/V-Vaal](https://github.com/V-Vaal)
- LinkedIn : [linkedin.com/in/valentin-valluet](https://linkedin.com/in/valentin-valluet)
- X : [@val2_x](https://x.com/val2_x)
