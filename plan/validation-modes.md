# Validation Modes

## Problem / Opportunity

The iteration prompt tells the agent to run build, tests, linters, or formatters when available. This gives the agent flexibility, but validation can become inconsistent:

- One iteration may run a targeted test.
- Another may run the full suite.
- An agent may spend too much time discovering commands.
- Expensive validation may be unnecessary for small doc changes.
- Weak validation may miss regressions before commit.

There is an opportunity to make validation intent explicit while still leaving room for agent judgment.

This must not change existing behavior unless a validation mode or validation command is configured. The current prompt-only guidance should remain the default path.

## Solution Options

### Option 1: Prompt-Only Validation Mode

Add a CLI/config option:

```sh
gnhf "fix parser bug" --validation quick
```

The iteration prompt explains the selected validation level:

- `none`: do not run validation unless essential.
- `quick`: run focused checks related to changed files.
- `normal`: run reasonable tests and type/lint checks when available.
- `thorough`: run full validation suites when feasible.

### Option 2: Deterministic Validation Commands

Let users define commands:

```yaml
validation:
  quick:
    - npm run typecheck
  thorough:
    - npm test
```

The agent can be instructed to run the configured commands. The orchestrator could also run them directly after agent completion.

If the orchestrator runs commands, commands should be represented in a structured form or executed through a carefully controlled runner. Avoid building shell command strings from config without clear quoting and platform rules.

### Option 3: Orchestrator-Enforced Validation

After agent success but before commit, `gnhf` runs configured validation commands. If validation fails, the iteration is treated like a failure or converted into a repair iteration.

This is more reliable but changes the contract: the orchestrator becomes an active validator, not just a commit manager.

Because coder changes are uncommitted at this point, validation failure must preserve those changes for repair or intentionally reset them according to a documented policy. It must not accidentally discard work that the agent could repair.

### Option 4: Agent-Reported Validation Evidence

Extend the schema with:

```json
{
  "validation": ["npm run typecheck", "npx vitest run src/foo.test.ts"]
}
```

The notes and exit summary can show what was verified.

## Test Strategy

- Unit test config parsing for validation modes and commands.
- Unit test prompt contains correct validation guidance for each mode.
- Unit test schema extension for validation evidence if implemented.
- Orchestrator tests for validation command success/failure if enforcement is implemented.
- E2E test with fake validation commands.
- Test command execution uses safe argument handling and avoids shell injection.
- Test Windows behavior for command execution if commands are user-configurable.
- Test validation is not run when no validation mode/commands are configured.
- Test validation failure preserves or discards coder changes according to the documented policy.

## Feature Enhancement

Start with prompt-only validation modes because they are low risk. Later add optional orchestrator-enforced commands for teams that need stronger guarantees.

Possible config:

```yaml
validation:
  mode: normal
  commands:
    quick:
      - npm run typecheck
    thorough:
      - npm test
```

Exit summary could include a validation section showing agent-reported checks.

The first implementation should be prompt-only and backward compatible. Orchestrator-enforced validation should be a later opt-in feature with explicit repair semantics.
