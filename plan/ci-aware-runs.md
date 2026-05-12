# CI-Aware Runs

## Problem / Opportunity

`gnhf` can push after each successful iteration with `--push`, but it does not currently observe remote CI status. A run can finish locally while GitHub Actions or another CI provider fails afterward.

For autonomous work, CI feedback is valuable. It can catch platform-specific failures, full-suite failures, integration problems, and packaging issues that local validation missed.

There is an opportunity for `gnhf` to close the loop between agent commits and CI results.

CI awareness must be opt-in and should not change the existing `--push` behavior unless a CI flag is provided.

## Solution Options

### Option 1: CI Status Summary Only

After pushing, fetch status checks and include them in the exit summary. This is low risk and does not change iteration behavior.

### Option 2: Wait For CI

Add:

```sh
gnhf "fix release build" --push --wait-for-ci
```

The run waits for checks on the pushed commit. It exits success only if checks pass or exits aborted/failed when checks fail.

### Option 3: CI Repair Iteration

If CI fails, gather failing job names and logs, then start another iteration with that failure context injected into the prompt.

This is powerful but must avoid infinite repair loops.

Because CI failures occur after a commit has already been created and often pushed, repair should create a follow-up commit rather than trying to reset or rewrite published history.

### Option 4: Provider Abstraction

Create a CI provider interface:

```text
GitHub Actions provider
Generic command provider
Webhook provider
```

Start with GitHub Actions because this repo is on GitHub and many users will expect it.

## Test Strategy

- Unit test CI provider parsing for success, pending, failed, cancelled, and timed-out states.
- Unit test retry and timeout behavior.
- Unit test failed CI context is formatted safely for the agent prompt.
- E2E test with a mocked GitHub API or local fake provider.
- Test `--push` without `--wait-for-ci` remains unchanged.
- Test CI repair creates follow-up work instead of rewriting pushed commits.
- Test stop behavior on SIGINT while waiting for CI.
- Test logs do not include secrets from CI output. Redaction may be needed.

## Feature Enhancement

Add flags:

```sh
--wait-for-ci
--ci-timeout <minutes>
--repair-ci-failures
```

Potential prompt addition for repair iterations:

```markdown
## CI Failure Context

The last pushed commit failed CI. Review the failing checks below and make the smallest fix that addresses the failure.
```

This would make `gnhf` much more useful for unattended overnight runs.

The first implementation should probably be CI status summary or wait-only. Automatic CI repair should come later with strict retry limits, log redaction, and clear authentication requirements.
