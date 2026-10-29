# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Idowu Thomas Adegoke
- **Lane:** Refresh / Content Opportunity Scoring (a ranked review queue with reason codes)
- **Repo:** https://github.com/Thomasayanfeoluwa/flyrank-data-pipeline
- **Paper:** https://thomasayanfeoluwa.github.io/flyrank-data-pipeline/
- **Notebook:** `work/notebooks/capstone.ipynb` (this report mirrors it; every number below matches `work/outputs/capstone_summary.json`)
- **Date:** October 2026

## 0. Abstract

A content team with a few review hours a week needs to know which pages to open first when some quietly lose search impressions. I asked whether a readable model can rank pages so that the top of the list holds more pages that lost 30% or more of their impressions (last 30 days against the prior 30) than random picking does. I compared a hand-written rule, Logistic Regression and gradient boosting on an anonymized FlyRank snapshot of 21,758 pages from 30 clients, holding out whole clients in every validation fold. On unseen clients, Logistic Regression's top 500 pages were decaying 79.4% of the time against a 51.4% base rate (lift 1.55×, 95% interval 1.22 to 1.87), while a random split had inflated gradient boosting's equivalent figure from 0.71 to 0.94. Most of the measured skill came from 90-day activity numbers that overlap the label window, and precision moved with the population rule, so the queue is observed, directional decision support for human screening, not a forecast and not evidence that refreshing a page helps.

## 1. Problem framing

- **Decision supported:** which pages an editor opens first for review (refresh, expand, leave alone, or monitor).
- **Unit of analysis:** one content page.
- **Output:** a score for every page in the review population, a ranked queue, and per page a reason code, a confidence label (`higher` at 500 or more impressions in 90 days, otherwise `lower`) and a protect check (page one, at least 180 days old).
- **Human action:** an editor works the queue top-down, about 500 pages per cycle (assumed capacity), verifies the decline, then decides.
- **Cost of a wrong call:** a false positive costs about 15 minutes of review (an assumption). 103 of the top 500 did not decay, which is about 26 hours of that cycle's roughly 125. A false negative leaves a decaying page unreviewed: 4,436 of the 11,174 decayed pages (40%) ranked in the bottom half.
- **Why ML helps:** the hand-written rule finds decaying pages only in a narrow group (7.0% of pages flagged). A model can rank all pages. Its gain over the rule is modest and uncertain (section 5).
- **Link to FlyRank's work:** FlyRank's March 2026 report, *The State of AI-Driven SEO*, advises refreshing strong content before it decays. This work asks which pages to look at first. It does not test the report's refresh findings, which are observational cohort comparisons.

## 2. Data safety

