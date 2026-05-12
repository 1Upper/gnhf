# Role-Based Multi-Agent Orchestration

## Problem / Opportunity

`gnhf` currently chooses one configured agent for the run and sends every coding iteration to that agent. That is simple and predictable, but autonomous development work is not one uniform task.

A run often contains several different kinds of work:

- Deciding which role should act next.
- Understanding the objective.
- Exploring the codebase.
- Making architecture and system design decisions.
- Producing an execution plan.
- Editing source code.
- Running or selecting tests.
- Diagnosing validation failures.
- Reviewing diffs for risk.
- Summarizing outcomes for future iterations.

Using the same model or agent for all of those tasks can be inefficient. A strong coding model may be too expensive for summarization. A fast cheap model may be good enough for initial classification but not for risky refactors. A model that is strong at test diagnosis may not be the best writer of production code.

There is also a conflict risk when multiple agents are allowed to modify the workspace. If a planner, tester, reviewer, and coder all edit files directly, they can overwrite each other, create noisy diffs, or make it unclear which role owns the final state.

The opportunity is to introduce role-based agent configuration, where each role has a clear responsibility, permission boundary, prompt, output schema, and handoff artifact.

This idea must be implemented as an extension, not a replacement for the current run model. A repository with no role configuration should behave exactly as it does today: one configured agent runs each iteration, successful iterations are committed, failures are reset or repaired according to the existing rules, and existing CLI flags/config continue to work.

Two orchestration layers should stay separate:

- The TypeScript runtime orchestrator remains authoritative for safety, git state, commits, resets, limits, signals, and lifecycle.
- An optional LLM orchestrator role can make advisory routing decisions at configured decision points.

System design should also have its own owner. The architect role focuses on architecture, module boundaries, data flow, config shape, compatibility, migration strategy, and rollout tradeoffs. The planner role turns that design into concrete execution steps.

### Current Architecture Constraints

This feature is implementable, but not as a thin config-only layer. The current architecture has constraints that the design must respect:

- `Agent.run(prompt, cwd, options)` always receives a working directory. A `workspace: none` role still needs a safe `cwd`, such as a temporary run-artifact directory or isolated empty directory, unless the agent interface is extended.
- Agents are currently created with one output schema. Some adapters receive the schema object at construction, while Codex/RovoDev use `runInfo.schemaPath`. Role-specific schemas require separate role schema files or a role-runner abstraction before they can be reliable.
- The current `Orchestrator` owns one `Agent` instance and closes one agent. Multi-role execution needs a role manager that creates, tracks, and closes all role agents.
- ACP agents keep session state under the run directory. Multiple ACP roles need role-scoped ACP session directories so planner, coder, tester, and reviewer sessions do not accidentally share state unless sharing is explicitly configured.
- Existing per-iteration logs use names like `iteration-1.jsonl`. Role runs need distinct log paths such as `role-planner-before-run.jsonl` or `iteration-1-role-reviewer.jsonl`.
- Advisory roles that inspect uncommitted coder changes cannot safely run in the target worktree unless their edits can be isolated from the coder's pending diff.

These constraints do not block the idea, but they mean the first implementation should add a small role runner layer rather than trying to reuse the existing single-agent iteration path unchanged.

Backward compatibility is a hard requirement. The role runner layer should sit beside the existing single-agent path and only activate when role orchestration is explicitly enabled by config, profile, or CLI flag.

## Solution Options

### Option 1: Configurable Roles With A Single Writer

Add role configuration while keeping only the `coder` role as the owner of workspace mutations by default.

Important feasibility constraint: the current native agent integrations do not provide a portable, hard read-only permission mode. `gnhf` can instruct advisory roles not to edit, detect unexpected workspace changes, run advisory roles in an isolated copy/worktree, or use agent-specific permission flags where available. It should not assume every supported agent can be made truly read-only by configuration alone.

Example config:

```yaml
roles:
  orchestrator:
    agent: claude
    workspace: none
    mayEditTarget: false
    when: decision-points
  architect:
    agent: claude
    workspace: target
    mayEditTarget: false
    when: before-planning
  planner:
    agent: copilot
    workspace: target
    mayEditTarget: false
    when: before-run
  coder:
    agent: claude
    workspace: target
    mayEditTarget: true
    when: each-iteration
  tester:
    agent: codex
    workspace: isolated
    mayEditTarget: false
    when: after-coder
  reviewer:
    agent: copilot
    workspace: isolated
    mayEditTarget: false
    when: before-commit
```

The default rule is: only `coder` is allowed to intentionally edit the target workspace. Other roles produce structured results that the runtime writes to artifacts under `.gnhf/runs/<runId>/`. Advisory roles should not be asked to write those files directly unless they run in a controlled artifact-only mode.

