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

```python
# 1. Digital Activity & Engagement Velocity
engagement_decay_ratio = (
    (count('screen_view', window='7d') / 7.0) / 
    ((count('screen_view', window='30d') / 30.0) + 1e-5)
)
browse_breadth_7d = (
    (count('view_item', window='7d') + count('view_item_list', window='7d')) / 
    (count('session_start', window='7d') + 1e-5)
)

# 2. Purchase Funnel & Cart Velocity
cart_abandonment_rate_30d = (
    1.0 - (count('purchase', window='30d') / (count('add_to_cart', window='30d') + 1e-5))
)
cart_purge_ratio_30d = (
    (count('remove_from_cart', window='30d') + count('trash_icon', window='30d')) / 
    (count('add_to_cart', window='30d') + 1e-5)
)
checkout_dropoff_velocity_7d = (
    count('checkout_coupon_leave', window='7d') / 
    (count('cart_proceed_to_checkout', window='7d') + 1e-5)
)
checkout_stall_rate_7d = (
    1.0 - (count('purchase', window='7d') / (count('cart_proceed_to_checkout', window='7d') + 1e-5))
)
refund_propensity_30d = (
    count('refund', window='30d') / (count('purchase', window='30d') + 1e-5)
)

# 3. Loyalty, Coins & Promotion Dynamics
coin_voucher_utilization_rate_30d = (
    count('redeem_coins_confirm', window='30d') / 
    (count('coupon_apply', window='30d') + 1e-5)
)
coin_redemption_velocity_30d = (
    count('redeem_coins_confirm', window='30d') / 
    (count('enable_use_lotuss_point', window='30d') + 1e-5)
)
coupon_abandonment_rate_30d = (
    (count('checkout_coupon_leave', window='30d') + count('coupon_cancel', window='30d')) / 
    (count('checkout_coupon_select', window='30d') + 1e-5)
)

# 4. Push & In-App Notification Receptivity
notification_receptivity_ratio_7d = (
    count('notification_open', window='7d') / 
    (count('notification_receive', window='7d') + 1e-5)
)
push_dismiss_ratio_7d = (
    count('notification_dismiss', window='7d') / 
    (count('notification_receive', window='7d') + 1e-5)
)
in_app_banner_dismiss_rate_30d = (
    (count('fiam_dismiss', window='30d') + count('firebase_in_app_message_dismiss', window='30d')) / 
    (count('fiam_impression', window='30d') + count('firebase_in_app_message_impression', window='30d') + 1e-5)
)

# 5. App Stability & Crash Telemetry
app_exception_rate_30d = (
    count('app_exception', window='30d') / 
    (count('session_start', window='30d') + 1e-5)
)
```

Idea: 

- gathering user engagement of click sessions 
- defining indicators


Step 2: Researching Models 

Status: Options Gathering

- Light GBM vs XGBoost?
