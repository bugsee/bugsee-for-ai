---
name: bugsee-cli
description: Reference for the Bugsee CLI (`bugsee-cli`) — the single binary that uploads debug symbols, source maps, and build artefacts, and resolves build metadata in CI. Use when installing or invoking bugsee-cli, wiring it into a build script or CI job, choosing a command or flag, pinning a minimum CLI version, or interpreting a bugsee-cli exit code or error.
license: MIT
category: workflow
parent: bugsee-workflow
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [Workflows](../bugsee-workflow/SKILL.md) > Bugsee CLI

# Bugsee CLI

`bugsee-cli` is one cross-platform binary that owns everything Bugsee needs at build time: it uploads **debug information files** (dSYM, ELF, PDB, R8/ProGuard mappings, JS source maps, Unity IL2CPP line maps), injects **debug IDs** into JS bundles, uploads **build artefacts and metadata** for size analysis, and resolves **build-environment metadata** (VCS, CI provider, iOS dependency graph, Xcode version, Mach-O UUIDs).

Every Bugsee build integration — the Android Gradle plugin, the iOS SDK's build script, the fastlane plugin, the Unity post-build hook, the .NET MSBuild target, the Flutter Dart plugin — shells out to this binary rather than reimplementing HTTP, compression, retry, and the presigned-upload handshake. So the CLI is usually already there, installed by the plugin.

**Reach for the CLI directly when** there is no plugin for the build system, the project is in CI without Xcode or Gradle, symbols need to be uploaded from a script, or a one-off upload has to be done by hand.

For the platform-by-platform symbol-upload story, use [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md) — this skill is the command and contract reference behind it.

---

## Install

Check for an existing install first — a build plugin may have already placed one:

```bash
bugsee-cli --version    # -> "bugsee-cli 0.7.10"
```

| Channel | Command | Use for |
|---|---|---|
| Installer script | `curl --proto '=https' --tlsv1.2 -sSfL https://download.bugsee.com/cli/install.sh \| sh` | macOS / Linux, generic CI |
| Installer script | `powershell -ExecutionPolicy ByPass -c "irm https://download.bugsee.com/cli/install.ps1 \| iex"` | Windows |
| npm | `npm i -D @bugsee/cli` then `npx bugsee-cli` | JS toolchains — React Native, Cordova, Capacitor, web |
| Homebrew | Bugsee tap | macOS developer machines |
| Maven Central / NuGet / UPM | bundled by the plugin | Android Gradle, .NET MAUI, Unity |

Both installers download and **SHA-256-verify** the binary for the host from `download.bugsee.com` — no GitHub dependency. Override with `BUGSEE_CLI_VERSION` (pin an exact `X.Y.Z`), `BUGSEE_CLI_INSTALL_DIR`, or `BUGSEE_CLI_BASE_URL` (internal mirror).

**Prefer `@bugsee/cli` over `@bugsee/bugsee-cli` in JS projects.** `@bugsee/cli` ships the binary in per-platform packages declared as `optionalDependencies`, so npm resolves exactly one and **nothing runs at install time** — it works under `--ignore-scripts`, under a lockfile-pinned CI install, and offline from a warm cache. `@bugsee/bugsee-cli` is the older single package whose `postinstall` downloads the binary on every fresh install; it still works and is still published, but it needs network at install time and does nothing under `--ignore-scripts`.

Published platforms: macOS arm64 + x86_64, Linux x86_64 + aarch64, Windows x86_64 + arm64.

Keep an installed binary current with `bugsee-cli update` (same-major only — never a breaking major bump). In a build script use `bugsee-cli update --max-age 12h`, which checks at most once per interval and is best-effort: any failure is logged and exits 0, so it can run on every build without ever breaking one.

---

## Version floors

Integrations activate a new capability by pinning a **minimum CLI version**. Before writing a command into a build script, check the binary is new enough — an older one fails with a clap usage error (exit 2), which reads like a broken script rather than an old tool.

