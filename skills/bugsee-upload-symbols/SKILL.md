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

Release builds strip or minify symbols, so crash stack traces arrive as raw addresses or mangled names. Upload the matching symbol artifacts so Bugsee can symbolicate traces back to your source. Each platform has its own workflow — pick the one that matches the project, and always verify the artifact's **version/build matches the build that shipped**.

## Invoke This Skill When

- Stack traces in Bugsee show hex addresses, minified names, or `<unknown>` frames
- The user asks to upload dSYMs, source maps, mapping files, ProGuard/R8 output, or IL2CPP symbols
- A release was shipped and crashes are coming in unsymbolicated

---

## iOS / iPadOS — dSYMs

Upload the debug symbol files (`.dSYM`) produced by the release build. Bugsee provides an Xcode build-phase run script that compresses and uploads dSYMs automatically, and supports manual upload (including dSYMs downloaded from App Store Connect when Bitcode/symbol stripping is involved).

- Docs: <https://docs.bugsee.com/sdk/ios/symbolication/>
- Tip: confirm `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` for the Release configuration.

## Android — mapping files (R8 / ProGuard)

When minification/obfuscation is enabled, upload the `mapping.txt`. The **Bugsee Android Gradle plugin** handles this for you: applied to the app module, it uploads the mapping (and native symbols) on each release build using the app token.

- Gradle plugin: <https://docs.bugsee.com/sdk/android/gradle-plugin/>
- Apply `id("com.bugsee.android.gradle")` and set the app token in the `bugsee { }` DSL via `appToken("<your-app-token>")`.

## React Native — JavaScript source maps

Use a **pinned** Bugsee CLI to inject debug IDs and upload JS source maps so JS frames symbolicate. Current npm latest (re-verified 2026-09-01): **`@bugsee/bugsee-cli@0.7.5`** (binary `bugsee-cli`).

```bash
npm install -g @bugsee/bugsee-cli@0.7.5
```

Token via `BUGSEE_APP_TOKEN` or `--app-token`. Inject after the bundler writes bundles and `.map` files, then upload:

```bash
export BUGSEE_APP_TOKEN="<APP_TOKEN>"

# 1. Inject debug IDs into the build output
bugsee-cli sourcemaps inject ./dist

# 2. Upload the injected source maps
bugsee-cli debug-files upload ./dist --type sourcemaps --version 1.2.3 --build 123
```

- CLI source maps: <https://docs.bugsee.com/cli/sourcemaps/>
- Debug-file uploads (including `--uuid` when the Gradle plugin owned the build UUID): <https://docs.bugsee.com/cli/debug-files/>
- The React Native SDK can also upload during native builds via the `bugsee-sourcemaps` helper (`make` / `generate` / `upload`) — see <https://docs.bugsee.com/sdk/react_native/crashes/>. That helper ships as a `react-native-bugsee` devDependency; do not `npm install -g bugsee-sourcemaps` unpinned.
- iOS and Android native frames in a React Native app still need dSYMs / mapping files — see the sections above.

## Flutter — symbolication

Flutter release builds use obfuscated Dart symbols; upload the split debug info so Dart frames resolve, plus native dSYMs/mapping for the platform layers.

- Docs: <https://docs.bugsee.com/sdk/flutter/symbolication/>

## .NET / MAUI — symbolication

Upload the symbol files for the release build so managed and native frames resolve.

- Docs: <https://docs.bugsee.com/sdk/dotnet/symbolication/>

## Xamarin — symbolication

- Docs: <https://docs.bugsee.com/sdk/xamarin/symbolication/>

## Kotlin Multiplatform (KMP)

There is no KMP-specific upload tool. Follow the **native** workflows on each target: iOS dSYMs and Android mapping files (above).

- Docs: <https://docs.bugsee.com/sdk/kmp/debug-symbols/>

## Unity

Upload the IL2CPP/native symbols for the platform you build to (iOS dSYMs, Android symbols). See the Unity SDK docs.

- Docs: <https://docs.bugsee.com/sdk/unity/crashes/>

---

## Verify

The Bugsee MCP server has no symbols-lookup tool ([usage](https://docs.bugsee.com/mcp/usage/) — eleven tools across applications, issues, and builds). Confirm symbolication from a **new** crash on the matching build, not from a lookup of uploaded files.

1. **Match version/build.** The artifact you uploaded must belong to the binary that shipped. Bugsee matches an iOS crash to the dSYM of that build ([symbolication](https://docs.bugsee.com/sdk/ios/symbolication/)). CLI uploads (`bugsee-cli` from `@bugsee/bugsee-cli@0.7.5`) record `--version` and `--build` on the symbol document — including `--type sourcemaps`. If the Android Gradle plugin embedded a build UUID, the mapping upload must use that same `--uuid` or the crash never resolves ([debug files](https://docs.bugsee.com/cli/debug-files/)).

2. **Trigger a fresh crash** on that build (or wait for the next real one). Existing issues keep the dump they were created with.

3. **Dashboard.** Open the new issue at <https://app.bugsee.com> and confirm application frames show file, symbol, and line — not hex addresses, minified names, or `<unknown>`.

4. **Issue tools.** With the MCP server connected, call `list_issues` (pass `version` when you know it) and then **`get_issue`** on the new issue key. Read the Exception section (add `include_all_threads: true` when you need the rest of the dump). Readable traces map to source; still-raw frames mean the version/build (or UUID) did not match — re-upload with the correct `--version` / `--build` (and `--uuid` when required) and repeat from step 2.
