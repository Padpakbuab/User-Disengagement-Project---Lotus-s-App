# User-Disengagement-Project---Lotus-s-App
!!Confidential Data Source!! This is just a journey of my research

Step 1: Defining Data

Look into Lotus's App Data bucket and see what I can play with.

1.1 Look into Customer Journey 

Key Event:

App Open > App usage > App Leave 



# Lotus's Active User Churn Prediction: Data Pipeline & Feature Engineering

This repository outlines the data requirements, entity relationships, and behavioral feature schema for predicting active user disengagement and churn in the Lotus's mobile application ecosystem.

---

## Data Schema & Entity Scope

| Entity / Table | Granularity | Required Columns / Fields (Mapped to Lotus's GA4 & POS) | Purpose |
| :--- | :--- | :--- | :--- |
| **1. User Profile Dimension** | 1 row per `user_id` | • `user_id`<br>• `signup_date`<br>• `home_store_id` (Preferred store branch)<br>• `primary_delivery_province` / `postcode`<br>• `opt_in_push_notification` (Boolean)<br>• `account_tier` (My Lotus's Member tier) | Controls for customer tenure, geography, channel permissions, and member segmentation. |
| **2. App Clickstream & Navigation** | 1 row per event log | • `user_id`<br>• `session_id`<br>• `event_timestamp`<br>• `event_name` (`session_start`, `screen_view`, `view_item_list`, `view_item`, `product_detail_related`, `homepage_icon`, `footer_menu`, `search_result_filter_tab`)<br>• `device_os` (iOS / Android)<br>• `app_version` | Captures digital velocity, browse breadth, search behavior, and navigation degradation. |
| **3. Omnichannel Transactions & Funnel** | 1 row per transaction / checkout event | • `user_id`<br>• `order_id` (or `session_id` for checkout funnel)<br>• `event_timestamp`<br>• `funnel_event` (`add_to_cart`, `remove_from_cart`, `header_cart_icon`, `cart_proceed_to_checkout`, `place_selected_confirm`, `purchase`, `refund`)<br>• `channel` (`ONLINE_DELIVERY`, `CLICK_COLLECT`, `IN_STORE_POS`)<br>• `order_status` (`COMPLETED`, `CANCELLED`, `REFUNDED`)<br>• `net_amount_paid`<br>• `product_category_id` | Tracks spend decay, cart abandonment, fulfillment selection drop-offs, and store-to-online switching. |
| **4. Loyalty & Promotions (My Lotus's)** | 1 row per reward interaction | • `user_id`<br>• `event_timestamp`<br>• `loyalty_event` (`mycoupon_category`, `coupon_apply`, `coupon_card`, `enable_use_lotuss_point`, `redeem_coins_button`, `redeem_coins_confirm`, `lotuss_privileges`, `store_privileges`)<br>• `points_balance`<br>• `coupon_discount_value` | Measures promotion sensitivity, Lotus's Coins redemption activity, and voucher fatigue. |
| **5. Friction, Messaging & Telemetry** | 1 row per friction / alert event | • `user_id`<br>• `event_timestamp`<br>• `friction_event` (`app_exception`, `forced_logout`, `app_clear_data`, `app_remove`, `trash_icon`, `place_selected_cancel`, `checkout_coupon_leave`, `coupon_cancel`, `notification_receive`, `notification_open`, `notification_dismiss`, `fiam_impression`, `fiam_dismiss`)<br>• `error_code` / `screen_fragment` (`PageNotFoundFragment`, etc.) | Detects involuntary churn drivers, technical crashes, push notification burnout, and UX roadblocks. |

---

## Derived Behavioral Feature Definitions

| Feature Name | Category | Plain Math Formula | Underlying GA4 Events | What It Means (Simple Explanation) |
| :--- | :--- | :--- | :--- | :--- |
| **Engagement Decay Ratio** | Navigation & Activity | `(Screen Views in 7d / 7) / (Screen Views in 30d / 30)` | `screen_view` | Compares recent daily browsing to the monthly average. Score $< 1.0$ means the user is opening and browsing the app less frequently. |
| **Browse Breadth** | Navigation & Activity | `(Product Views in 7d + List Views in 7d) / Total Sessions in 7d` | `view_item`, `view_item_list`, `session_start` | Shows how deeply a user browses per visit. Low numbers mean they only open the app briefly and leave without looking at products. |
| **Cart Abandonment Rate** | Purchase Funnel | `1 - (Purchases in 30d / Items Added to Cart in 30d)` | `purchase`, `add_to_cart` | Measures how often a user puts items in the cart but never actually buys them. |
| **Cart Purge Ratio** | Purchase Funnel | `(Items Removed + Trash Clicks in 30d) / Items Added to Cart in 30d` | `remove_from_cart`, `trash_icon`, `add_to_cart` | High ratio means the user frequently clears out their cart due to high prices, extra fees, or missing items. |
| **Checkout Drop-off Velocity** | Purchase Funnel | `Coupons Left at Checkout in 7d / Checkout Starts in 7d` | `checkout_coupon_leave`, `cart_proceed_to_checkout` | Measures users who start paying but cancel or leave right at the coupon screen. |
| **Checkout Stall Rate** | Purchase Funnel | `1 - (Purchases in 7d / Checkout Starts in 7d)` | `purchase`, `cart_proceed_to_checkout` | The percentage of checkout attempts in the last week that failed to finish as a completed order. |
| **Refund Propensity** | Purchase Funnel | `Refunds in 30d / Purchases in 30d` | `refund`, `purchase` | High refund rates indicate a bad delivery or product experience, leading directly to app abandonment. |
| **Coin-to-Voucher Utilization Rate** | Loyalty & Rewards | `Coins Redeemed in 30d / Coupons Applied in 30d` | `redeem_coins_confirm`, `coupon_apply` | Checks if the user actively spends My Lotus's Coins when using discount coupons. |
| **Coin Redemption Velocity** | Loyalty & Rewards | `Coins Redeemed in 30d / Times Points Enabled in 30d` | `redeem_coins_confirm`, `enable_use_lotuss_point` | Shows if a user stops using their reward points even though their account has points available. |
| **Coupon Abandonment Rate** | Loyalty & Rewards | `(Vouchers Left + Vouchers Cancelled in 30d) / Vouchers Selected in 30d` | `checkout_coupon_leave`, `coupon_cancel`, `checkout_coupon_select` | Measures voucher frustration (e.g., trying to apply a discount code that doesn't meet minimum spend conditions). |
| **Notification Receptivity Ratio** | Communication Fatigue | `Push Opens in 7d / Push Notifications Received in 7d` | `notification_open`, `notification_receive` | Low ratio means the user is ignoring push notifications and may soon turn off app alerts. |
| **Push Dismiss Ratio** | Communication Fatigue | `Push Dismissals in 7d / Push Notifications Received in 7d` | `notification_dismiss`, `notification_receive` | Measures how often a user actively swipes away push notifications without reading them. |
| **In-App Message Dismiss Rate** | Communication Fatigue | `Pop-ups Dismissed in 30d / Pop-ups Shown in 30d` | `fiam_dismiss`, `fiam_impression`, `firebase_in_app_message_dismiss`, `firebase_in_app_message_impression` | Shows banner fatigue—users immediately closing in-app promotional pop-ups. |
| **App Exception Rate** | Technical Friction | `Crashes & App Errors in 30d / Total Sessions in 30d` | `app_exception`, `session_start` | Tracks technical bugs and crashes per visit, which cause involuntary user drop-off. |
```

Idea: 

- gathering user engagement of click sessions 
- defining indicators


Step 2: Researching Models 

Status: Options Gathering

- Light GBM vs XGBoost?
