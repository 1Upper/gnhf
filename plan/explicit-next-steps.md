# Explicit Next Steps

## Problem / Opportunity

The current agent output schema captures `success`, `summary`, `key_changes_made`, and `key_learnings`. That is enough for commit decisions and iteration history, but it does not give the next iteration a formal handoff.

Iteration N must infer what to do next from notes, the objective, and the current codebase. This can work, but it can also cause repeated exploration, uneven prioritization, or loss of useful intent from the previous iteration.

There is an opportunity to make the loop more coherent by asking every successful or informative iteration to state likely next steps explicitly.

This should be optional at first. The current output schema is used by every agent adapter, and changing it unconditionally can break integrations that rely on strict schemas or existing parsing behavior.

## Solution Options

### Option 1: Add `next_steps` To The Schema

Extend the output schema with:

```json
{
  "next_steps": ["Add e2e coverage for resume prompt handling"]
}
```

The iteration prompt can describe this as a short, prioritized list of concrete follow-up work. The orchestrator appends it to `notes.md` after each iteration.

### Option 2: Add `remaining_work` Instead

Use a broader field:

```json
{
  "remaining_work": ["Validation still needs Windows coverage"]
}
```

This may be better when the next step is not obvious, but it is less action-oriented.

### Option 3: Add A Structured Plan File

Create `.gnhf/runs/<runId>/plan.json` or `plan.md` and let the orchestrator update it from agent output. The prompt then tells the next iteration to use the plan as the backlog.

This is more powerful but also introduces plan maintenance complexity.

### Option 4: Optional Field Based On Config

Only include `next_steps` when a config flag is enabled. This avoids changing the default output schema for every agent at once.

This is the safest first implementation because the existing schema remains unchanged unless the user opts in.

## Test Strategy

- Unit test `buildAgentOutputSchema` includes `next_steps` when enabled.
- Unit test `buildAgentOutputSchema` is byte-for-byte compatible with the current schema when the feature is disabled.
- Unit test schema compatibility with commit-message fields and `should_fully_stop`.
- Unit test `buildIterationPrompt` explains the expected `next_steps` content.
- Unit test note appending includes next steps in a readable section.
- Add an orchestrator test where iteration 1 returns next steps and iteration 2 prompt points the agent to notes containing them.
- Test failure/no-op behavior: when `success=false`, next steps may still be useful if the agent learned something.

## Feature Enhancement

Add config:

```yaml
iterationHandoff:
  nextSteps: true
```

The notes format could become:

```markdown
### Next steps suggested by the agent

- Add focused unit coverage for config parsing.
- Run e2e tests on Windows path handling.
```

This is a small change with a large potential payoff for multi-iteration coherence.

Implementation should treat `next_steps` as guidance, not a command queue. The agent still chooses the next smallest useful unit of work, and existing stop conditions, max iteration limits, commit behavior, and failure rollback behavior remain authoritative.
