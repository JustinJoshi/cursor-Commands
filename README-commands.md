# Cursor Commands Guide

This document covers every command in `.cursor/commands/` -- what each one does, how to use it, and the intended workflow that ties them together.

## How to invoke a command

- Open Cursor Agent chat.
- Type `/command-name` (e.g., `/audit-all`).
- If the command expects input, include it in the same message (e.g., `/audit-a11y-browser http://localhost:3000/dashboard`).

## Quick reference

Commands listed in workflow order.

| Command | Step | What it does | Recommended model |
|---|---|---|---|
| `/audit-all` | 1 -- Audit | Runs selected audit roles and writes consolidated report | Opus |
| `/audit-principal` | 1 -- Audit | Architecture and code quality review | Opus |
| `/audit-security` | 1 -- Audit | OWASP-focused security audit | Opus |
| `/audit-devops` | 1 -- Audit | Production readiness and operability audit | Sonnet |
| `/audit-a11y` | 1 -- Audit | WCAG code-level accessibility audit | Sonnet |
| `/audit-a11y-browser` | 1 -- Audit | Live URL accessibility audit via Playwright + axe-core | Sonnet |
| `/audit-dry` | 1 -- Audit | DRY/pattern duplication and abstraction audit | Sonnet |
| `/generate-tests` | 2 -- Test generation | Generates complete Playwright e2e suite and setup | Opus |
| `/fix-plan` | 3 -- Fix | Batches and sequences audit findings into execution plan | Sonnet/Opus split |
| `/fix` | 3 -- Fix | Executes one specific audit finding fix | Sonnet (most), Opus (architectural) |
| `/fix-all` | 3 -- Fix | Runs all fix batches with test gates and rollback | Sonnet (most), Opus (specific PE fixes) |
| `/fix-rollback` | 3 -- Fix | Reverts problematic fix commits/files/branches | Sonnet |
| `/debug-tests` | 4 -- Debug | Iterative Playwright failure debugging with resumable sessions | Sonnet/Opus as needed |

## Workflow

The commands are designed to work as a pipeline. Each step builds on the output of the previous one.

```text
/audit-all → /generate-tests → /fix-plan → /fix-all → /debug-tests
```

**Step 1 -- Audit.** Start with `/audit-all` to get a comprehensive picture of what needs attention across architecture, security, operations, accessibility, and code patterns. The consolidated report becomes the input for everything downstream. Use individual audit commands (`/audit-security`, `/audit-devops`, etc.) when you only need one perspective.

**Step 2 -- Generate tests.** Run `/generate-tests` after auditing so that Playwright test gates exist before you start making changes. This gives `/fix-all` a safety net to validate each batch of fixes against, and gives you confidence that fixes don't introduce regressions.

**Step 3 -- Plan and execute fixes.** Run `/fix-plan` to turn the audit report into a sequenced execution plan -- batched by file, ordered by severity, with model recommendations per batch. Then run `/fix-all` to execute the plan with automated test gating after every commit. For high-risk or architectural findings you want to review manually, use `/fix` to handle one finding at a time. If anything regresses, run `/fix-rollback` before re-planning.

**Step 4 -- Debug.** After fixes are applied, run `/debug-tests` to iteratively fix any remaining Playwright failures. Each failure gets a fresh worker agent with focused context. The loop continues until all tests pass or stop conditions are reached.

**Tips:**

- Use `/audit-all` for periodic full-project quality gates, not just before fix runs.
- Prefer `/fix` over `/fix-all` for principal-engineer-level architectural findings where you want to review each change.
- Keep `/generate-tests` and `/debug-tests` in your regular loop so fixes remain verifiable.

---

## Command details

### Step 1 -- Audit

#### `/audit-all`

Orchestrates selected audit roles using fresh Task sub-agents and consolidates the results.

- **Scope:** Audits the active file if one is provided; otherwise audits full `src/`.
- **Interactive configuration:** Before running, prompts you to select which roles to run and which model tier to use per role.
- **Execution:** Launches up to 4 concurrent Task workers. Each worker reads its role instruction file and produces a role-specific report.
- **Output:**
  - Per-role reports at `audit-reports/AUDIT-[timestamp]/individual/[role].md`.
  - Consolidated report at `audit-reports/AUDIT-[timestamp]/consolidated/CONSOLIDATED.md`.
  - In-chat summary table with severity counts by role.
