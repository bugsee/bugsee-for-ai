---
name: bugsee-sdk-setup
description: Set up Bugsee in any mobile or cross-platform project. Detects the user's platform and loads the right SDK skill. Use when asked to add Bugsee, install a Bugsee SDK, or set up bug reporting, crash reporting, or video recording in an app.
license: MIT
role: router
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md)

# Bugsee SDK Setup

Set up Bugsee bug reporting, crash reporting, video recording, and network monitoring in any supported platform. This page helps you find the right SDK skill for the project.

## How to Use These Skills

Each SDK skill is a complete, opinionated setup wizard. Read the whole skill before acting — these files are detailed and fetch tools often summarize them, losing steps that matter.

- **Installed as a plugin (Claude Code / Cursor):** the SDK skills are bundled locally. Open the skill linked in the **Skill** column below and follow it.
- **Standalone / other agents:** download the full skill with `curl` from the docs, then read and follow it:

      curl -sL https://docs.bugsee.com/ai/agent-skills/sdk/ios/SKILL.md

  Append the path from the **Online docs** column. Do not guess or shorten URLs.

## Start Here — Read This Before Doing Anything

**Do not skip this section.** Do not assume which SDK the project needs and do not start installing packages or editing files until you have confirmed the platform.

1. **Detect the platform** from project files (`Podfile`/`*.xcodeproj`, `build.gradle`(`.kts`), `pubspec.yaml`, `package.json`, `config.xml`, `*.csproj`, `*.unity`/`ProjectSettings`, `*.sln`, etc.).
2. **Tell the user what you found** and which SDK you recommend.
3. **Wait for confirmation**, then open the matching skill and follow it step by step.

Each SDK skill contains its own detection logic, prerequisites, and step-by-step configuration. Trust the skill — do not improvise or take shortcuts.

---

## SDK Skills

| Platform | Skill | Online docs |
|---|---|---|
| iOS / iPadOS (Swift, Objective-C) | [`bugsee-ios-sdk`](../bugsee-ios-sdk/SKILL.md) | `sdk/ios/SKILL.md` |
| Android (Kotlin, Java) | [`bugsee-android-sdk`](../bugsee-android-sdk/SKILL.md) | `sdk/android/SKILL.md` |
| Android 7.x (Beta) | [`bugsee-android-sdk-7x`](../bugsee-android-sdk-7x/SKILL.md) | `sdk/android/v7/SKILL.md` |
| Flutter / Dart | [`bugsee-flutter-sdk`](../bugsee-flutter-sdk/SKILL.md) | `sdk/flutter/SKILL.md` |
| React Native | [`bugsee-react-native-sdk`](../bugsee-react-native-sdk/SKILL.md) | `sdk/react-native/SKILL.md` |
| Unity | [`bugsee-unity-sdk`](../bugsee-unity-sdk/SKILL.md) | `sdk/unity/SKILL.md` |
| .NET / MAUI | [`bugsee-dotnet-sdk`](../bugsee-dotnet-sdk/SKILL.md) | `sdk/dotnet/SKILL.md` |
| Xamarin | [`bugsee-xamarin-sdk`](../bugsee-xamarin-sdk/SKILL.md) | `sdk/xamarin/SKILL.md` |
| Cordova | [`bugsee-cordova-sdk`](../bugsee-cordova-sdk/SKILL.md) | `sdk/cordova/SKILL.md` |
| Kotlin Multiplatform (KMP) | [`bugsee-kmp-sdk`](../bugsee-kmp-sdk/SKILL.md) | `sdk/kmp/SKILL.md` |

Online docs paths are relative to `https://docs.bugsee.com/ai/agent-skills/`.

### Platform Detection Priority

When more than one SDK could match, prefer the more specific one:

- **Flutter** (`pubspec.yaml` with a `flutter:` section or `bugsee_flutter`) → `bugsee-flutter-sdk`
- **React Native** (`package.json` with `react-native` or `react-native-bugsee`) → `bugsee-react-native-sdk` over a bare iOS/Android setup
- **Cordova** (`config.xml` + `www/`) → `bugsee-cordova-sdk`
- **Unity** (`ProjectSettings/`, `*.unity` scenes) → `bugsee-unity-sdk`
- **.NET MAUI** (`*.csproj` with `net*-ios`/`net*-android` target frameworks) → `bugsee-dotnet-sdk`
- **Xamarin** (`*.csproj` referencing `Xamarin.*`, `Xamarin.Forms`) → `bugsee-xamarin-sdk`
- **Kotlin Multiplatform** (`*.gradle.kts` with `kotlin("multiplatform")`, `bugsee-kotlin-multiplatform`) → `bugsee-kmp-sdk`
- **Android only** (`build.gradle`/`build.gradle.kts` with the Android plugin, no cross-platform framework) → `bugsee-android-sdk`. Use `bugsee-android-sdk-7x` **only** when the user explicitly asks for 7.x / beta.
- **iOS only** (`Podfile`, `*.xcodeproj`/`*.xcworkspace`, no cross-platform framework) → `bugsee-ios-sdk`
- **No match** → point the user to the [Bugsee Docs](https://docs.bugsee.com/).

## Quick Lookup

Match the project to a skill by keywords.

| Keywords | Skill |
|---|---|
| ios, ipados, swift, objective-c, cocoapods, spm, swiftui, uikit, xcode | `bugsee-ios-sdk` |
| android, kotlin, java, gradle, jetpack compose, okhttp, ktor | `bugsee-android-sdk` |
| android 7, android beta, apm, auto-init, extension modules | `bugsee-android-sdk-7x` |
| flutter, dart, pubspec | `bugsee-flutter-sdk` |
| react native, expo, react-native-bugsee, metro | `bugsee-react-native-sdk` |
| unity, c#, game, il2cpp, mono | `bugsee-unity-sdk` |
| .net, maui, dotnet, blazor hybrid | `bugsee-dotnet-sdk` |
| xamarin, xamarin.forms, xamarin.ios, xamarin.android | `bugsee-xamarin-sdk` |
| cordova, phonegap, ionic, config.xml | `bugsee-cordova-sdk` |
| kmp, kotlin multiplatform, compose multiplatform, shared module | `bugsee-kmp-sdk` |

---

## Finding Your App Token

Every Bugsee SDK is initialized with an **app token** that identifies the application. If the user doesn't have theirs:

1. Sign in at [`https://app.bugsee.com`](https://app.bugsee.com).
2. Open the application (or create a new one with **New App**).
3. Copy the **app token** shown in the app's settings.

You can help the user open the dashboard directly:

```bash
open https://app.bugsee.com         # macOS
xdg-open https://app.bugsee.com     # Linux
start https://app.bugsee.com        # Windows
```

> **Note:** Apps that ship on both iOS and Android are registered as **two separate Bugsee apps** and each gets its own token. Pass the correct per-platform token at launch. The app token is not a server secret, but treat it like other build configuration — keep it out of public repos where practical and inject it per build/environment.
