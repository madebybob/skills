---
name: process-pr-review
description: Process and act on review comments left on your GitHub PR. Trigger when the user shares a PR URL or number and wants to handle/address/process review feedback, Copilot suggestions, or code review comments. Fetches all unresolved review threads, assesses each one, proposes actions, and after approval: makes code changes, commits, and marks threads as resolved.
---

# Process PR Review Comments

## Input Parsing

Accept as argument:
- Full URL: `https://github.com/{owner}/{repo}/pull/{number}`
- PR number only (uses current repo via `gh repo view`)

Extract `owner`, `repo`, and `pr_number`.

## Phase 1: Fetch & Analyze

**1. Check current branch**

```bash
gh pr view {pr_number} --json headRefName -q '.headRefName'
git branch --show-current
```

If not on the PR branch, ask the user: "You're on `{current}`, not the PR branch `{head}`. Should I checkout?"

**2. Fetch all review threads via GraphQL** (REST lacks thread IDs and resolution status):

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 20) {
              nodes {
                databaseId
                body
                path
                line
                originalLine
                diffHunk
                author { login }
                createdAt
              }
            }
          }
        }
      }
    }
  }
' -F owner='{owner}' -F repo='{repo}' -F pr={pr_number}
```

**3. Filter threads** — exclude:
- `isResolved: true`
- Threads where all comments are from the authenticated user (`gh api user -q '.login'`)
- Bot-only informational threads (e.g. pure CI status, dependency update bots with no actionable suggestion)

**4. For each remaining thread**, identify:
- Root comment (first in thread)
- File path and line number
- Whether it's a suggestion (has a code block in body)

## Phase 2: Present Plan

For each unresolved thread, output:

```
### Comment #N — {path}:{line} by @{author}
> {abbreviated comment body — max 3 lines}

**Assessment**: {Agree | Partially agree | Disagree} — {1–2 sentence reasoning}
**Action**: {Fix code | Refactor | Dismiss — {reason} | Needs discussion}
**Details**: {What specifically will change, or why dismissing}
```

Then show a commit grouping plan:
- Group logically related changes into shared commits (same function/class, same concern)
- Unrelated changes get separate commits
- Style/nit-only changes group together per file

End with:
```
Ready to process {n} comment(s) across {m} commit(s). Approve to proceed, or tell me which to skip or override.
```

Wait for user confirmation before proceeding. User may approve all, skip specific numbers, or override assessments.

## Phase 3: Execute

For each approved comment/group in order:

1. Read the relevant file(s) and understand the context around the changed line
2. Make the code change
3. Run the most relevant test(s):
   ```bash
   php artisan test --filter={relevant_test_name}
   ```
   If tests fail: stop, report to user, do not continue
4. Stage specific files:
   ```bash
   git add {specific files}
   ```
5. Commit:
   ```bash
   git commit -m "$(cat <<'EOF'
   {short description of change}

   Addresses review comment by @{author} on PR #{pr_number}

   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   EOF
   )"
   ```
6. Resolve the thread:
   ```bash
   gh api graphql -f query='
     mutation($threadId: ID!) {
       resolveReviewThread(input: {threadId: $threadId}) {
         thread { isResolved }
       }
     }
   ' -f threadId='{thread_node_id}'
   ```

For dismiss-only items (no code change): resolve the thread without creating a commit.

## Phase 4: Summary

Report:
- Comments addressed: {n}
- Commits created: {m}
- Skipped: {list with reasons}

Ask: "Want me to push to the remote?"

