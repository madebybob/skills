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
