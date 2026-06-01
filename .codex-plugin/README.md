# Codex Plugin Metadata

This directory contains OpenAI Codex-specific plugin configuration.

- `plugin.json` — the Codex plugin manifest (name, version, description, `interface` block, and the paths to the shared `skills/` and `.mcp.json`).

Shared content lives at the repo root and is auto-discovered by Codex: skills from `./skills` and the read-only Bugsee MCP server from `./.mcp.json`.

> **Note on `disable-model-invocation`:** the shared SDK and workflow skills set `disable-model-invocation: true` so Claude Code keeps them hidden behind the router skills. Codex's runtime does not read this field — skills load and are discoverable normally — so it is intentionally left in place. Codex's optional `validate_plugin.py` authoring lint flags it; that check is for plugin authors and does not affect loading or using the plugin.

For the plugin specification, see the [Codex plugin docs](https://developers.openai.com/codex).
