# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Syed Ahmed
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/syedhahmed1/Starter_Repo
- **Date:** August 2026

## 1. Problem framing

Content teams can't manually review every page on a site to decide what needs a refresh, so
they need a way to prioritize. This project supports that decision: **given a page's traffic
history, staleness, and position trend, which pages are the best candidates to prioritize for
content refresh this month?**

The unit of analysis is a single page. The output is a ranked refresh-opportunity score per
page (0-100), along with a reason code and a suggested action (e.g. `refresh`,
`refresh_and_review_ctr`, `expand_and_refresh`, `monitor`). The action a human takes from it is
straightforward: a content editor works down the ranked queue, starting with the highest-scored
pages, and either refreshes the content, reviews it for a specific issue (low click-through
rate, low engagement), or expands it if it's thin.

The cost of a wrong call is asymmetric: missing a page that genuinely needs a refresh means
lost traffic that could have been recovered; refreshing a page that didn't need it costs editor
time but little else. That asymmetry is why a ranked queue (prioritize the top candidates) is
more useful than a hard yes/no cutoff.

Data/ML helps here because "is this page declining" isn't obvious from any single metric — it's
a combination of traffic volume, position, staleness, and trend, and the relationships between
those signals are exactly what a model can learn that a single hand-written threshold cannot.

## 2. Data safety

**Data used:** `data/raw/content_refresh_anonymized.csv` — a 30,000-row anonymized teaching
slice of the FlyRank warehouse, one row per page, metrics aggregated over a trailing 90-day
window, covering 32 pseudonymized clients.

**Columns deliberately excluded, and why:**
- `trend_direction` / `trend_pct` — these are the source of the label itself
  (`is_declining_label` is derived directly from `trend_direction == "down"`). Using either as
  a model feature would be leakage: the model would essentially be given the answer.
- `content_id` / `client_id` — pseudonymous identifiers, used only to build a **client-holdout**
  train/test split (so the model is never tested on a client it trained on), never used as
  model inputs.
- `provider_used` / `model_used` — which AI tool generated the article isn't a defensible causal
  signal for real-world search performance, so it was left out of the feature set.

**Leakage risks considered:** the label-derived fields above were the primary leakage risk and
were confirmed absent from both the numeric and categorical feature lists actually used to train
the model. No client names, URLs, or private query data appear anywhere in this repo — the
source CSV is already anonymized before this project touches it.

## 3. Baseline

The baseline is a transparent, hand-written rule — no fitted weights, every term readable:

```
baseline_refresh_score = 0.40 * visibility_score        (log impressions, percentile-ranked)
                        + 0.30 * freshness_risk_score    (days since last update, percentile-ranked)
                        + 0.25 * position_opportunity_score  (good position + visible)
                        + 0.05 * depth_gap_score          (thin content + visible)
```

This is a fair comparison because it's evaluated on the exact same held-out test rows and the
exact same metric (precision@K) as the trained model — same data, same yardstick, no home-field
advantage for either side.

**Baseline numbers on the held-out test split:**

| Metric | Baseline rule |
|---|---:|
| Precision@20 | 0.150 |
| Precision@50 | 0.240 |
| Precision@100 | 0.360 |
| ROC AUC | 0.627 |

The base rate (share of pages actually declining) is **0.542** — meaning the baseline rule's
precision@20 (0.150) is actually *worse* than randomly guessing at the top of the list, and its
precision@50 (0.240) is well below the base rate too. This is an honest, useful result: the
hand-written rule is not simply "correct by construction," and beating it is a real bar, not a
formality.

## 4. Model / analysis

**Method:** a random forest classifier (200 trees, max depth 10, `class_weight="balanced_subsample"`),
chosen after comparing against logistic regression and a decision tree. Random forest fits this
lane because the relationship between traffic/position/staleness signals and decline is
non-linear and involves interactions (e.g. "stale AND visible" matters more than either alone) —
exactly what tree-based ensembles capture better than a linear model.

**Feature list (52 total: numeric + one-hot categorical):**
- Numeric: `search_volume`, `competition`, `cpc`, `word_count`, `char_count`,
  `log_impressions_90d`, `log_clicks_90d`, `log_sessions_90d`, `log_ai_sessions_90d`,
  `days_with_impressions`, `days_with_sessions`, `content_age_days`, `days_since_last_update`,
  `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`
- Categorical (one-hot): `competition_level`, `content_type`, `main_intent`, `age_tier`,
  `freshness_tier`, `word_count_tier`, `impression_tier`, `position_tier`

Left out on purpose: `trend_direction`, `trend_pct`, `content_id`, `client_id`, `provider_used`,
`model_used` (see Data safety above).

**Target/proxy definition, in one sentence:** `is_declining_label` = 1 when a page's impressions
dropped more than 20% over the last 30 days compared to the prior 30 days (`trend_direction ==
"down"`), else 0 — a measurable proxy for "this page is losing search visibility," not a direct
editorial judgment call.

## 5. Evaluation

