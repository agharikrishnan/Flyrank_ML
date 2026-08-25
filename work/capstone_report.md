# Capstone Report — Content Refresh Opportunity Scoring

- **Author:** Sita Lakshmi R Shetty
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/agharikrishnan/Flyrank_ML
- **Date:** August 2026

## 0. Abstract

This project asks which content pages should be prioritized for review using measurable search-performance and content signals. I use the 30,000-row FlyRank starter dataset and a client-grouped holdout to test a Logistic Regression scoring approach against a transparent Week-4 baseline. On the held-out clients, the model measured Precision@50 of 0.54 compared with 0.28 for the baseline, with a test-set declining base rate of 0.511. The model therefore showed a measured improvement in ranking observed declining pages, although the starter data does not support a clean prospective forecasting claim. The output is a ranked, public-safe review queue with reason codes that can support human content-review decisions.

## 1. Problem framing

The practical decision is which content pages a content team should review first when it cannot review every page at once.

The unit of analysis is a content item/page. The output is a model score and ranked review queue. A human editor can use the ranking to decide which pages deserve closer inspection, while considering additional editorial and business context.

The target is the observed declining outcome:

`is_declining_label = 1` when `trend_direction == "down"`.

The goal is decision-support rather than prediction of Google's ranking system or a claim that refreshing a page will cause performance to improve.

A wrong call has an opportunity cost: a team may spend limited review time on a page that does not need attention while overlooking a page with stronger measured signs of decline. A ranking model is useful here because it can combine multiple measurable signals into one repeatable prioritization score.

## 2. Data safety

This analysis uses the FlyRank 30,000-row starter dataset, `content_refresh_anonymized.csv`. It contains one row per pseudonymized content item across 32 clients. The measurements are aggregated over a trailing 90-day window, with recent and previous 30-day comparison fields.

This project uses the starter slice rather than the full warehouse release. Therefore, its results are limited to this dataset and its available measurement window.

The following fields are deliberately excluded from model features:

- `content_id` — pseudonymous identifier; used only for grouping/joins and not as a feature.
- `client_id` — pseudonymous identifier; used for the grouped train/test split and not as a feature.
- `trend_direction` — source of the declining label and therefore direct label leakage.
- `trend_pct` — label-derived trend field and therefore excluded.
- `is_declining_label` — target, not a feature.
- `provider_used` — documented metadata that is not a model feature.
- `model_used` — documented metadata that is not a model feature.

Missing numeric values are handled using training-set medians in the current modeling notebook. This avoids treating missing values as genuine zero measurements.

The evaluation uses a grouped client split so that content from the same client does not appear in both training and testing. The observed split contains 25 training clients and 7 test clients, with zero client overlap.

No client names, domains, URLs, private queries, credentials, or raw client exports are included in this public report.

### Leakage and interpretation warning

The starter dataset's 30-day impression fields are closely connected to the construction of the observed declining label. Therefore, this experiment should be interpreted as an observed classification/ranking analysis, not as a clean prospective forecast.

A stronger production-style study would construct features from a completed historical window and predict a separate future outcome window using the daily warehouse.

## 3. Baseline

The baseline is the transparent Week-4 rule used for comparison.

It assigns visibility points from `impressions_90d`, one staleness point when `days_since_last_update` is between 91 and 180 days, and combines those components into a baseline score.

Pages are ranked by this score and the top 50 pages are evaluated.

On the same held-out test pages, the Week-4 baseline achieved **Precision@50 = 0.28**.

The test-set declining base rate was **0.511**.

The baseline provides a transparent reference for judging whether the learned ranking adds measured signal.

## 4. Model / analysis

I use Logistic Regression as the learned scoring model.

Logistic Regression was selected because it is relatively simple, reproducible, and interpretable. It produces a score for the observed declining outcome, which can naturally be used to rank pages.

The workflow is:

1. Create the observed binary target from `trend_direction`.
2. Split the data into train and test groups by `client_id`.
3. Fill missing numeric values using medians calculated from the training set.
4. Standardize numeric features.
5. Train Logistic Regression with random seed 42.
6. Generate the estimated probability of the observed declining outcome.
7. Rank held-out pages by this score.
8. Evaluate the top 50 using Precision@50.

