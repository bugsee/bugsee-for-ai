---
name: bugsee-fix-issues
description: Triage and fix Bugsee crashes, errors, memory/thread leaks, and bug reports using the Bugsee MCP server. Use when asked to fix a Bugsee issue, debug a crash, exception, or memory leak, investigate a bug report, root-cause a production issue, or when given a Bugsee issue key like MYAPP-123.
license: MIT
category: workflow
parent: bugsee-workflow
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [Workflows](../bugsee-workflow/SKILL.md) > Fix Issues

# Fix Bugsee Issues

Pull full crash/error/bug context from Bugsee into the editor, root-cause it in the local codebase, and propose a fix. This skill drives the **Bugsee MCP server** — every issue's stack trace, breadcrumbs, network log, and console log is fetched live, not guessed.

## Invoke This Skill When

- The user asks to "fix a Bugsee issue", "debug this crash", "look at this exception", or "triage overnight crashes"
- The user provides a Bugsee issue key (`APPKEY-NUMBER`, e.g. `MYAPP-123`)
- The user invokes the `/bugsee_fix` prompt from an MCP-capable agent
- The user wants to find which release introduced a regression, or turn a bug report into a code change

## Prerequisites: the Bugsee MCP Server

This skill needs the Bugsee MCP server connected. The issue tools it uses are read-only: `list_applications`, `list_issues`, `get_issue`, and `get_issue_resource` (a presigned URL for an issue's video, screenshot, network log, or attachment). The server also exposes build tools — for size, dependency, timing, or vulnerability questions use [`bugsee-build-insights`](../bugsee-build-insights/SKILL.md) instead.

If the tools are not available, help the user connect the server (base URL `https://api.bugsee.com/mcp`):

- **OAuth (recommended):** add an MCP server with the URL `https://api.bugsee.com/mcp`. The client opens a browser to sign in to Bugsee and approve once.
- **Personal access token:** generate a token in the Bugsee dashboard and use `https://api.bugsee.com/mcp/<token>`.

Full per-client setup: <https://docs.bugsee.com/mcp/configuration/>.

> If MCP cannot be connected at all, fall back to asking the user to paste the issue report from the Bugsee dashboard, then continue from Step 4.

---

## Step 1: Identify the Application

If you don't already know the application key, call **`list_applications`** (no parameters). Each entry has `key` (used in issue keys), `name`, `type` (`ios`/`android`/`web`), and `subtype` (e.g. `react_native`, `flutter`, `unity`). Confirm with the user which app to work in if there is more than one.

## Step 2: Find the Issue

- If the user gave an issue key (`MYAPP-123`), skip to Step 3.
- Otherwise call **`list_issues`** with `application_id_or_key` and narrow with the optional filters:
  - `type` — `"crash"`, `"error"`, or `"bug"`
  - `status` — `"open"` or `"closed"`
  - `version` — e.g. `"1.2.3"`
  - `reporter_email` — for a specific user's bug report
  - `sort` — `"events_desc"` (most frequent), `"users_desc"` (most users), `"date_desc"` (newest, default)
- Present a short ranked shortlist (key, type, severity, summary, events/users) and let the user pick, or proceed with the top item for "triage" requests. Use `nextCursor` to page when needed.

## Step 3: Pull Full Context

Call **`get_issue`** with the `issue_key`. Tuning parameters:

- `include_all_threads: true` — for deadlocks, ANRs, or when the crashed thread alone doesn't explain the failure (native dumps can have many threads, so the report grows).
- `include_logs: { entries: "errors", max_log_entries: 50, deduplicate_errors: true }` — start with errors-only for triage; switch to `"all"` (last N entries closest to the crash) when you need the full breadcrumb trail.

Memory/thread-leak issues (they arrive as `error`-type issues) include a **`# Memory leak analysis`** section inline — leaking class, retained size, GC-root reference chain, and a memory snapshot. No extra parameter needed.

Need the session video, a screenshot, the full network log, or a user attachment the report doesn't surface inline? Call **`get_issue_resource`** with the `issue_key` and a `resource_type` (e.g. `"video"`, `"screenshot"`, `"network"`, `"attachment"`, or `"memory.leak"` for the raw leak bundle) for a presigned URL. For text log analysis, prefer `get_issue` with `include_logs` (inline, no fetch round trip).

Read the timing, environment, exception details, stack frames, breadcrumbs, network events, any memory-leak analysis, and logs.

## Step 4: Root-Cause in the Codebase

- Map the top application stack frames (file, symbol, line) to files in the repo. Skip framework/system frames.
- Open those files and read the relevant code paths. Use the breadcrumbs, network log, and console log to reconstruct what happened just before the failure.
- For `subtype` apps (React Native, Flutter, Unity, etc.), match symbolicated frames to the source language. If frames are unsymbolicated/minified, the symbols haven't been uploaded — use the [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md) skill, then re-fetch.
- State the root cause in one or two sentences and cite the exact file and line before changing anything.

## Step 5: Propose and Apply the Fix

- Describe the fix and the reasoning. For anything non-trivial, confirm the approach with the user first.
- Make the smallest change that addresses the root cause. Match the surrounding code style. Add a regression test where the project has a test suite.

## Step 6: Verify

- Build / run the project's tests or the affected path.
- Note the issue key in the commit message so the fix is traceable (e.g. `Fix NPE in checkout (MYAPP-123)`).
- The Bugsee MCP exposes no issue-mutating tool — it cannot close or resolve the issue. Tell the user to verify the fix in a new build and resolve the issue from the dashboard.

---

## Common Workflows

- **Triage overnight crashes** — `list_issues` with `type:"crash"`, `status:"open"`, `sort:"events_desc"`; summarize the top few, then deep-dive the worst.
- **Regression hunting** — compare `list_issues` filtered by `version` across two releases to find what a release introduced.
- **Bug report → code change** — start from a `type:"bug"` issue (or `reporter_email`), read its description and attached session, then implement.

## The `/bugsee_fix` Prompt

Agents that support MCP prompts can invoke `/bugsee_fix` with an issue key; it loads the issue and walks through analysis and a proposed fix — the same flow as this skill.
