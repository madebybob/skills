# Rebase vs Merge — Conflict Semantics

Understanding which operation you're in changes how you interpret `--ours` and `--theirs`.
This is one of the most confusing aspects of Git conflict resolution.

---

## The Ours/Theirs Swap Problem

In a **merge**, the semantics are intuitive:
- `--ours` = your current branch (HEAD)
- `--theirs` = the branch you're merging in

In a **rebase**, the semantics are *reversed* (counterintuitive!):
- `--ours` = the TARGET branch (the base you're rebasing onto)
- `--theirs` = YOUR commits being replayed on top

**Why?** During rebase, Git temporarily checks out the target branch and replays your
commits as if they were "incoming changes." So your own commits become `theirs`.

```
# During merge: git merge feature-branch
--ours   = main (your current branch) ✓ intuitive
--theirs = feature-branch (incoming)  ✓ intuitive

# During rebase: git rebase main (while on feature-branch)
--ours   = main (the base)          ← SWAP: this is NOT your branch
--theirs = your commits             ← SWAP: this IS your branch
```

**Rule:** When in doubt during rebase, use `git show :2:<file>` (always "ours" in the
index regardless of operation) and `git show :3:<file>` (always "theirs").

---

## Detecting Which Operation You're In

```bash
git status   # message will say "You are currently rebasing" or "merging"

# More reliable:
ls .git/MERGE_HEAD    # exists during merge
ls .git/rebase-merge/ # exists during rebase
ls .git/CHERRY_PICK_HEAD # exists during cherry-pick
```

---

## Operation-Specific Commands

### Merge
```bash
git merge --abort            # abort entirely
git merge --continue         # after resolving (just git commit also works)
```

### Rebase
```bash
git rebase --abort           # abort entirely, return to pre-rebase state
git rebase --continue        # after resolving one commit's conflicts
git rebase --skip            # skip this commit entirely (use rarely)
```

During rebase, conflicts may appear **per commit**. Each `git rebase --continue`
advances to the next commit. You may need to resolve conflicts multiple times.

### Cherry-pick
```bash
git cherry-pick --abort
git cherry-pick --continue
```

---

## The rerere Feature

Enable this — there is no downside:
```bash
git config --global rerere.enabled true
```

Git records how you resolved each conflict. When the same conflict appears again (e.g.
during a rebase retry), Git applies the previous resolution automatically.

You'll see this in output:
```
Resolved 'path/to/file' using previous resolution.
```

---

## When to Use Merge vs Rebase

| Situation | Prefer |
|-----------|--------|
| Shared/public branch | Merge (preserves history) |
| Private feature branch updating from main | Rebase (cleaner linear history) |
| Already pushed to remote and others have pulled | Merge (don't rewrite shared history) |
| Squashing commits before PR | Rebase + squash |
| Hotfix to multiple release branches | Cherry-pick |

Martin Fowler's guidance: integrate frequently regardless of strategy. The longer a branch
diverges, the more painful any merge or rebase becomes — exponentially so.

---

## Rebase Conflict Tips

From codeinthehole.com's guide on rebase conflicts:

1. **Resolve by applying YOUR change to the HEAD (target) block** — you understand your
   changes better and are less likely to inadvertently break the base branch's logic.

2. **Read the conflict using diff3 format** — enables you to see the common ancestor:
   ```bash
   git config merge.conflictstyle diff3
   ```
   Output includes a `|||||||` section showing the common base, making it easier to
   understand what each side changed relative to the original.

3. **Atomic commits make rebase conflicts much easier** — each commit motivated by a single
   reason means conflicts are isolated to the specific change, not tangled with unrelated
   edits (Single Responsibility Principle applied to commits).