- **Flags:**
  - `--dry-run` -- shows the execution plan (roles, models, concurrency batches, report paths) without launching workers.
  - `--debug` -- writes resolved worker prompts and execution traces to `audit-reports/AUDIT-[timestamp]/debug/`.
- **Model recommendation:** Opus.

#### `/audit-principal`

Principal Engineer review focused on architecture, code quality, React/Next.js patterns, and tech debt.

- **Key checks:** Responsibility boundaries, hook correctness, server/client component usage, complexity flags, async patterns.
- **Output:** Inline findings and a `PRINCIPAL ENGINEER AUDIT` summary block.
- **Model recommendation:** Opus.

#### `/audit-security`

Security audit aligned with OWASP Top 10.

- **Key checks:** Access control, cryptographic handling, injection risks, insecure design/misconfig, auth/session handling.
- **Output:** Confirmed security risks only, with OWASP mapping and summary block.
- **Model recommendation:** Opus.

#### `/audit-devops`

DevOps/platform readiness audit for production reliability.

- **Key checks:** Logging/observability, error handling and timeouts, env/config validation, scalability, deployment hygiene.
- **Output:** Operational risk findings and `DEVOPS AUDIT` summary block.
- **Model recommendation:** Sonnet.

#### `/audit-a11y`

WCAG 2.1 AA accessibility audit for React/Next.js code.

- **Key checks:** Semantic HTML, ARIA usage, keyboard accessibility, focus management, form/input assistance, state announcements.
- **Output:** Accessibility findings with fix patterns and summary block.
- **Model recommendation:** Sonnet.

#### `/audit-a11y-browser`

Live accessibility test against a running URL using Playwright MCP and axe-core.

- **Usage:** `/audit-a11y-browser http://localhost:3000` or `/audit-a11y-browser http://localhost:3000/dashboard`.
- **Key checks:** axe-core violations, keyboard tab order, modal focus trapping, skip links, aria-live behavior, desktop and mobile viewport checks.
- **Output:** In-chat quick summary and markdown report at `audit-reports/a11y-browser-[timestamp].md`.
- **Model recommendation:** Sonnet.

#### `/audit-dry`

Finds duplication and abstraction opportunities.

- **Key checks:** DRY violations, extractable hooks/components/utils, repeated constants/routes/types/schemas.
- **Output:** Actionable extraction suggestions and `DRY / PATTERNS AUDIT` summary block.
- **Model recommendation:** Sonnet.

---

### Step 2 -- Test generation

#### `/generate-tests`

Generates a full Playwright e2e suite for a Next.js app.

- **Discovery:** Reads route/component/backend structure, the latest audit report, and any existing tests.
- **Workflow:** Produces a test plan first and waits for your confirmation, then generates config, setup, spec files, a verification checklist, and install instructions.
- **Expected outputs:**
  - `playwright.config.ts` setup pattern.
  - Auth bootstrap (`e2e/global.setup.ts`, `.env.test`, helpers).
  - Spec files for auth, teams, documents, security, accessibility (adjusted to your app).
  - Verification checklist and setup instructions.
- **Model recommendation:** Opus.

---

### Step 3 -- Fix planning and execution

#### `/fix-plan`

Turns an audit report into a sequenced execution plan.

- **Input:** Pasted audit report or the latest file in `audit-reports/`.
- **What it produces:**
  - Batches grouped by file, ordered by severity.
  - New shared-file prerequisites (created before dependent batches).
  - Ordered execution list.
  - Model assignment summary (Sonnet vs Opus per batch).
- **Behavior:** Planning only -- does not execute any fixes.

#### `/fix`

Executes a single pre-diagnosed finding from an audit report.

- **Required input:** The finding text (ID, issue, fix instruction) and the relevant current code.
- **Output:** Short fix summary, complete updated file content, change log, and verification checklist.
- **Model guidance:** Sonnet for mechanical fixes; Opus for architectural `[PRINCIPAL]` work.

#### `/fix-all`

Executes all planned fix batches sequentially using fresh Task workers, with test gating after every commit.

- **Interactive configuration:** Prompts you to select which batches to execute, the worker model strategy, and whether auto-fix mode is enabled on post-commit test failures.
- **Prerequisites:** Requires a fix plan from `/fix-plan` and a working Playwright suite from `/generate-tests`. Verifies both before starting.
- **Core behavior:**
  - Creates an `audit-fixes/[date]` branch.
  - Spawns one fresh Task worker per batch.
  - Commits changes and runs the full Playwright suite after each batch.
  - On test failure with auto-fix enabled: spawns one auto-fix worker, re-runs tests, reverts the batch if still failing, and continues to the next batch.
