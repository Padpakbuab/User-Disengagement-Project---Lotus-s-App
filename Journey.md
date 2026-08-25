# User-Disengagement-Project---Lotus-s-App
!!Confidential Data Source!! This is just a journey of my research


# Lotus's Active User Disengagement & Churn Prediction

This document outlines the testing strategy, operational roadmap, data schema, and feature definitions for predicting active customer disengagement within the Lotus's mobile application.

---

## 1. Project Overview & Testing Strategy

### Step 1: Define the Audience & Target
* **Target Audience:** Customers who opened the Lotus’s app at least once during a past 30-day window.
* **The "Leaving" Rule (Ground Truth):** If an active user records **zero app visits and zero purchases** across the subsequent 14 days, they are labeled as **Leaving** (`churn = 1`). Otherwise, they are **Retained** (`churn = 0`).

### Step 2: Time-Based Splitting (Fair Testing)
* **Training Data (Past Period):** Use activity from Month 1 to teach the model how engagement decay patterns appear prior to leaving.
* **Testing Data (Future Period):** Validate the model on unseen data from Month 2 to evaluate predictive accuracy without historical leakage.

### Step 3: Model Benchmarking
* **Primary Model:** Train a Gradient Boosted Tree (e.g., LightGBM) on rolling behavioral and transaction features.
* **Baseline Heuristic:** Compare against a basic business rule (e.g., *"Flag users inactive for > 7 days"*). The AI model must achieve superior precision among the top 10% highest-risk customers.

