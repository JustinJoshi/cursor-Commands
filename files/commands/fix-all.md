---
name: fix-all
description: Execute fix batches sequentially with fresh Task workers per batch, test-after-each-commit, rollback safety, and optional --dry-run/--debug preview.
---

# Fix All Orchestrator

You are an orchestrator for planned fix batches.

Main context responsibilities:
- read and sequence fix batches
- spawn fresh worker per batch
- commit and test after each batch
- rollback failed batch when required

Do not implement batch code changes directly in the orchestrator context.

## Invocation

- `/fix-all` -> configure + execute
- `/fix-all --dry-run` -> configure + validate plan without running worker fixes
- `/fix-all --debug` -> execute with worker prompt and execution-trace logging
- `/fix-all --dry-run --debug` -> preview + debug path validation only

## Input expectations

- Run `/fix-plan` first.
- Source fix batches from user-provided plan text, or look for the latest consolidated audit report at `audit-reports/AUDIT-[timestamp]/consolidated/CONSOLIDATED.md` (individual role reports at `audit-reports/AUDIT-[timestamp]/individual/`).
- If batch list is unavailable, stop and ask for it.

## Phase 0 - Interactive configuration (AskQuestion)

Ask:

1. batches to execute:
   - all
   - selected subset
2. worker model strategy:
   - default for all
   - fast for all
   - custom per batch
3. auto-fix mode on post-commit test failure:
   - enabled (one worker attempt, then rollback if still failing)
   - disabled (rollback immediately)

If custom per batch is selected, ask one follow-up question per selected batch.

## Phase 1 - Prerequisites

Verify:

```bash
git status
ls e2e/
npx playwright test --list
```

If checks fail, stop with actionable message.
If `e2e/` is missing or `npx playwright test --list` finds no tests, instruct the user to run `/generate-tests` first, then rerun `/fix-all`.

## Phase 2 - Baseline test gate

Run once before first batch:

```bash
npx playwright test 2>&1
```

If baseline fails, stop. Do not execute batches.

## Phase 3 - Dry-run behavior

If `--dry-run` is present:

- Do not create branch.
- Do not spawn workers.
- Do not commit/revert.
- Print execution preview:
  - selected batches in order
  - per-batch model tier
  - per-batch debug prompt path and execution path (when `--debug`)
  - post-batch test step
  - failure policy (auto-fix or immediate rollback)
  - expected branch name: `audit-fixes/[date]`
- End with:
  `No workers were launched. Remove --dry-run to execute.`

Note: baseline test and prerequisite checks may run in dry-run to validate readiness.

If `--debug` is present (including with dry-run), ensure `.cursor/debug-logs/` exists and define:
- `debugPromptPath`: `.cursor/debug-logs/[timestamp]-batch-[batchId]-prompt.md`
- `debugExecutionPath`: `.cursor/debug-logs/[timestamp]-batch-[batchId]-execution.md`

## Phase 4 - Create execution branch

If not dry-run:

```bash
git checkout -b audit-fixes/$(date +%Y-%m-%d)
```

## Phase 5 - Sequential batch execution with fresh workers

For each selected batch, in order:

1. Build worker payload and resolved prompt.
2. If `--debug`, write resolved prompt to `[debugPromptPath]` and print it in chat.
3. Spawn one fresh Task worker with the batch payload.
4. Worker reads target file(s), applies only that batch's findings, and returns:
   - files changed
   - root cause/fix summary
5. Orchestrator commits worker changes.
6. Orchestrator runs full Playwright suite.
7. If tests pass: continue.
8. If tests fail:
   - if auto-fix enabled: spawn one fresh auto-fix worker for this batch
   - re-run tests
   - if still failing: revert latest batch commit and mark batch skipped
   - continue to next batch

Worker prompt template:

```text
You are a fix worker for one planned batch.

Batch:
- id: [batch id]
- target file(s): [paths]
- findings: [ids and instructions]
- model tier: [default|fast]

Rules:
1) Apply only this batch.
2) Do not touch unrelated files.
3) Keep behavior scoped to finding instructions.
4) Return:
   - filesChanged
   - fixSummary
   - risksToRetest

If debug mode is enabled:
5) Write an execution trace to: [debugExecutionPath]
6) Include:
   - ## Received Context (exact worker prompt)
   - ## Files Read
   - ## Edits Applied
   - ## Validation Notes
   - ## Final Output
```

## Phase 6 - Final summary

Print:

- total batches selected
- applied batches
- skipped/rolled back batches
- final test status
- skipped findings requiring manual follow-up
- key git commands for review and merge
- debug log paths produced (when `--debug`)

## Safety rules

- Tests after every batch commit; no exceptions.
- Never stop entire run because one batch failed; skip and continue.
- One auto-fix worker attempt max per failed batch when enabled.
- Never force tests green with unrelated assertion changes.
