# Experiment Plan — Buddy-Personalize PR 1

## Hypothesis

Reframing `CONTENT_CONTEXT` from recommendation-mode to buddy-mode (own-show-watched + drop-off + buddy-first rules + cohort hint) will increase D1→D2 chat return rate by **3pp absolute on the Trial cohort within 14 days post-launch**.

**Mechanism:** Bot can acknowledge what the user has watched naturally, which increases user-side disclosure (the playbook's strongest depth correlate), which increases probability the user feels enough connection to return tomorrow.

## Metrics

### Primary
| Metric | Definition | Threshold |
|---|---|---|
| **D1→D2 return rate** | % of users chatting on D1 who also chat on D2 (IST), per cohort | TRT ≥ CTL + 3pp absolute, p<0.05 (one-sided) |

### Secondary (pre-registered, Bonferroni-corrected α=0.0125)
| Metric | Definition | Threshold |
|---|---|---|
| Own-show mention rate | Bot turns mentioning character's own show title in first 10 turns of users who watched it ≥60% | ≥30% absolute (baseline ~0%) |
| User disclosure rate | % of users with ≥1 playbook disclosure regex (`achha lagta`, `akela`, `family`, `papa`, `dil ki baat`) within first 20 turns | ≥1.5× CTL |
| Avg user msgs/session D1 | Median user-sent messages per first-day session | ≥1.2× CTL |

### Guardrails (must not regress)
| Metric | Threshold | Action if breach |
|---|---|---|
| Hallucination rate | <5% absolute (TRT mentions of unwatched shows / total show mentions) | Investigate at <5%, rollback if ≥10% for 2 days |
| Creepiness flag rate | <0.5% absolute (manual audit, double-coded) | Rollback if ≥1% in any day |
| Latency p95 | TRT ≤ CTL + 50ms | Investigate; check Redis cache hit |
| Token cost per session | TRT ≤ CTL + 5% | Trim cohort hint if needed |
| Card emission rate | TRT within ±30% of CTL | Investigate prompt drift if outside |

## Treatment

| Variable | Control | Treatment |
|---|---|---|
| GB flag `character_chat_buddy_personalization_v1` | false | true |
| `CONTENT_CONTEXT` block shape | Current (recommendation-flavored) | NEW (buddy-first with own-show, drop-off, cohort hint, two-section rules) |
| Everything else | Unchanged | Unchanged |
| Carousel infra | Active | Active |

## Randomization

- Unit: user_id (sticky)
- Method: GrowthBook deterministic hash
- Pre-stratification: by lifecycle cohort (Trial/NewSub/Renewer/Non-trial)
- Exclusions: Stage employees (allowlist)
- No-overlap: orthogonal to `character_chat_content_recommendation_v2`

## Sample size

Power calc (α=0.05, β=0.20, baseline Trial D2 ≈ 5.13%):
- Minimum detectable lift = 3pp → ~8,500 users/arm
- Trial daily chatter volume ≈ 15K → ramp Stage 2 (25%) reaches power in ~3 days
- Each stage runs ≥3 days regardless of statistical power

## Ramp schedule

| Stage | Audience | % | Min duration | Statistical gate | Operational gate |
|---|---|---:|---:|---|---|
| 0 | Stage employees (allowlist) | 100% | 24h | n/a | 30 test sessions across 8 scenarios pass internal QA |
| 1 | Trial only | 5% | 3 days | Primary trending +ve or null, no breach | All guardrails green |
| 2 | Trial only | 25% | 4 days | Primary p<0.10, secondaries ≥1.3× | All guardrails green |
| 3 | Trial + NewSub + Non-trial | 50% | 7 days | Primary ≥2pp lift OR secondaries ≥1.3× | All guardrails green, no cohort regresses >2pp |
| 4 | All cohorts | 100% | stable | — | — |

Total: ~3 weeks employee-to-100%.

## Stopping rules

- **Early harm:** any guardrail ≥2× threshold for 2 consecutive days → Redis kill switch within 30 min; root cause within 24h
- **Early win:** primary ≥5pp lift p<0.01 + all guardrails green for 5 days → ramp to next stage
- **Inconclusive at 14 days:** ±1.5pp on primary, overlapping CIs → ship if secondaries green; iterate prompt via Iteration Agent if all flat

## Statistical method

- **Primary:** Two-proportion z-test, one-sided (testing for improvement). Confidence intervals reported alongside.
- **Secondaries:** Welch's t-test (continuous) or two-proportion z (rates). Bonferroni-corrected α=0.0125.
- **Subgroups:** Sequential testing (Pocock boundaries) — flag if any subgroup shows >2× others.

## Subgroup analyses (pre-registered)

| Slice | What we test |
|---|---|
| Cohort | Trial vs NewSub vs Renewer vs Non-trial — different cohorts may respond differently |
| Own-show status | YES vs STARTED vs NO — YES bucket should show the largest treatment effect |
| Dialect | Rajasthani / Haryanvi / Bhojpuri / Gujarati / Marathi — no dialect-specific regression |
| Recency of first watch | <1 day / 1-7 days / >7 days — recency may moderate response |
| Days since app installed | New users (<7 days) vs tenured (≥30 days) |

## Pre-mortem — failure modes

| Failure mode | Detection | Action |
|---|---|---|
| Primary flat across all cohorts | Day 7 check, p>0.5 CI spans 0 | Move to PR #2; investigate via Iteration Agent |
| Own-show mention rate stays low | Day 3 check | Prompt-Craft Agent iteration |
| Hallucination ≥5% | Audit at 24h, 72h | Rollback, tighten "only mention this list" rule |
| Latency +>50ms | Real-time monitoring | Redis cache hit rate check |
| Cost +>5% | Daily cost dashboard | Trim block to ~330 tokens; cohort hint cuttable |
| One cohort regresses while others lift | Subgroup analysis at 50% | Rollback that cohort via GB targeting |
| Carousel CTR drops >30% | Daily metric | Verify recommendation rules still trigger |

## Post-experiment

- **Win:** Ramp to 100%, archive results, ship PR #2 (Day-1 close)
- **Mixed:** Document cohort × treatment-effect heatmap. Iterate via Iteration Agent. Likely keep change but adjust per-cohort phrasing in PR 1.1
- **Loss:** Rollback. Decide: re-craft prompt (PR 1.1) or pivot to PR #2 (Day-1 close is the bigger lever)

## Reproducibility

All analysis SQL templates in `/tmp/buddy_rollout_metrics_<date>.csv` (output by Rollout Monitor Agent). All audit sampling JSONs in `/tmp/audit-<date>.json` (output by Audit Agent). GrowthBook assignments visible to anyone with GB access.