Role responsibilities:

| Role | Main question | Typical output |
| --- | --- | --- |
| Orchestrator | What should happen next? | Routing decision and next-role instructions |
| Architect | What design should we use? | Architecture brief, tradeoffs, constraints |
| Planner | What are the concrete steps? | Ordered plan and candidate files |
| Coder | How do we implement the next step? | File changes and normal iteration JSON |
| Tester | Does it work? | Validation results and failure diagnosis |
| Reviewer | Is the change acceptable? | Findings, approval, repair brief |
| Summarizer | What should future iterations remember? | Compact notes and learnings |

Possible artifacts:

```text
.gnhf/runs/<runId>/context.md
.gnhf/runs/<runId>/architecture.md
.gnhf/runs/<runId>/plan.md
.gnhf/runs/<runId>/validation.md
.gnhf/runs/<runId>/review.md
.gnhf/runs/<runId>/handoff.md
```

The orchestrator becomes a coordinator:

```text
setupRun
  -> optional LLM orchestrator chooses initial roles
  -> architect returns design output; runtime writes architecture.md when design is needed
  -> planner returns planning output; runtime writes context.md and plan.md
  -> coder performs one scoped iteration
  -> runtime runs configured validation, tester diagnoses output when useful
  -> reviewer inspects the diff for risk
  -> optional LLM orchestrator recommends repair or stop
  -> runtime commits only when coder output, validation, and review policy allow it
  -> append notes
  -> repeat
```

This offers model routing while keeping conflict control simple.

If `roles` is absent, the runtime should not infer a multi-role workflow. It should continue using the existing `agent` configuration as the single coder agent.

### Option 2: Built-In Role Pipeline

Start with a fixed pipeline and only allow users to choose the agent for each role.

```yaml
roleAgents:
  orchestrator: claude
  architect: claude
  planner: copilot
  coder: claude
  tester: codex
  reviewer: copilot
```

The pipeline itself is not configurable at first. This reduces complexity and makes testing easier.

The initial phases could be:

1. Optional orchestrator decision before the first iteration.
2. Optional architect pass when design is needed.
3. Optional planner before the first coding iteration.
4. Coder each iteration.
5. Optional runtime validation after coder success.
6. Optional tester diagnosis if validation fails or if configured for extra analysis.
7. Optional reviewer before commit.
8. Optional orchestrator decision after tester/reviewer feedback.

### Option 3: Role Profiles

Combine this feature with run profiles. A profile can define the role set for a workflow.

```yaml
profiles:
  cautious-refactor:
    roles:
      planner:
        agent: claude
      coder:
        agent: claude
      tester:
        agent: codex
      reviewer:
        agent: copilot
```

This lets users choose between simple single-agent runs and more careful multi-role runs.

### Option 4: Fully Pluggable Workflow Graph

Represent the run as a configurable graph of phases:

```yaml
workflow:
  - role: orchestrator
    when: decision-points
  - role: architect
    when: before-planning
  - role: planner
    when: before-run
  - role: coder
    when: each-iteration
  - role: tester
    when: after-coder
    onFailure: coder-repair
  - role: reviewer
    when: before-commit
    onFailure: coder-repair
```

This is powerful, but it is probably too much for a first implementation. It increases state management, error handling, user support burden, and the risk of routing loops. It should be treated as a future direction rather than an initial deliverable.

### Option 5: Advisory Roles Only

Keep the main run exactly as it works today, but allow optional advisory calls:

- An orchestrator advisory call at decision points.
- An architect advisory call for system design or ambiguous refactors.
- A planner advisory call before iteration 1.
- A reviewer advisory call before commit.
- A summarizer advisory call after commit.

The coder still owns the normal agent output schema. This is the lowest-risk stepping stone.

For the first implementation, advisory roles should be limited to planner and architect before any product-code edits exist. Reviewer/tester roles that run after coder changes need extra isolation because the workspace is already dirty.

### Option 6: Isolated Advisory Execution

Run advisory roles in an isolated environment instead of the target worktree.

Possible approaches:

- Create a temporary git worktree at the same commit/diff state.
- Create a temporary copy of the repository, excluding heavy ignored folders when safe.
- Feed advisory roles a generated diff and relevant files instead of giving them filesystem access.

This is the safest way to review or diagnose uncommitted coder changes without risking accidental edits to the target workspace. It is more expensive to implement, especially for large repositories and Windows path handling, but it avoids relying on unsupported read-only guarantees from every agent CLI.

## Test Strategy

