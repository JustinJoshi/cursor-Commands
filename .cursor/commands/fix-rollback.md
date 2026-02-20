---
name: fix-rollback
description: Roll back audit fixes — either a single commit, a specific batch, or the entire fix branch. Use when something breaks after running /fix-all.
---

# Fix Rollback

You are helping the user recover from a broken state after applying audit fixes. Diagnose what broke, identify the right rollback scope, and execute the safest recovery path.

## Usage

The user can invoke this in several ways:

```
/fix-rollback                        — shows all options and current fix commits
/fix-rollback all                    — roll back the entire audit-fixes branch
/fix-rollback last                   — revert only the most recent fix commit
/fix-rollback SEC-H1                 — revert the commit that fixed a specific finding
/fix-rollback convex/email.ts        — revert the commit that touched a specific file
```

---

## Step 1 — Assess the situation

First, show the user where they are:

```bash
git log --oneline audit-fixes/$(date +%Y-%m-%d) 2>/dev/null || git log --oneline | grep "^.*fix(" | head -20
```

Print the fix commits in a readable format:
```
Fix commits on this branch:
  abc1234  fix(email): SEC-H1, OPS-M1, OPS-M4
  def5678  fix(documentVersions): SEC-H2
  ghi9012  fix(auth): OPS-H1
  ...

Current HEAD: [hash] [message]
Clean working tree: [yes/no]
```

---

## Rollback Options

### Option A — Revert a single commit (safest)

Use when: one specific fix broke something and others are fine.

```bash
git revert [commit-hash] --no-edit
```

This creates a new commit that undoes only that change. All other fixes remain. Safe to use on shared branches.

### Option B — Roll back the last N commits

Use when: the last few fixes are bad but earlier ones are good.

```bash
# Revert last 3 commits (adjust N as needed)
git revert HEAD~3..HEAD --no-edit
```

### Option C — Reset to before a specific commit (destructive)

Use when: you want to completely remove a commit from history. Only safe on a local branch not yet pushed.

```bash
# Roll back to just before a specific commit
git reset --hard [commit-hash-before-the-bad-one]
```

### Option D — Abandon the entire fix branch

Use when: too many things broke, start the fix run over.

```bash
git checkout main
git branch -D audit-fixes/[date]
```

Then re-run `/fix-plan` and `/fix-all` after addressing the root cause.

### Option E — Restore a single file

Use when: only one file is broken, everything else is fine.

```bash
# Restore file to its state on main
git checkout main -- [filepath]
git commit -m "revert: restore [filepath] — fix caused regression"
```

---

## Decision Guide

Walk the user through this:

```
What broke?
├── One specific feature stopped working
│   └── → Find which commit touched that feature → Option A (revert that commit)
├── Multiple things broke after a specific point
│   └── → Find the bad commit → Option B or C
├── App won't start / build fails
│   └── → Check the most recent commit first → Option A on HEAD
│         If that doesn't fix it → Option B rolling back 2-3 commits
└── Everything is broken / not sure what's wrong
    └── → Option D (abandon branch, start over)
```

---

## Finding Which Commit Broke Something

Help the user bisect if needed:

```bash
# See what changed in a specific commit
git show [commit-hash]

# See all files changed in fix commits
git log --oneline --name-only | grep -A5 "fix("

# Find which commit last touched a specific file
git log --oneline -- [filepath]

# Compare current state of a file to main
git diff main -- [filepath]
```

---

## After Rolling Back

Once the rollback is done, confirm the state:

```bash
git status
git log --oneline -5
```

Then advise:
```
✅ Rollback complete.

What to do next:
- If you reverted a specific fix: re-examine that finding in the audit report
  and implement a corrected version manually before re-running /fix-all
- If you abandoned the branch: the issue that caused the break is likely
  [describe based on what broke] — address it in the fix plan before retrying
- Your other fixes are [safe on branch / lost — specify]
```

---

## Behavior Rules

- Always show the user what you're about to do before doing it — one line summary
- Default to the **least destructive** option that solves the problem
- Never force-push or rewrite history on anything that could be a shared branch
- If the working tree is dirty (uncommitted changes), stash first:
  ```bash
  git stash && git revert [hash] && git stash pop
  ```
- After any rollback, confirm with `git log --oneline -5` and print the result
