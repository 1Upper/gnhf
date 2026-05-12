# GitHub Issue And PR Workflow

## Problem / Opportunity

Many coding tasks begin from a GitHub issue, pull request comment, or review thread. Today, the user needs to copy relevant context into the prompt manually. That is workable, but it loses structure and makes it harder to link results back to the source request.

There is an opportunity for `gnhf` to understand GitHub issue and PR workflows directly while keeping the current CLI simple.

GitHub integration must be explicit. A plain URL prompt should not silently change meaning unless the user opts into URL fetching, because today any string can be the coding objective.

## Solution Options

### Option 1: Issue URL As Prompt

Allow:

```sh
gnhf https://github.com/owner/repo/issues/123
```

`gnhf` fetches the issue title, body, labels, and selected comments, then uses that as the objective.

To avoid breaking existing prompt behavior, this should likely be implemented as `gnhf issue <url>` or `gnhf --from-github <url>` rather than automatic URL detection in the base prompt argument.

### Option 2: PR Review Repair Mode

Add a mode that fetches unresolved PR review comments and asks the agent to address them.

```sh
gnhf pr-review https://github.com/owner/repo/pull/456
```

This naturally fits the iteration loop: fix one coherent group of comments, commit, repeat.

### Option 3: Automatic PR Creation

After a successful run, open a PR with:

- Objective.
- Commit list.
- Iteration summaries.
- Validation evidence.
- Links or summaries of notes and logs if appropriate. Local filesystem paths should not be posted to GitHub unless the user explicitly asks for them.

### Option 4: Comment Back Results

Post a summary comment to the issue or PR when the run completes. This should be opt-in to avoid noisy automation.

## Test Strategy

- Unit test URL parsing for issues and PRs.
- Unit test GitHub context formatting avoids excessive prompt size.
- Unit test private/sensitive fields are not logged or sent to telemetry.
- E2E test with mocked GitHub API responses.
- Test authentication missing, expired, or insufficient scope errors.
- Test PR body generation from run summary.
- Test issue comments with large logs are truncated safely.
- Test a plain URL prompt still behaves as a normal prompt unless GitHub fetching is explicitly enabled.

## Feature Enhancement

Add commands or flags:

```sh
gnhf issue <url>
gnhf pr-review <url>
gnhf "fix bug" --create-pr
```

Potential run metadata additions:

```text
.gnhf/runs/<runId>/github-context.md
.gnhf/runs/<runId>/pr-description.md
```

This makes `gnhf` more useful in real team workflows where the source of truth is often GitHub, not a manually written prompt.

The first implementation should fetch GitHub context into run metadata and the prompt. Mutating GitHub state, such as PR creation or comments, should be separate opt-in behavior with clear authentication errors and no telemetry of issue/PR content.
