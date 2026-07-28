---
name: sms
description: Analyze Michaels SMS campaign performance for a specific campaign. Includes sends, clicks, click rate, click-attributed revenue and transactions, and baseline comparison vs similar-size campaigns in the last 90 days. Renders an HTML dashboard.
---

# Michaels SMS Campaign Analysis

---

## Step 0: Campaign Selection (ALWAYS run first)

**Ask the user:**

> "Which SMS campaign would you like to analyze?
> - **Name a campaign directly** — paste the message name (e.g. `MASS_SummerClearanceEndingSoon`)
> - **Show me recent campaigns** — I'll pull the last 15 mass SMS campaigns with sends and clicks"

**This skill analyzes one campaign at a time.**

### If user wants recent campaigns:
```sql
SELECT
    message_name,
    MIN(substring(timestamp,1,10)) AS send_date,
    COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS sends,
    COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END) AS clicks,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END)
        / NULLIF(COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END), 0), 2) AS click_rate_pct
FROM mk_src.attentive_general_histunion
WHERE type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
  AND (upper(message_name) LIKE 'MASS%' OR upper(message_name) LIKE 'PROMO%')
  AND lower(message_name) NOT LIKE '%welcome%'
  AND lower(message_name) NOT LIKE '%makerplace%'
  AND upper(message_name) NOT LIKE '%_QUE%'
  AND upper(message_name) NOT LIKE '%_CAN%'
GROUP BY 1
ORDER BY send_date DESC
LIMIT 15;
```

Present the list and ask: **"Which campaign would you like to analyze?"**

---

## Step 1: Resolve campaign date range

Get the send window for `{input_campaign}` — used to anchor the attribution window:

```sql
SELECT
    MIN(substring(timestamp,1,10)) AS send_start,
    MAX(substring(timestamp,1,10)) AS send_end
FROM mk_src.attentive_general_histunion
WHERE message_name = '{input_campaign}'
  AND type = 'MESSAGE_RECEIPT';
```

Store `send_start` and `send_end`. The 7-day attribution window runs from `send_start` to `send_start + 7 days`.

---

## Step 2: Run all queries in parallel

### Q1 — Campaign Engagement Metrics

```sql
WITH campaign_raw AS (
    SELECT
        phone,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN timestamp END) AS sends,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN timestamp END) AS clicks,
        MIN(CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN CAST(substring(timestamp,1,10) AS DATE) END) AS first_click_date
    FROM mk_src.attentive_general_histunion
    WHERE message_name = '{input_campaign}'
      AND type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
    GROUP BY 1
)
SELECT
    COUNT(DISTINCT phone) AS unique_recipients,
    SUM(sends) AS total_sends,
    SUM(clicks) AS total_clicks,
    COUNT(DISTINCT CASE WHEN clicks > 0 THEN phone END) AS unique_clickers,
    ROUND(100.0 * SUM(clicks) / NULLIF(SUM(sends), 0), 2) AS click_rate_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN clicks > 0 THEN phone END)
        / NULLIF(COUNT(DISTINCT phone), 0), 2) AS unique_click_rate_pct
FROM campaign_raw;
```

### Q2 — Click-Attributed Revenue (7-day window)

Uses the attribution logic: clicker phones → `enrich_attentive_optstatus` → `crafter_id` → `enrich_resa_transaction_head_fact` within 7 days of first click.

