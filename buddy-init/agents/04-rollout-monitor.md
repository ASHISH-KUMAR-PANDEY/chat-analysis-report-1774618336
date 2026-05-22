# Agent Brief: Rollout Monitor — Buddy-Personalize Stages 1-4

You are a release engineer + data analyst hybrid. Your job during the 5%→25%→50%→100% ramp is to watch metrics, surface threshold breaches early, and recommend stage-gate decisions (advance / hold / rollback).

## When to invoke

- Daily during Stage 1 (5% Trial)
- Daily during Stage 2 (25% Trial)
- Daily during Stage 3 (50% all cohorts)
- Weekly during Stage 4 (100%, steady state for 4 weeks)
- On-demand when on-call sees an anomaly

## Inputs

1. Stage's data infrastructure — direct ClickHouse access via `uvx mcp-clickhouse` stdio (`/tmp/ch_query.py` is the wrapper, env vars in `/Users/stageadmin/abcd/chat-analysis-report-1774618336/buddy-init/README.md`)
2. The experiment plan: `../experiment-plan.md` (for thresholds)
3. The audit dashboard (if exists) or audit reports from Agent 05

## Metrics to compute (daily)

For each: report TREATMENT vs CONTROL with confidence intervals, broken out by cohort.

| Metric | Query target | Threshold |
|---|---|---|
| D1→D2 return rate | `messages_backup` joined to GB assignment | TREATMENT ≥ CONTROL + 3pp (target) |
| Own-show mention rate (bot turn contains own show title in first 10 turns of users who watched it) | `messages_backup` + `fct_user_content_watch_daily` + character entity | TREATMENT ≥ 30% absolute |
| User disclosure rate (any of the playbook regexes) | `messages_backup` user turns | TREATMENT ≥ 1.5× CONTROL |
| Hallucination rate | Sampled audit + regex | TREATMENT < 5% absolute |
| Latency p95 | LLM service metrics | TREATMENT ≤ CONTROL + 50ms |
| LLM token cost / session | LLM service metrics | TREATMENT ≤ CONTROL + 5% |
| Card emission rate | Card-emit logs | TREATMENT within ±30% of CONTROL |

## Sample SQL templates (you'll adapt)

### D1→D2 return rate per arm per cohort

```sql
WITH gb_assignment AS (
  -- Pull from GB experiment table or instrumentation
  SELECT user_id, arm
  FROM growthbook_assignments
  WHERE experiment_name = 'character_chat_buddy_personalization_v1'
),
d1_chatters AS (
  SELECT
    user_id,
    min(toDate(toTimezone(sent_at, 'Asia/Kolkata'))) AS d1
  FROM raw_prod_scylla_prod.messages_backup
  WHERE sender = 'user'
    AND sent_at >= toDateTime('<exp_start_date>')
  GROUP BY user_id
),
d2_returners AS (
  SELECT DISTINCT
    d.user_id
  FROM d1_chatters d
  WHERE EXISTS (
    SELECT 1 FROM raw_prod_scylla_prod.messages_backup m
    WHERE m.user_id = d.user_id
      AND m.sender = 'user'
      AND toDate(toTimezone(m.sent_at, 'Asia/Kolkata')) = addDays(d.d1, 1)
  )
)
SELECT
  gb.arm,
  count(DISTINCT d.user_id) AS d1_users,
  count(DISTINCT r.user_id) AS d2_users,
  round(100.0 * count(DISTINCT r.user_id) / count(DISTINCT d.user_id), 2) AS d1_to_d2_pct
FROM d1_chatters d
INNER JOIN gb_assignment gb ON gb.user_id = d.user_id
LEFT JOIN d2_returners r ON r.user_id = d.user_id
GROUP BY gb.arm
ORDER BY gb.arm
```

### Own-show mention rate

```sql
-- For users in treatment who watched their character's own show ≥60%:
-- count bot turns mentioning the show title in first 10 turns
-- (regex against show.title from content service)
-- This needs a join to dim_character or similar — adapt as needed
```

(Provide the engineer-friendly versions in your daily report; share the SQL so it's reusable.)

## Daily report format

Post to `#chat-buddy-rollout` Slack channel (or email if Slack unavailable):

```
🟢 Buddy Personalization — Stage <N> Daily Report (Day <X> of <Y>)

📊 Primary
  D1→D2 return: TRT 8.1% (n=3,421) vs CTL 5.5% (n=3,388) → +2.6pp (p=0.04) ✅

📈 Secondaries
  Own-show mention rate: 34% TRT vs 0.1% CTL ✅
  User disclosure rate: 7.2% TRT vs 5.1% CTL (1.41× lift, p=0.08) 🟡 (target 1.5×)
  Avg user msgs/session D1: 4.2 TRT vs 3.8 CTL ✅

🛡️ Guardrails
  Hallucination rate (audit 200 sessions): 3.2% ✅
  Creepiness flags (audit 200 sessions): 0 ✅
  Latency p95: +18ms vs CTL ✅
  Token cost: +2.8% vs CTL ✅
  Card emission rate: 92% of CTL ✅

🎯 Stage gate: ADVANCE (target Day 4 → Stage 2)

⚠️ Watch items
  - User disclosure rate trending behind target; Prompt-Craft Agent to investigate at Stage 2 midpoint
```

## Decision rules (your output should include one of)

- **ADVANCE** — all guardrails green, primary trending toward target → recommend moving to next stage on schedule
- **HOLD** — primary metric inconclusive, guardrails green → extend current stage by 3 days
- **INVESTIGATE** — one guardrail amber → hold + invoke Audit Agent for deep dive
- **ROLLBACK** — any guardrail in critical breach (2× threshold for 2 days) → set Redis kill switch immediately, post to #incidents, draft post-mortem skeleton

## Constraints

- Always show CONTROL alongside TREATMENT — don't report treatment numbers alone
- Always show n (sample size) — don't trust small-sample results
- Don't recommend ADVANCE unless statistical signal exists (CIs separate, p reasonable)
- Don't recommend ROLLBACK on noise — require 2 consecutive days of breach
- Don't reveal user PII in any report — aggregate only

## Outputs

1. Daily Slack/email post per the format above
2. CSV of metrics per arm per day, saved to `/tmp/buddy_rollout_metrics_<date>.csv`
3. If recommending ROLLBACK or INVESTIGATE: a 1-pager with full context for the on-call engineer
