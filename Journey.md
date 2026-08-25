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

## 2. Data Schema & Entity Scope

To build the predictive pipeline, extract and join the following five data entities using `user_id` as the primary key:

| Entity / Table | Granularity | Required Columns / Fields (Mapped to Lotus's GA4 & POS) | Purpose & Description |
| :--- | :--- | :--- | :--- |
| **1. User Profile Dimension** | 1 row per `user_id` | • `user_id`<br>• `signup_date`<br>• `home_store_id` (Preferred store branch)<br>• `primary_delivery_province` / `postcode`<br>• `opt_in_push_notification` (Boolean)<br>• `account_tier` (My Lotus's Member tier) | Controls for customer tenure, geography, channel permissions, and member segmentation. |
| **2. App Clickstream & Navigation** | 1 row per event log | • `user_id`<br>• `session_id`<br>• `event_timestamp`<br>• `event_name` (`session_start`, `screen_view`, `view_item_list`, `view_item`, `product_detail_related`, `homepage_icon`, `footer_menu`, `search_result_filter_tab`)<br>• `device_os` (iOS / Android)<br>• `app_version` | Captures digital velocity, browse breadth, search behavior, and navigation degradation. |
| **3. Omnichannel Transactions & Funnel** | 1 row per transaction / checkout event | • `user_id`<br>• `order_id` (or `session_id` for checkout funnel)<br>• `event_timestamp`<br>• `funnel_event` (`add_to_cart`, `remove_from_cart`, `header_cart_icon`, `cart_proceed_to_checkout`, `place_selected_confirm`, `purchase`, `refund`)<br>• `channel` (`ONLINE_DELIVERY`, `CLICK_COLLECT`, `IN_STORE_POS`)<br>• `order_status` (`COMPLETED`, `CANCELLED`, `REFUNDED`)<br>• `net_amount_paid`<br>• `product_category_id` | Tracks spend decay, cart abandonment, fulfillment selection drop-offs, and store-to-online switching. |
| **4. Loyalty & Promotions (My Lotus's)** | 1 row per reward interaction | • `user_id`<br>• `event_timestamp`<br>• `loyalty_event` (`mycoupon_category`, `coupon_apply`, `coupon_card`, `enable_use_lotuss_point`, `redeem_coins_button`, `redeem_coins_confirm`, `lotuss_privileges`, `store_privileges`)<br>• `points_balance`<br>• `coupon_discount_value` | Measures promotion sensitivity, Lotus's Coins redemption activity, and voucher fatigue. |
| **5. Friction, Messaging & Telemetry** | 1 row per friction / alert event | • `user_id`<br>• `event_timestamp`<br>• `friction_event` (`app_exception`, `forced_logout`, `app_clear_data`, `app_remove`, `trash_icon`, `place_selected_cancel`, `checkout_coupon_leave`, `coupon_cancel`, `notification_receive`, `notification_open`, `notification_dismiss`, `fiam_impression`, `fiam_dismiss`)<br>• `error_code` / `screen_fragment` (`PageNotFoundFragment`, etc.) | Detects involuntary churn drivers, technical crashes, push notification burnout, and UX roadblocks. |

---

## 3. Feature Prioritization & Definitions

Indicators are categorized from highest direct signal strength to secondary and edge-case signals.

| Feature Name | Priority Tier | Plain Math Formula | Underlying GA4 Events | What It Means (Simple Explanation) |
| :--- | :--- | :--- | :--- | :--- |
| **Engagement Decay Ratio** | Tier 1: Core | `(Screen Views in 7d / 7) / (Screen Views in 30d / 30)` | `screen_view` | Compares recent daily browsing to the monthly baseline. Values $< 1.0$ indicate declining app visits. |
| **Checkout Stall Rate** | Tier 1: Core | `1 - (Purchases in 7d / Checkout Starts in 7d)` | `purchase`, `cart_proceed_to_checkout` | Percentage of recent checkout attempts that failed to convert into completed orders. |
| **Cart Abandonment Rate** | Tier 1: Core | `1 - (Purchases in 30d / Items Added to Cart in 30d)` | `purchase`, `add_to_cart` | Measures how often items added to the cart are left unpurchased over 30 days. |
| **Browse Breadth** | Tier 1: Core | `(Product Views in 7d + List Views in 7d) / Total Sessions in 7d` | `view_item`, `view_item_list`, `session_start` | Measures exploration depth per visit; shallow visits signal passive disengagement. |
| **Cart Purge Ratio** | Tier 2: Secondary | `(Items Removed + Trash Clicks in 30d) / Items Added to Cart in 30d` | `remove_from_cart`, `trash_icon`, `add_to_cart` | High ratios reflect price sensitivity, high delivery thresholds, or missing item friction. |
| **Checkout Drop-off Velocity** | Tier 2: Secondary | `Coupons Left at Checkout in 7d / Checkout Starts in 7d` | `checkout_coupon_leave`, `cart_proceed_to_checkout` | Captures users abandoning purchases specifically during coupon selection. |
| **Coupon Abandonment Rate** | Tier 2: Secondary | `(Vouchers Left + Vouchers Cancelled in 30d) / Vouchers Selected in 30d` | `checkout_coupon_leave`, `coupon_cancel`, `checkout_coupon_select` | Measures voucher friction, such as unmet minimum spend thresholds. |
| **Coin Redemption Velocity** | Tier 2: Secondary | `Coins Redeemed in 30d / Times Points Enabled in 30d` | `redeem_coins_confirm`, `enable_use_lotuss_point` | Detects users who stop utilizing loyalty points despite having an active point balance. |
| **Coin-to-Voucher Utilization Rate** | Tier 2: Secondary | `Coins Redeemed in 30d / Coupons Applied in 30d` | `redeem_coins_confirm`, `coupon_apply` | Checks if loyalty coins are actively combined with regular discount vouchers. |
| **App Exception Rate(flags)** | Tier 3: Edge Case | `Crashes & App Errors in 30d / Total Sessions in 30d` | `app_exception`, `session_start` | Measures crash frequency per visit, identifying involuntary technical churn. |
| **Refund Propensity(flags)** | Tier 3: Edge Case | `Refunds in 30d / Purchases in 30d` | `refund`, `purchase` | Tracks post-purchase order dissatisfaction. |

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
