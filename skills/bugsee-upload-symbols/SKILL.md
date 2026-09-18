---
name: bugsee-upload-symbols
description: Make Bugsee stack traces readable by uploading debug symbols, source maps, and mapping files. Use when crash traces are unsymbolicated, minified, or obfuscated, or when asked to upload dSYMs, source maps, Android mapping/ProGuard/R8 files, or IL2CPP symbols.
license: MIT
category: workflow
parent: bugsee-workflow
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [Workflows](../bugsee-workflow/SKILL.md) > Upload Symbols

# Upload Symbols to Bugsee

Release builds strip or minify symbols, so crash stack traces arrive as raw addresses or mangled names. Upload the matching symbol artifacts so Bugsee can symbolicate traces back to your source.

Two ways to get symbols to Bugsee, and the choice is the same on every platform:

1. **The build plugin for your build system** — the Android Gradle plugin, the iOS SDK's post-action script, the Unity post-build hook, the .NET MSBuild target. Wire it once and every release build uploads automatically. **Prefer this** when the project uses that build system.
2. **[`bugsee-cli`](../bugsee-cli/SKILL.md) directly** — one binary, every format, works from any script or CI job. Use it when there is no plugin for the build system, when uploading from CI without Xcode or Gradle, or for a one-off upload of symbols that already exist.

The plugins all shell out to `bugsee-cli` underneath, so the two paths upload the same thing. See [`bugsee-cli`](../bugsee-cli/SKILL.md) for install, flags, exit codes, and version floors.

**Whichever path you take, the artifact's version and build number must match the build that shipped.** A mismatch uploads a symbol that is accepted and then never resolves a crash — the most common reason traces stay raw after a "successful" upload.

## Invoke This Skill When

- Stack traces in Bugsee show hex addresses, minified names, or `<unknown>` frames
- The user asks to upload dSYMs, source maps, mapping files, ProGuard/R8 output, or IL2CPP symbols
- A release was shipped and crashes are coming in unsymbolicated

---

## iOS / iPadOS — dSYMs

Three shapes, in order of preference.

### 1. Scheme post-action — the full flow

The iOS SDK's `BugseeAgent` script runs from the scheme's **Archive → Post-actions** stage and delegates to `bugsee-cli xcode post-action`. It uploads dSYMs *and* registers the build (build info, dependency graph, build timings, optional size analysis and an in-build size gate). It is gated to Release + Archive by default and never fails an already-signed build.

