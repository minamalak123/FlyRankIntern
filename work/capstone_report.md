# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Mina Malak
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/minamalak123/FlyRankIntern
- **Date:** 2026-09-14

## 0. Abstract

This project asks whether observable search-performance signals can help prioritize content items for review by estimating the likelihood of a future `went_dark` outcome. The modeling frame uses February 2026 content-level performance aggregated from the FlyRank internship warehouse, with March 2026 GSC clicks used as the future outcome when GSC measurement was available. I compared a transparent impressions-based baseline with Logistic Regression, Decision Tree, and Random Forest models using a client-grouped 80/20 holdout and ranking-oriented metrics. In the recorded Week-5 run, Random Forest achieved the strongest measured Average Precision (0.8803) and ROC AUC (0.8445), compared with 0.4817 and 0.2103 for the recorded simple baseline, while its top 10/50/100/500 ranked items had observed `went_dark` rates of 100%, 98%, 96%, and 95.2%. The resulting score is intended as decision-support for prioritizing human content review, not as proof that a refresh will improve performance or as a prediction of any search-engine algorithm.

## 1. Problem framing

The decision is **which content items should receive limited review attention first**.

- **Unit of analysis:** `client_hash_id × content_hash_id`.
- **Output:** a risk score and ranked queue.
- **Human action:** review the highest-ranked content first for possible refresh or further investigation.
- **Target/proxy:** `went_dark = 1` when March 2026 measured GSC clicks are zero, among content items with at least one March day with GSC measurement.
- **Cost of a wrong call:** a false positive can waste editorial time on content that does not need intervention; a false negative can leave a potentially important item low in the review queue.

ML is useful here because the queue is a ranking problem: the goal is not simply to classify every item correctly, but to concentrate likely future-risk items near the top of a finite review queue.

## 2. Data safety

The source data is the pseudonymized FlyRank internship warehouse. The final modeling frame uses five February performance features:

1. `impressions_feb`
2. `clicks_feb`
3. `ctr_feb`
4. `avg_position_feb`
5. `engagement_rate_feb`

`client_hash_id` is used only for grouping the holdout split and auditing, never as a predictive feature. `content_hash_id` is used only as an identifier for the modeling unit and audit output, never as a predictive feature.

The project deliberately avoids using future March outcome fields as predictors. In the earlier baseline work, `trend_direction` and other target-derived fields were explicitly treated as audit-only rather than scoring features. The final model likewise does not use `went_dark` or `clicks_mar` as predictive inputs.

The warehouse has an unbalanced panel and different source-availability histories, so unavailable GSC/GA4 measurement is not automatically interpreted as a real zero. The final March label requires observed GSC measurement. No client names, domains, raw queries, credentials, or other client-identifying information are included in the public report.

## 3. Baseline

Week 4 established a transparent refresh-prioritization rule based on **staleness plus observed search visibility**: content was prioritized when it was at least 180 days since update and had at least 300 impressions in the 90-day window. The Week-4 run audited 30,000 rows and produced 22 `refresh` actions. Its signal audit marked both staleness and search volume as `CONFIRMED` directional signals under the notebook's stated thresholding procedure.

For the final Week-5 model comparison, the notebook used a same-frame **Simple Baseline** based on February impressions. Its recorded test metrics were:

| Method | Average Precision | ROC AUC | Precision@10 | Precision@50 | Precision@100 |
|---|---:|---:|---:|---:|---:|
| Simple Baseline | 0.481691 | 0.210273 | 0.10 | 0.02 | 0.01 |
| Logistic Regression | 0.874937 | 0.841181 | 1.00 | 0.92 | 0.93 |
| Decision Tree | 0.874933 | 0.841293 | 1.00 | 0.96 | 0.96 |
| Random Forest | **0.880301** | **0.844464** | **1.00** | **0.98** | **0.96** |

The same-data baseline is the appropriate comparator for the final model result. The Week-4 staleness/visibility rule remains the human-readable starting heuristic rather than being retroactively treated as a directly comparable score on the Week-5 test frame.

**Base rate:** 60.8943% of the final Week-5 modeling frame had `went_dark = 1`.

## 4. Model / analysis

The primary model is **Random Forest** with 300 trees, maximum depth 8, minimum leaf size 20, balanced class weights, and random seed 42. Logistic Regression and a Decision Tree were also evaluated as simpler alternatives.

