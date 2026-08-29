Capstone Report — CTR / Engagement Opportunity Scoring
Author: Maher Hasni
Lane: CTR / Engagement Opportunity Scoring
Repo: https://github.com/UnsoundMouse/flyrankaiw01_research_question
Date: 2026-08-29
0. Abstract

Which pages already rank well in search but under-capture the clicks their position should earn, and can that gap be ranked well enough to prioritize a content editor's limited review time? Using a 90-day, 30,000-page starter slice across 32 clients, a position-tier-adjusted CTR gap as a proxy target, and a client-grouped validation split, a Logistic Regression model ranked genuine underperformers roughly 1.7x more precisely than a position-only baseline (precision@20: 0.85 vs. 0.50, base rate 0.44) — while a more complex Random Forest model did not beat that same baseline. The output is a five-action, human-reviewed content playbook for FlyRank editors, not an automated editing system.

1. Problem framing

Decision supported: which page a content editor reviews and potentially rewrites (title/ meta) next, out of a portfolio too large to review by hand. Unit of analysis: one page, for one client, at a 90-day snapshot. Output: a ranked queue with a model score, a reason code, and an action label. Action a human takes: an editor works down the queue, either rewriting a page's title/meta, scheduling it for a content refresh, or leaving it alone. Cost of a wrong call: a false positive costs an editor's hour on a page that was already fine; a false negative leaves real, already-earned search traffic unclaimed on a page that has the hard part — ranking, demand — already solved, with nobody flagged to look. Why data/ML helps at all: a single fixed CTR threshold cannot work, because "good CTR" depends heavily on position — mean CTR ranges from 2.76% (top 3) down to 0.15% (deep) in this data — so any useful rule has to be tier-relative, and a model can combine several weak, tier-relative signals (engagement, freshness, content type) more effectively than a hand-tuned rule can.

2. Data safety

Data used: the FlyRank ML Internship starter release, content_refresh_anonymized.csv — a single 90-day snapshot, 30,000 rows, 32 pseudonymous clients. The full warehouse (79M rows, 104 clients, 17 months, Hugging Face-gated) was used in Week 3 for data-contract practice, but every model and figure in the final paper uses the starter slice, for direct comparability across every weekly stage of the build.

Deliberately excluded:

client_id / content_id — pseudonymous grouping keys only, never model features.
trend_pct / trend_direction — label-trap columns, computed downstream of the same signal any outcome would be validated against; excluded from every model regardless of task.
ctr, clicks_90d, tier_median_ctr, ctr_gap — the label itself and everything it's built from; including any of these as a feature would let a model trivially reconstruct the label rather than predict it independently.
Rows with avg_position == 0 — this value means "no position data," not rank zero, and silently files under the top_3 tier label if not filtered out first.

