---
name: solve-merge-conflicts
description: >
  Resolves Git merge conflicts intelligently by deeply understanding code context, commit
  history, branch intent, and semantic meaning — not just textual diffs. Use this skill
  whenever git conflict markers appear in files (the less-than, equals, and greater-than
  fence lines), when a git merge, rebase, cherry-pick, or stash pop reports conflicts,
  when a git pull results in conflicting changes, or when you're asked to help resolve,
  understand, or explain any merge conflict. Also triggers when the user says things like
  "fix the conflicts", "help me merge this", "there's a conflict in", "merge is failing",
  or "rebase conflict". This skill always asks clarifying questions before auto-resolving
  any ambiguous conflict — it never silently discards code.
---

# Solve Merge Conflicts

A skill for resolving Git merge conflicts with deep contextual understanding. Inspired by
the insight that **most merge tool failures are semantic, not textual** — as Martin Fowler
notes in *Patterns for Managing Source Code Branches*, "even a clean merge can hide semantic
conflicts." This skill treats every conflict as requiring understanding of *intent*, not
just *text*.

---

## Phase 1: Orient — Understand What Happened

Before touching a single conflict marker, run the orientation commands below. Do not skip
this phase. Resolving without context is the primary cause of bad merges.

```bash
# 1. Identify ALL conflicted files and the current operation type
git status

# 2. Understand the branches involved
git log --oneline --graph --decorate -20

# 3. For each conflicted file, understand WHO changed WHAT and WHY
git log --oneline --follow -10 -- <conflicted-file>

# 4. Read commit messages on both sides to understand intent
git log --oneline MERGE_HEAD..HEAD -- <conflicted-file>   # our changes
git log --oneline HEAD..MERGE_HEAD -- <conflicted-file>   # their changes

# 5. See the three-way diff (base, ours, theirs)
git diff --diff-filter=U                                  # unified conflict view
git show :1:<file>   # common ancestor (base)
git show :2:<file>   # our version (HEAD)
git show :3:<file>   # their version (incoming)
```

Build a mental model: *What was each side trying to accomplish?* Write it out before
resolving. If you cannot answer this from the git log, **ask the user** (see Phase 3).

---

## Phase 2: Classify Each Conflict

Every conflict falls into one of these types. Classify before resolving.

| Type | Description | Default Strategy |
|------|-------------|-----------------|
| **Textual — clear winner** | One side is clearly correct (e.g. a hotfix) | Accept winner, annotate why |
| **Textual — both needed** | Both sides add independent code (e.g. imports, config entries) | Merge both |
| **Semantic — additive** | Both sides added logic that must coexist (e.g. two features) | Synthesize |
| **Semantic — contradictory** | Both sides changed the same logic differently | ⚠️ Ask user |
| **Structural** | Rename/move/delete vs. modify | ⚠️ Ask user |
| **Lock files / generated** | `package-lock.json`, `yarn.lock`, `*.generated.*` | See references/lockfiles.md |
| **Config / env** | `.env`, config values, feature flags | ⚠️ Ask user — high risk |

**Rule: When in doubt, classify up.** If it might be contradictory, treat it as
contradictory and ask. Never silently discard code.

---

## Phase 3: Ask Before Acting (Ambiguous Conflicts)

For any conflict that is **semantic-contradictory**, **structural**, or **config-related**,
stop and ask the user before resolving. Use this template:

```
I found a conflict in `<filename>` that I need your input on.

**Our side** (`<branch-name>`): <1-sentence summary of what our code does>
**Their side** (`<branch-name>`): <1-sentence summary of what their code does>

These are semantically contradictory — both change the same logic in different directions.

My question: <specific, concrete question>
Options:
  A) Keep ours — <consequence>
  B) Keep theirs — <consequence>  
  C) Merge both — <only if feasible, explain how>
  D) Something else — describe

Which should we use?
```

Do not ask multiple questions at once. One conflict block, one question.

For **clearly additive** conflicts (e.g. both sides added a new import, a new config key,
or a new test case), you may resolve automatically and report what you did.

---

## Phase 4: Resolve

### Resolution Patterns