- Unit test config parsing for role definitions.
- Unit test unknown role names, invalid workspace modes, invalid `mayEditTarget` combinations, and invalid lifecycle phases produce helpful errors.
- Unit test fallback behavior when no roles are configured: current single-agent behavior remains unchanged.
- Unit test role-specific prompt builders for orchestrator, architect, planner, coder, tester, reviewer, and summarizer.
- Unit test role-specific output schemas. Planner output should not be confused with coder output.
- Unit test artifact creation under `.gnhf/runs/<runId>/`.
- Unit test advisory roles do not intentionally modify files. A practical implementation can snapshot git state before and after a pre-edit advisory role and fail if tracked files changed.
- Unit test post-coder advisory roles do not call `resetHard` on the target worktree, because that would discard the coder's pending changes.
- Unit test isolated advisory execution receives the intended diff/context and leaves the target worktree unchanged.
- Unit test tester/reviewer failure creates repair context rather than committing the change.
- Unit test LLM orchestrator decisions are treated as advisory and cannot override hard runtime limits.
- Unit test architect output is included in planner and coder context when present.
- Unit test planner output does not replace architect output; the two artifacts stay separate.
- Orchestrator tests for the happy path: architect -> planner -> coder -> runtime validation -> reviewer -> commit.
- Orchestrator tests for validation failure: planner -> coder -> validation fails -> no commit -> repair prompt includes validation artifact.
- Orchestrator tests for tester diagnosis: validation fails -> tester receives failure output -> repair prompt includes tester artifact.
- Orchestrator tests for reviewer failure: reviewer findings are added to notes or repair context.
- Orchestrator tests for LLM orchestrator failure: deterministic fallback routing is used.
- E2E test with fake agents for each role, confirming role order and artifacts.
- E2E test that only the coder fake agent modifies target-worktree files.
- Windows tests for artifact paths and process spawning because role orchestration increases agent process count.

## Feature Enhancement

### Role Definitions

Introduce role metadata:

```ts
type AgentRole =
  | "orchestrator"
  | "architect"
  | "planner"
  | "coder"
  | "tester"
  | "reviewer"
  | "summarizer";

interface RoleConfig {
  agent: AgentSpec;
  workspace: "target" | "isolated" | "none";
  mayEditTarget: boolean;
  when:
    | "decision-points"
    | "before-planning"
    | "before-run"
    | "each-iteration"
    | "after-coder"
    | "before-commit"
    | "after-commit";
  enabled?: boolean;
}
```

`workspace` and `mayEditTarget` are more honest than a simple `permissions: read` flag:

- `target`: the role runs in the active run worktree.
- `isolated`: the role runs against a temporary copy/worktree or receives prepared diff context.
- `none`: the role receives artifacts only and should run with a safe temporary `cwd`, not the target repository. With the current `Agent.run` interface, this still requires a real directory.
- `mayEditTarget`: only `coder` should set this to true by default.

The runtime may still pass agent-specific permission flags when available, but the portable enforcement mechanism is isolation plus git-state checks, not trust in a universal CLI read-only mode.

The default behavior maps the existing `agent` config to the `coder` role:

```yaml
roles:
  coder:
    agent: <existing config.agent>
    workspace: target
    mayEditTarget: true
    when: each-iteration
```

This mapping should be internal compatibility behavior, not a breaking config migration. Users should not need to rewrite existing config files to keep current behavior.

### Role Prompts

Each role should receive a different prompt.

LLM orchestrator prompt:

```text
You are the advisory orchestrator for this gnhf run. Read the run artifacts and recommend the next role/action. Do not edit files. The TypeScript runtime orchestrator enforces all hard safety rules, so your output is advisory and must be structured. You cannot approve commits directly.
```

Architect prompt:

```text
You are the architect for this gnhf run. Focus on system design: module boundaries, config shape, data flow, compatibility, migration strategy, and rollout risks. Do not edit files. Produce an architecture brief for the planner and coder.
```

Planner prompt:

```text
You are the planner for this gnhf run. Read the project and architecture brief, do not edit files, and produce a concise execution plan and project context for the coder.
```

Coder prompt:

```text
You are the coder for this gnhf iteration. Read the context, notes, and any repair artifacts. Make the next smallest useful change. Do not commit.
```

Tester prompt:

```text
You are the tester for this gnhf iteration. Inspect validation output and the relevant diff context. Diagnose failures or recommend targeted follow-up validation. Do not edit files.
```

Reviewer prompt:

```text
You are the reviewer for this gnhf iteration. Inspect the diff for correctness, risk, missing tests, and scope creep. Do not edit files.
```

Summarizer prompt:

```text
You are the summarizer for this gnhf run. Condense role artifacts and iteration results into durable notes for future iterations. Do not edit product files.
```

### Role Outputs

Use separate schemas.