- **Data:** the FlyRank ML Internship starter release (`content_refresh_anonymized.csv`): 30,000 pages, 44 columns, 32 clients, one snapshot, hashed identifiers.
- **Population:** 26,612 pages have a prior-30-day baseline. The review population is the 21,758 of those with at least 100 impressions in 90 days (30 clients).
- **Excluded columns:** `trend_direction`, `trend_pct` and the label itself; every `_last_30d` and `_prev_30d` window column (they define or sit inside the label); `content_id` and `client_id` (client_id is used only to group folds). 34 features remain.
- **Leakage checks:**
  1. The label and window columns are excluded, with assertions in the code.
  2. A deliberate leak (adding the two label-window columns) lifted gradient boosting's AUC from 0.667 to 0.999 and P@500 from 0.710 to 1.00; removing them restored 0.667 and 0.710.
  3. The 90-day totals still overlap the label windows (the two windows hold a median of 56% of each page's 90-day impressions). Section 6 measures what that costs.
- **Public safety:** no client names, domains, URLs, titles, keywords or private queries appear in `work/`, the paper or the figures. No dataset files are committed. The ranked queue file is not committed.

## 3. Baseline

- **Rule (Week-4, no training):** flag pages with 100 to 499 impressions in 90 days that have not been updated for 91 days or more, scored as `min(days_since_update, 365)/365 × (500 − impressions)/400`. All other pages score 0. It flags 1,531 pages (7.0%).
- **Why it is a fair comparison:** it is scored on the same 21,758 pages, the same five client-grouped folds and the same metric as the models. Its cutoffs were chosen after looking at two signal tables on this same data, so it is somewhat optimistic and the comparison is conservative toward the models.
- **Its numbers:** P@500 0.618, lift 1.20× (0.92 to 1.35), AUC 0.511 against a 0.514 base rate.

## 4. Model / analysis

- **Label (proxy):** a page is *decaying* if its impressions in the last 30 days fell 30% or more against the prior 30 days. The cutoff is an assumption, not tuned, and is stricter than FlyRank's report, which calls a page "down" at a 10% decline.
- **Models:** Logistic Regression (one-hot categories, scaled numbers) and `HistGradientBoostingClassifier`, with fixed settings and seed 42.
- **Features:** every column except the excluded ones in section 2, in six families (visibility consistency, volume, position and CTR, on-page engagement, page profile, keyword and source). The exact list is `FEATURES` in the notebook's setup cell.
- **Queue:** the final queue uses the Logistic Regression score from client-grouped out-of-fold scoring.

## 5. Evaluation

- **Split:** five-fold cross-validation grouped by client (zero shared clients between train and test). A time-aware split is not possible with one snapshot. The folds are uneven; fold 1 is a single client.
- **Metrics (pooled out-of-fold, base rate 0.514):**

| Method | P@500 | Lift@500 (95% interval) | AUC |
|---|---|---|---|
| Baseline rule | 0.618 | 1.20× (0.92 to 1.35) | 0.511 |
| Logistic Regression | 0.794 | 1.55× (1.22 to 1.87) | 0.638 |
| Gradient boosting | 0.710 | 1.38× (1.04 to 1.74) | 0.667 |

- **Paired differences in lift@500:** Logistic Regression minus rule +0.35 (0.00 to 0.75); gradient boosting minus rule +0.21 (−0.14 to 0.59); Logistic Regression minus gradient boosting +0.15 (0.01 to 0.29).
- **Random versus grouped split (P@500):** Logistic Regression 0.862 then 0.794; gradient boosting 0.936 then 0.710.
- **Error analysis:** 103 of the Logistic Regression's top 500 did not decay (45 grew, 40 were stable, 18 fell a median of 25%). Missed decaying pages are larger (median 2,530 impressions against 1,728) and rank lower (average position 15.7 against 12.2).

## 6. Interpretation

- Logistic Regression's top 500 hold more decaying pages than random (0.794 against 0.514) on unseen clients. Its advantage over the rule is directional, not settled. Gradient boosting cannot be told apart from the rule.
- **Inputs overlap the label.** Using only the 17 page-attribute features, P@500 falls to 0.574 (Logistic Regression) and 0.484 (gradient boosting) against a 0.514 base rate. Part of what the models find is a decline that had already started. This work does not show that a model can anticipate decay.
- **Population sensitivity.** Using prior-30-day impressions of at least 100 instead of 90-day impressions moved P@500 by 0.11 (Logistic Regression) and 0.21 (gradient boosting), while AUC moved by about 0.01 to 0.02. Precision figures are tied to the population rule and are directional.
- **Reason codes in the top 2,000** (decay rate against a 0.514 base rate): `weak_position_or_ctr` 0.876, `visibility_gaps` 0.776, `keyword_or_source` 0.71, `weak_engagement` 0.69, `page_age_or_length` 0.64. Differences between codes were not tested, and a reason code describes what the model reacts to, not a cause.
- **Negative and mixed results kept:** the random split was optimistic; the attribute-only models are near chance; the queue favors small pages (308 of the top 500 have under 500 impressions in 90 days) and holds only 0.4% of click-equivalent value (clicks × CPC, a proxy and not revenue).

## 7. Recommendation

1. Use the queue as a screening list, top-down, about 500 pages per cycle.
2. Verify the decline before any action (103 of the top 500 did not decay).
3. Use the reason code to decide what to check on a page, not which page to open next.
4. Protect healthy page-one pages (the 4,898 page-one pages at least 180 days old decayed at 0.458).
5. Check value before spending the time.
6. Treat `lower`-confidence pages with extra care.
7. Keep publishing, rewriting, pruning, merging, redirecting and client promises manual.
8. Re-score monthly. Re-audit if confirmed-decay precision falls below 0.62 in two batches, a batch's base rate leaves 0.38 to 0.72, reviewers reject more than 41% of top pages, a new client is added, or the label changes.

**Confidence and limits:** observed on one snapshot of 30 clients. Directional decision support only. No evidence about interventions or how Google ranks pages. The intervals reflect only which clients are in the data, not the choice of label, cutoff or population rule.

## 8. Reproducibility

- **From a fresh clone:** open `work/notebooks/capstone.ipynb` in Colab (badge at the top of the notebook) and choose Runtime → Run all. The setup cell clones the public course starter repo and copies its `data/` folder.
- **Seed:** 42. **Environment:** scikit-learn 1.6.1, pandas 2.2.3.
- **Receipts committed:** `work/outputs/capstone_summary.json` (headline numbers, no ids, no data) and `work/figures/` (the four paper figures). The ranked queue file is regenerated by the notebook and is not committed.
- **No sealed or holdout evaluation is claimed.**
- **Small numerical differences between runs** can move a page or two in the top 500 (for example 397 against 398 of 500).

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset, from [FlyRank](https://flyrank.ai). Thank you to the FlyRank team for the data and the programme.

---

> Claims language: observed / measured / directional / decision-support. Base rate (0.514) is reported next to every precision figure. No causal claims. No client-identifying details. Numbers match `work/outputs/capstone_summary.json`.