| Capability | Needs |
|---|---|
| `xcode upload-dsyms` (dSYM upload from a build phase) | **0.7.7** |
| Re-uploading a symbol the server already has is a skip, not a failure | **0.7.8** |
| `sourcemaps inject` registers a debug ID another tool (Rollup 4) wrote | **0.7.9** |
| `debug-files upload --type sourcemaps --concurrency N` / `--allow-empty` | **0.7.10** |

Current release: **0.7.10**. When a script needs a floor, gate on it:

```bash
bugsee-cli --version   # parse the X.Y.Z and compare, or just require a known-good install
```

---

## Auth

Two global values, each with a flag and an environment variable. A flag wins over its variable.

| | Flag | Env var | Default |
|---|---|---|---|
| App token | `--app-token` | `BUGSEE_APP_TOKEN` | — (required by upload commands) |
| API endpoint | `--endpoint` | `BUGSEE_ENDPOINT` | `https://api.bugsee.com` |

Prefer the environment variable in CI so the token never lands in a log or a shell history. Only the upload commands (`debug-files upload`, `upload build`, `upload build-info`, `xcode post-action`, `xcode upload-dsyms`) consume them; the metadata resolvers do no network I/O and ignore them.

**Never write a real app token into a file you commit.** Read it from the CI secret store.

---

## Commands

| Command | What it does |
|---|---|
| `debug-files upload <paths>...` | Discover, package, and upload debug files. `--type dsym\|elf\|pdb\|proguard\|sourcemaps\|il2cpp-linemap\|rust` (default `proguard`) |
| `sourcemaps inject <paths>...` | Embed a deterministic debug ID into JS bundles and their `.map` files. Run before the upload |
| `xcode post-action` | The whole iOS build-publish flow from an Xcode scheme post-action — build registration, build-info, dSYMs, size analysis, size gate |
| `xcode upload-dsyms` | dSYMs only, from an Xcode Run Script **build phase**, with no build registration |
| `upload build` / `upload build-info` | Build artefact (size analysis) and the per-build metadata bundle |
| `pack` | Pack an artefact + mapping into the upload ZIP locally, without uploading |
| `vcs-metadata` | Resolve provider / commit SHA / branch / PR from CI env vars or `git`. JSON to stdout |
| `ios-deps collect` | Parse `Podfile.lock` / `Package.resolved` / `Cartfile.resolved` / vendored frameworks. JSON to stdout |
| `build-env xcode-version\|machine-label\|read-plist` | Build-environment resolvers |
| `dsym uuid <path>` / `dsym slices <path>` | Mach-O UUIDs from a `.dSYM` bundle or binary. JSON to stdout |
| `update` | Self-update in place, same-major only |

`bugsee-cli <command> --help` prints the full, authoritative flag list for the installed version — **read it before writing a flag into a script**, especially for `xcode post-action`, whose `BUGSEE_*` environment variables are documented at the bottom of its help.

### Uploading debug files

```bash
# Android R8 / ProGuard mapping (the default --type)
bugsee-cli debug-files upload ./app/build/outputs/mapping/release \
    --version 1.4.0 --build 1400

# Apple dSYMs from an archive
bugsee-cli debug-files upload "$ARCHIVE/dSYMs" --type dsym \
    --version 1.4.0 --build 1400

# JS source maps, after injecting debug IDs
bugsee-cli sourcemaps inject ./dist
bugsee-cli debug-files upload ./dist --type sourcemaps \
    --version 1.4.0 --build 1400
```

`--version` and `--build` are required and must match the **shipped build** — a mismatch uploads a symbol that never resolves a crash. A symbol the server already has is skipped and the batch continues, so rebuilding uploads only what changed; `--force` re-uploads anyway. `--dry-run` discovers and packs without uploading.

### Source maps, specifically

