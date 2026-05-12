# First-Class Agent Skills

## Problem / Opportunity

Some coding agents support skills, instruction files, specialized prompts, or reusable workflows. Today, `gnhf` delegates to each agent and passes through extra native arguments when configured, but skills are not a first-class `gnhf` concept.

This means skill behavior depends on the selected agent:

- Some agents may auto-discover skills from project or user directories.
- Some may require command-line flags.
- Some may not support skills at all.
- `gnhf` cannot currently list, select, log, or inject skills consistently.

There is an opportunity to make skills portable across agents by treating them as prompt modules managed by `gnhf`.

This must be opt-in. Existing runs should not receive extra skill text unless the user configures skills or selects a profile that enables them.

## Solution Options

### Option 1: Prompt-Injected Skills

Add config that points to markdown skill files. `gnhf` reads selected files and injects them into the iteration prompt.

```yaml
skills:
  - .gnhf/skills/testing.md
  - .gnhf/skills/typescript.md
```

This works for every agent because it is just prompt text. The tradeoff is increased prompt length and possible instruction conflicts.

### Option 2: Skill Manifest

Support a manifest with metadata:

```yaml
skills:
  testing:
    path: .gnhf/skills/testing.md
    applyWhen:
      - "tests"
      - "coverage"
```

The orchestrator can select skills based on prompt keywords or explicit CLI flags.

### Option 3: Native Agent Skill Bridge

For agents with native skill support, map `gnhf` skills to their native mechanism. For example, one adapter might pass extra arguments, while another relies on a project directory.

This may be more efficient, but it creates adapter-specific complexity. It should be treated as an optimization, not the portable baseline, because not every supported agent exposes the same skill mechanism.

### Option 4: Explicit Skill Selection CLI

Add:

```sh
gnhf "fix flaky tests" --skill testing --skill ci
```

This makes skill usage deliberate and easy to audit.

## Test Strategy

- Unit test config parsing for skill paths and names.
- Unit test invalid skill paths fail with helpful messages.
- Unit test skill content is injected into the prompt in a stable order.
- Unit test redaction/logging does not leak sensitive skill paths if that matters for telemetry.
- E2E test with a fake agent verifies the final prompt contains selected skill instructions.
- Test prompt size guardrails for very large skill files.
- Test conflict handling when two skills define incompatible instructions.
- Test no skill section appears in the prompt when skills are not configured.

## Feature Enhancement

Add a `SkillProvider` module:

```text
config -> resolve skills -> validate files -> render skill section -> buildIterationPrompt
```

The prompt section could look like:

```markdown
## Skills

The following reusable project guidance applies to this iteration.

### testing

<skill content>
```

Longer term, add `gnhf skills list`, `gnhf skills validate`, and templates for common skills.

The first implementation should use prompt-injected skills because it works across all agents. Native agent bridges can be added per adapter later without changing the user-facing skill config.