```sql
WITH clickers AS (
    SELECT phone, MIN(CAST(time AS BIGINT)) AS first_click_time
    FROM mk_src.attentive_general_histunion
    WHERE message_name = '{input_campaign}'
      AND type = 'MESSAGE_LINK_CLICK'
      AND substring(timestamp,1,10) BETWEEN '{send_start}' AND '{send_end}'
    GROUP BY 1
),
clicker_crafters AS (
    SELECT DISTINCT c.phone, c.first_click_time, o.crafter_id
    FROM clickers c
    JOIN cdp_unification_mk.enrich_attentive_optstatus o
      ON c.phone = o.phone
    WHERE o.crafter_id IS NOT NULL AND o.crafter_id <> ''
      AND o.opt_in_status = 'join'
)
SELECT
    COUNT(DISTINCT cc.crafter_id) AS customers,
    COUNT(DISTINCT t.transaction_id) AS transactions,
    ROUND(SUM(t.trans_total_amt), 2) AS revenue,
    ROUND(SUM(t.trans_total_amt) / NULLIF(COUNT(DISTINCT t.transaction_id), 0), 2) AS aov,
    ROUND(SUM(t.trans_total_amt) / NULLIF(COUNT(DISTINCT cc.crafter_id), 0), 2) AS rev_per_clicker
FROM clicker_crafters cc
JOIN cdp_unification_mk.enrich_resa_transaction_head_fact t
  ON cc.crafter_id = t.crafter_id
WHERE t.time BETWEEN cc.first_click_time AND cc.first_click_time + 7*86400
  AND t.trans_total_amt > 0;
```

> **Attribution:** 7-day post first-click window. Identity bridge via `enrich_attentive_optstatus` (opt-in only). Single-touch last-click. `trans_total_amt > 0` excludes returns.

### Q3 — Baseline: Similar Send-Size Campaigns (last 90 days)