- **Flags:**
  - `--dry-run` -- shows the execution plan (batches, models, failure policy, branch name) without launching workers.
  - `--debug` -- writes resolved worker prompts and execution traces to `.cursor/debug-logs/`.
- **Output:** Batch-by-batch status and final run summary with applied/skipped batches and git commands for review.
- **Model guidance:** Sonnet generally; Opus for PE-level architectural fixes.

#### `/fix-rollback`

Safely recovers from regressions after fix execution.

- **Usage patterns:**
  - `/fix-rollback` -- shows options and current fix commits.
  - `/fix-rollback all` -- abandon the entire fix branch.
  - `/fix-rollback last` -- revert the most recent fix commit.
  - `/fix-rollback SEC-H1` -- revert the commit for a specific finding.
  - `/fix-rollback convex/email.ts` -- restore a specific file from main.
- **Rollback options:** Single commit revert, revert last N, destructive reset (local-only), abandon branch, or restore one file.
- **Behavior:** Defaults to the least destructive option and confirms state after rollback.

---

### Step 4 -- Debugging

#### `/debug-tests`

Runs a retry loop that fixes failing Playwright tests using fresh Task workers with focused context.

- **Interactive configuration:** Prompts for mode (interactive or auto), session behavior (resume or fresh), worker model tier, and max retries (2, 3, or 5).
- **Invocation shortcuts:**
  - `/debug-tests` -- starts with interactive configuration.
  - `/debug-tests -in` -- interactive mode (pauses after each failed attempt).
  - `/debug-tests -auto` -- unattended mode (no pauses).
- **Core behavior:**
  - Runs Playwright, reads `test-results.json`, and classifies failures as independent or coupled.
  - Independent failures are fixed in parallel (up to 4 concurrent workers). Coupled failures are fixed sequentially.
  - Each failure gets a fresh worker with focused context: error details, relevant code, and prior attempt summaries.
  - After fixes are applied, re-runs the full suite and compares results.
- **Interactive controls:** After a failed attempt in interactive mode, reply:
  - `continue` -- run the next attempt.
  - `auto` -- switch to unattended mode for remaining attempts.
  - `stop` -- end and print the final summary.
- **Model switching:** In interactive mode, change the model in the Cursor dropdown at any pause, then reply `continue`. Switching to `auto` does not change the fresh-worker architecture -- it only removes pause prompts.
- **Stop conditions:** All tests pass, retry limit reached, no-progress guard (same failure set repeats), or user replies `stop`.
- **Session resume:** State is stored in `.cursor/debug-session.json`. If you stop or get interrupted, re-run `/debug-tests` and choose `resume` to continue from the last completed attempt. Only completed attempts are preserved; the next run re-validates actual test state before continuing.
- **Flags:**
  - `--dry-run` -- discovers failures and shows the worker plan (parallel/sequential grouping, model, debug paths) without launching workers.
  - `--debug` -- writes resolved worker prompts and execution traces to `.cursor/debug-logs/`.
- **Output:** Final report with attempts used, what was fixed per attempt, remaining failures, and recommended next steps.

---

## Troubleshooting

- **`test-results.json` missing:** Run `npx playwright test --project=chromium` once and verify the reporter config in `playwright.config.ts`.
- **Stale debug session:** Choose `fresh` when `/debug-tests` prompts, or delete `.cursor/debug-session.json`.
- **Visual failure context:** Run `npx playwright show-report` to see screenshots and traces.
- **Audit workers not spawning:** If `/audit-all` produces a single in-chat audit with no report files, workers were not launched. Check for errors and re-run.

## `.cursor` folder reference

- **`.cursor/mcp.json`** -- Configures MCP servers. This repo points to Playwright MCP via `npx @playwright/mcp@latest`.
- **`.cursor/commands/`** -- Contains the 13 command definitions documented above.
- **`.cursor/debug-session.json`** -- Created by `/debug-tests` to persist retry state across stops and resumes. Safe to delete for a fresh start.
- **`.cursor/debug-logs/`** -- Created by `--debug` runs of `/fix-all` and `/debug-tests`. Contains resolved worker prompts and execution traces.
