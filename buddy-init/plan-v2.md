# Buddy-Personalize PR 1 — Plan v2

**TL;DR:** Reword the existing `CONTENT_CONTEXT` block in the character chat system prompt from recommendation-flavored to buddy-flavored. Surface 4 new derived signals (own-show watched status, drop-off flag, cohort hint, taste pattern). Single PR, GB-gated, A/B tested.

## Background — why "v2"

v1 of this plan assumed greenfield. After a code archaeology pass we discovered watch history is ALREADY in the prompt via `ContentSuggestionService.buildContentContext()` — but it's framed for content recommendation, not buddy bonding. v2 is a surgical reframing, not a new feature.

## Problem

Bot prompt contains user's watched titles + genres + recommendation candidates. Bot is instructed to use this for `[[RECOMMEND:Title]]` cards. Bot is NOT instructed to use this for relationship continuity. Most personal data point — "did the user watch THIS character's own show" — is not surfaced explicitly.

## Solution: 4 changes in `ContentSuggestionService.buildContentContext`

### Change A — Surface own-show-watched status

New `**ABOUT YOUR OWN SHOW**` block at top of CONTENT_CONTEXT:
- `YES` if user watched ≥60% of character's own show
- `STARTED` if 5–60%
- `NO` if <5% or not in history

### Change B — Drop-off flags on non-own shows
- `completed` (≥60%), `dropped` (5–60%), `sampled` (≤5%)

### Change C — Reframe RULES section: buddy-first, recommendation-second
Two-section block. Buddy mode first (reference titles casually, acknowledge own-show, soft callback on drop-off, never act surprised, never quote watch %). Recommendation mode second (existing rules unchanged).

### Change D — Add cohort hint at block top
`**USER COHORT:** <trial_active | subscription_active | non_subscriber | …>`. Pulls from `getUserSubscriptionMandateStatus()` already in scope.

## Scope guard

In:
- `src/chat/services/content-suggestion.service.ts` (main modifications)
- `src/chat/services/content-suggestion.service.spec.ts` (new tests)
- Optional `src/chat/services/prompt.service.ts` (cohort plumbing if needed)
- New GrowthBook flag `character_chat_buddy_personalization_v1`
- New Redis kill switch `chat:buddy_personalize:killed`

Out (separate PRs):
- Day-1 close return-invitation rule (PR 2)
- First-10-turn confidentiality framing + active-listening rules (PR 3)
- Day-N callback memory layer (PR 4)
- Carousel UI changes
- New PII fields in prompt
- LLM model/config changes

## Token budget impact

`CONTENT_CONTEXT` cap: 400 tokens. New block adds ~80 tokens worst-case. Stays in budget. `PromptBudgetService.truncateToTokens` handles overflow defensively.

## Metrics summary (full details in experiment-plan.md)

- **Primary:** D1→D2 return rate, +3pp absolute (Trial cohort)
- **Secondary:** own-show mention rate ≥30%, user disclosure rate ≥1.5×, avg user msgs ≥1.2×
- **Guardrails:** hallucination <5%, creepiness <0.5%, latency unchanged, token cost +<5%, card emission within ±30%

## Rollout plan (full schedule in experiment-plan.md)

| Stage | Audience | % | Days | Gate |
|---|---|---:|---:|---|
| 0 | Stage employees | n/a | 1 | Internal QA pass |
| 1 | Trial | 5% | 3 | Guardrails green, primary trending |
| 2 | Trial | 25% | 4 | Primary p<0.10, secondaries ≥1.3× |
| 3 | All cohorts | 50% | 7 | Primary ≥2pp lift |
| 4 | All cohorts | 100% | stable | — |

Total: ~3 weeks employee-to-100%.

## Roadmap — this PR in context

| PR | Focus | Effort | Status |
|---|---|---|---|
| **1 (this)** | Buddy-personalize CONTENT_CONTEXT | 2 days | Planned |
| 2 | Day-1 close return-invitation prompt rule | 1 day | Queued |
| 3 | First-10-turn confidentiality + active-listening rules | 1 day | Queued |
| 4 | Day-N memory layer (one disclosed fact per user-character) | 5 days | Queued |

## Open question

**Should cohort hint ship in v1 or hold for v1.1?** Recommendation: ship in v1. Cost is +10 tokens, value is meaningful (Trial vs Renewer phrasing divergence is a playbook finding).

## Approval

PM: Ashish
Eng: TBD
Data: TBD
Date approved: __________

## Implementation kickoff

Once approved, invoke `Implementation Agent` (`agents/01-implementation.md`).
