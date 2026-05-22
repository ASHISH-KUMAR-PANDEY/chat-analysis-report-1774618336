# Agent Brief: Buddy-Personalize Content Context — Implementation

You are an experienced NestJS engineer implementing a focused PR on the Stage chat backend. Your job is to make the existing `CONTENT_CONTEXT` block in the character-chat system prompt buddy-flavored instead of recommendation-flavored, behind a GrowthBook flag.

## Working directory

`/Users/stageadmin/stage-ott/stage-nest-backend/`

## Critical reading before writing any code

1. `CLAUDE.md` at the repo root — the project's coding standards. Follow these strictly:
   - No standard NestJS decorators (`@Get`, `@Post`, `@Body`, etc. — use the custom @app ones)
   - No direct exception throws — use the existing error pattern
   - No direct `@InjectModel` — use the repository pattern
   - No `console.log` — use the Logger
   - No `common/` import paths (use `@app/common/...`)

2. The plan document: `/Users/stageadmin/abcd/chat-analysis-report-1774618336/buddy-init/plan-v2.md`

3. The current implementation you'll be modifying:
   - `src/chat/services/content-suggestion.service.ts` (main file — methods `buildPersonalizedContext`, `buildNewUserContext`, `getUserWatchHistory`)
   - `src/chat/services/prompt.service.ts` (orchestrator — see how content-context is wired in)
   - `src/chat/services/prompt-budget.service.ts` (token allocator — your block lives in `CONTENT_CONTEXT` slot, 400-token cap)

4. Reference (recently merged) example of similar pattern with GrowthBook flag:
   - `src/chat/services/recommendation-guard.service.ts` — uses `growthBookService.checkFeatureGate` + Redis kill switch

## Scope of work

### File: `src/chat/services/content-suggestion.service.ts`

Modify these methods:

1. **`getUserWatchHistory`** — keep the raw `timestamp` field in the returned `WatchHistoryItem` (already there) but also compute a `daysAgo: number` derived field at call-site. Don't change the interface; derive inline in the consumers.

2. **Add helper `deriveOwnShowStatus(watchHistory, characterContentSlug)`** — returns `{ watched: 'YES' | 'STARTED' | 'NO', watchPercent?: number, daysAgo?: number }`:
   - YES if `watchPercent >= 60` for an entry where `contentSlug === characterContentSlug`
   - STARTED if `5 <= watchPercent < 60` for that entry
   - NO if no such entry exists

3. **Add helper `deriveDropoffFlag(watchPercent: number): 'completed' | 'dropped' | 'sampled'`**:
   - `completed` if `watchPercent >= 60`
   - `dropped` if `5 < watchPercent < 60`
   - `sampled` if `watchPercent <= 5`

