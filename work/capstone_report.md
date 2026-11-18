# Capstone Report: Refresh / Content Opportunity Scoring

- **Author:** Azizul Abedin Azmi
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/azizulabedinazmi/FlyRank-ML-Internship-Starter
- **Paper:** https://azizulabedinazmi.is-a.dev/flyrank-paper.html
- **Date:** November 2026

## 0. Abstract

Can a ranked queue help an editor find pages that are probably losing ground, before anyone reviews the whole inventory by hand? I used the FlyRank anonymized starter release: 30,000 content items from 32 pseudonymized clients, with trailing-90-day activity and two back-to-back 30-day comparison windows. I compared a simple "stale but still visible" rule against logistic regression, a shallow decision tree and a random forest, testing all four on the same set of clients the models never saw in training. The random forest got ROC AUC 0.747 and Precision@50 0.680, against 0.627 and 0.240 for the rule. The output is a ranked review queue with reason codes for a human editor. It does not publish anything on its own and it says nothing about what causes search performance.

## 1. Problem framing

A content team usually has more existing pages than it can review in one cycle, so the order of review matters. The question I worked on is narrow: which pages should an editor open first when looking for refresh candidates, and which can wait?

- **Unit:** one content item.
- **Output:** a 0–100 priority score, a suggested action (refresh, review CTR, review engagement, expand and refresh, or monitor) and the reason codes behind it.
- **Human action:** the editor opens the top of the queue, checks the page, and decides whether a refresh is worth it.
- **Cost of a wrong call:** a false positive wastes an editor's time on a page that was fine. A false negative means a declining page waits another cycle. Neither is a disaster, which is why a ranking that a person checks is a better fit here than an automatic decision.

ML helps because the obvious rule ("old and still getting impressions") ignores most of what is in the data: position, engagement, how many days the page actually showed up, content length. A model can weigh those together. The comparison below tests whether that pays off.

## 2. Data safety

**Used:** `data/raw/content_refresh_anonymized.csv`, 30,000 rows × 44 columns, 32 pseudonymized clients. It's a trailing-90-day snapshot with two 30-day comparison windows, not a daily warehouse extract.

**Not used:** the gated 79M-row Hugging Face warehouse. The Refresh lane is documented as "starter playground plus warehouse support", and I kept every claim inside the starter slice instead of borrowing numbers from data I didn't evaluate.

**Excluded from features, and why:**

| Excluded | Reason |
|---|---|
| `trend_direction`, `trend_pct` | The label is built from these. Using them would be answering the question with the answer. |
| `content_id`, `client_id` | Pseudonymous IDs. `client_id` is used only to group the train/test split, never as a feature. |
| Provider / model identifiers | Not a property of the content. |
| Titles, URLs, domains, raw queries | Could identify a client. Not loaded into features or shown anywhere. |

**Leakage risk I can't fully remove:** the 90-day traffic totals overlap the two 30-day windows the label compares. A page that lost traffic in the last month will also tend to have different 90-day totals. Features like `days_with_impressions` therefore carry some of the same signal as the label. Dropping all 90-day totals would leave almost nothing to model on, so I kept them and treat the result as "this ranking agrees with observed decline", not "this ranking forecasts future decline". A real forecast needs the warehouse daily facts, with features taken strictly before the label window (see section 8).

Nothing in `work/` names a client, a domain, a URL or a query.

## 3. Baseline

The rule: flag content where `days_since_last_update >= 180` **and** `impressions_90d >= 500`, then rank by visibility. It's what an editor would do without a model: old pages that still get seen.

It's a fair comparison because it runs on the same client-holdout test set (2,325 rows) and is scored with the same metrics.

| Metric | Rule baseline |
|---|---:|
| ROC AUC | 0.627 |
| Average precision | 0.468 |
| Precision@50 | 0.240 |
| Precision@100 | 0.360 |

A Precision@50 of 0.240 is well below the 54.2% base rate. In other words, "old and visible" by itself points the editor at the wrong pages. Plenty of old pages are stable.

## 4. Model

**Target:** `is_declining_label = 1` when the item's trend between the two 30-day windows is "down", otherwise 0. That gives 16,262 positives out of 30,000 (54.2%).

**Candidates:** logistic regression, a shallow decision tree, and a class-balanced random forest (200 trees, max depth 10, min leaf 25, `random_state=42`). Selection metric: Precision@50, because the real use is "how good is the first batch an editor looks at".

**Features (52 after encoding):** log-transformed impressions, clicks and sessions; days with impressions and days with sessions; average position and position tiers; CTR; engagement and scroll rate; content age, days since last update and age tiers; word and character count; search volume and keyword context; content type and a few categorical tiers.

