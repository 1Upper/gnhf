# Run Context Layer

## Problem / Opportunity

Today, each iteration asks the agent to read `.gnhf/runs/<runId>/notes.md` and then inspect the repository as needed. That keeps the orchestrator simple, but it means iteration 1 starts cold and later iterations may repeat codebase discovery work.

On large repositories, repeated discovery can be slow and expensive. The agent may repeatedly inspect top-level docs, package files, test scripts, source layout, and the same search results. The current `notes.md` file is iteration history, not a stable project briefing.

There is an opportunity to separate two kinds of context:

- Stable project orientation: repo layout, package manager, useful commands, local conventions, known relevant files.
- Iteration history: what previous iterations changed, learned, failed, or left behind.

This must be an additive feature. If context generation is disabled or not configured, the existing prompt, notes, schema, and iteration loop should behave exactly as they do today.

## Solution Options

### Option 1: Static Context File

Create `.gnhf/runs/<runId>/context.md` during `setupRun`. The first implementation could contain deterministic facts gathered without an LLM:

- Repo root path.
- Current branch and base commit.
- Detected package manager and scripts from `package.json`.
- Existing instruction files such as `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md`, or similar.
- Test file conventions discovered from filenames.
- Top-level directory summary.

The iteration prompt would tell the agent to read both files:

```text
Read .gnhf/runs/<runId>/context.md for project orientation.
Read .gnhf/runs/<runId>/notes.md for previous iteration history.
```

This keeps behavior deterministic and avoids another model call.

### Option 2: Agent-Generated Briefing

Add an optional planning phase before iteration 1. The selected agent inspects the repo and returns structured context, and the runtime writes `context.md`. The planning phase should be instructed not to edit product code, but the implementation should also detect unexpected git changes because native agents do not all provide a portable hard read-only mode.

This can produce richer context, but it costs an extra agent call and depends on agent quality.

### Option 3: Hybrid Briefing

Generate deterministic context first, then optionally ask the agent to add objective-specific findings:

- Relevant search terms.
- Candidate files.
- Suspected test commands.
- Risk areas.

This gives the agent a structured starting point while still allowing deeper analysis when useful.

### Option 4: Refreshable Context

Allow context refresh after major changes or after N iterations. The orchestrator could append a compact summary of committed files and changed project assumptions.

## Test Strategy

- Unit test `setupRun` creates `context.md` for new runs when enabled.
- Unit test no `context.md` is created and the iteration prompt is unchanged when context mode is `off` or absent.
- Unit test `resumeRun` preserves existing `context.md` and does not overwrite user or prior run context.
- Unit test `buildIterationPrompt` includes the context read instruction only when the run has a context file or when the feature is enabled.
- Add fixtures for repositories with and without `package.json`, instruction files, and test scripts.
- E2E test that a run directory contains `context.md` and that the agent prompt references it.
- Regression test that `.gnhf/runs/` remains excluded from git tracking.

## Feature Enhancement

Add a configuration option:

```yaml
context:
  mode: deterministic
```

Possible modes:

- `off`: current behavior, and the safest default for a first release.
- `deterministic`: create context from local repo facts.
- `agent`: run a planning-only agent pass.
- `hybrid`: deterministic context plus optional agent additions.

The default should begin as `off`, or remain effectively off unless a profile or flag enables it. A careful rollout avoids surprising users with extra token cost or changed prompts.
