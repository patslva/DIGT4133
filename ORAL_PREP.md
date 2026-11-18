# Oral exam prep: NorthLine delivery-exception notebook

20 minutes, individual, on Zoom. Expect to run or inspect selected code and to explain individual records, transformations and metrics. Slides are not expected.

## Before the oral
- Run the notebook once on your own machine (about 27 minutes) so you have seen it work. Small numbers may differ slightly from the report if your Java or core count differs.
- Open the executed notebook before the call starts. Do not rerun the tuning cell (Cell 16) during the oral; if asked to run something, pick a small cell.
- Check Zoom, screen sharing, and that Java and Spark start.
- Open the report PDF and this file on a second screen if you have one.

## 60-second story
1. **Problem:** at dispatch, will this order have a delivery exception within 7 days? Positive class = 1. About 1 order in 7.
2. **Data:** 121,150 rows. After removing duplicates, conflicting copies and bad dates, 118,802 orders.
3. **Split:** by dispatch date (train, validate, test) with 7-day gaps, so the model is tested on the future.
4. **Leakage:** five columns only exist after dispatch, so I excluded them even though they look very predictive.
5. **Models:** logistic regression baseline vs a tuned random forest, tuned on validation PR-AUC.
6. **Cutoff:** chosen on validation (F1, cost and capacity rules), frozen before testing.
7. **Result:** the forest catches about half the exceptions, and about one alert in four is real. It is not clearly better than the baseline.
8. **Limits:** weak signal (ROC-AUC about 0.64), one synthetic period, assumed costs.

## Cell map (1-based, as numbered in the notebook)
| Cell | What it does |
|---|---|
| 4 | Load the CSV as text, size, schema, blanks |
| 5 | Duplicates, conflicting order IDs, unusable dates |
| 7 | Chronological split, integrity check, class balance, customer overlap |
| 9 | Cleaning rules and feature definitions |
| 10 | Applies rules, numeric audit |
| 12 | Leakage table and data-quality entries |
| 14 | Distribution and category figures |
| 16 | Pipeline, hyperparameter search, winners |
| 18 | Feature ablation and class-weight checks |
| 20 | Alert cutoffs (F1, cost, capacity) and cost sensitivity |
| 22 | Single test evaluation, curves, confusion matrices |
| 24 | Customer-level bootstrap |
| 26 | Error analysis and one traced record |
| 27 | Forest importances and logistic coefficients |
| 29 | Full data-quality log |

## Key numbers to remember
| Item | Value |
|---|---|
| Rows / columns | 121,150 / 33 |
| Exact duplicates | 500 |
| Order IDs with conflicting copies | 650 (1,300 rows dropped) |
| Unusable dispatch dates | 548 |
| Usable orders | 118,802 |
| Split sizes | Train 82,214, Validation 16,868, Test 17,887 |
| Exception rate | 12.9% train, 14.9% validation, 16.1% test |
| Orders from customers seen in training | about 84% (validation and test) |
| Leakage single-column AUC | support tickets 0.90, delivery days 0.87, attempts 0.77, refund 0.73 |
| Forest winner | depth 12, min 25 per node, one-third of features, 100 trees; validation PR-AUC 0.265 |
| Baseline winner | L1 logistic regression, regParam 0.001; validation PR-AUC 0.256 |
| Forest on test | caught 1,467 of 2,882 (recall 50.9%), 6,047 alerts, precision 24.3%, F1 0.329, accuracy 66.5%, ROC-AUC 0.646, PR-AUC 0.272 |
| Baseline on test | recall 45.2%, precision 25.0%, F1 0.322, accuracy 69.3%, ROC-AUC 0.639, PR-AUC 0.267 |
| Always predict "no exception" | accuracy 83.9%, catches nothing |
| No-skill PR-AUC | the exception rate, 0.161 |
| Bootstrap (forest minus baseline) | PR-AUC +0.005 (-0.003 to +0.014), ROC-AUC +0.006, F1 +0.007; all intervals include 0 |
| Expected cost per 1,000 orders (test) | forest about 652 vs 806 for alerting nobody (assumed 5:1 cost) |
| Forest PR-AUC by split | train 0.329, validation 0.265, test 0.272 |

## Likely questions and answers
**Why split by date, not randomly?** It mirrors real use: train on the past, score the future. A random split lets the model learn from orders dispatched after the ones it is tested on, and hides drift. The 7-day gaps let earlier orders finish their outcome window.

**How do you know identical records are not in two splits?** Duplicates and conflicting copies were removed before splitting, and Cell 7 asserts that every `order_id` appears once, in exactly one split, with splits in strict date order.

**What is leakage in this dataset?** Information only known after dispatch. Five columns (`actual_delivery_days`, `final_delivery_status`, `delivery_attempts_total`, `refund_issued_after_dispatch`, `post_delivery_support_tickets`) are excluded. Cell 12 shows their single-column AUC and that "Delivered" orders almost never have an exception.

