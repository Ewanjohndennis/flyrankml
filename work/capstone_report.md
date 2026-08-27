# Capstone Report — Lane 2: Content Refresh Opportunity Scoring

- **Author:** Ewan John Dennis
- **Lane:** Lane 2 — Content Refresh / Opportunity Scoring
- **Repo:** https://github.com/Ewanjohndennis/flyrankml
- **Date:** August 2026

---

## 1. Problem framing

This project supports content operations teams, SEO strategists, and editorial managers in prioritizing content maintenance across portfolio-scale client websites. 

- **Unit of Analysis:** Individual content pages per client snapshot (`content_hash_id` scoped by `client_hash_id`).
- **Output:** A continuous risk score (`model_score` from `0.0` to `100.0`) mapped to a 3-tier actionable priority queue (`IMMEDIATE_REFRESH`, `SCHEDULED_UPDATE`, `MONITOR`) with primary reason codes.
- **Action Taken:** Human editors use the ranked queue to perform targeted copy updates, structural refreshes, and search intent alignment, rather than conducting unassisted site-wide audits.
- **Cost of Wrong Call:** 
  - *False Positive (Flagging a stable page as decaying):* Wastes editorial budget and writer hours on content that is already performing well.
  - *False Negative (Missing a decaying page):* Leads to continued loss of search index visibility, organic traffic decay, and revenue erosion.
- **Why Data/ML Helps:** Rule-based heuristics over-weight raw impression scale (flagging large, stable pages while ignoring decaying long-tail URLs). Machine learning captures non-linear interactions between historical volume, indexing frequency, and content staleness to isolate true decay risk.

---

## 2. Data safety

- **Data Source:** FlyRank ML Internship Warehouse (`hf://datasets/FlyRank/internship-warehouse`), specifically tables `fact_content_daily_performance` and `dim_content`.
- **Snapshot Date Window:** Evaluation snapshot `month = '2026-03'` (Days 1–30 for historical features `imp_prev30`; Days 31–60 for target outcome `imp_last30`).
- **Feature Set Used:** `log_imp_prev30`, `days_with_impressions`, `avg_position`, `avg_daily_clicks`, `days_since_last_update`.
- **Deliberately Excluded Columns:** 
  - `imp_last30`: Excluded from inputs as it forms the target label outcome window.
  - `trend_direction` & `trend_pct`: Excluded to eliminate label-derived feature leakage.
  - `client_hash_id` & `content_hash_id`: Excluded from feature matrix $X$ and used strictly for data joining and `GroupKFold` cross-validation splitting.
- **Public Safety Confirmation:** No raw domain names, URLs, client names, or proprietary search queries exist in `work/`. All client entity references utilize pseudonymous hashes (`client_hash_id`).

---
## 3. Baseline

- **Transparent Rule Baseline:** We constructed a composite heuristic score (`baseline_score`) integrating impression volume tiers, content staleness age, and historical log scales:
  `baseline_score = tier_score + (12 * log10(imp_prev30)) + ((days_since_update / 365) * 15)`
- **Fairness of Comparison:** Evaluated on the exact same snapshot dataset (`175,205` total pages) and the identical 5-fold `GroupKFold` cross-validation split as the machine learning models.
- **Baseline Performance Metrics:**
  - **Average Precision (PR-AUC):** `0.998828`
  - **ROC-AUC Score:** `0.903117`
  - **AP Standard Deviation:** `0.000309`
---

## 4. Model / analysis

- **Method:** Evaluated Random Forest Classifier (`max_depth=6`, `n_estimators=100`) and LightGBM Classifier (`max_depth=4`, `learning_rate=0.05`). Tree-based ensembles naturally capture non-linear thresholds and complex feature interactions without requiring linear feature normalization.
- **Exact Feature List:**
  1. `log_imp_prev30`: Base-10 log of total GSC impressions in days 1–30.
  2. `days_with_impressions`: Total active indexing days with $>0$ impressions.
  3. `avg_position`: Trailing mean Search Console ranking position.
  4. `avg_daily_clicks`: Mean daily organic click volume.
  5. `days_since_last_update`: Days elapsed between `content_updated_date` and snapshot evaluation date.
- **Target Proxy Definition (One Sentence):** `is_declining = 1` if organic search impressions in days 31–60 (`imp_last30`) drop by more than 20% relative to impressions in days 1–30 (`imp_prev30`), and `0` otherwise.