Leakage risk actually found and fixed: a first-pass model included sessions_90d (a GA4 session count) and produced a suspiciously high score. Permutation importance flagged it as the dominant feature; its correlation with clicks_90d (the label's numerator) was 0.83 — a near-duplicate measurement of the same underlying visits, not an independent signal. Removed; every downstream result is the honest, post-removal number.

Confirmed: no client names, domains, URLs, or private query text appear anywhere in work/ or the deployed paper.

3. Baseline

The rule: rank eligible pages (impressions_90d >= 500, avg_position <= 20) by avg_position alone (best position first). This is a deliberately fair reduction of the original Week 4 tier-median rule down to the one signal it can offer without touching CTR — since CTR is exactly what the label is built from, a baseline that used CTR directly would trivially "win" by definition, not by being useful.

Numbers, same data and metric as the model: precision@20 = 0.50, precision@50 = 0.48, on the same held-out, client-grouped test split as the model (n=824, base rate 0.439).

4. Model / analysis

Method: Logistic Regression, chosen first because the task ("is this page's CTR below its own tier's median?") is a yes/no question with a label observable from current data — the textbook case for starting with a readable linear model. Random Forest was also trained, to check whether nonlinear complexity earns its keep rather than assuming it would.

Feature list: avg_position, word_count, content_age_days, days_since_last_update, engagement_rate, scroll_rate, ai_traffic_pct, content_type, main_intent, freshness_tier. Left out on purpose: ctr, clicks_90d, tier_median_ctr, ctr_gap (label-derived), sessions_90d (near-duplicate of the label's numerator, confirmed via correlation), trend_pct/trend_direction (label-trap columns per the data dictionary, excluded regardless of task).

Target/proxy, in one sentence: underperforms = 1 if a page's CTR sits below the median CTR of other visible pages sharing its position tier, else 0 — a same-window proxy for "opportunity," not an observed future outcome.

5. Evaluation

Split: GroupShuffleSplit by client_id, 75/25, confirmed zero client overlap between train and test — chosen because pages from the same client could share confounders (a CMS, an editorial voice) that would let a model quietly memorize client identity rather than learn a generalizable pattern. A same-task comparison against a plain random split showed only a small gap here (precision@20: 0.86 random vs. 0.85 grouped) — this task's signal turned out to live mostly at the page level, not the client level, so the risk was real to check but modest in this instance.

Metrics, model vs. baseline, same split:

Method	precision@20	precision@50
Baseline (position-only)	0.50	0.48
Logistic Regression	0.85	0.72
Random Forest	0.45	0.56

Test base rate: 0.439 (n=824).

Error analysis: the three concrete false positives inspected by hand all shared a pattern — striking/page_3_5 tier, transactional/informational intent, and CTR gaps close to zero (i.e., borderline cases near the tier median rather than clear misses). This suggests the model's confidence at the boundary is less reliable than its confidence on clear cases, which is exactly why the playbook (Section 7) routes borderline scores to monitor_only rather than a direct action.

6. Interpretation

What the model found: permutation importance on the honest feature set puts engagement_rate clearly in the lead (≈0.15), well ahead of content_age_days (≈0.03) and every other feature. Checked for the same kind of leak that caught sessions_90d: engagement_rate's correlation with ctr (0.08) and clicks_90d (0.02) is low, unlike sessions_90d's 0.83 — so this reads as a genuinely separate, informative signal rather than a disguised copy of the label.

A negative result, reported honestly: Random Forest does not beat the position-only baseline at precision@20 (0.45 vs. 0.50) — a direct, kept-in-the-report example of added model complexity not paying for itself on this task and this amount of data (~9,000 training rows).

A surprise: the random-vs-grouped split gap (Section 5) was smaller than expected going in. The likely explanation is that this label is defined and varies at the page level even within one client's own portfolio, so there wasn't much client-level "fingerprint" available for a model to exploit — a reassuring, well-understood negative result, not a failure to find one.

7. Recommendation

Five ranked actions, all requiring human review before publishing — nothing here is approved for direct automated edits:

rewrite_title_meta — high model score, non-commercial intent: direct title/meta review.
review_before_rewrite — high score, commercial/transactional intent: human review first, given the intent-comparison bias found in the Week 6 audit.
schedule_refresh_review — content_age_days between 271-365 (the decay-cliff window independently identified in FlyRank's own March 2026 portfolio report): proactive refresh review ahead of the CTR signal showing up.
monitor_only — borderline model score: watch, no action yet.
no_action — low score: not flagged this round (not the same as "confirmed healthy").

How an editor uses this tomorrow: pull the current week's queue, start at the top of rewrite_title_meta, sort by impressions_90d within each bucket when time is scarce (a wrong call on a high-traffic page costs more attention than one on a low-traffic page), and treat review_before_rewrite and schedule_refresh_review entries as needing a second look before any edit.

Confidence and limits, stated directly: this is decision support, not a decision. The underlying label is a same-window proxy, not an observed future outcome — nothing here has been validated against real before/after click data, and no claim in this report or the deployed paper goes further than "observed," "directional," or "decision-support" language.

8. Reproducibility

Environment: Python 3, pandas, numpy, scikit-learn, matplotlib — no unusual versions required; any recent Colab runtime has all of these preinstalled.

Exact commands to re-run from a fresh clone: open any of work/notebooks/w01_...ipynb through w07_...ipynb and capstone.ipynb in Google Colab (or Jupyter), and use Runtime → Run all. Every notebook loads the same public starter CSV directly from this repo's raw GitHub URL — no local files or credentials needed for w01, w02, w04, w05, w06, w07, or capstone.ipynb. Only w03_data_contract.ipynb needs a personal, gated Hugging Face HF_TOKEN (stored as a Colab Secret, never pasted into a cell) to query the full warehouse.

Random seed: 42, fixed throughout every split and every model that accepts one.

No sealed/holdout claim is made anywhere in this work — worth stating explicitly, since a claim not being made deserves the same clarity as one that is. All reported metrics come from a single, reproducible GroupShuffleSplit(random_state=42), not a separately sealed final evaluation set.

Committed receipts: work/outputs/capstone_metrics.json (the exact numbers above, in machine-readable form) and work/figures/capstone_precision_comparison.png (the results chart) are committed alongside this report, so the numbers in this document can be checked against a fresh run rather than taken on faith.

9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — flyrank.ai. This report mirrors, and is fully consistent with, the deployed research paper, which carries this same acknowledgment and data-credit link at the bottom of the page.
