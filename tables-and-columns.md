# Swym Reference – Tables & Columns (Query Building)

This repository documents commonly used Swym warehouse tables/columns referenced in wishlist adoption and engagement queries.

> **How to use:** Every time a new reference query is shared, this file is updated with any new tables and columns found. When building a new query, refer to this file for correct table and column names.

---

## Date range convention used in queries

Use an inclusive start date and exclusive end date:

- `>= YYYY-MM-01`
- `<  YYYY-MM-01` of next month

Example (Feb 2026):
- `>= '2026-02-01'` and `< '2026-03-01'`

Example (Mar 2026):
- `>= '2026-03-01'` and `< '2026-04-01'`

---

## Sessions tables

### `swymbi.site_sessions_summary`
Merchant-level sessions aggregated by day.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `summarization_date` | date | Aggregation date |
| `session_count` | integer | Total sessions for the merchant on that date |

**Typical usage**

```sql
SELECT merchant_id, SUM(session_count) AS total_sessions
FROM swymbi.site_sessions_summary
WHERE summarization_date >= '2026-03-01'
  AND summarization_date <  '2026-04-01'
GROUP BY merchant_id;
```

---

### `swymbi.swym_sessions_summary`
Merchant-level sessions aggregated by day (Swym-specific).

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `summarization_date` | date | Aggregation date |
| `session_count` | integer | Total sessions |
| `purchase_sessions_count` | integer | Sessions that resulted in purchase (if applicable) |

---

### `swymbi.site_unique_sessions`
Session-level table useful for deduping/unique user analysis.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `session_date` | date | Session date |
| `session_id` | text | Session identifier |
| `device_id` | text | Device identifier |
| `user_id` | text | User identifier |

> **Important:** None of the session tables above include `merchant_pid`. Any "PID-level sessions" metric cannot be computed from these tables alone.

---

## Wishlist actions tables

### `swymbi.wishlist_items` (event-level)
Used to compute wishlist actions at merchant + PID level.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `merchant_pid` | text | Product identifier |
| `add_date` | date | Date the item was wishlisted |
| `source` | text | e.g. `'auto-wishlist'` |
| `channel` | text | e.g. `'customer-accounts'` |

**Typical usage (PID-level actions)**

```sql
SELECT merchant_id, merchant_pid, COUNT(*) AS total_wishlist_actions
FROM swymbi.wishlist_items
WHERE add_date >= '2026-03-01'
  AND add_date <  '2026-04-01'
GROUP BY merchant_id, merchant_pid;
```

---

### `swymbi.wishlist_items_summary` (merchant-level daily/monthly summary)
Used in merchant-level action qualification.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `summarization_date` | date | Aggregation date |
| `wishlist_items_count` | integer | Total wishlist actions for the merchant on that date |

---

### `swymbi.save_for_later_items_summary`
Used to include Save-for-Later actions in merchant-level action totals.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `summarization_date` | date | Aggregation date |
| `items_count` | integer | Total save-for-later actions for the merchant on that date |

**Typical usage (combined wishlist + save for later)**

```sql
SELECT merchant_id, SUM(action_count) AS total_actions
FROM (
  SELECT merchant_id, summarization_date, wishlist_items_count AS action_count
  FROM swymbi.wishlist_items_summary
  UNION ALL
  SELECT merchant_id, summarization_date, items_count AS action_count
  FROM swymbi.save_for_later_items_summary
) x
WHERE summarization_date >= '2026-02-01'
  AND summarization_date <  '2026-03-01'
GROUP BY merchant_id;
```

---

## Merchant & install metadata

### `swymbi.merchants`
Core merchant table. Used for Shopify plan enrichment and email filtering.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `platform_url` | text | Store URL (use this for store URL — **not** `store_url`) |
| `platform_plan` | text | Shopify plan. Values: `'professional'`, `'unlimited'`, `'shopify_plus'` |
| `admin_email` | text | Admin email — exclude `%@swymcorp.com%` in queries |
| `wishlist_install_email` | text | Install email — exclude `%@swymcorp.com%` in queries |

