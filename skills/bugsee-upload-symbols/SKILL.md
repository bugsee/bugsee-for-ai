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

## React Native — JavaScript source maps (`bugsee-sourcemaps`)

Use the `bugsee-sourcemaps` CLI to generate and upload JS source maps so JS frames symbolicate.

```bash
npm install -g bugsee-sourcemaps      # or: yarn global add bugsee-sourcemaps
```

`make` (generate + upload), `generate`, and `upload` are the available commands. Key options: `-t/--app-token` (required), `-v/--app-version`, `-p/--platform` (`ios` or `android`), `-c/--configuration` (`debug`/`release` — required for `upload`), `-s/--source-map` + `-b/--bundle` (paths, for `upload`), `-o/--overwrite`.

```bash
# Generate and upload for both configs of one platform:
bugsee-sourcemaps make -t <APP_TOKEN> -p ios -v 1.2.3 ./

# Upload a source map/bundle you already produced:
bugsee-sourcemaps upload -t <APP_TOKEN> -p android -c release \
  -s ./android-release.map -b ./android-release.bundle -v 1.2.3 ./
```

- Docs: <https://docs.bugsee.com/tools/sourcemaps/>
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

After uploading, trigger a fresh crash on the matching build (or wait for the next real one) and confirm the trace in the Bugsee dashboard is fully symbolicated. If frames are still raw, the artifact's **version/build number didn't match** the shipped build — re-upload with the correct `-v`/build value.