The feature set contains measurable content, search, traffic, engagement, age, freshness, and position signals after excluding identifiers, label-derived fields, and documented metadata fields.

The target is:

> A page is labeled 1 when its documented `trend_direction` is `down`; otherwise it is labeled 0.

The model is a ranking and prioritization tool. Its coefficients describe model associations with the observed label; they are not causal effects.

## 5. Evaluation

### Split design

The evaluation uses one fixed client-grouped holdout:

- Train rows: **23,837**
- Test rows: **6,163**
- Training clients: **25**
- Test clients: **7**
- Client overlap: **0**
- Train declining rate: **0.550**
- Test declining rate: **0.511**
- Random seed: **42**

### Model vs baseline

Both methods are evaluated on exactly the same held-out pages.

| Method | Precision@50 |
|---|---:|
| Week-4 baseline | **0.28** |
| Logistic Regression | **0.54** |
| Test base rate | **0.511** |

The Logistic Regression model measured an improvement of **0.26 Precision@50 points** over the Week-4 baseline on this holdout.

At the top 50, this corresponds to approximately 14 declining pages for the baseline and 27 for Logistic Regression.

This is an observed ranking result, not proof that the model will always outperform the baseline or that refreshing recommended pages will improve performance.

### Error analysis

The model makes both false-positive and false-negative decisions. Some stable or flat pages receive high scores, while some genuinely declining pages receive low scores. This reinforces the need for human review.

The largest coefficient magnitudes in the current fit include:

- `content_age_days`
- `pageviews_90d`
- `sessions_90d`
- `days_since_last_update`
- `clicks_90d`
- `word_count`
- `impressions_90d`
- `avg_position`

These are model signals, not evidence that any feature causes decline.

## 6. Interpretation

The model found that content-age, traffic, freshness, and activity signals contributed strongly to its ranking of the observed declining outcome.

`content_age_days`, `pageviews_90d`, and `sessions_90d` had relatively large coefficient magnitudes. `days_since_last_update` also contributed meaningfully to the fitted score.

These relationships should be interpreted carefully because the features can be correlated and standardized. A coefficient describes the fitted model relationship; it does not establish a causal mechanism.

The useful finding is narrower:

> On this client-grouped holdout, a learned combination of available signals produced a substantially higher top-50 concentration of observed declining pages than the transparent Week-4 rule.

That is a measured decision-support result, not a causal or prospective claim.

## 7. Recommendation

The model output should be used as a ranked review queue rather than an automatic refresh queue.

### Action playbook

| Priority | Recommended action | Reason |
|---|---|---|
| **High priority** | Review first | Higher measured model score |
| **Review** | Assess for refresh | Moderate measured model signal |
| **Monitor** | Continue monitoring | Lower measured model signal |

Reason codes can include:

- `recent_impressions_below_previous`
- `older_update`
- `deep_position`
- `measurable_volume`

An editor can start with the highest-ranked pages, check whether the page has enough measurable activity, inspect freshness and performance context, review the content itself, and then decide whether to refresh, investigate, monitor, or take no action.

The model should not automatically trigger a rewrite or refresh.

Confidence in the current result is limited to this evaluation setup. The model outperformed the baseline on the current holdout, but the experiment uses the 30,000-row starter slice and an observed label whose measurement window overlaps available trend features.

## 8. Reproducibility

The main analysis is implemented in:

`work/notebooks/capstone.ipynb`

The notebook uses Python, pandas, NumPy, scikit-learn, Matplotlib, Logistic Regression, `GroupShuffleSplit`, and random seed **42**.

To reproduce the result:

1. Load the starter dataset.
2. Recreate the documented target.
3. Exclude identifiers and label-derived fields.
4. Create the client-grouped split with seed 42.
5. Train Logistic Regression.
6. Generate test-set scores.
7. Calculate Precision@50 for the model and baseline.
8. Generate the recommendation queue and paper artifacts.
9. Run the notebook from top to bottom and confirm the reported numbers.

If a fresh run changes the metrics, update this report and the deployed paper so the numbers match the fresh run.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
