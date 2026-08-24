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

| Feature Name | Category | Mathematical / Logic Formula | Underlying GA4 Events | Behavioral Interpretation |
| :--- | :--- | :--- | :--- | :--- |
| **Engagement Decay Ratio** | Navigation & Activity | $\frac{\text{screen\_view}_{7d} \,/\, 7}{(\text{screen\_view}_{30d} \,/\, 30) + \epsilon}$ | `screen_view` | Ratio $< 1.0$ indicates an active drop-off in daily browsing volume compared to historical baseline. |
| **Browse Breadth** | Navigation & Activity | $\frac{\text{view\_item}_{7d} + \text{view\_item\_list}_{7d}}{\text{session\_start}_{7d} + \epsilon}$ | `view_item`, `view_item_list`, `session_start` | Measures exploration depth; shallow opens with low item views signal passive disengagement. |
| **Cart Abandonment Rate** | Purchase Funnel | $1 - \left(\frac{\text{purchase}_{30d}}{\text{add\_to\_cart}_{30d} + \epsilon}\right)$ | `purchase`, `add_to_cart` | Spikes when high purchase intent fails to convert into completed orders over a 30-day window. |
| **Cart Purge Ratio** | Purchase Funnel | $\frac{\text{remove\_from\_cart}_{30d} + \text{trash\_icon}_{30d}}{\text{add\_to\_cart}_{30d} + \epsilon}$ | `remove_from_cart`, `trash_icon`, `add_to_cart` | High ratio signals hesitation, pricing friction, or out-of-stock items encountered in cart. |
| **Checkout Drop-off Velocity** | Purchase Funnel | $\frac{\text{checkout\_coupon\_leave}_{7d}}{\text{cart\_proceed\_to\_checkout}_{7d} + \epsilon}$ | `checkout_coupon_leave`, `cart_proceed_to_checkout` | Captures users backing out during coupon application at final checkout step. |
| **Checkout Stall Rate** | Purchase Funnel | $1 - \left(\frac{\text{purchase}_{7d}}{\text{cart\_proceed\_to\_checkout}_{7d} + \epsilon}\right)$ | `purchase`, `cart_proceed_to_checkout` | Measures immediate 7-day payment drop-offs after initiating the checkout flow. |
| **Refund Propensity** | Purchase Funnel | $\frac{\text{refund}_{30d}}{\text{purchase}_{30d} + \epsilon}$ | `refund`, `purchase` | Spikes in refund activity strongly correlate with post-purchase dissatisfaction and app abandonment. |
| **Coin-to-Voucher Utilization Rate** | Loyalty & Rewards | $\frac{\text{redeem\_coins\_confirm}_{30d}}{\text{coupon\_apply}_{30d} + \epsilon}$ | `redeem_coins_confirm`, `coupon_apply` | Measures whether a user actively uses My Lotus's Coins alongside standard promotional coupons. |
| **Coin Redemption Velocity** | Loyalty & Rewards | $\frac{\text{redeem\_coins\_confirm}_{30d}}{\text{enable\_use\_lotuss_point}_{30d} + \epsilon}$ | `redeem_coins_confirm`, `enable_use_lotuss_point` | A declining ratio signals the user is no longer burning points/coins despite having points enabled. |
| **Coupon Abandonment Rate** | Loyalty & Rewards | $\frac{\text{checkout\_coupon\_leave}_{30d} + \text{coupon\_cancel}_{30d}}{\text{checkout\_coupon\_select}_{30d} + \epsilon}$ | `checkout_coupon_leave`, `coupon_cancel`, `checkout_coupon_select` | Identifies coupon frustration where vouchers fail minimum spend rules or provide poor discounts. |
| **Notification Receptivity Ratio** | Communication Fatigue | $\frac{\text{notification\_open}_{7d}}{\text{notification\_receive}_{7d} + \epsilon}$ | `notification_open`, `notification_receive` | Low/collapsing ratio signals push notification fatigue before the user disables permissions or uninstalls. |
| **Push Dismiss Ratio** | Communication Fatigue | $\frac{\text{notification\_dismiss}_{7d}}{\text{notification\_receive}_{7d} + \epsilon}$ | `notification_dismiss`, `notification_receive` | High manual dismissal rate indicates push messaging irrelevance to the active user. |
| **In-App Message Dismiss Rate** | Communication Fatigue | $\frac{\text{fiam\_dismiss}_{30d} + \text{firebase\_in\_app\_message\_dismiss}_{30d}}{\text{fiam\_impression}_{30d} + \text{firebase\_in\_app\_message\_impression}_{30d} + \epsilon}$ | `fiam_dismiss`, `fiam_impression`, `firebase_in_app_message_dismiss`, `firebase_in_app_message_impression` | Measures active banner/pop-up fatigue and reflexive in-app dismissals. |
| **App Exception Rate** | Technical Friction | $\frac{\text{app\_exception}_{30d}}{\text{session\_start}_{30d} + \epsilon}$ | `app_exception`, `session_start` | Involuntary churn trigger capturing app crashes and client-side unhandled errors per session. |
```

Idea: 

- gathering user engagement of click sessions 
- defining indicators


Step 2: Researching Models 

Status: Options Gathering

- Light GBM vs XGBoost?
