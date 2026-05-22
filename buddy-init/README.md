# Buddy Initiative — PR 1: Buddy-Personalize Content Context

**Status:** Planning complete, awaiting "go" to start implementation.
**Owner:** Ashish (PM), Claude (initial code + ops support)
**Initiative:** PR 1 of 4 from the chat-buddy-playbook (see `../chat-buddy-playbook.md`).

## Documents in this folder

- `README.md` — this file
- `plan-v2.md` — the focused plan (the v2 plan after the code-archaeology pass)
- `experiment-plan.md` — A/B test design, ramp schedule, stopping rules
- `testing-plan.md` — unit/integration/manual/audit testing strategy
- `agents/01-implementation.md` — agent brief: write the code
- `agents/02-prompt-craft.md` — agent brief: refine prompt language
- `agents/03-code-review.md` — agent brief: review the PR before merge
- `agents/04-rollout-monitor.md` — agent brief: watch metrics during ramp
- `agents/05-audit.md` — agent brief: daily session sampling for hallucination/creepiness
- `agents/06-iteration.md` — agent brief: propose prompt v2 if primary flat

## How to invoke an agent

Each agent file is self-contained. From a Claude Code session:

```
Agent({
  description: "Implement PR 1 (buddy content context)",
  subagent_type: "general-purpose",
  prompt: <contents of agents/01-implementation.md>
})
```

Or use the Agent tool with the file content as the prompt verbatim.

## Sequence of agent invocations

1. **Implementation Agent** → produces branch + PR
2. **Code-Review Agent** → reviews the PR (run BEFORE merge)
3. **Prompt-Craft Agent** → invoked if Code-Review flags prompt-quality issues (optional)
4. Merge + deploy
5. **Audit Agent** → starts running daily from stage-1 launch
6. **Rollout Monitor Agent** → runs during ramp, surfaces metrics
7. **Iteration Agent** → invoked at stage 2/3 if primary metric is flat

## Status tracker

- [ ] Plan approved
- [ ] Branch created
- [ ] PR drafted (Implementation Agent)
- [ ] PR reviewed (Code-Review Agent)
- [ ] PR merged
- [ ] Internal soak (Stage 0) — 24h, 30 test sessions
- [ ] Stage 1: 5% Trial cohort — 3 days
- [ ] Stage 2: 25% Trial — 4 days
- [ ] Stage 3: 50% all cohorts — 7 days
- [ ] Stage 4: 100% — stable
- [ ] Post-experiment writeup
