# Cursor Commands Guide

This document covers every command in `.cursor/commands/` -- what each one does, how to use it, and the intended workflow that ties them together.

# Initial Setup

`git clone https://github.com/JustinJoshi/cursor-Commands.git`

- Clone this repo into a new directory on your desktop
- Copy the contents of "files" into the .cursor file of your project
- Start auditing!

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
| `/debug-tests` | 3 -- Debug | Iterative Playwright failure debugging with resumable sessions | Sonnet/Opus as needed |

## Workflow

The commands are designed to work as a pipeline. Each step builds on the output of the previous one.

```text
/audit-all → /generate-tests → /debug-tests
```

**Step 1 -- Audit.** Start with `/audit-all` to get a comprehensive picture of what needs attention across architecture, security, operations, accessibility, and code patterns. The consolidated report becomes the input for everything downstream. Use individual audit commands (`/audit-security`, `/audit-devops`, etc.) when you only need one perspective.

**Step 2 -- Generate tests.** Run `/generate-tests` after auditing so that Playwright test gates exist before you start making changes. This gives you a baseline to validate against and confidence that changes don't introduce regressions.

**Step 3 -- Debug.** After tests are generated, run `/debug-tests` to iteratively fix any failing Playwright tests. Each failure gets a fresh worker agent with focused context. The loop continues until all tests pass or stop conditions are reached.

**Tips:**

- Use `/audit-all` for periodic full-project quality gates, not just before debug runs.
- Keep `/generate-tests` and `/debug-tests` in your regular loop so changes remain verifiable.

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

### Step 3 -- Debugging

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
- **`.cursor/commands/`** -- Contains the 9 command definitions documented above.
- **`.cursor/debug-session.json`** -- Created by `/debug-tests` to persist retry state across stops and resumes. Safe to delete for a fresh start.
- **`.cursor/debug-logs/`** -- Created by `--debug` runs of `/debug-tests`. Contains resolved worker prompts and execution traces.
