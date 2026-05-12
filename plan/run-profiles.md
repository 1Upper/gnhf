# Run Profiles

## Problem / Opportunity

`gnhf` has several knobs: agent, max iterations, max tokens, stop condition, current branch, worktree mode, push, commit message preset, sleep prevention, and agent argument overrides. As teams use the tool for different workflows, repeating these options becomes error-prone.

Common workflows may need different defaults:

- Bug fixes.
- Refactors.
- Test coverage.
- Documentation.
- PR review repair.
- CI repair.
- Exploration-only runs.

There is an opportunity to bundle options into named profiles.

Profiles must be additive. Running `gnhf` without `--profile` should continue to use the existing config and CLI precedence rules.

## Solution Options

### Option 1: Config-Only Profiles

Add profiles to `~/.gnhf/config.yml`:

```yaml
profiles:
  bugfix:
    agent: claude
    maxIterations: 3
    commitMessage:
      preset: conventional
```

Then run:

```sh
gnhf --profile bugfix "fix login timeout"
```

### Option 2: Project Profiles

Allow repo-local profiles in `.gnhf/config.yml` or another project file. This lets teams share workflow defaults.

The risk is mixing local project config with user-level config. Precedence rules must be very clear, and repo-local config should not reuse `.gnhf/runs/` because that directory is local runtime metadata. A tracked project config path should be chosen deliberately if this option is implemented.

### Option 3: Built-In Profiles

Ship opinionated built-ins:

- `quick`
- `thorough`
- `bugfix`
- `docs`
- `ci-repair`

Users can override them. This improves discoverability but may create expectations that do not fit every repo.

### Option 4: Prompt Prefix Profiles

Profiles could also include prompt guidance:

```yaml
profiles:
  tests:
    promptPrefix: "Focus on test coverage and avoid product behavior changes."
```

This makes profiles more powerful, but also raises instruction conflict risks.

## Test Strategy

- Unit test profile config parsing and validation.
- Unit test CLI profile selection overrides default config.
- Unit test explicit CLI flags override profile values.
- Unit test behavior is unchanged when no profile is selected.
- Unit test unknown profile returns a clear error.
- E2E test that `--profile bugfix` affects max iterations, agent selection, and commit config.
- Test telemetry records only the mode/profile name if acceptable and does not record prompt content.
- Test docs examples stay aligned with actual config schema.

## Feature Enhancement

Add:

```sh
gnhf --profile bugfix "fix crash on startup"
gnhf profiles list
```

Potential profile schema:

```yaml
profiles:
  bugfix:
    agent: claude
    maxIterations: 3
    maxTokens: 120000
    validation: normal
    push: false
    commitMessage:
      preset: conventional
```

This feature makes `gnhf` friendlier for repeated workflows and team standards.

Implementation should start with user-level config profiles only. Project-level profiles can follow once the config path, trust model, and precedence rules are settled.
