# Testing Plan — Buddy-Personalize PR 1

Four layers: unit, integration, manual QA, and continuous production audit.

## Layer 1: Unit tests (BLOCKING before merge)

File: `src/chat/services/content-suggestion.service.spec.ts`

| # | Test case | Setup | Expected |
|---|---|---|---|
| 1 | Own-show YES path | Watch history includes character's `content_slug` at 87% | Output contains `**ABOUT YOUR OWN SHOW** … WATCHED: YES (87%)` |
| 2 | Own-show STARTED path | Same slug at 42% | Output contains `STARTED but not finished (42%)` |
| 3 | Own-show NO path | Watch history without own-slug | Output contains `User has NOT watched it yet` |
| 4 | Drop-off flag on other shows | Other show at 30% | Output contains `(30% — started, dropped)` |
| 5 | Cold start | Empty watch history | Falls to `buildNewUserContext`; own-show NO block present |
| 6 | Heavy watcher | 50+ titles | List truncates to top 10; total ≤400 tokens |
| 7 | Null total_duration | Repo record has `total_duration=null` | `watchPercent`=0; marked "not watched" |
| 8 | Cohort hint Trial | `getUserSubscriptionMandateStatus` → `trial_active` | Output starts with `**USER COHORT:** trial_active` |
| 9 | Cohort hint Renewer | Returns `subscription_active` (renewer) | Output starts with `**USER COHORT:** subscription_active` |
| 10 | Cohort hint Non-trial | Returns `non_subscriber` | Output starts with `**USER COHORT:** non_subscriber` |
| 11 | GB flag OFF | `growthBookService.checkFeatureGate` returns false | Returns OLD prompt shape (no new fields) |
| 12 | Kill switch ON | Redis `chat:buddy_personalize:killed=1` | Returns OLD prompt shape regardless of GB |
| 13 | Redis cache hit | Pre-populate cache | Repo NOT called; returns cached value |
| 14 | Repo failure fails open | WatchVideoEventRepository throws | Empty context returned; no exception; logged |

Coverage target: 95%+ on `content-suggestion.service.ts` changes.

## Layer 2: Integration tests (BLOCKING before merge)

File: `test/chat/buddy-personalization.e2e-spec.ts`

| # | Scenario | Expected |
|---|---|---|
| 1 | Happy path: Trial user, 5 watched, own-show 87% | System prompt sent to LLM contains own-show block, drop-off flags, buddy rules, cohort hint = `trial_active` |
| 2 | Cohort routing: same for NewSub, Renewer, Non-trial | Cohort hint matches each |
| 3 | Cold start: new user, 0 watches | New-user block in prompt, no drop-off flags |
| 4 | GB variation: same user, treatment vs control | Control gets OLD block, treatment gets NEW block |
| 5 | Kill switch: Redis flag set | Both arms get OLD block |
| 6 | End-to-end with mock LLM: full chat round-trip | LLM call receives expected system prompt; bot response returned to user |

## Layer 3: Manual QA scenarios (BLOCKING before Stage 1)

Run on staging with internal employee accounts. Capture 30+ turns per scenario. Log system prompt sent to LLM (debug logging enabled).

| # | Scenario | Test user setup | Pass criteria |
|---|---|---|---|
| A | Trial + own-show 87% | Employee Trial account, ensure character_id matches | Bot acknowledges own show within first 5 turns ("tune meri kahani dekhi hi hai") |
| B | Trial + own-show 30% (dropped) | Same | Bot does Pattern C — soft callback ("tu aage dekhega na?") once |
| C | Trial + own-show not watched | Same | Bot does NOT pretend; uses it as a hook once |
| D | NewSub + heavy watcher (20+ titles) | Existing internal NewSub | Bot acknowledges taste *pattern* not enumerated titles |
| E | Renewer + 5 watches in 30 days | Internal Renewer | Bot tone is lighter (less emotional), more chitchat |
| F | Non-trial + 0 watches | Anonymous-ish test | Cold start works; no fake watch references |
| G | Edge: user mentions show NOT in prompt list | Manual injection | Bot does not confirm watching ("main nahi suna ki tune woh dekha tha") |
| H | Multi-character mid-session switch | Switch from Char A to Char B | Char B's prompt acknowledges Char B's own show, NOT Char A's |

Sign-off: each scenario reviewed by at least one Hindi-belt internal user (PM, designer, eng). Capture system prompt diff (NEW vs OLD) per scenario in PR description.

## Layer 4: Continuous production audit (Stage 1 onwards)

See `agents/05-audit.md`. Summary:

- **Daily sample:** 200 sessions stratified by arm × cohort
- **Hallucination check:** bot mentions a show NOT in user's watch history → flag (true positive = assertion of viewing; false positive = recommendation context)
- **Creepiness check:** specific timestamps, watch percentages, possessive surveillance language
- **Tone match check:** playbook buddy phrases present, own-show acknowledged in first 10 turns, soft questions invite disclosure
- **Output:** daily Slack post; flagged sessions JSON for human review

## Smoke gates between stages

| Before stage | Required smoke | Pass criteria |
|---|---|---|
| Stage 1 (5%) | 24h employee soak | All 8 manual QA scenarios pass; <2 hallucinations in 30 sessions |
| Stage 2 (25%) | 3 days of audit clean | Hallucination <5%; creepiness <1%; primary trending +ve |
| Stage 3 (50%) | Subgroup analysis | No cohort regresses on primary >2pp |
| Stage 4 (100%) | 7 days at 50% | All guardrails green |

## Rollback drill (run BEFORE Stage 1)

Manually verify in staging:
1. `redis-cli SET chat:buddy_personalize:killed 1` → confirm next chat session falls back to OLD path within 30s
2. GB dashboard: set flag to 0% → new sessions get OLD path immediately
3. **Runbook drafted:** on-call has a 1-pager for "buddy personalization regressed, what do I do" — link in PR description

## Test data fixtures

Create reusable fixtures for unit + integration tests:

- `watch_history_own_show_completed.json` — user with 87% on character's slug
- `watch_history_own_show_dropped.json` — user with 42% on character's slug
- `watch_history_no_own_show.json` — user with other shows watched but not character's
- `watch_history_empty.json` — new user
- `watch_history_heavy_watcher.json` — 50 titles, varied completion
- `watch_history_null_duration.json` — edge case

Store under `test/fixtures/chat/`.

## Test execution

```bash
# Pre-PR
cd /Users/stageadmin/stage-ott/stage-nest-backend
npm run lint
npm run type:check
npm run test  # full suite
npm run test:e2e  # e2e suite

# CI runs these on PR open. Must pass.
```

## Sign-off

Implementation Agent runs Layer 1-3. Code-Review Agent verifies Layer 1-2 coverage. PM signs off on Layer 3. Audit Agent owns Layer 4 from Stage 1 onwards.
