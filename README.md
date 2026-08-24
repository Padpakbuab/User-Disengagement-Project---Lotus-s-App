# User-Disengagement-Project---Lotus-s-App
!!Confidential Data Source!! This is just a journey of my research

Step 1: Defining Data

Look into Lotus's App Data bucket and see what I can play with.

1.1 Look into Customer Journey 

Key Event:

App Open > App usage > App Leave 



| Entity / Table | Granularity | Required Columns / Fields | Purpose |
| :--- | :--- | :--- | :--- |
| **1. User Profile Dimension** | 1 row per `user_id` | • `user_id`<br>• `signup_date`<br>• `home_store_id`<br>• `primary_delivery_province`<br>• `opt_in_push_notification`<br>• `account_tier` | Controls for user tenure, geography, channel permissions, and baseline segmentation. |
| **2. App Clickstream & Sessions** | 1 row per event / session | • `user_id`<br>• `session_id`<br>• `event_timestamp`<br>• `event_name` (`app_open`, `search`, `product_view`, `add_to_cart`, `checkout_start`)<br>• `device_os`<br>• `app_version` | Captures digital engagement velocity, browse depth, funnel friction, and app version stability. |
| **3. Omnichannel Transactions** | 1 row per order item / order | • `order_id`<br>• `user_id`<br>• `order_timestamp`<br>• `channel` (`ONLINE_DELIVERY`, `CLICK_COLLECT`, `IN_STORE_POS`)<br>• `order_status` (`COMPLETED`, `CANCELLED`, `RETURNED`)<br>• `net_amount_paid`<br>• `discount_amount`<br>• `product_category_id` | Tracks monetary velocity, channel mix shifts, basket shrinkage, and category-level disengagement. |
| **4. Loyalty & Promotions** | 1 row per reward interaction | • `user_id`<br>• `transaction_timestamp`<br>• `action_type` (`POINTS_EARNED`, `POINTS_BURNED`, `COUPON_COLLECTED`, `COUPON_REDEEMED`, `COUPON_EXPIRED`)<br>• `points_balance`<br>• `coupon_discount_value` | Measures promotion sensitivity, incentive fatigue, and reward wallet activity. |
| **5. Friction & Service Logs** | 1 row per friction event | • `user_id`<br>• `timestamp`<br>• `friction_type` (`PAYMENT_FAILED`, `APP_CRASH`, `ORDER_DELAYED`, `CS_TICKET_OPENED`)<br>• `error_code`<br>• `cs_category` | Captures involuntary churn drivers (payment failures, technical crashes, or logistics/CS complaints). |

Idea: 

- gathering user engagement of click sessions 
- defining indicators


Step 2: Researching Models 

Status: Options Gathering

- Light GBM vs XGBoost?
