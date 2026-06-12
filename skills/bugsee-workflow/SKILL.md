---
name: bugsee-workflow
description: Debug Bugsee issues, inspect builds, and keep stack traces readable. Use when asked to fix or triage Bugsee crashes, errors, or bug reports, investigate a Bugsee issue, check a build's size or whether it regressed, audit a build's dependency vulnerabilities, or upload debug symbols, source maps, or mapping files.
license: MIT
role: router
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md)

# Bugsee Workflows

Use Bugsee data to debug production issues and keep crash reports readable. This page helps you find the right workflow skill for the task.

## How to Use These Skills

- **Installed as a plugin (Claude Code / Cursor):** these skills are bundled locally. Open the skill linked below and follow it.
- **Standalone / other agents:** open the matching skill file in this repository and follow it.

## Start Here — Read This Before Doing Anything

Pick the workflow that matches the request:

1. The user wants to **fix, triage, or root-cause a Bugsee crash, error, or bug report**, or mentions a Bugsee issue key / the `/bugsee_fix` prompt → [`bugsee-fix-issues`](../bugsee-fix-issues/SKILL.md)
2. The user has **unreadable / unsymbolicated stack traces**, or wants to **upload dSYMs, source maps, mapping files, or symbols** → [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md)
3. The user asks about a **build's size, a size / dependency / timing regression, what a release added, the latest build of each variant, or dependency vulnerabilities** → [`bugsee-build-insights`](../bugsee-build-insights/SKILL.md)

When it is unclear which workflow is needed, **ask the user**. Do not guess.

---

## Workflow Skills

| Task | Skill | Path |
|---|---|---|
| Triage and fix crashes, errors, and bug reports (uses the Bugsee MCP server) | [`bugsee-fix-issues`](../bugsee-fix-issues/SKILL.md) | `bugsee-fix-issues/SKILL.md` |
| Inspect builds — size / dependency / timing regressions and dependency vulnerabilities (uses the Bugsee MCP server) | [`bugsee-build-insights`](../bugsee-build-insights/SKILL.md) | `bugsee-build-insights/SKILL.md` |
| Upload debug symbols / source maps / mapping files so traces are readable | [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md) | `bugsee-upload-symbols/SKILL.md` |

To set up a Bugsee SDK in a new project instead, see [`bugsee-sdk-setup`](../bugsee-sdk-setup/SKILL.md).
