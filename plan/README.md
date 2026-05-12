# Enhancement Plan Index

This directory tracks possible product and system design enhancements for `gnhf`.

Each idea is split into its own document so it can grow independently. The common structure is:

- Problem / opportunity
- Solution options
- Test strategy
- Feature enhancement

## Compatibility Principle

Every idea in this folder should be treated as an additive extension. Existing `gnhf` behavior should remain unchanged unless a user explicitly enables a new feature through config, a profile, or a CLI flag. Existing CLI flags, config files, run metadata, commit behavior, failure handling, and the single-agent iteration loop must continue to work.

## Ideas

- [Run Context Layer](run-context-layer.md)
- [Explicit Next Steps](explicit-next-steps.md)
- [First-Class Agent Skills](first-class-agent-skills.md)
- [Run Profiles](run-profiles.md)
- [Validation Modes](validation-modes.md)
- [Cost And Budget Controls](cost-and-budget-controls.md)
- [Role-Based Multi-Agent Orchestration](role-based-multi-agent-orchestration.md)
- [CI-Aware Runs](ci-aware-runs.md)
- [GitHub Issue And PR Workflow](github-issue-pr-workflow.md)