> **Note:** Always use `m.platform_url AS store_url` for the store URL column. Do NOT use a column called `store_url` directly from this table.

---

### `swymrevenue.merchant_app_installs`
Used for install/uninstall date and active install window checks.

| column | type | notes |
|---|---|---|
| `platform_url` | text | Store URL — join key to `swymbi.merchants.platform_url` |
| `app_name` | text | e.g. `'Wishlist'` |
| `install_date` | date | Date the app was installed |
| `uninstall_date` | date | Date the app was uninstalled (NULL if still active) |
| `last_activated_plan` | text | Last activated Swym plan |
| `is_shop_closed` | boolean | Whether the shop is closed |

**Typical usage (installed on / uninstalled date)**

```sql
SELECT
  platform_url AS store_url,
  MIN(install_date)   AS installed_on,
  MAX(uninstall_date) AS uninstalled_date
FROM swymrevenue.merchant_app_installs
WHERE app_name = 'Wishlist'
GROUP BY platform_url;
```

---

### `swymrevenue.shopify_app_events`
Used to determine the current/latest Swym plan for a merchant.

| column | type | notes |
|---|---|---|
| `platform_url` | text | Store URL — join key |
| `app_name` | text | e.g. `'Wishlist'` |
| `app_plan` | text | Swym plan name. Values: `'Starter'`, `'Pro'`, `'Premium'`, `'Enterprise'` |
| `event_type` | text | e.g. `'SUBSCRIPTION_CHARGE_ACTIVATED'` |
| `occured_at` | timestamp | When the event occurred |
| `id` | text | Event/subscription ID |

**Typical usage (latest Swym plan)**

```sql
SELECT platform_url, app_plan AS swym_plan
FROM (
  SELECT platform_url, app_plan,
    ROW_NUMBER() OVER (PARTITION BY platform_url ORDER BY occured_at DESC) AS rn
  FROM swymrevenue.shopify_app_events
  WHERE app_name = 'Wishlist'
    AND event_type = 'SUBSCRIPTION_CHARGE_ACTIVATED'
) t
WHERE rn = 1;
```

---

### `swymrevenue.shopify_app_subscriptions`
Used to filter out trial subscriptions.

| column | type | notes |
|---|---|---|
| `id` | text | Subscription ID — join to `shopify_app_events.id` |
| `trial_days` | integer | Number of trial days (`0` or NULL = not a trial) |

---

## Feature configuration (Wishlist)

### `swymtelemetry.feature_state`
Feature enablement flags per merchant.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `app_name` | text | e.g. `'Wishlist'` |
| `feature_name` | text | Name of the feature |
| `is_enabled` | boolean | Whether the feature is enabled |
| `created_at` | timestamp | When the record was created |
| `expired_at` | timestamp | When the record expired (NULL if still active) |

---

### `swymtelemetry.feature_attribute`
Feature configuration values per merchant.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `app_name` | text | e.g. `'Wishlist'` |
| `feature_name` | text | Name of the feature |
| `attribute_name` | text | Configuration key |
| `attribute_value` | text | Configuration value |
| `created_at` | timestamp | When the record was created |
| `expired_at` | timestamp | When the record expired (NULL if still active) |

---

## Notes on sessions-to-wishlist-actions ratio

If you compute wishlist actions at **PID level** but sessions only exist at **merchant level**, then:

- `sessions_to_wishlist_actions_ratio = merchant_sessions_in_month / pid_wishlist_actions_in_month`

This is useful as a proxy, but it is not a true product-level conversion rate.

---

## Common filters applied in most queries

```sql
-- Exclude Swym internal stores
AND LOWER(COALESCE(m.admin_email, '')) NOT LIKE '%@swymcorp.com%'
AND LOWER(COALESCE(m.wishlist_install_email, '')) NOT LIKE '%@swymcorp.com%'

-- Shopify plan filter
AND m.platform_plan IN ('professional', 'unlimited', 'shopify_plus')

-- Swym paying plan filter
AND e.app_plan IN ('Starter', 'Pro', 'Premium', 'Enterprise')

-- Exclude trials
AND COALESCE(s.trial_days, 0) = 0
```