Implementation note: current agents are constructed with one output schema per agent instance, and some adapters read the schema from `runInfo.schemaPath`. Role-specific schemas are feasible, but they require either separate role agent instances with separate schema files, or a role runner that can provide a schema per invocation. This should be built before relying on the schemas below.

LLM orchestrator output:

```json
{
  "next_action": "run-coder",
  "target_role": "coder",
  "reason": "Reviewer found missing config documentation.",
  "instructions_for_next_role": "Update README config docs and add a config parsing test.",
  "should_stop": false
}
```

Architect output:

```json
{
  "summary": "Use fixed advisory roles first and keep the existing single-agent coder flow as the default.",
  "design_decisions": ["Only coder has write permission by default"],
  "module_boundaries": ["Role config belongs in config.ts", "Role artifacts belong in run.ts"],
  "risks": ["Workflow graph configuration may be too complex for the first release"],
  "rollout_plan": ["Add planner artifact first", "Add reviewer before commit", "Add tester after coder"]
}
```

Planner output:

```json
{
  "summary": "Repo uses TypeScript ESM and Vitest.",
  "relevant_files": ["src/core/orchestrator.ts"],
  "plan": ["Add failing test", "Implement fix", "Run targeted test"],
  "risks": ["Windows path handling"]
}
```

Tester output:

```json
{
  "passed": false,
  "commands_run": ["npm run typecheck"],
  "failures": ["Type error in config.ts"],
  "repair_brief": "Fix the type mismatch before commit."
}
```

Reviewer output:

```json
{
  "approved": false,
  "findings": ["New config option is not documented"],
  "repair_brief": "Add README documentation or remove the option."
}
```

Summarizer output:

```json
{
  "summary": "Role orchestration design now separates architect, planner, coder, tester, reviewer, and advisory orchestrator responsibilities.",
  "durable_learnings": ["Only coder should write files by default"],
  "next_context": ["Architect artifacts should feed planner and coder prompts"]
}
```

### Conflict Avoidance

The first implementation should enforce a single-writer rule:

- `coder` is the only role allowed to intentionally modify the target worktree by default.
- Pre-edit advisory roles can run in the target worktree with git-state checks because no coder changes are pending yet.
- Post-coder advisory roles should run in isolation or receive prepared diff/artifact context. They should not run directly in the target worktree unless the implementation can preserve the coder's pending changes.
- If a pre-edit advisory role modifies tracked files, the runtime can fail the role and reset because there are no target changes to preserve.
- If a post-coder advisory role modifies tracked files in the target worktree, the runtime must not blindly call `resetHard`; that would destroy the coder's work. Prefer isolated execution or abort with a clear diagnostic.
- Tester and reviewer findings become repair artifacts, not direct edits.
- The LLM orchestrator role cannot override runtime safety checks such as max iterations, max tokens, stop requests, dirty worktree rules, commit failures, or push failures.
- The architect role owns design guidance, but the planner owns sequencing. This avoids mixing high-level architecture with task decomposition.

### Validation Ownership

The runtime should own command execution for validation when commands are configured. The tester role should primarily diagnose validation output and suggest repair context.

This avoids asking a supposedly read-only LLM role to run arbitrary shell commands in the target worktree. A later version can allow tester-run commands, but only with explicit configuration and the same process-safety rules used elsewhere in `gnhf`.

### When To Invoke Architect

The architect role should be optional and trigger only when design work is likely to matter. Possible triggers:

- The user objective mentions architecture, design, refactor, API, config, migration, integration, or plugin behavior.
- The planned change touches core orchestration, agent interfaces, config loading, run metadata, git behavior, or telemetry.
- The planner reports that the implementation approach is ambiguous.
- The reviewer flags architecture risk or scope creep.
- The user selects a profile such as `cautious-refactor` or passes an explicit role option.

The architect should not run for every tiny fix by default because that could add latency and cost.

### Incremental Rollout

Recommended phases:

1. Preserve the existing single-agent path as the default and add tests proving behavior is unchanged when no roles are configured.
2. Add advisory planner role whose structured output is written by the runtime to `context.md` and `plan.md` before iteration 1.
3. Add advisory architect role for explicit design/refactor runs, with runtime-written `architecture.md`.
4. Add role-specific schema support through separate role schema files or a role runner abstraction.
5. Add runtime-owned validation commands and runtime-written `validation.md`.
6. Add tester diagnosis of validation output without target-worktree editing.
7. Add isolated reviewer execution before commit.
8. Add optional LLM orchestrator decisions at limited decision points with deterministic fallback.
9. Add role profiles.
10. Consider configurable workflow graphs only after the fixed pipeline is stable.

This design supports cost efficiency, faster execution, better model fit, and clearer feature management while keeping the existing single-agent flow as the default.