4. **Modify `buildPersonalizedContext`** to:
   - Prepend an `**ABOUT YOUR OWN SHOW**` block using `deriveOwnShowStatus`
   - Annotate each watched entry with the drop-off flag
   - Replace the existing `**RULES:**` section with the new two-section block (see Prompt Rewrite below)
   - Add a `**USER COHORT:** <status>` line at the top (read from `getUserSubscriptionMandateStatus()` — you'll need to inject this service or accept the cohort as a parameter)

5. **Modify `buildNewUserContext`** similarly — own-show NOT-watched block, cohort hint, same new two-section rules.

6. **Wrap the whole block in a GrowthBook flag check** `character_chat_buddy_personalization_v1`. Method signature change: the GB flag check happens at the entry point (`buildContentContext`), and the call branches to either the OLD shape or the NEW shape. Keep the OLD code paths intact during rollout for control-arm safety.

7. **Add a Redis kill-switch** `chat:buddy_personalize:killed` similar to the existing `chat:recommendation:killed` pattern.

### File: `src/chat/services/prompt.service.ts`

- If `buildContentContext` needs a new parameter (cohort), pass it from here using the same `getUserSubscriptionMandateStatus()` call that already happens in `buildPriorityInstructions`. **Do not make a second DB/Redis call** — share the result.

### New unit tests in `src/chat/services/content-suggestion.service.spec.ts`

Cover at minimum:
- Own-show YES path
- Own-show STARTED path
- Own-show NO path
- Drop-off flag on other shows
- Cold start (empty history) with NEW rules
- 50+ titles (heavy watcher, no token blow-up)
- Null `total_duration` (defensive)
- Cohort hint present (`trial_active`, `subscription_active`, `non_subscriber`)
- GB flag OFF returns OLD prompt shape (regression test)
- Kill switch ON returns OLD prompt shape
- Redis cache hit short-circuits
- WatchVideoEventRepository throws → empty context, no exception propagation

### Prompt rewrite (must be IDENTICAL to the spec)

The new two-section RULES block to be used in both methods. Use this verbatim (you can tweak whitespace but not the words):

```
**HOW TO USE THIS HISTORY (buddy mode — PRIMARY):**
- Reference titles casually, not constantly. Once per session max unless user brings it up.
- If user has watched YOUR show: acknowledge naturally — "Tune meri kahani dekh li hai na — woh wala part yaad hai?"
- If user has NOT watched YOUR show: do NOT pretend they did. Invite them.
- If user dropped off mid-show: ONE soft callback, no guilt — "Tu aage dekhega na yaar?"
- Use lifetime taste to invite disclosure, not to recommend — "Romance dekhta hai... tujhe asli zindagi mein kabhi pyaar hua?"
- NEVER mention watch percentages or timestamps verbatim. They're context for YOU, not lines for the user.
- NEVER act surprised by their taste ("achha tu woh dekhta hai!" implies you didn't know before — breaks continuity).

**RECOMMENDATION MODE — secondary, only after 10+ messages:**
- Do NOT recommend anything in the first 10 messages — build the friendship first
- After 10+ messages: recommend ONLY when user asks, praises a story, or the vibe is right
- Recommend like a friend sharing gossip, NOT like a salesman
- Use [[RECOMMEND:Title]] tag so the app can show a card
- Max 1-2 recommendations total, then STOP pushing. Never repeat the same one.
```

### Logging & metrics

- Log `buddy_personalize_active` when treatment path executes (sample 1% or use existing log levels)
- Log `buddy_personalize_gate_off` and `buddy_personalize_killed` when fallback to OLD path
- Statsd/Datadog metric `chat.buddy_personalize.active_count` increment per treatment session
- Metric `chat.buddy_personalize.fallback_count` increment per fallback session

### Branch + PR

- Branch: `feat/chat-buddy-personalize-content-context`
- PR title: "feat(chat): buddy-personalize CONTENT_CONTEXT block (PR 1/4 of buddy initiative)"
- PR description must include:
  - Link to `../buddy-init/plan-v2.md` and `../buddy-init/experiment-plan.md`
  - Token-budget impact (+~80 tokens worst case, stays under 400 cap)
  - GrowthBook flag name + Redis kill switch key
  - Rollout schedule (the four ramp stages)
  - Manual QA results from the 8 internal scenarios (run these yourself before opening the PR)
  - Test results (unit + integration)

## Constraints (hard)

- Do not modify the safety prompt (`SAFETY_SYSTEM_PROMPT` in chat.constants.ts).
- Do not change the carousel infrastructure — `RecommendationGuardService` and the `[[RECOMMEND:]]` tag pipeline must keep working.
- Token budget after change: total `CONTENT_CONTEXT` must stay ≤400 tokens for any user. If your block exceeds, fall back to the OLD shape (let `PromptBudgetService.truncateToTokens` handle the worst case — but verify it works).
- Do not introduce new Mongo or Scylla calls. Reuse what's already fetched (`getUserWatchHistory`, `getUserSubscriptionMandateStatus`).
- No new external dependencies in `package.json`.
- All linter checks pass (`npm run lint`).
- Type checks pass (`npm run type:check`).

## Manual QA before opening PR

After implementing, run the 8 scenarios in `../testing-plan.md` §2.3 on your local dev environment (or staging). Capture the system prompt that gets sent to Grok in each case (log it once with debug enabled), confirm:
- Own-show block appears correctly for YES / STARTED / NO
- Drop-off flags on watched list
- Cohort hint matches the user's actual cohort
- Two-section rules block present
- Total length within budget

## Deliverables

1. Branch `feat/chat-buddy-personalize-content-context` pushed to origin
2. PR opened with full description per above
3. Unit + integration tests passing in CI
4. A short writeup in the PR description showing the 8 internal QA scenarios with the system prompt diff (NEW vs OLD) for each

## When you're done

Reply with:
1. PR URL
2. Confirmation of all test passes (`npm run test`, `npm run lint`, `npm run type:check`)
3. Sample of the NEW prompt block for a Trial user with own-show 87% watched (paste verbatim)
4. Any deviations from this brief, flagged explicitly

## What you should NOT do

- Do NOT also implement PR #2 (Day-1 close), PR #3 (confidentiality framing), or PR #4 (Day-N memory). Those are separate.
- Do NOT modify the LLM model, temperature, max_tokens, or any OpenAI/Grok client setting.
- Do NOT change the recommendation carousel UI on Flutter — this is backend-prompt-only.
- Do NOT add user PII (phone, name, location) to the prompt. Only what's already there + the four new fields (own-show status, drop-off, cohort, derived).
- Do NOT ship the feature without GB-gated control path. Treatment and control must coexist.
