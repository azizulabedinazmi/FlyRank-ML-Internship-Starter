# Finding Refresh Opportunities in Anonymized Search Content

- **Author:** Azizul Abedin Azmi
- **Lane:** Content refresh prioritization
- **Repo:** [FlyRank-ML-Internship-Starter](https://github.com/azizulabedinazmi/FlyRank-ML-Internship-Starter)
- **Date:** 2026-11-11

## Abstract

Can a transparent content-refresh queue help an editor find likely declining pages before reviewing the whole inventory? Using the bundled FlyRank anonymized starter release, I evaluated 30,000 content items from 32 pseudonymized clients with trailing-90-day activity and two consecutive 30-day comparison windows. I compared a hand-written visibility-and-staleness rule with logistic regression, a shallow decision tree, and a random forest using a client holdout and leakage-aware features. The random forest reached ROC AUC 0.747 and Precision@50 0.680 on the same holdout where the rule reached ROC AUC 0.627 and Precision@50 0.240. The result is a reviewer-aid and ranked action queue, not an automatic publishing decision or a causal claim about search performance.

## 1. Introduction / problem statement

Editors often have more existing pages than they can manually inspect. The decision supported here is: **which content items should be reviewed first for a possible refresh, and which should be monitored instead?** The unit of analysis is one anonymized content item. A wrong positive costs editorial time; a wrong negative can delay a useful refresh. A ranked model is useful only if it improves the first few review slots over a rule that an editor can understand.

## 2. Data

The final evaluation uses the repository's bundled `content_refresh_anonymized.csv` starter release: 30,000 rows, 44 source columns, one row per pseudonymized content item, and 32 pseudonymized clients. The snapshot contains trailing-90-day activity totals and 30-day-versus-previous-30-day comparisons; it is not a daily warehouse extract. The gated FlyRank warehouse release (including its daily performance and query tables) was **not** scanned for this paper, so no 79M-row claim is made here.

The label is `is_declining_label = 1` when the source trend direction is `down`, producing 16,262 positive rows (54.2%). I excluded `trend_direction` and `trend_pct` from features because they define the label. I also excluded pseudonymous IDs, provider/model identifiers, titles, URLs, domains, and raw queries; IDs are used only to group the validation split. Numeric missing values are imputed after feature selection and categorical missingness is represented as `unknown`.

## 3. Methodology

The baseline ranks items with a transparent rule: stale content (`days_since_last_update >= 180`) with meaningful visibility (`impressions_90d >= 500`). The learned candidates use the same prepared frame and include traffic volume transforms, content age and freshness, position and rate measures, engagement measures, keyword-context fields, and selected categorical tiers. The best model is a class-balanced random forest with 200 trees, maximum depth 10, minimum leaf size 25, and seed 42.

The split is a deterministic client holdout: 20% of pseudonymized clients are held out, with 27,675 training rows and 2,325 test rows. Model selection uses Precision@50, matching the operational question of choosing a small first-review batch. Leakage checks explicitly removed label-derived trend fields and pseudonymous identifiers; the holdout is grouped by client rather than by randomly mixing rows from the same client across train and test.

## 4. Results

All reported model and baseline metrics use the same client-holdout test set. The positive-class base rate is 54.2%, so top-of-queue precision is reported with that context.

| Method | ROC AUC | Average precision | Precision@50 | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Rule baseline | 0.627 | 0.468 | 0.240 | — | — |
| Logistic regression | 0.700 | 0.522 | 0.400 | 0.567 | 0.566 |
| Decision tree | 0.742 | 0.575 | 0.620 | 0.716 | 0.634 |
| **Random forest** | **0.747** | **0.610** | **0.680** | **0.741** | **0.638** |

The model's Precision@50 is 0.440 higher than the rule baseline on this split (about 2.8x the baseline precision). The strongest feature importances were `days_with_impressions`, `log_impressions_90d`, `avg_position`, and `content_age_days`; these are associations used for prioritization, not evidence that changing a feature will cause a ranking change.

### What the charts say

The final queue contains 3,576 high-confidence, 11,424 medium-confidence, and 15,000 low-confidence items. Suggested actions are `monitor` (13,069), `refresh` (8,207), `refresh_and_review_ctr` (6,655), `refresh_and_review_engagement` (1,987), and `expand_and_refresh` (82). The operational pattern is therefore a broad triage queue with a smaller, evidence-rich review set rather than a single automatic action.

## 5. Limitations and honest framing

- This is a retrospective, anonymized starter slice, not a randomized intervention. The paper cannot claim that refreshing a page causes more traffic.
- The target is a directional decline label derived from observed trend windows; it is not a human-verified content-quality label.
- Client holdout improves separation from the training clients, but the sample is still one release and may not represent future sites or seasons.
- The model scores review priority, not the expected lift from a particular edit. Editorial context, intent, seasonality, and page quality still require human review.
- The final evaluation did not use the gated 79M-row warehouse release. Extending this result to that release requires a separately documented, sealed evaluation.

## 6. Ranked recommendations

1. **Review the highest-scoring visible pages first.** Start with the high-confidence queue and verify whether the page still matches search intent, has current evidence, and has a realistic update path.
2. **Use CTR review as a diagnostic, not an instruction.** Pages with visibility and low CTR are candidates for title/snippet and intent review; do not rewrite automatically.
3. **Separate engagement review from search review.** Where sessions exist but engagement is weak, inspect content structure and relevance before changing targeting.
4. **Monitor low-confidence items.** Keep them in a watchlist rather than spending equal editorial effort on every scored item.
5. **Log outcomes for a future prospective test.** Record the intervention, date, and comparable follow-up window so later work can test whether the queue improves decisions.

## 7. Reproducibility

From a fresh clone with the bundled starter CSV:

```text
pip install -r requirements.txt
python scripts/run_all.py
```

The pipeline uses seed 42 and writes the prepared frame, baseline, model results, queue, charts, and PDF to `outputs/`. The filled capstone notebook is [`work/notebooks/capstone.ipynb`](notebooks/capstone.ipynb). The model receipt is [`outputs/model_results.json`](../outputs/model_results.json), and the executable pipeline is [`scripts/run_all.py`](../scripts/run_all.py).

## 8. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). Thanks to FlyRank for providing an anonymized educational release and the internship workflow used to frame, validate, and communicate this analysis.
