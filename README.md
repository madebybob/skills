# madebybob/skills

Personal Claude Code skills.

## Skills

### `process-pr-review`

Processes and acts on review comments left on a GitHub PR.

Invoke with a PR URL or number:

```
/process-pr-review https://github.com/owner/repo/pull/123
/process-pr-review 123
```

**What it does:**
1. Fetches all unresolved review threads via the GitHub GraphQL API
2. Assesses each comment and proposes an action (fix, refactor, or dismiss with reasoning)
3. After your approval, makes code changes, runs tests, commits, and resolves threads
4. Optionally pushes to the remote

**Requirements:** `gh` CLI, authenticated with repo access.

---

### `solve-merge-conflicts`

Resolves Git merge conflicts with deep contextual understanding — semantic, not just textual.

Triggers automatically when conflict markers appear, or invoke explicitly:

```
/solve-merge-conflicts
```

**What it does:**
1. Orients by reading git history to understand the *intent* on both sides
2. Classifies each conflict (textual, semantic-additive, semantic-contradictory, structural, lock file, config)
3. Asks clarifying questions before resolving any ambiguous or contradictory conflict — never silently discards code
4. Resolves each conflict using the appropriate strategy (accept one side, merge both, or synthesize)
5. Validates the result by running build/lint/tests before completing the merge, rebase, or cherry-pick
6. Reports a summary of every resolution with reasoning

**Requirements:** `git` with a repository that has active conflict markers or a pending merge/rebase/cherry-pick operation.

---

## Installation

Uses [`npx skills`](https://github.com/vercel-labs/skills) — the open agent skills CLI.

```bash
npx skills add madebybob/skills/code -a claude-code
```

To install non-interactively:

```bash
npx skills add madebybob/skills/code -a claude-code -y
```

Skills are available immediately in new Claude Code sessions. Invoke them with `/skill-name`.