**Which assumption could still be a leak?** `prior_order_count` and `prior_exception_count`. I assume they are snapshots from before this order. The file has no timestamps to prove it, but they are only weakly predictive (AUC about 0.53 and 0.56), so a small leak could not explain the results.

**Why drop the 650 conflicting order IDs instead of picking one copy?** There is no authoritative version, and the copies differ in two feature columns. Picking one adds arbitrary noise. It is about 1% of rows, and their exception rate is close to the overall rate, so bias is small.

**Why median imputation?** It is robust to extreme values, and it is learned from training data only inside the pipeline.

**What does `StringIndexer(handleInvalid="keep")` do?** It turns categories into numbers using codes learned on training data. A category never seen in training goes to an extra "unseen" bucket instead of crashing.

**Why one-hot encoding?** Carrier and zone have no natural order. One-hot gives each level its own 0/1 column.

**Why not use accuracy?** Predicting "no exception" for every order scores 83.9% and catches nothing. PR-AUC focuses on the rare class across all cutoffs; precision, recall, F1 and the confusion matrix describe one cutoff.

**Why average precision for PR-AUC?** The trapezoid area gives a model that outputs one constant score a PR-AUC near 0.57 instead of the exception rate. Two over-penalized logistic models in my grid are exactly such models, and average precision scores them 0.149 (the exception rate), so it behaves correctly.

**What did you tune, and why those hyperparameters?** Forest: `maxDepth` (how detailed the rules), `minInstancesPerNode` (regularization), `featureSubsetStrategy` (tree diversity), `numTrees` (stability). Logistic: `regParam` and `elasticNetParam`. Selection metric: validation PR-AUC. Test data was never used for selection.

**Why does `minInstancesPerNode` matter?** Depth 12 with one order per leaf memorizes training data (train PR-AUC 0.63 vs validation 0.26). Requiring 25 orders per node keeps most of the accuracy with a much smaller gap (0.33 vs 0.27).

**Is the model overfitting or underfitting?** Slight overfitting after tuning (0.329 train, 0.265 validation, 0.272 test). ROC-AUC is about 0.64 on every split for both models, and a flexible forest gains almost nothing over logistic regression, so the limit is weak signal in the inputs, not model capacity.

**Which model is better?** The forest is at best marginally better. The bootstrap intervals for PR-AUC, ROC-AUC and F1 differences include zero. The forest's higher recall comes mostly from alerting on more orders (33.8% vs 29.1%), which is a different operating point, not a better model.

**Why bootstrap customers, not orders?** Orders from the same customer are not independent, and about 84% of later orders come from customers seen in training.

**How did you choose the cutoff?** On validation only, with three rules: F1-optimal (main), cost-optimal (assumed miss = 5 times a wasted check) and capacity (alert the top 20%). All were frozen before testing. Scores from a forest are not calibrated probabilities, so cutoffs are score thresholds.

**What if the business cost ratio is different?** The cost-optimal cutoff is very sensitive: at 2:1 only about 3% of orders are alerted, at 5:1 about 30%, at 10:1 about 80%. The ratio has to come from the business.

**Why does logistic regression score higher on later splits?** PR-AUC rises with the exception rate (12.9%, then 14.9%, then 16.1%). ROC-AUC stays near 0.64.

**What did the model get wrong?** It alerts mostly in rural and storm orders (high recall there) and misses express orders and orders with no obvious risk factor (express recall about 36%). Cell 26 shows the six most extreme errors.

**Why Spark for this size of data?** The workflow is built to scale and the course asks for it. scikit-learn is used only to compute metrics on the collected scores (under 18,000 rows), and I cross-checked with Spark's own evaluator.

**What would you do next?** Add more dispatch-time information (carrier or lane history, warehouse load), calibrate the scores, get real intervention costs, use time-based cross-validation, and monitor drift.

## Tracing one record (Cell 26)
Practice saying these steps for the traced order:
1. **Raw row:** the CSV text for that order, with post-dispatch fields hidden.
2. **Cleaning:** spellings standardized, invalid numbers set to missing, weekday and distance per promised day added.
3. **Pipeline:** missing numbers filled with the training median, categories turned into 0/1 columns.
4. **Score:** the forest's score for the order vs the frozen cutoff (0.1344). At or above means alert.
5. **Check:** compare the decision with the true label (false alert or miss) and say why the dispatch-time data could or could not show the problem.

## Be honest about
- The cost ratio (5:1) and capacity (20%) are my assumptions, not data.
- One future period of synthetic data; returning customers are included.
- No label timestamps, so seven-day outcomes are assumed complete.
- The forest is not clearly better than logistic regression.
