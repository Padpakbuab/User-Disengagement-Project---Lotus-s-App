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

| user_id | snapshot_date | days_since_last_session | engagement_decay_ratio | cart_abandonment_rate_30d | checkout_stall_rate_7d | delivery_failure_rate_30d | is_app_uninstalled_30d | coin_to_voucher_rate_30d | notification_receptivity_7d | is_churned_next_14d |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `U_1001` | 2026-08-01 | 3 | 0.85 | 0.20 | 0.00 | 0.00 | 0 | 0.80 | 0.40 | **0** |
| `U_1002` | 2026-08-01 | 18 | 0.12 | 1.00 | 1.00 | 0.50 | 0 | 0.00 | 0.02 | **1** |
| `U_1003` | 2026-08-01 | 25 | 0.00 | 0.00 | 0.00 | 1.00 | 1 | 0.00 | 0.00 | **1** |

---

## 3. Feature Prioritization & Definitions

Indicators are categorized from highest direct signal strength to secondary and edge-case signals.

## Derived Behavioral Feature Definitions

| Feature Name | Priority Tier | Plain Math Formula | Underlying Events & Logs | What It Means (Simple Explanation) |
| :--- | :--- | :--- | :--- | :--- |
| **Engagement Decay Ratio** | Tier 1: Core | `(Screen Views in 7d / 7) / (Screen Views in 30d / 30)` | `screen_view` | Compares recent daily browsing to the monthly baseline. Values $< 1.0$ indicate declining app activity. |
| **Delivery Failure Rate** | Tier 1: Core | `Failed Deliveries in 30d / Total Deliveries in 30d` | Delivery Status Logs (`FAILED`, `CANCELLED_BY_RIDER`) | Measures fulfillment issues; customers experiencing failed deliveries are at immediate risk of leaving. |
| **Return / Refund Propensity** | Tier 1: Core | `(Refunds + Returned Orders in 30d) / Total Orders in 30d` | `refund`, Order Status (`RETURNED`) | Spikes in returned items reflect product dissatisfaction and drive churn. |
| **App Uninstall Flag** | Tier 1: Core | `1 if app uninstalled in 30d else 0` | `app_remove`, OS Uninstall Callback | Direct signal of churn; indicates the user deleted the app. |
| **Checkout Stall Rate** | Tier 1: Core | `1 - (Purchases in 7d / Checkout Starts in 7d)` | `purchase`, `cart_proceed_to_checkout` | Percentage of recent checkout attempts that failed to finish as a completed order. |
| **Cart Abandonment Rate** | Tier 1: Core | `1 - (Purchases in 30d / Items Added to Cart in 30d)` | `purchase`, `add_to_cart` | Measures how often items added to the cart are left unpurchased over 30 days. |
| **Spend Velocity (Order Value Drift)** | Tier 2: Secondary | `Total Spend in 7d / (Total Spend in 30d / 4)` | Order Info (`net_amount_paid`) | Measures whether weekly spending is slowing down compared to the user's monthly average. |
| **Browse Breadth** | Tier 2: Secondary | `(Product Views in 7d + List Views in 7d) / Total Sessions in 7d` | `view_item`, `view_item_list`, `session_start` | Measures exploration depth per visit; shallow visits signal fading user interest. |
| **Notification Receptivity Ratio** | Tier 2: Secondary | `Push Opens in 7d / Push Notifications Received in 7d` | `notification_open`, `notification_receive` | Low rates indicate push notification fatigue before users disable alerts. |
| **Cart Purge Ratio** | Tier 2: Secondary | `(Items Removed + Trash Clicks in 30d) / Items Added to Cart in 30d` | `remove_from_cart`, `trash_icon`, `add_to_cart` | High ratios reflect price sensitivity, high delivery thresholds, or missing item friction. |
| **Coin-to-Voucher Utilization Rate** | Tier 2: Secondary | `Coins Redeemed in 30d / Coupons Applied in 30d` | `redeem_coins_confirm`, `coupon_apply` | Checks if loyalty coins are actively combined with regular discount vouchers. |
| **Coupon Abandonment Rate** | Tier 2: Secondary | `(Vouchers Left + Vouchers Cancelled in 30d) / Vouchers Selected in 30d` | `checkout_coupon_leave`, `coupon_cancel`, `checkout_coupon_select` | Measures voucher friction, such as unmet minimum spend thresholds. |
| **App Exception Rate** | Tier 3: Edge Case | `Crashes & App Errors in 30d / Total Sessions in 30d` | `app_exception`, `session_start` | Measures crash frequency per visit, identifying technical app instability. |

## 4. LightGBM Model Implementation Guide

Follow these 4 core steps to train, validate, and interpret the churn model:

1. **Format Features & Handle Imbalance:**
   * Convert categorical features (`account_tier`, `device_os`, `home_store_id`) directly to `category` dtype.
   * Calculate positive class weighting: `scale_pos_weight = total_negative_users / total_churned_users`.

2. **Configure Baseline Hyperparameters:**
   * `objective = 'binary'`
   * `boosting_type = 'gbdt'`
   * `learning_rate = 0.03`
   * `num_leaves = 31`
   * `max_depth = 6`
   * `min_child_samples = 50`
   * `subsample = 0.8` (bagging fraction)
   * `colsample_bytree = 0.8` (feature fraction)

3. **Train with Early Stopping:**
   * Fit the model using an out-of-time validation snapshot (e.g., train on Month 1, validate on Month 2).
   * Monitor evaluation metrics: `eval_metric = ['average_precision', 'auc']` with `stopping_rounds = 50`.

4. **Evaluate & Explain Risk Drivers:**
   * Evaluate ranking performance using **PR-AUC (Average Precision)** and **Precision@Top 10%**.
   * Run `shap.TreeExplainer(model)` on high-risk users to output the top 3 contributing friction factors per customer for CRM targeting.