`sourcemaps inject` stamps a content-derived debug ID into each bundle (`//# debugId=` plus a `globalThis._bugseeDebugIds` runtime registration) and into its `.map`. It is idempotent, and a bundle already carrying another tool's debug ID keeps that ID and gains only the registration. The upload then keys each map by that ID.

> **It only rewrites `.js`, `.cjs`, and `.mjs` files.** Any other extension — React Native's `main.jsbundle`, for instance — is skipped **silently**: `js_injected=0` and exit 0, with the failure surfacing later as a map with no debug ID. Check the `js_injected` count in the log.

Maps upload **several at a time** (0.7.10+). `--concurrency N` (1–32) is a ceiling, not a fixed width; left unset it scales with the batch. `--concurrency 1` restores strictly sequential uploads. An explicit `--uuid` forces sequential uploads regardless, because it keys every map under one ID.

Three behaviours worth knowing before wiring this into a build:

- **A path that does not exist is an error** (exit 10), even when other paths hold maps — a typo or a build that never ran cannot half-upload a release's symbols.
- **Nothing to upload is an error too**, unless `--allow-empty` says otherwise. Use it for a monorepo package that legitimately builds without maps.
- **A map without a debug ID fails the run** before anything uploads — it means `inject` never stamped that bundle. Stylesheet and type-declaration maps (`.css.map`, `.d.ts.map`, …) are skipped, not failed.

`--concurrency` and `--allow-empty` are rejected (exit 20) for any other `--type`, rather than accepted and ignored.

---

## Exit codes

The exit code is a stable contract — branch on it rather than on the message text.

| Code | Meaning | What an agent should do |
|---|---|---|
| `0` | Success — uploaded, or the server already had it, or a resolver returned empty output | Continue |
| `1` | Unexpected error | Report; a fallback path is reasonable |
| `2` | Usage / argv error | **Usually an outdated binary** — check `--version` against the floor above |
| `10`–`19` | Input / discovery problem (path missing, unparseable file) | Fix the path or the build settings; do not retry blindly |
| `20`–`29` | Configuration problem (missing or rejected token, invalid flags) | Fix the token or the flags |
| `30`–`39` | Upload problem (network, server 4xx/5xx) | Retry is reasonable; the CLI already retried with backoff |
| `40` | A deliberate build gate failed (size check) | Report the gate — this is the intended outcome, not a bug |

## Output contract

**stdout carries only the command's machine-readable result** — JSON for the metadata resolvers (`vcs-metadata`, `ios-deps`, `build-env`, `dsym`), a one-line report for `xcode post-action`. All logging, progress, and diagnostics go to **stderr**.

When shelling out and parsing, read stdout alone:

```bash
commit=$(bugsee-cli vcs-metadata | jq -r .commit_sha)
```

The metadata resolvers exit 0 with empty output (`[]`, `{}`, `null`, empty string) when they cannot resolve something, so check the output shape rather than the exit code for those.

---

## Documentation

- [CLI overview](https://docs.bugsee.com/cli/) · [Installation](https://docs.bugsee.com/cli/installation/) · [Configuration](https://docs.bugsee.com/cli/configuration/)
- [Debug information files](https://docs.bugsee.com/cli/debug-files/) · [Source maps](https://docs.bugsee.com/cli/sourcemaps/) · [Builds](https://docs.bugsee.com/cli/builds/)
- [iOS build publishing](https://docs.bugsee.com/cli/xcode/) · [Metadata resolvers](https://docs.bugsee.com/cli/metadata/) · [Exit codes](https://docs.bugsee.com/cli/exit-codes/) · [Self-update](https://docs.bugsee.com/cli/update/)

The docs site trails the binary for the newest additions — `xcode upload-dsyms`, `--concurrency`, and `--allow-empty` are not on it yet. **`bugsee-cli <command> --help` is authoritative for the installed version**; check it before telling a user a flag does not exist.
