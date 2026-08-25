---
name: bugsee-kmp-sdk
description: Full Bugsee SDK setup for Kotlin Multiplatform. Use when asked to add Bugsee to KMP, install bugsee-kotlin-multiplatform, or set up bug reporting, crash reporting, and video recording for KMP applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > KMP SDK

# Bugsee KMP SDK

Opinionated wizard that scans your Kotlin Multiplatform project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, and console logs on iOS and Android.

## Invoke This Skill When

- User asks to "add Bugsee to KMP" or "set up Bugsee" in a Kotlin Multiplatform app
- User wants bug reporting, crash reporting, video recording, or network monitoring in KMP
- User mentions `bugsee-kotlin-multiplatform`, Bugsee for Kotlin Multiplatform, or Bugsee KMP

> **Note:** Always verify against [docs.bugsee.com/sdk/kmp/installation/](https://docs.bugsee.com/sdk/kmp/installation/) before implementing.

---

## Phase 1: Detect

```bash
# Confirm KMP project
ls build.gradle.kts 2>/dev/null || ls build.gradle 2>/dev/null

# Check for kotlin multiplatform plugin
grep -i "kotlinMultiplatform\|kotlin.*multiplatform\|kotlin(\"multiplatform\")" build.gradle.kts 2>/dev/null

# Check for existing Bugsee dependency
grep -ri bugsee build.gradle.kts */build.gradle.kts 2>/dev/null

# Detect shared module (the one with commonMain)
find . -path "*/src/commonMain" -maxdepth 3 2>/dev/null

# Detect Android source set
find . -path "*/src/androidMain" -maxdepth 3 2>/dev/null

# Detect iOS source set
find . -path "*/src/iosMain" -maxdepth 3 2>/dev/null

# Check for Application class (Android)
grep -rl "Application()" --include="*.kt" */src/androidMain 2>/dev/null

# Check for ComposeUIViewController (iOS entry point)
grep -rl "ComposeUIViewController" --include="*.kt" */src/iosMain 2>/dev/null

# Check for Kotlin CocoaPods plugin
grep -i "kotlinCocoapods\|cocoapods" build.gradle.kts */build.gradle.kts 2>/dev/null
```

| Question | Impact |
|----------|--------|
| KMP `build.gradle.kts` exists? | Confirm KMP project |
| Already has `bugsee-kotlin-multiplatform`? | Skip install |
| Shared module with `commonMain` found? | Target for dependency |
| `Application` class found? | Android initialization location |
| `ComposeUIViewController` found? | iOS initialization location |
| Kotlin CocoaPods plugin used? | iOS pod setup location |

---

## Phase 2: Install

Add to the shared module's `build.gradle.kts` (the module containing `commonMain`). The native Android SDK is pulled in transitively. For iOS, add the Bugsee pod via the Kotlin CocoaPods plugin:

```kotlin
kotlin {
    cocoapods {
        // ... your existing cocoapods configuration

        pod("Bugsee")  // native iOS SDK; trunk latest 6.3.2. CocoaPods publishing ends Aug 2026 — see the iOS skill for SPM/Carthage if you are not using the Kotlin CocoaPods plugin.
    }

    sourceSets {
        commonMain.dependencies {
            implementation("com.bugsee:bugsee-kotlin-multiplatform:0.1.2")
        }
    }
}
```

Current Maven latest (re-verified 2026-08-25): **0.1.2**. Pin that version — docs still show `+`, which the KMP installation page itself warns against (`Pin your dependency to a specific version rather than using a floating range`).

> **Note:** The CocoaPods integration approach can't be used together with the `embedAndSignAppleFrameworkForXcode` mechanism used for direct integration. See [Kotlin CocoaPods overview](https://kotlinlang.org/docs/multiplatform/multiplatform-cocoapods-overview.html#set-up-an-environment-to-work-with-cocoapods).

Then sync Gradle:

```bash
./gradlew --refresh-dependencies
```

---

## Phase 3: Initialize

For the best Bugsee functionality, initialize the SDK as early as possible in the app lifecycle.

### Android

Add `Bugsee.launch()` in the `Application.onCreate()`:

```kotlin
import com.bugsee.kmp.Bugsee
import com.bugsee.kmp.BugseeLaunchOptions

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val options = BugseeLaunchOptions()
        options.monitorNetwork = true
        options.captureLogs = true
        options.videoEnabled = true
        options.viewHierarchyEnabled = true
        options.shakeToReport = true
        Bugsee.launch("<your_app_token>", options)
    }
}
```

In KMP projects the `AndroidManifest.xml` often has no `android:name` attribute on the `<application>` tag. Make sure to add it so the `Application` subclass is actually used:

```xml
<application
    android:name=".MyApplication"
    ...>
```

### iOS

Add `Bugsee.launch()` in your `ComposeUIViewController` entry point:

```kotlin
import androidx.compose.ui.window.ComposeUIViewController
import com.bugsee.kmp.Bugsee
import com.bugsee.kmp.BugseeLaunchOptions
import platform.UIKit.UIViewController

fun MainViewController(): UIViewController {
    val options = BugseeLaunchOptions()
    options.monitorNetwork = true
    options.captureLogs = true
    options.videoEnabled = true
    options.viewHierarchyEnabled = true
    options.shakeToReport = true
    Bugsee.launch("<your_app_token>", options)

    return ComposeUIViewController {
        App()
    }
}
```

> Replace `<your_app_token>` with the token from your Bugsee dashboard. It is common to use different tokens for iOS and Android apps.

---

## Phase 4: Configure (Optional)

Use `BugseeLaunchOptions` for common settings, or the platform-specific `BugseeLaunchOptionsAndroid` / `BugseeLaunchOptionsIos` classes:

```kotlin
import com.bugsee.kmp.*

// Android-specific
val androidOptions = BugseeLaunchOptionsAndroid()
androidOptions.videoMode = BugseeVideoMode.V3
androidOptions.handleAnr = true
Bugsee.launch("<your_app_token>", androidOptions)

// iOS-specific
val iosOptions = BugseeLaunchOptionsIos()
iosOptions.killDetection = true
Bugsee.launch("<your_app_token>", iosOptions)
```

Common options:

| Option | Default | Description |
|--------|---------|-------------|
| `videoEnabled` | `true` | Enable video recording |
| `crashReport` | `true` | Catch and report crashes |
| `monitorNetwork` | `true` | Capture network traffic |
| `captureLogs` | `true` | Capture console logs |
| `maxRecordingTime` | `60` | Max recording duration (seconds) |
| `shakeToReport` | `true` (Android) | Shake gesture to trigger report |
| `viewHierarchyEnabled` | `true` | Enable view hierarchy capturing |
| `wifiOnlyUpload` | `false` | Upload only on WiFi |

Full options: [docs.bugsee.com/sdk/kmp/configuration/](https://docs.bugsee.com/sdk/kmp/configuration/)

---

## Phase 5: Debug symbols (Production builds)

For readable stack traces from release builds, upload platform debug symbols. There is no KMP-specific upload tool — follow the native workflows on each platform.

### Android

In a typical KMP/Compose Multiplatform project, wire the Bugsee Gradle plugin into `composeApp/build.gradle.kts`:

```kotlin
// gradle/libs.versions.toml
//
// [versions]
// bugseeGradle = "4.0.5"
//
// [plugins]
// bugsee-gradle-plugin = { id = "com.bugsee.android.gradle", version.ref = "bugseeGradle" }


// composeApp/build.gradle.kts
plugins {
    alias(libs.plugins.androidApplication)
    alias(libs.plugins.kotlinMultiplatform)
    // ...other plugins...

    alias(libs.plugins.bugsee.gradle.plugin)
}

android {
    // ...your existing android configuration...

    bugsee {
        appToken("<your_app_token>")
        ndk {
            enabled.set(true)   // upload NDK debug symbols (only if your app ships native libraries)
        }
    }
}
```

The plugin uploads the ProGuard/R8 `mapping.txt` automatically at build time.

### iOS

Add a `Run Script` build phase to the **Post-actions** stage of your **iosApp** scheme — the standalone Xcode project (typically generated alongside `composeApp`) that hosts the iOS app and embeds the Kotlin/Native shared framework. The script must be added in the *iosApp* scheme, not in the Gradle/Kotlin layer.

Set `DEBUG_INFORMATION_FORMAT` to `dwarf-with-dsym` for the configurations you want to symbolicate. When using the Kotlin CocoaPods plugin, apply this in the Podfile `post_install` hook so all pods (including the Bugsee pod) produce dSYMs.

Full instructions and the BugseeAgent script: [docs.bugsee.com/sdk/kmp/debug-symbols/](https://docs.bugsee.com/sdk/kmp/debug-symbols/).

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/kmp/installation/)
- [Configuration](https://docs.bugsee.com/sdk/kmp/configuration/)
- [Manual invocation](https://docs.bugsee.com/sdk/kmp/manual/)
- [Custom data](https://docs.bugsee.com/sdk/kmp/custom/)
- [Console logs](https://docs.bugsee.com/sdk/kmp/logs/)
- [Privacy](https://docs.bugsee.com/sdk/kmp/privacy/overview/)
- [Network capture](https://docs.bugsee.com/sdk/kmp/network/)
- [Feedback](https://docs.bugsee.com/sdk/kmp/feedback/)
- [Appearance](https://docs.bugsee.com/sdk/kmp/appearance/)
- [Lifecycle events](https://docs.bugsee.com/sdk/kmp/lifecycle/)
- [Debug symbols](https://docs.bugsee.com/sdk/kmp/debug-symbols/)
- [Release notes](https://docs.bugsee.com/sdk/kmp/release-notes/)