## 5. Evaluation

**Split:** a deterministic 20% client holdout, giving 27,675 training rows and 2,325 test rows. Splitting by client means the test pages come from sites the model never saw. That's closer to real use than a random row split, where pages from the same site leak patterns into both sides.

| Method | ROC AUC | Avg precision | Precision@50 | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Rule baseline | 0.627 | 0.468 | 0.240 | — | — |
| Logistic regression | 0.700 | 0.522 | 0.400 | 0.567 | 0.566 |
| Decision tree | 0.742 | 0.575 | 0.620 | 0.716 | 0.634 |
| **Random forest** | **0.747** | **0.610** | **0.680** | 0.741 | 0.638 |

**Read it next to the base rate.** 54.2% of items are positive, so picking 50 at random would get about 27 right. The forest gets 34. The rule gets 12. The honest summary is that the model is clearly better than the rule, and moderately better than chance. AUC (0.747 vs 0.627) is the cleaner measure of how well each one separates the two groups.

**Where it goes wrong:** the test set is only a handful of clients, so one unusual client can move Precision@50 a lot. It's 50 items, and each one is worth 2 points. Pages with very little traffic are the hardest: a "down" trend on 30 impressions is mostly noise, and the model has little to go on. That's why the final queue needs both a high model score and real visibility before it marks something high-confidence.

## 6. Interpretation

The strongest features were `days_with_impressions` (0.161), `log_impressions_90d` (0.128), `avg_position` (0.108) and `content_age_days` (0.095). Length, CTR, clicks and scroll rate follow well behind.

In plain words, the model mostly looks at how consistently a page shows up in search, how much it shows up, where it ranks, and how old it is. These are associations inside this dataset. They are not levers. Nothing here says that changing a page's age or position would change its trend.

Two things surprised me:

- **Days since last update barely mattered** (importance 0.021), even though it's half of the baseline rule. Staleness alone doesn't predict decline in this data.
- **The shallow decision tree came close to the forest** (P@50 0.620 vs 0.680). If a team wants something they can read on one page, the tree costs little in accuracy.

## 7. Recommendation

The final queue scores all 30,000 items:

| Suggested action | Items |
|---|---:|
| Monitor | 13,069 |
| Refresh | 8,207 |
| Refresh + review CTR | 6,655 |
| Refresh + review engagement | 1,987 |
| Expand + refresh | 82 |

3,576 items are high-confidence. To get there, an item needs a score above the 80th percentile, at least 500 impressions, at least 10 sessions, and a model probability of 0.50 or more.

How an editor would use it tomorrow:

1. **Start with high-confidence, visible pages.** Check search intent, whether the facts are out of date, and whether there's a clear update to make. If there isn't, skip it.
2. **Treat a CTR flag as a question, not an instruction.** Look at the title, snippet and intent before rewriting anything.
3. **Keep engagement problems separate from search problems.** A page people find but leave quickly needs different work from a page nobody finds.
4. **Don't review low-confidence items.** Put them on a watch list and re-score next cycle.
5. **Log what you change and when.** Without a record of interventions and comparable follow-up windows, nobody can tell later whether refreshing helped.

**Confidence:** moderate. The ranking is a reasonable way to order a review queue for this data. It isn't evidence that refreshing pages raises traffic, and it hasn't been tested on new sites or later time periods.

## 8. Reproducibility

From a fresh clone:

```bash
pip install -r requirements.txt
python scripts/run_all.py
```

On Windows, set UTF-8 output first or the step banners fail to print:

```bash
set PYTHONUTF8=1
```

Then open `work/notebooks/capstone.ipynb` and run all cells. It reads `outputs/model_results.json` and `outputs/summary.json`, so its tables match the paper exactly.

- **Seed:** 42 for the split and every model.
- **Environment:** Python 3.12, versions pinned in `requirements.txt`.
- **Check:** I re-ran the full pipeline on 2026-11-18 and got the same metrics as the committed receipts (random forest P@50 0.680, baseline 0.240, 3,576 high-confidence rows).
- **Receipts:** `outputs/model_results.json`, `outputs/summary.json`, `outputs/model_report.md`, `work/outputs/baseline_action_score_metrics.json`.
- **Weekly notebooks:** `work/notebooks/w01`–`w07`, one per assignment.

No sealed evaluation is claimed. The next honest step would be one: rebuild the label from warehouse daily facts, take every feature from before the label window, and score a later time window exactly once.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). Thanks to FlyRank for the anonymized educational release and the internship workflow this analysis follows.

---

**Claims check:** observed, measured, directional, decision-support language throughout. No causal claims. No claims about Google's algorithm. No client-identifying details. Every number matches a fresh re-run.