**Pattern 1 — Accept one side (clear winner)**
```bash
git checkout --ours   <file>    # keep our version of entire file
git checkout --theirs <file>    # keep their version of entire file
git add <file>
```
⚠️ Only use for the *whole file* when you are 100% certain. Prefer manual editing for
partial conflicts.

**Pattern 2 — Manual synthesis (most common for semantic conflicts)**
1. Open the file
2. Remove ALL conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
3. Write the correct merged result — often neither verbatim ours nor theirs
4. Run `git diff` to verify only intended changes remain
5. `git add <file>`

**Pattern 3 — Keep both (additive)**
```
# Example: both sides added a new import
import { featureA } from './featureA';   # ours
import { featureB } from './featureB';   # theirs
# Result: keep both, alphabetically sorted
```

**Pattern 4 — Lock files / generated files**
→ See `references/lockfiles.md` for file-type-specific instructions.

### After Every File
```bash
# Verify no markers remain
grep -rn "<<<<<<\|=======\|>>>>>>>" <file>

# Mark resolved
git add <file>
```

---

## Phase 5: Validate Before Committing

Never commit without running validation. The level of validation scales with conflict
severity.

```bash
# 1. Confirm all conflicts are resolved
git diff --check
git status   # no "Unmerged paths" remaining

# 2. Build / lint (adapt to your stack)
# Node:   npm run build && npm run lint
# Python: python -m py_compile **/*.py
# Go:     go build ./...

# 3. Run tests (critical — semantic conflicts often pass textual checks but fail tests)
# npm test / pytest / go test ./... / etc.

# 4. Complete the operation
git commit                 # for merge
git rebase --continue      # for rebase
git cherry-pick --continue # for cherry-pick
```

If tests fail after resolution, **do not commit**. Re-examine the conflict — there is
likely a semantic conflict hiding behind the textual one.

---

## Phase 6: Report to User

After resolution, always summarize what was done:

```
## Merge conflict resolution summary

**Files resolved:** N
**Conflicts by type:**
  - Textual (kept ours): file1.ts — reason
  - Textual (kept theirs): file2.ts — reason  
  - Semantic (synthesized): file3.ts — description of synthesis
  - Asked user: file4.ts — user chose option B

**Validation:** build ✓ / lint ✓ / tests ✓ (or list failures)

**⚠️ Watch out for:** <any semantic risks you spotted but couldn't fully verify>
```

---

## Key Principles (Research-Backed)

1. **Semantic > textual** — As Fowler warns, code can merge cleanly at the text level while
   breaking at runtime. Always run tests. Git only sees lines; it doesn't see logic.

2. **Commit messages are first-class context** — `git log` is not optional. The commit
   message tells you the *intent* behind a change, which is what you need to resolve
   conflicts correctly.

3. **git blame identifies ownership** — When the git log doesn't clarify, `git blame`
   on the conflicting lines identifies which developer made the change and when, helping
   you trace the right person to ask.

4. **Never blindly --ours or --theirs a whole repo** — As the CHATMERGE research (IEEE
   2023) shows, most conflicts that "look" resolvable by choosing one side are actually
   synthesizable. Discarding the other side silently is the most common source of merge-
   introduced bugs.

5. **Test after every resolution** — DeployHQ's AI merge analysis (2026) identifies that
   the most valuable class of conflicts are "code that merges cleanly with no markers but
   silently breaks business logic." Tests catch these; code review alone often misses them.

6. **rerere for repeated conflicts** — Enable `git config rerere.enabled true`. Git will
   remember how you resolved a conflict and replay it automatically during future rebases
   of the same branch.

---

## Reference Files

- `references/lockfiles.md` — How to handle package-lock.json, yarn.lock, Gemfile.lock,
  poetry.lock, go.sum, and other generated/lock files
- `references/conflict-patterns.md` — Common conflict patterns by file type (imports,
  config, schema migrations, test files) with worked examples
- `references/rebase-vs-merge.md` — How --ours/--theirs semantics flip in rebase vs merge,
  and when to use each operation

Read the relevant reference file when you encounter that file type or pattern.