---

## 5. Evaluation

- **Validation Split Strategy:** 5-fold cross-validation grouped strictly by `client_hash_id` (`GroupKFold`). Standard random splits cause severe domain-level authority memorization; grouped splits enforce honest zero-shot evaluation on completely unseen client domain portfolios.
- **Task Base Rate:** Positive decay prevalence (`is_declining = 1`) across the snapshot portfolio is **85.35%**.

### Model Performance vs Baseline (Grouped 5-Fold CV):

| Model / Strategy | Average Precision (PR-AUC) | ROC-AUC Score | AP Std Dev |
|:---|:---:|:---:|:---:|
| **Week 4 Rule Baseline** | `0.998828` | `0.903117` | `0.000309` |
| **Random Forest (Grouped)** | `0.999659` | `0.967508` | `0.000102` |
| **LightGBM (Grouped)** | **`0.999733`** | **`0.973368`** | **`0.000104`** |

- **Error Analysis:**
  - *False Positives:* Primarily occur on newly published pages (<30 days old) experiencing natural post-launch impression settling rather than structural content decay.
  - *False Negatives:* Occur on high-authority head terms where impression volume remains high despite slight ranking position drops, leading the model to underestimate subtle decay.

---

## 6. Interpretation

- **Feature Importance:**
  1. `log_imp_prev30` & `days_with_impressions` account for over 65% of split importance, proving that active indexing frequency and prior volume scale are the primary anchors for decay risk.
  2. `days_since_last_update` provides crucial secondary signal, accelerating risk scores for pages exceeding 180+ days of staleness.
- **Empirical Findings & Surprises:**
  - Naïve random train/test splits produced an artificially inflated ROC-AUC of `0.975394` via client domain memorization. Enforcing strict `GroupKFold` splits revealed an honest zero-shot ROC-AUC of `0.967508` (RF) and `0.973368` (LightGBM).
  - Pearson correlation checks confirmed all feature correlations with the target outcome remained between `+0.0115` and `+0.1598`, verifying zero temporal or target leakage.

---

## 7. Recommendation

### Content Action Playbook (Calibrated Portfolio Distribution)
Outputs are mapped to a 3-tier actionable queue using quantile score thresholds on `175,205` scored URLs:

1. **`IMMEDIATE_REFRESH` (24.42% of portfolio | Score $\ge 99.98$):**
   - *Reason Code:* `STALE_HIGH_TRAFFIC_DECAY`
   - *Action:* High-priority decay assets (>180d stale). Perform immediate editorial copy updates, update statistics, and submit for recrawl.
2. **`SCHEDULED_UPDATE` (20.98% of portfolio | $99.96 \le \text{Score} < 99.98$):**
   - *Reason Code:* `STALE_MODERATE_DECAY`
   - *Action:* Moderate decay risk (>90d stale). Slated for routine quarterly content maintenance.
3. **`MONITOR` (54.60% of portfolio | Score $< 99.96$):**
   - *Reason Code:* `STABLE_NO_ACTION`
   - *Action:* Stable performance or recent updates. No editorial effort required.

### Governance & No-Go Rules
- **Decision-Support Scope:** Scores serve as directional recommendation queues, not deterministic rank forecasts.
- **No-Go Automation List:** Full automation of live copy publishing, page sunsetting/deletions, or modifying legal/brand compliance content is strictly prohibited without human editorial sign-off.

---

## 8. Reproducibility

### Execution Workflow
To re-run the entire pipeline from a fresh clone:

```bash
# 1. Clone repository
git clone [https://github.com/Ewanjohndennis/flyrankml.git](https://github.com/Ewanjohndennis/flyrankml.git)
cd flyrankml

# 2. Install dependencies
pip install -r requirements.txt

# 3. Execute capstone notebook
python -m jupyter execute work/notebooks/capstone.ipynb

```

### Environment Details & Seeds

* **Python Version:** `3.13`
* **Core Dependencies:** `duckdb==1.2.0`, `pandas==2.2.3`, `scikit-learn==1.6.1`, `lightgbm==4.6.0`, `matplotlib==3.10.0`, `seaborn==0.13.2`
* **Random Seed:** Set strictly to `42` across all model fits (`RandomForestClassifier`, `LGBMClassifier`, `GroupKFold`).
* **Generated Receipts:**
* `work/outputs/baseline_action_score.csv`
* `work/outputs/playbook_metrics.json`
