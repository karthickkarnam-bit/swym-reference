# Swym Reference – Tables & Columns (Query Building)

This repository documents commonly used Swym warehouse tables/columns referenced in wishlist adoption and engagement queries.

## Date range convention used in queries

Use an inclusive start date and exclusive end date:

- `>= YYYY-MM-01`
- `<  YYYY-MM-01` of next month

Example (Mar 2026):

- `>= '2026-03-01'` and `< '2026-04-01'`

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

### `swymbi.swym_sessions_summary`
Merchant-level sessions aggregated by day (Swym-specific).

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `summarization_date` | date | Aggregation date |
| `session_count` | integer | Total sessions |
| `purchase_sessions_count` | integer | Sessions that resulted in purchase (if applicable) |

### `swymbi.site_unique_sessions`
Session-level table useful for deduping/unique user analysis.

| column | type | notes |
|---|---|---|
| `merchant_id` | text | Merchant identifier |
| `session_date` | date | Session date |
| `session_id` | text | Session identifier |
| `device_id` | text | Device identifier |
| `user_id` | text | User identifier |

> Important: none of the session tables above include `merchant_pid` (product id). Any “PID-level sessions” metric cannot be computed from these tables alone.

## Wishlist actions tables

### `swymbi.wishlist_items` (event-level)
Used to compute wishlist actions at merchant + PID level.

Commonly referenced columns (from existing queries):

- `merchant_id`
- `merchant_pid`
- `add_date`
- `source` (example: `'auto-wishlist'`)
- `channel` (example: `'customer-accounts'`)

**Typical usage (PID-level actions)**

```sql
SELECT merchant_id, merchant_pid, COUNT(*) AS total_wishlist_actions
FROM swymbi.wishlist_items
WHERE add_date >= '2026-03-01'
  AND add_date <  '2026-04-01'
GROUP BY merchant_id, merchant_pid;
```

### `swymbi.wishlist_items_summary` (merchant-level daily/monthly summary)
Used in merchant-level action qualification.

Commonly referenced columns:

- `merchant_id`
- `summarization_date`
- `wishlist_items_count`

### `swymbi.save_for_later_items_summary`
Used to include Save-for-Later actions in merchant-level action totals.

Commonly referenced columns:

- `merchant_id`
- `summarization_date`
- `items_count`

## Merchant + install metadata

### `swymrevenue.merchant_app_installs`
Used for active install window checks.

Commonly referenced columns:

- `platform_url`
- `app_name`
- `is_shop_closed`
- `install_date`
- `uninstall_date`
- `last_activated_plan`

### `swymbi.merchants`
Used for Shopify plan enrichment.

Commonly referenced columns:

- `merchant_id`
- `platform_plan`

### `merchants`
Used to map platform URL to merchant id (as used in existing queries).

Commonly referenced columns:

- `merchant_id`
- `platform_url`

## Feature configuration (Wishlist)

### `swymtelemetry.feature_state`
Feature enablement flags.

Commonly referenced columns:

- `merchant_id`
- `app_name`
- `feature_name`
- `is_enabled`
- `created_at`
- `expired_at`

### `swymtelemetry.feature_attribute`
Feature attributes/configuration values.

Commonly referenced columns:

- `merchant_id`
- `app_name`
- `feature_name`
- `attribute_name`
- `attribute_value`
- `created_at`
- `expired_at`

## Notes on sessions-to-wishlist-actions ratio

If you compute wishlist actions at **PID level** but sessions only exist at **merchant level**, then:

- `sessions_to_wishlist_actions_ratio = merchant_sessions_in_month / pid_wishlist_actions_in_month`

This is useful as a proxy, but it is not a true product-level conversion rate.