### Step 4: Business Validation (A/B Experiment)
* **Control Group (50%):** Standard marketing and communication flow.
* **Treatment Group (50%):** Automated intervention (e.g., targeted push notification with a tailored Lotus's Coins or voucher incentive).
* **Evaluation:** Compare 14-day retention rates between groups to measure direct business lift.

---
## 2. Data Table Preview

| user_id | snapshot_date | days_since_last_session | days_since_last_purchase | engagement_decay_ratio | browse_breadth_7d | product_engagement_decay_ratio | checkout_stall_rate_7d | spend_velocity_7_vs_30 | user_dissatisfaction_service_count_30d | is_app_uninstalled_30d | app_exception_rate_30d | is_churned_next_14d |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `U_1001` | 2026-08-01 | 2 | 5 | 0.95 | 4.80 | 1.05 | 0.00 | 1.10 | 0 | 0 | 0.00 | **0** |
| `U_1002` | 2026-08-01 | 12 | 22 | 0.28 | 1.10 | 0.20 | 1.00 | 0.00 | 2 | 0 | 0.15 | **1** |
| `U_1003` | 2026-08-01 | 28 | 30 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 4 | 1 | 0.50 | **1** |
| `U_1004` | 2026-08-01 | 6 | 14 | 0.65 | 2.40 | 0.58 | 0.00 | 0.45 | 0 | 0 | 0.00 | **0** |

---

## 3. Feature Prioritization & Definitions

Indicators are categorized from highest direct signal strength to secondary and edge-case signals.

## Derived Behavioral Feature Definitions

| Feature Name | Priority Tier | Plain Math Formula | Underlying Events & Logs | What It Means (Simple Explanation) |
| :--- | :--- | :--- | :--- | :--- |
| **Engagement Decay Ratio** | Tier 1: Core | `(Screen Views in 7d / 7) / (Screen Views in 30d / 30)` | `screen_view` | Compares recent daily browsing to the monthly baseline. Values $< 1.0$ indicate declining overall app activity. |
| **User Dissatisfaction Service Received Amount** | Tier 1: Core | `Count of (Refunds + Returns + Failed Deliveries + CS Complaints in 30d)` | `refund`, Order Status (`RETURNED`, `FAILED`), CS Tickets | Quantifies the total volume of negative service experiences and order failures that trigger brand churn. |
| **App Uninstall Flag** | Tier 1: Core | `1 if app uninstalled in 30d else 0` | `app_remove`, OS Uninstall Callback | Direct behavioral signal indicating the customer has deleted the application. |
| **Checkout Stall Rate** | Tier 1: Core | `1 - (Purchases in 7d / Checkout Starts in 7d)` | `purchase`, `cart_proceed_to_checkout` | Percentage of checkout initiations over the last week that failed to convert into completed orders. |
| **Spend Velocity (Order Value Drift)** | Tier 2: Secondary | `Total Spend in 7d / (Total Spend in 30d / 4)` | Order Info (`net_amount_paid`) | Tracks spending acceleration or slowdown relative to the customer's average 30-day weekly baseline. |
| **Browse Breadth** | Tier 2: Secondary | `(Product Views in 7d + List Views in 7d) / Total Sessions in 7d` | `view_item`, `view_item_list`, `session_start` | Measures exploration depth per visit; shallow opens with low item views signal passive disengagement. |
| **Product Engagement Decay Ratio** | Tier 2: Secondary | `((Product Views in 7d + List Views in 7d) / 7) / ((Product Views in 30d + List Views in 30d) / 30)` | `view_item`, `view_item_list` | Compares daily catalog browsing volume in the past week against the 30-day baseline to detect intent drop-off. |
| **App Exception Rate** | Tier 3: Edge Case | `Crashes & App Errors in 30d / Total Sessions in 30d` | `app_exception`, `session_start` | Measures error and crash frequency per session, isolating involuntary technical friction. |

## 4. LightGBM Model Implementation Guide

Follow these 4 core steps to train, validate, and interpret the active user churn prediction model using the finalized feature set:

### Step 1: Feature Matrix & Imbalance Setup
* **Feature Selection (10 Predictors):**
  * *Recency Baselines:* `days_since_last_session`, `days_since_last_purchase`
  * *Digital & Catalog Velocity:* `engagement_decay_ratio`, `browse_breadth_7d`, `product_engagement_decay_ratio`
  * *Funnel & Spend:* `checkout_stall_rate_7d`, `spend_velocity_7_vs_30`
  * *Friction & Dissatisfaction:* `user_dissatisfaction_service_count_30d`, `is_app_uninstalled_30d`, `app_exception_rate_30d`
* **Target Label:** `is_churned_next_14d` (Binary: `1` for churned, `0` for retained).
* **Imbalance Handling:** Active users dropping off are a minority class. Set positive weighting dynamically:
  $$\text{scale\_pos\_weight} = \frac{\text{count}(\text{retained\_users})}{\text{count}(\text{churned\_users})}$$

---

### Step 2: Baseline Hyperparameters & Configuration

| Parameter | Recommended Value | Purpose |
| :--- | :--- | :--- |
| `objective` | `'binary'` | Standard log-loss for binary churn classification. |
| `boosting_type` | `'gbdt'` | Gradient Boosted Decision Trees. |
| `learning_rate` | `0.03` | Conservative step size to generalize across seasonality. |
| `num_leaves` | `31` | Controls tree complexity (kept low to prevent overfitting on ratios). |
| `max_depth` | `6` | Limits tree depth. |
| `min_child_samples` | `50` | Prevents leaves from isolating single outlier users or extreme ratios. |
| `subsample` | `0.8` | Randomly samples 80% of rows per tree to reduce variance. |
| `colsample_bytree` | `0.8` | Randomly samples 80% of features per tree to prevent dominant collinearity. |
| `scale_pos_weight` | `neg_count / pos_count` | Re-weights churn instances to account for class imbalance. |

---

### Step 3: Out-of-Time Training & Early Stopping

Train on a historical snapshot ($T_0$) and evaluate on a separate future snapshot ($T_1$) to prevent temporal data leakage:

```python
import lightgbm as lgb
from sklearn.metrics import average_precision_score, roc_auc_score

# 1. Split Data by Snapshot Date (Out-of-Time Validation)
X_train, y_train = train_df[features], train_df['is_churned_next_14d']
X_valid, y_valid = valid_df[features], valid_df['is_churned_next_14d']

# 2. Compute Class Weighting
pos_weight = (len(y_train) - y_train.sum()) / y_train.sum()

# 3. Initialize and Train Model
model = lgb.LGBMClassifier(
    objective='binary',
    boosting_type='gbdt',
    n_estimators=1000,
    learning_rate=0.03,
    num_leaves=31,
    max_depth=6,
    min_child_samples=50,
    subsample=0.8,
    colsample_bytree=0.8,
    scale_pos_weight=pos_weight,
    random_state=42
)

model.fit(
    X_train, y_train,
    eval_set=[(X_valid, y_valid)],
    eval_metric=['average_precision', 'auc'],
    callbacks=[lgb.early_stopping(stopping_rounds=50, verbose=True)]
)