**Split:** client-holdout — 20% of the 32 clients (by `client_id`) were held out entirely, so
every test-set page comes from a client the model never saw during training. This is a
time-agnostic but *client-aware* split, chosen because the realistic deployment scenario is
scoring pages for clients/sites the model may not have deep history on, not just unseen pages
from already-familiar clients. Result: 27,675 training rows, 2,325 test rows.

**Metrics, model vs. baseline, same split:**

| Metric | Baseline rule | Random forest |
|---|---:|---:|
| Precision@20 | 0.150 | 0.700 |
| Precision@50 | 0.240 | 0.680 |
| Precision@100 | 0.360 | 0.700 |
| ROC AUC | 0.627 | 0.747 |

The model roughly **triples** the baseline's precision@50 (0.24 → 0.68) and comfortably clears
the 0.542 base rate at every K — the baseline alone does not.

**Error analysis:** the model's precision@50 (0.68) means 16 of the top 50 flagged pages were
not actually declining by the label's definition. Manually spot-checking the top-10 queue,
every one of the top 10 rows was flagged for a page with `trend_direction == "down"`, high
impression volume, and either a below-average CTR or below-average engagement rate — consistent
with the intended pattern, not an obviously wrong pick. A well-understood miss here (page flagged
as an opportunity but not actually declining) typically had high traffic and a stable-to-good
position, suggesting the model sometimes over-weights raw visibility over trend stability — a
reasonable, explainable failure mode rather than a random one.

## 6. Interpretation

**Top model features, in plain words:**
1. `days_with_impressions` (0.160) — how consistently a page shows up in search over the window matters more than any single traffic number
2. `log_impressions_90d` (0.128) — raw visibility, log-scaled because traffic is heavy-tailed
3. `avg_position` (0.108) — where the page ranks
4. `content_age_days` (0.095) — older pages are more likely candidates
5. `word_count` / `char_count` (0.041 / 0.040) — thin content shows up as a real signal

**What this means in plain words:** the model is mostly rewarding pages that are *consistently
visible* (not just occasionally spiking) and *aging*, over pages that are simply large-traffic
or well-ranked in isolation. That matches editorial intuition: a page quietly losing steady
traffic over time is a better refresh candidate than a page with one good week.

**Surprises / negative results:** the baseline rule's precision@20 (0.150) being *below* the
base rate (0.542) was a genuine negative result worth stating plainly — a well-explained "this
simple rule doesn't work well at the very top of the list" is more useful than hiding it, since
it's exactly what justifies building the model in the first place.

## 7. Recommendation

**The ranked action queue (30,000 pages scored):**

| Action | Count |
|---|---:|
| `monitor` | 13,063 |
| `refresh` | 8,211 |
| `refresh_and_review_ctr` | 6,657 |
| `refresh_and_review_engagement` | 1,987 |
| `expand_and_refresh` | 82 |

**How an editor would use this tomorrow:** start at the top of the ranked queue (highest
`final_refresh_score`) and work down. For pages tagged `refresh_and_review_ctr`, check the
title/meta description alongside refreshing content, since low click-through despite decent
position suggests a snippet problem, not just a content-freshness problem. For
`expand_and_refresh` pages, the issue is thinness (short content), not just staleness — these
need net-new content, not just a touch-up. `monitor` pages need no action now.

**Confidence and limits, stated explicitly:** treat this as a reviewer aid, not an automatic
publishing decision. The top of the queue (precision@50 = 0.68) is meaningfully better than
guesswork, but roughly 1 in 3 flagged pages in the top 50 will not actually be declining by this
label's definition — a human should still sanity-check before committing real editorial time,
especially outside the top 20 where precision is likely lower still.

## 8. Reproducibility

**To re-run everything from a fresh clone:**

```bash
cd flyrank-ml-internship-starter
pip install -r requirements.txt   # pandas, numpy, scikit-learn
cd scripts
python3 run_all.py
```

This regenerates, in order: the prepared feature vector (`data/processed/refresh_feature_vector.csv`),
the baseline queue (`data/processed/baseline_refresh_queue.csv`), the trained models and metrics
(`outputs/model_results.json`), and the final ranked queue + charts (`outputs/refresh_queue.csv`,
`outputs/charts/*.svg`). The capstone notebook (`work/notebooks/capstone.ipynb`) then loads these
same output files directly, so notebook and report always stay consistent with each other.

**Random seed:** `RANDOM_STATE = 42`, fixed in `scripts/03_train_model.py` for both the
train/test client split and the model training itself, so results are exactly reproducible on
the same input file.

**Environment:** Python 3.14, `pandas`, `numpy`, `scikit-learn` (see `requirements.txt` for
exact versions).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language used throughout — no causal claims without an experiment, no claim of predicting
> Google's algorithm, no client-identifying details, and the numbers above match a fresh re-run
> of `scripts/run_all.py` on the bundled anonymized dataset.
> **Metrics vs. base rate:** base rate (0.542) is reported directly alongside precision@K
> throughout Sections 3-5 — the baseline's precision@20 (0.150) being *below* base rate, and the
> model's precision@50 (0.68) being well above it, are both stated plainly rather than left to
> look impressive out of context.
