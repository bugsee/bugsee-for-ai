# Bugsee for AI

The official Bugsee plugin for AI coding assistants. It teaches Claude Code, Cursor, and other agents how to **set up Bugsee SDKs**, **debug Bugsee crashes and bug reports**, **track build size and dependency vulnerabilities**, and **upload symbols** for readable stack traces — using the Bugsee MCP server and a library of opinionated, platform-specific skills.

## What You Can Do

- **Add Bugsee to an app** — detect the platform and run a complete, verified SDK setup (install → initialize → configure) for iOS, Android, Flutter, React Native, Unity, .NET/MAUI, Xamarin, Cordova, or Kotlin Multiplatform.
- **Fix issues from Bugsee** — pull a crash, error, or bug report's full context (stack trace, breadcrumbs, network, logs) into your editor and root-cause it against your code.
- **Track build health** — check build size, what a release added or removed, size/dependency/timing regressions, and dependency vulnerabilities.
- **Keep traces readable** — upload dSYMs, JavaScript source maps, Android mapping files, and other symbols so production stack traces resolve to your source.

## Installation

### Claude Code

```bash
/plugin marketplace add bugsee/bugsee-for-ai
/plugin install bugsee@bugsee-plugin-marketplace
```

This enables the Bugsee skills and connects the Bugsee MCP server.

### Cursor

Point Cursor at this repository — skills and the MCP config live at the repo root. For MCP specifically, you can also follow the per-client setup at [docs.bugsee.com/mcp/configuration/](https://docs.bugsee.com/mcp/configuration/).

### OpenAI Codex

This repo is also a Codex plugin (`.codex-plugin/plugin.json`). Codex auto-discovers the skills from `./skills` and the Bugsee MCP server from `./.mcp.json`.

- **Skills:** point Codex at this repository, or install individual skills into `~/.codex/skills/` from this repo, e.g.:

  ```bash
  mkdir -p ~/.codex/skills/bugsee-ios-sdk
  curl -o ~/.codex/skills/bugsee-ios-sdk/SKILL.md \
    https://raw.githubusercontent.com/bugsee/bugsee-for-ai/main/skills/bugsee-ios-sdk/SKILL.md
  ```

  Other platforms use the matching `skills/bugsee-*-sdk/SKILL.md` path. Do not install from `https://docs.bugsee.com/ai/agent-skills/` — those published copies currently lag this plugin and must not be used for version pins or the iOS SPM URL.

- **MCP:** installing the plugin wires the Bugsee MCP via `.mcp.json`. To add it manually, configure `mcp_servers` in `~/.codex/config.toml` per the [Codex MCP docs](https://developers.openai.com/codex).

> Codex's runtime ignores the `disable-model-invocation` flag the skills carry for Claude Code, so every skill is normally discoverable in Codex.

### From Source

Clone the repository and load it as a local plugin/marketplace in your assistant. The skills live in `skills/`, and the MCP server is declared in `.mcp.json` and the Claude plugin manifest.

## Skills

### SDK Setup

`bugsee-sdk-setup` detects your platform and loads the right wizard: `bugsee-ios-sdk`, `bugsee-android-sdk` (7.x, the current Android SDK; `bugsee-android-sdk-6x` for the legacy 6.x line), `bugsee-flutter-sdk`, `bugsee-react-native-sdk`, `bugsee-unity-sdk`, `bugsee-dotnet-sdk`, `bugsee-xamarin-sdk`, `bugsee-cordova-sdk`, `bugsee-kmp-sdk`.

### Workflows

`bugsee-workflow` routes to:

- `bugsee-fix-issues` — triage and fix crashes/errors/bug reports using the Bugsee MCP server.
- `bugsee-build-insights` — inspect builds for size/dependency/timing regressions and dependency vulnerabilities via the Bugsee MCP server.
- `bugsee-upload-symbols` — upload dSYMs, source maps, and mapping files.

See [`SKILL_TREE.md`](SKILL_TREE.md) for the full index.

## Bugsee MCP Server

The plugin connects the Bugsee MCP server at `https://api.bugsee.com/mcp`. It exposes eleven tools across three families — applications (`list_applications`), issues (`list_issues`, `get_issue`, `get_issue_resource`), and builds (`list_builds`, `list_latest_builds`, `get_build`, `get_build_by_commit`, `get_build_regressions`, `list_build_vulnerabilities`, `trigger_build_vuln_scan`) — plus the `/bugsee_fix` prompt. All are read-only except `trigger_build_vuln_scan`. Authenticate with OAuth 2.1 (recommended) or a personal access token (`https://api.bugsee.com/mcp/<token>`).

- MCP overview & tools: <https://docs.bugsee.com/mcp/>
- Per-client setup: <https://docs.bugsee.com/mcp/configuration/>

## Prerequisites

- A Bugsee account and at least one application — sign up at <https://app.bugsee.com>.
- An app token per application (found in the app's settings) for SDK setup.
- For issue debugging, the Bugsee MCP server connected in your assistant.

## Contributing

Skills are single `SKILL.md` files under `skills/`. To add or update one, use the `bugsee-skill-creator` skill or follow `AGENTS.md`, then run:

```bash
scripts/build-skill-tree.sh        # regenerate SKILL_TREE.md + validate
scripts/build-skill-tree.sh --check
```

CI validates the skill tree on every PR. A weekly drift-detection workflow is included; it requires one-time setup (API/SDK-read secrets and a verified repo mapping) before it produces useful results.

## License

MIT. See [`LICENSE`](LICENSE).
