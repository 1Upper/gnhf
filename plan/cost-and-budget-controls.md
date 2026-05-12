# Cost And Budget Controls

## Problem / Opportunity

`gnhf` supports `--max-tokens`, and the renderer tracks token usage. That protects users from unbounded runs, but it does not actively optimize cost.

Large repositories and repeated iterations can spend tokens on duplicated repo discovery, verbose prompts, full-file reads, or validation loops. Different agents may also have very different cost profiles.

There is an opportunity to make cost visible, controllable, and actionable.

Cost controls should not change existing iteration behavior unless explicitly enabled. Existing `--max-tokens` behavior must remain compatible.

## Solution Options

### Option 1: Budget-Aware Prompting

When token budget is low, adjust the iteration prompt:

- Prefer targeted searches before reading large files.
- Keep final summaries concise.
- Avoid broad refactors.
- Choose the smallest useful change.
- Stop if useful progress is unlikely within the remaining budget.

### Option 2: Per-Iteration Usage Artifact

Write `.gnhf/runs/<runId>/usage.jsonl`:

```json
{"iteration":1,"inputTokens":10000,"outputTokens":2000,"estimated":false}
```

This enables local analysis without exposing private repo details.

### Option 3: Cost Estimates

Allow optional pricing config by agent/model:

```yaml
cost:
  currency: USD
  models:
    claude:
      inputPerMillion: 3.00
      outputPerMillion: 15.00
```

The exit summary can display estimated cost.

### Option 4: Cheap Triage Agent

Use a cheaper agent or deterministic search to create a short briefing before invoking an expensive coding agent. This pairs well with the run context layer.

This requires either role-based orchestration or a small preflight runner. It should not be implemented by silently replacing the configured coding agent.

### Option 5: Adaptive Stop Rules

Stop early when:

- Consecutive iterations produce no commits.
- Summaries repeat.
- Token spend per useful commit exceeds a threshold.
- Remaining budget cannot cover a likely next iteration.

## Test Strategy

- Unit test token accounting remains correct with estimated and authoritative usage.
- Unit test usage artifact appends one record per iteration.
- Unit test exit summary displays cost only when pricing config exists.
- Unit test budget-aware prompt content appears when enabled.
- Orchestrator test for adaptive stop rules.
- E2E test that `--max-tokens` still aborts as expected.
- Test cost display is absent or clearly marked unavailable when pricing config is absent.
- Test telemetry remains anonymous and does not include sensitive pricing or prompt details unless explicitly intended.

## Feature Enhancement

Add config:

```yaml
budget:
  mode: balanced
  maxEstimatedCost: 5.00
  usageLog: true
```

Modes could be:

- `speed`: spend more to reduce wall-clock time.
- `balanced`: normal behavior.
- `economy`: prioritize concise prompts and targeted work.

This improves user trust because long-running autonomous loops feel safer when costs are visible and bounded.

Cost estimates should be local and optional. They should not be sent to telemetry unless the telemetry policy is explicitly updated and documented.
