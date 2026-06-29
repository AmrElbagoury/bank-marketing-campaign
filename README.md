# Bank Marketing Campaign — Classifier Comparison

## Business Understanding

A Portuguese bank runs phone-based marketing campaigns to sell long-term term deposit products. The majority of calls result in a "no," making the campaigns costly and inefficient. Using data from 17 campaigns (May 2008 – November 2010) covering 41,188 client contacts, the goal is to **predict which customers are most likely to subscribe to a term deposit**, so the bank can focus its calls on those people, reduce wasted effort, and improve the return on each campaign.

This is a **binary classification problem**: predict whether a client will subscribe (`yes`) or not (`no`) based on their personal profile and how they were contacted.

---

## The Data

- **Source:** UCI Machine Learning Repository — Bank Marketing Dataset
- **Size:** 41,188 contacts, 20 input features, 1 target (`y`)
- **Class imbalance:** ~88.7% of contacts did not subscribe — only ~11.3% did

**Features used:**

| Category | Features |
|---|---|
| Client demographics | age, job, marital status, education, credit default, housing loan, personal loan |
| Campaign contact | contact type, month, day of week, number of contacts, days since last campaign, previous contacts, previous outcome |

**Feature excluded:** `duration` (call length in seconds) — this is only known *after* the call ends, so including it would give the model unfair information it wouldn't have in a real scenario.

---

## Data Cleaning

- **No true missing values** were found (`isnull()` returned 0 for all columns).
- **Implicit missing values** exist as the string `'unknown'` in 6 columns. Rather than deleting those rows (which would remove up to 21% of data for `default`), `'unknown'` was retained as its own encoded category since it may carry signal (e.g., clients who decline to share financial information may behave differently).
- **`pdays` engineering:** A value of `999` means the client was never previously contacted. This was split into:
  - A binary flag `contacted_before` (1 = yes, 0 = no)
  - `pdays` reset to 0 for never-contacted clients
- All categorical features were one-hot encoded. All numeric features were standardized using `StandardScaler`.

---

## Findings

### Baseline

A model that simply predicts "no" for every customer (without looking at any features) achieves **88.7% accuracy**. This is the floor our models must beat in a meaningful way.

### Model Performance Summary (after tuning)

| Model | Test Accuracy | Test AUC | Train Time |
|---|---|---|---|
| **Logistic Regression** | **89.8%** | **0.765** | 12s |
| KNN | 89.9% | 0.739 | 14s |
| Decision Tree | 89.5% | 0.736 | 3s |
| SVM | 89.7% | 0.702 | 425s |

> **AUC (Area Under the ROC Curve)** is the right metric here. It measures how well the model distinguishes subscribers from non-subscribers, regardless of the decision threshold. A score of 0.5 is random guessing; 1.0 is perfect. All models beat random, with Logistic Regression performing best.

### Key Insight: Accuracy Is Misleading

All four models score around 89–90% accuracy, barely above the 88.7% baseline. But looking under the hood, the initial model (using only demographics) predicted "no" for **every single customer**. Accuracy looked fine on paper while the model was completely useless for the bank's goal.

Switching to AUC and expanding the feature set (adding campaign contact information) provided a meaningful improvement in the model's ability to identify likely subscribers.

### What Drives Subscription?

Based on model behavior and feature engineering:

- **Contact timing matters:** Certain months (especially end-of-quarter: March, June, September, December) see higher subscription rates.
- **Previous campaign history helps:** Clients who were successfully converted in a prior campaign (`poutcome = success`) are far more likely to subscribe again.
- **Being contacted before is a positive signal:** Clients with `contacted_before = 1` tend to convert at higher rates.
- **Demographics alone are not enough:** Job, age, and marital status have weak predictive power on their own. Campaign behavior is more telling.

---

## Actionable Recommendations

1. **Prioritize clients with a successful prior campaign outcome.** If a client said "yes" before, they are your highest-probability target. Focus call resources here first.

2. **Schedule campaigns in March, June, September, and December.** Subscription rates are noticeably higher at the end of each quarter. Aligning campaigns to these months can improve conversion without adding calls.

3. **Deprioritize clients who have never been contacted.** First-time contacts convert at a much lower rate. Use limited call capacity on warm leads first.

4. **Use the Logistic Regression model to score and rank your contact list.** Before each campaign, run client records through the model to get a probability score for each client. Call the top-ranked clients first and set a cut-off threshold based on how many calls the team can handle.