Finds campaigns with similar audience size (within ±30% of this campaign's sends) sent in the last 90 days, excluding the current campaign. Uses these for baseline click rate, click-attributed revenue, and AOV comparisons.

```sql
WITH this_campaign AS (
    SELECT
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS this_sends,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END) AS this_clicks
    FROM mk_src.attentive_general_histunion
    WHERE message_name = '{input_campaign}'
      AND type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
),
peer_sends AS (
    SELECT
        message_name,
        MIN(substring(timestamp,1,10)) AS send_date,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_RECEIPT' THEN phone END) AS sends,
        COUNT(DISTINCT CASE WHEN type = 'MESSAGE_LINK_CLICK' THEN phone END) AS clicks
    FROM mk_src.attentive_general_histunion
    WHERE type IN ('MESSAGE_RECEIPT','MESSAGE_LINK_CLICK')
      AND message_name != '{input_campaign}'
      AND (upper(message_name) LIKE 'MASS%' OR upper(message_name) LIKE 'PROMO%')
      AND lower(message_name) NOT LIKE '%welcome%'
      AND lower(message_name) NOT LIKE '%makerplace%'
      AND upper(message_name) NOT LIKE '%_QUE%'
      AND upper(message_name) NOT LIKE '%_CAN%'
      AND substring(timestamp,1,10) >= CAST(DATE_ADD('day', -90, DATE('{send_start}')) AS VARCHAR)
      AND substring(timestamp,1,10) <= '{send_end}'
    GROUP BY 1
),
peers AS (
    SELECT ps.*
    FROM peer_sends ps
    CROSS JOIN this_campaign tc
    WHERE ps.sends BETWEEN tc.this_sends * 0.7 AND tc.this_sends * 1.3
      AND ps.sends > 0
),
peer_revenue AS (
    SELECT
        p.message_name,
        p.sends,
        p.clicks,
        ROUND(100.0 * p.clicks / NULLIF(p.sends, 0), 2) AS click_rate_pct,
        rev.revenue,
        rev.customers,
        rev.transactions,
        ROUND(rev.revenue / NULLIF(rev.customers, 0), 2) AS rev_per_clicker,
        ROUND(rev.revenue / NULLIF(rev.transactions, 0), 2) AS aov
    FROM peers p
    LEFT JOIN (
        SELECT
            a_clickers.message_name,
            COUNT(DISTINCT cc.crafter_id) AS customers,
            COUNT(DISTINCT t.transaction_id) AS transactions,
            ROUND(SUM(t.trans_total_amt), 2) AS revenue
        FROM (
            SELECT
                message_name,
                phone,
                MIN(CAST(time AS BIGINT)) AS first_click_time
            FROM mk_src.attentive_general_histunion
            WHERE type = 'MESSAGE_LINK_CLICK'
              AND message_name IN (SELECT message_name FROM peers)
            GROUP BY 1, 2
        ) a_clickers
        JOIN cdp_unification_mk.enrich_attentive_optstatus o
          ON a_clickers.phone = o.phone
         AND o.crafter_id IS NOT NULL AND o.crafter_id <> ''
         AND o.opt_in_status = 'join'
        JOIN cdp_unification_mk.enrich_resa_transaction_head_fact t
          ON o.crafter_id = t.crafter_id
         AND t.time BETWEEN a_clickers.first_click_time AND a_clickers.first_click_time + 7*86400
         AND t.trans_total_amt > 0
        GROUP BY 1
    ) rev ON p.message_name = rev.message_name
)
SELECT
    COUNT(*) AS peer_campaign_count,
    APPROX_PERCENTILE(click_rate_pct, 0.5) AS median_click_rate_pct,
    AVG(click_rate_pct) AS avg_click_rate_pct,
    APPROX_PERCENTILE(COALESCE(revenue, 0), 0.5) AS median_revenue,
    AVG(COALESCE(revenue, 0)) AS avg_revenue,
    APPROX_PERCENTILE(COALESCE(rev_per_clicker, 0), 0.5) AS median_rev_per_clicker,
    AVG(COALESCE(rev_per_clicker, 0)) AS avg_rev_per_clicker,
    APPROX_PERCENTILE(COALESCE(aov, 0), 0.5) AS median_aov,
    AVG(COALESCE(aov, 0)) AS avg_aov
FROM peer_revenue;
```

> **Baseline peer logic:** Mass/Promo campaigns only, last 90 days, within ±30% of this campaign's send volume. Click-attributed revenue computed with identical 7-day attribution window.

---

## Step 3: Dashboard Output

Render a self-contained single-page HTML dashboard.

**Header:** "SMS Campaign Analysis — {input_campaign}" with send date below.

### Tab 1 — Overview

**KPI row (6 tiles):**
- Total Sends
- Unique Clickers
- Click Rate (unique clickers / sends, %)
- Click-Attributed Revenue
- Transactions
- AOV

**Engagement vs Baseline comparison table:**

| Metric | This Campaign | Baseline Median | vs Baseline |
|--------|---------------|-----------------|-------------|
| Click Rate | x% | x% | +/- bps |
| Revenue | $x | $x | +/- % |
| Rev / Clicker | $x | $x | +/- % |
| AOV | $x | $x | +/- % |

Show baseline peer count (e.g. "vs 8 similar campaigns in last 90 days").

> **Delta display:** Click rate delta in bps (basis points). Revenue/AOV/Rev per clicker deltas as %.

### Tab 2 — Baseline Detail

Show each peer campaign as a row:

| Campaign | Send Date | Sends | Click Rate | Revenue | AOV |
|----------|-----------|-------|------------|---------|-----|
| ...      | ...       | ...   | ...%       | $...    | $.. |

Sorted by send date descending. Helps the user see which peer campaigns were included.

---

## Calculated Metrics

- **Click Rate:** `Unique Clickers / Sends`
- **Conv Rate:** `Customers (buyers) / Unique Clickers`
- **Rev / Clicker:** `Revenue / Unique Clickers`
- **AOV:** `Revenue / Transactions`
- **bps delta:** `(campaign% − baseline%) × 100`
- **% delta:** `(campaign − baseline) / baseline × 100`

---

## Data Tables Reference

| Table | Purpose |
|---|---|
| `mk_src.attentive_general_histunion` | Raw SMS events — sends, clicks. Filter on `message_name` + `type`. |
| `mk_src.attentive_optstatus` | Subscriber opt-in status and current subscriber count. |
| `cdp_unification_mk.enrich_attentive_optstatus` | Phone → `crafter_id` bridge. Use `opt_in_status = 'join'` only. |
| `cdp_unification_mk.enrich_resa_transaction_head_fact` | Transaction header. Has `crafter_id`, `time` (Unix epoch), `trans_total_amt`, `transaction_id`. |
| `cdp_unification_mk.bq_date_dim` | Calendar / fiscal week dimension. |

> **Attribution window:** 7 days post first click (Unix epoch: `first_click_time + 7*86400`). Exclude returns with `trans_total_amt > 0`.
