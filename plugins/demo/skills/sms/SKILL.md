---
name: sms
description: Analyze Michaels SMS campaign performance for a given fiscal week. Includes total sends, clicks, click rate, subscriber count, and segment breakdown (Mass vs Journey/Trigger). Renders an HTML dashboard.
---

# Michaels SMS Analysis

## Step 0: Ask for the fiscal week

**Always ask first:**

> "Which fiscal week would you like to analyze? Please provide the fiscal week ID (e.g., `202620` for week 20 of fiscal year 2026)."

Substitute the user's input as `{input_week}` in the queries below.

---

## Queries (run all in parallel)

### Q1 — Overall Metrics
```sql
WITH TY_week_messages AS (
    SELECT DISTINCT message_name
    FROM mk_src.attentive_general_histunion a
    JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
    WHERE b.wk_idnt = '{input_week}'
      AND a.type = 'MESSAGE_RECEIPT'
      AND (lower(message_name) NOT LIKE '%welcome%'
           AND lower(message_name) NOT LIKE '%% off%'
           AND lower(message_name) NOT LIKE '%makerplacesellerevent%'
           AND lower(message_name) NOT LIKE '%makerplace%'
           AND upper(message_name) NOT LIKE '%_QUE%'
           AND upper(message_name) NOT LIKE '%_CAN%')
),
SMS_Raw AS (
    SELECT
        a.phone,
        a.message_name,
        CASE WHEN (upper(a.message_name) LIKE '%WALLET%'
                   OR lower(a.message_name) LIKE '%journey%'
                   OR lower(a.message_name) LIKE '%abandon%')
             THEN 'Journey/Trigger'
             ELSE 'Mass'
        END AS message_segment,
        COUNT(DISTINCT CASE WHEN a.type = 'MESSAGE_RECEIPT' THEN a.timestamp END) AS sends,
        COUNT(DISTINCT CASE WHEN a.type = 'MESSAGE_LINK_CLICK' THEN a.timestamp END) AS clicks
    FROM mk_src.attentive_general_histunion a
    JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
    JOIN TY_week_messages wm ON a.message_name = wm.message_name
    WHERE b.wk_idnt = '{input_week}'
    GROUP BY 1, 2, 3
),
segment_summary AS (
    SELECT
        message_segment,
        SUM(sends) AS total_sends,
        SUM(clicks) AS total_clicks,
        ROUND(100.0 * SUM(clicks) / NULLIF(SUM(sends), 0), 2) AS click_rate_pct
    FROM SMS_Raw
    GROUP BY 1
),
total_summary AS (
    SELECT
        SUM(total_sends) AS total_sends,
        SUM(total_clicks) AS total_clicks,
        ROUND(100.0 * SUM(total_clicks) / NULLIF(SUM(total_sends), 0), 2) AS click_rate_pct,
        COUNT(DISTINCT phone) AS unique_recipients
    FROM (SELECT phone, SUM(sends) AS total_sends, SUM(clicks) AS total_clicks FROM SMS_Raw GROUP BY 1) t
)
SELECT 'TOTAL' AS segment, total_sends, total_clicks, click_rate_pct, unique_recipients
FROM total_summary
UNION ALL
SELECT message_segment, total_sends, total_clicks, click_rate_pct, NULL
FROM segment_summary
ORDER BY segment;
```

### Q2 — Subscriber Count (as of this week)
```sql
SELECT COUNT(DISTINCT a.phone) AS total_subscribers
FROM mk_src.attentive_optstatus_histunion a
JOIN mk_src.bq_date_dim b ON substring(a.timestamp,1,10) = b.day_dt
WHERE CAST(b.wk_idnt AS INT) <= CAST('{input_week}' AS INT)
  AND opt_in_status = 'JOIN';
```

---

## Dashboard Output

Render a clean single-page HTML dashboard with:

**Header:** "SMS Performance — Fiscal Week {input_week}"

**KPI Cards (row of 4):**
- Total Sends
- Total Clicks
- Click Rate (%)
- Total Subscribers

**Segment Breakdown Table:**

| Segment | Sends | Clicks | Click Rate |
|---------|-------|--------|------------|
| Mass | ... | ... | ...% |
| Journey/Trigger | ... | ... | ...% |
| **Total** | ... | ... | ...% |

**Bar Chart:** Sends vs Clicks side-by-side for Mass and Journey/Trigger segments.

**Design:** Professional, clean, minimal. Neutral palette. Use inline CSS — no external dependencies.