The five predictive features are exactly the February fields listed above. Missing numeric values were median-imputed using the training partition.

The prediction moment is the end of the February feature window. The outcome window is March 2026. This keeps the future March outcome separate from February predictors.

### Deliberate leakage control

The final capstone should include a leakage demonstration in which exactly one label-derived feature is temporarily added, evaluated, and then removed before the final model. The current recorded Week-5 notebook does not contain that experiment, so **no fabricated leakage score is reported here**. The capstone notebook package includes a rerunnable leakage-test cell rather than claiming a result that was not measured.

## 5. Evaluation

Week 5 used a **client-grouped 80/20 holdout** with `GroupShuffleSplit(random_state=42)`. The resulting split contained 105,652 training rows from 33 clients and 55,887 test rows from 9 clients, with zero client overlap.

This design tests generalization to clients held out from training. It is stronger than a random row split for this particular evaluation because it prevents the model from seeing other content from the same client during training. It is not a chronological multi-period backtest, so the result should not be interpreted as proof of performance across all future months.

### Recorded model performance

Random Forest was the strongest measured model on the recorded test split:

- Average Precision: **0.8803**
- ROC AUC: **0.8445**
- Precision@10: **1.00**
- Precision@50: **0.98**
- Precision@100: **0.96**

The simple baseline recorded Average Precision of **0.4817** and ROC AUC of **0.2103**.

### Error analysis

The recorded false-positive examples were generally high-scored items with very low February impressions/clicks that did not subsequently go dark. This shows that low observed activity can be a useful but imperfect risk signal.

The recorded false-negative examples included items with substantial February impressions and nonzero clicks that nevertheless had zero March clicks. This is an important failure mode: content can appear healthy in the feature month and still experience a future zero-click outcome.

## 6. Interpretation

Permutation importance from the recorded Random Forest run ranked the February features as follows:

| Feature | Mean permutation importance |
|---|---:|
| `impressions_feb` | 0.079231 |
| `clicks_feb` | 0.018777 |
| `ctr_feb` | 0.008149 |
| `avg_position_feb` | 0.005228 |
| `engagement_rate_feb` | 0.000331 |

The largest measured signal was February impressions, followed by clicks and CTR. Engagement rate contributed comparatively little in this particular run.

The strongest practical result is concentration at the top of the queue: the Random Forest's top-ranked test items had observed `went_dark` rates of 100% for the top 10, 98% for the top 50, 96% for the top 100, and 95.2% for the top 500.

These are **observed** results from one held-out split. They do not establish causality, do not prove that refreshing a page will reverse the outcome, and do not show that the model predicts Google's ranking algorithm.

## 7. Recommendation

Use the Random Forest score as a **review-prioritization signal**, not an automatic action.

A practical workflow is:

1. Sort content by predicted `went_dark` risk.
2. Start human review with the highest-risk items.
3. Check whether the item is actually appropriate for refresh, rather than assuming the score means it should be edited.
4. Use the model's input signals as reason codes: unusually low observed impressions/clicks, weak CTR, poor average position, or missing/limited engagement evidence.
5. Keep a human override for evergreen, intentionally stable, newly changed, or otherwise exceptional content.

Confidence should be highest when several observable signals point in the same direction and when measurement coverage is strong. Confidence should be lower when source coverage is sparse or the item falls into a failure pattern seen in the error analysis.

## 8. Reproducibility

The recorded notebooks use Python, pandas, NumPy, DuckDB, Hugging Face Hub, and scikit-learn. Random seed: **42**.

From a fresh clone, the intended environment is:

```bash
pip install -r requirements.txt
```

Set `HF_TOKEN` in a local `.env` file. Never commit the token or place it in a notebook.

Run the notebooks in this order:

```text
work/notebooks/w04_baseline_score.ipynb
work/notebooks/w05_model.ipynb
work/notebooks/capstone.ipynb
```

The capstone notebook is the final synthesis. A fresh run should regenerate the reported evaluation artifacts rather than relying on hard-coded metrics.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. Data and project context are credited to [FlyRank](https://flyrank.ai).

---

> **Submission framing:** observed / measured / directional / decision-support. No causal claims without a causal design, no client-identifying details, and no claims that the model predicts a search-engine algorithm.
