---
name: bugsee-build-insights
description: Inspect builds and catch regressions with the Bugsee MCP server — app/install size, dependency changes, build timings, and dependency vulnerabilities (SCA). Use when asked about a build's size, whether a release regressed, what a build added or removed, the latest build of each variant, or to audit or re-scan a build's dependency vulnerabilities.
license: MIT
category: workflow
parent: bugsee-workflow
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [Workflows](../bugsee-workflow/SKILL.md) > Build Insights

# Bugsee Build Insights

Answer build-health questions from your editor using the Bugsee MCP server: app/install **size** and size regressions, **dependency** changes, **build-timing** regressions, and **dependency vulnerabilities** (software composition analysis).

## Invoke This Skill When

- "How big is build X / the latest release?" or "what did this release add or remove?"
- "Did this build/release regress?" — size, dependencies, or build timings vs the baseline
- "Show me the latest build of each variant" / "what's the current state of my builds?"
- "What does the build for commit `<sha>` look like?"
- "How many vulnerabilities are in this build? Any new ones since the last scan?" or "re-scan this build"

For crash/error/bug-report debugging, use [`bugsee-fix-issues`](../bugsee-fix-issues/SKILL.md) instead.

## Prerequisites: the Bugsee MCP Server

Needs the Bugsee MCP server connected (base URL `https://api.bugsee.com/mcp`). The build tools are read-only **except** `trigger_build_vuln_scan`, which queues a scan and requires `modify` permission on the application. If the tools aren't available, connect the server with OAuth, or the token URL `https://api.bugsee.com/mcp/<token>` — see [MCP configuration](https://docs.bugsee.com/mcp/configuration/).

## The build tools

| Tool | Use it to | Key parameters |
|---|---|---|
| `list_builds` | Page/filter build history | `application_id_or_key`; optional `format`, `build_configuration`, `version`, `package_id`, `query`, `sort` (`date_desc`/`size_desc`/…), `cursor` |
| `list_latest_builds` | One build per `(package_id, format, build_configuration)` lineup — "current state" | `application_id_or_key`; optional `limit` (≤ 200) |
| `get_build` | Full detail for one build; heavy fields opt-in | `application_id_or_key`, `build_id`, optional `include` tags |
| `get_build_by_commit` | Find a build by VCS commit SHA | `application_id_or_key`, `commit_sha` |
| `get_build_regressions` | Flat "did it regress?" across size / deps / timings | `application_id_or_key`, `build_id` |
| `list_build_vulnerabilities` | Vulnerability summary + diff vs the build's previous scan | `application_id_or_key`, `build_id` |
| `trigger_build_vuln_scan` | **(mutating)** queue a fresh vulnerability scan | `application_id_or_key`, `build_id` |

`get_build` `include` tags: `size_summary`, `size_diff_summary`, `dependencies_summary`, `dependencies_diff_summary`, `timings_diff_summary`, `vuln_scan_summary`, `vuln_scan_diff_summary`, `build_metadata`, `urls`, or `all`. Supply only the tags you need to keep responses small.

## Workflows

### Find the build

- **Current state** → `list_latest_builds` (one row per lineup; best for "what's the latest of each variant?").
- **History / filtered** → `list_builds` (page or filter by version, format, configuration, or `query`).
- **By commit** → `get_build_by_commit` with the `commit_sha`.

`list_builds` and `list_latest_builds` return the light projection with `has_*` discoverability flags; `get_build_by_commit` returns the same light fields without them. Use the returned `id` with the detail tools below. If you don't yet know the application, call `list_applications` first.

### Did it regress?

Prefer **`get_build_regressions`** for the flat answer (size / deps / timings vs the baseline). It returns `has_regressions` plus per-area deltas — e.g. `artifact_size_delta` / `artifact_size_pct`, added/removed/changed dependency counts, and `wall_clock_delta_ms`. Reach for `get_build` with specific `include` tags only when the user wants the raw summary blobs.

### Dependency vulnerabilities

- **Read** with `list_build_vulnerabilities` — severity counts (critical/high/medium/low/info), `vuln_scan_status`, scanned-at, and a `diff_summary` (new/resolved/unchanged) vs the build's previous scan. Use the in-band counts; only fetch the presigned `full_findings_url` when the user wants specific CVEs / advisory IDs.
- **Re-scan** with `trigger_build_vuln_scan` only when there's a concrete reason (explicit request, refreshed advisory databases, or a never-scanned build). It is **not idempotent** — branch on the returned `outcome`:
  - `queued` → poll `list_build_vulnerabilities` in a few minutes.
  - `already_in_progress` → poll, don't re-trigger.
  - `cooldown` → wait `retry_after_minutes`.
  - `dependencies_not_ready` → the dependency pipeline must finish first (check `get_build`).
  - `access_denied` → the user lacks `modify` permission.

## Reference

Full parameters, response fields, every `include` tag, and the scan-outcome envelope are documented at <https://docs.bugsee.com/mcp/usage/>.