- Docs: [iOS symbolication](https://docs.bugsee.com/sdk/ios/symbolication/) · [CLI reference](https://docs.bugsee.com/cli/xcode/)
- Configured entirely through `BUGSEE_*` environment variables, each with a matching CLI flag. `bugsee-cli xcode post-action --help` lists both.

### 2. Run Script build phase — dSYMs only

`bugsee-cli xcode upload-dsyms` uploads dSYMs from a **build phase**, with none of the build-info gating — it neither registers a build nor uploads build-info, so it is safe on every build. This is the shape a React Native or Flutter config plugin can generate (it edits `project.pbxproj`), as opposed to the post-action, which means editing `.xcscheme` XML.

```bash
# Run Script build phase, after "Embed Frameworks"
"$SRCROOT/path/to/bugsee-cli" xcode upload-dsyms --app-token "$BUGSEE_APP_TOKEN"
```

It reads `DWARF_DSYM_FOLDER_PATH` (which Xcode sets in every Run Script phase) and falls back to `<ARCHIVE_PATH>/dSYMs`. Requires CLI **0.7.7+**.

**A genuine failure fails the build, on purpose** — a build phase that swallows errors means symbolication silently stops working and nobody notices until a crash report is unreadable. "Nothing to upload" (no dSYM folder, or a folder with no `.dSYM` bundles) is a success, not a failure.

Failing the build and detaching the upload are independent:

| Invocation | Fails the build | Waits for the upload |
|---|---|---|
| *(default)* | yes | yes |
| `--no-fail` | no | no — detaches |
| `--no-fail --no-background` | no | **yes** |
| `--fail --background` | *refused* — exit `2` (flags) or `20` (env) | — |

`--no-fail --no-background` is usually what CI wants: never break the build, but still wait, so a runner tearing down its process tree the moment `xcodebuild` returns cannot kill the upload mid-flight. `--fail --background` is refused rather than honoured, because a detached process's exit code reaches nobody. Each pair also has an environment variable (`BUGSEE_DSYM_UPLOAD_NO_FAIL`, `BUGSEE_DSYM_UPLOAD_BACKGROUND`); a flag overrides its variable.

> **Xcode 15+:** `ENABLE_USER_SCRIPT_SANDBOXING` defaults to `YES`, which stops a build phase from reading the dSYM folder. Set it to `NO` on the target, or declare the folder in the phase's input file lists. The scheme post-action is unaffected.

A detached run logs to `$PROJECT_TEMP_DIR/bugsee-cli.log` rather than the Xcode build log, so `--no-background` is also how you keep warnings visible.

### 3. Manual upload of existing dSYMs

For an archive you already have, or dSYMs downloaded from App Store Connect (Bitcode / symbol stripping):

```bash
bugsee-cli debug-files upload "$ARCHIVE/dSYMs" --type dsym \
    --version 1.4.0 --build 1400
```

It discovers every bundle recursively and skips ones the server already has. The dashboard's "Manual upload" button also accepts a `.zip` of dSYMs.

- Tip: confirm `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` for the Release configuration — without it there are no dSYMs to upload at all.

---

## Android — mapping files (R8 / ProGuard) and native symbols

The **Bugsee Android Gradle plugin** is the right answer for a Gradle project: applied to the app module, it uploads the R8/ProGuard mapping on each release build using the app token, and NDK symbols when enabled.

- Docs: [Gradle plugin](https://docs.bugsee.com/sdk/android/gradle-plugin/)
- Apply `id("com.bugsee.android.gradle")` and set the token in the `bugsee { }` DSL via `appToken("<your-app-token>")`. Enable native symbols with `ndk { enabled.set(true) }`.

Without Gradle — a prebuilt APK, or a CI job that only has the artifacts:

```bash
# R8 / ProGuard mapping (proguard is the default --type)
bugsee-cli debug-files upload ./app/build/outputs/mapping/release \
    --version 1.4.0 --build 1400

# Native ELF symbols — --uuid is required, and must match what the SDK reports
bugsee-cli debug-files upload ./path/to/libnative.so --type elf \
    --uuid <build-uuid> --version 1.4.0 --build 1400
```

`--type elf` requires `--uuid` because the upload-side ID must match the one the SDK reports at crash time. The Gradle plugin owns that value; if you are uploading by hand, get it from the plugin's resolved build ID rather than inventing one.

---

## JavaScript source maps — React Native, web, Cordova, Capacitor

Two steps: stamp a debug ID into the bundles, then upload the maps.

```bash
# 1. Inject debug IDs into bundles and their .map files (idempotent)
bugsee-cli sourcemaps inject ./dist

# 2. Upload the injected maps
bugsee-cli debug-files upload ./dist --type sourcemaps \
    --version 1.4.0 --build 1400
```

`inject` must run **after** the bundler and **before** the upload, on the same output — it writes a `//# debugId=` comment plus a runtime registration into each bundle and the matching ID into each `.map`. A bundle that already carries another tool's debug ID (Rollup 4's `output.sourcemapDebugIds`) keeps that ID and gains only the registration.

> **`inject` only rewrites `.js`, `.cjs`, and `.mjs` files.** Any other extension — notably React Native's default `main.jsbundle` — is skipped **silently**: the run reports `js_injected=0` and exits 0, and the upload then fails because the map has no debug ID. Check the `js_injected` count in the log, or emit the bundle with a `.js` name.

Three behaviours to know before wiring this into a build:

- **A path that does not exist is an error** (exit 10), even when other paths hold maps — so a typo or a build that never ran cannot half-upload a release's symbols.
- **Nothing to upload is an error too**, unless `--allow-empty` says otherwise (CLI 0.7.10+). Use it for a monorepo package that legitimately builds without maps.
- **A map without a debug ID fails the run** before anything uploads — it means `inject` never stamped that bundle. Stylesheet and type-declaration maps (`.css.map`, `.d.ts.map`) are skipped rather than failed.

Maps upload several at a time on CLI 0.7.10+; `--concurrency N` sets a ceiling and `--concurrency 1` restores sequential uploads. Re-running after a rebuild uploads only the chunks that changed.

**For React Native, the `bugsee-sourcemaps` helper remains the documented path** (`make` / `generate` / `upload`, with `-t/--app-token`, `-p/--platform`, `-c/--configuration`, `-v/--app-version`) — see [React Native crashes](https://docs.bugsee.com/sdk/react_native/crashes/) and [docs.bugsee.com/tools/sourcemaps](https://docs.bugsee.com/tools/sourcemaps/). It handles the `.jsbundle` naming that `inject` skips. It ships as a `react-native-bugsee` devDependency — **do not `npm install -g bugsee-sourcemaps` unpinned.**

Use `bugsee-cli` for React Native only when the bundle is emitted with a `.js` name. For web and other JS builds it is the better choice, since one binary covers JS maps *and* the native symbols the same app needs.

Pin the CLI rather than floating on latest — current release **0.7.10**:

```bash
npm i -D @bugsee/cli@0.7.10     # then: npx bugsee-cli sourcemaps inject ...
```

iOS and Android **native** frames in a React Native app still need dSYMs and mapping files — see the sections above.

---

## Flutter

Flutter release builds obfuscate Dart symbols; upload the split debug info so Dart frames resolve, plus native dSYMs / mapping files for the platform layers.

- Docs: [Flutter symbolication](https://docs.bugsee.com/sdk/flutter/symbolication/)
- The Flutter integration downloads and invokes `bugsee-cli` for the upload.

---

## Unity — IL2CPP line maps

An IL2CPP build needs the line-number mapping in addition to the platform's native symbols (iOS dSYMs, Android ELF):

```bash
bugsee-cli debug-files upload path/to/Symbols/LineNumberMappings.json \
    --type il2cpp-linemap \
    --version 1.2.3 --build 45 \
    --uuid <arm64-build-id>,<armeabi-build-id>
```

The mapping is keyed by the IL2CPP module UUID(s) (`libil2cpp` / `UnityFramework`) — comma-separate them, or repeat `--uuid`, for a multi-ABI Android build. Sibling `MethodMap.tsv` / `il2cppFileRoot.txt` are picked up automatically when they sit next to the JSON.

- Docs: [Unity crashes](https://docs.bugsee.com/sdk/unity/crashes/)

---

## .NET / MAUI and Xamarin

Upload the symbol files for the release build so managed and native frames resolve. The .NET MAUI MSBuild target bundles and invokes `bugsee-cli`.

- Docs: [.NET symbolication](https://docs.bugsee.com/sdk/dotnet/symbolication/) · [Xamarin symbolication](https://docs.bugsee.com/sdk/xamarin/symbolication/)

---

## Kotlin Multiplatform (KMP)

There is no KMP-specific upload tool. Follow the **native** workflows on each target: iOS dSYMs and Android mapping files, both above.

- Docs: [KMP debug symbols](https://docs.bugsee.com/sdk/kmp/debug-symbols/)

---

## Rust

A Rust project has no single symbol format — it is a `.dSYM` for Apple targets, a `.pdb` for `*-pc-windows-msvc`, and the ELF binary itself for Linux/Android. `--type rust` discovers whichever the build produced:

```bash
bugsee-cli debug-files upload --type rust target/release --version 1.4.0 --build 250
```

A stock `cargo build --release` emits nothing uploadable. Set `debug = 1` and `split-debuginfo = "packed"` under `[profile.release]`, and on Linux add `-C link-arg=-Wl,--build-id` to `rustflags`. The command reports whichever setting is missing and exits 10 when it finds no symbols at all; `--dry-run` shows the diagnosis without uploading.

---

## Verify

Use **`get_symbol_by_uuid`** to diagnose whether a module UUID is uploaded and in what status. Confirm symbolication from a **new** crash on the matching build — a lookup that finds a `ready` file is not by itself proof that existing issues will resolve.

1. **Diagnose with `get_symbol_by_uuid`.** When a crash frame or upload has a module/debug UUID, call `get_symbol_by_uuid` with that `uuid` (optional `application_id_or_key` to narrow). Matches return in any status (`uploading`, `processing`, `ready`, `broken`, `deleted`), which distinguishes never-uploaded from still-processing or broken — the usual reason a crash sits in `missing_sym`. An empty `symbols` array is **not** proof the upload is missing (matches may exist only on applications the caller cannot read). Multiple entries can be expected (co-resident formats sharing a UUID, e.g. `elf` plus `il2cpp-linemap`). See [MCP usage](https://docs.bugsee.com/mcp/usage/).

2. **Match version/build.** The artifact you uploaded must belong to the binary that shipped. Bugsee matches an iOS crash to the dSYM of that build ([symbolication](https://docs.bugsee.com/sdk/ios/symbolication/)). CLI uploads (`bugsee-cli` from `@bugsee/bugsee-cli@0.7.5`) record `--version` and `--build` on the symbol document — including `--type sourcemaps`. If the Android Gradle plugin embedded a build UUID, the mapping upload must use that same `--uuid` or the crash never resolves ([debug files](https://docs.bugsee.com/cli/debug-files/)).

3. **Trigger a fresh crash** on that build (or wait for the next real one). Existing issues keep the dump they were created with.

4. **Dashboard.** Open the new issue at <https://app.bugsee.com> and confirm application frames show file, symbol, and line — not hex addresses, minified names, or `<unknown>`.

5. **Issue tools.** With the MCP server connected, call `list_issues` (pass `version` when you know it). Issues still waiting on symbols have `symbolication_status` of `"missing_sym"` and often an empty `key` — pass the `id` to `get_issue` as `issue_id`. After a new crash on the matching build, call **`get_issue`** (by key, or `issue_id` if the key is empty). Read the Exception section (add `include_all_threads: true` when you need the rest of the dump). Readable traces map to source; still-raw frames mean the version/build (or UUID) did not match — re-upload with the correct `--version` / `--build` (and `--uuid` when required) and repeat from step 3.
