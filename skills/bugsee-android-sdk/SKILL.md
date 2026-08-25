---
name: bugsee-android-sdk
description: Full Bugsee SDK setup for Android (7.x — the current SDK). Use when asked to add Bugsee to Android, install bugsee-android, or set up bug reporting, crash reporting, video recording, APM, or network monitoring for Android apps. For legacy 6.x apps, use bugsee-android-sdk-6x.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Android SDK

# Bugsee Android SDK

Opinionated wizard that scans the Android project and wires up Bugsee 7.x — core SDK, Gradle plugin, extension modules for network clients / Compose / feedback / NDK, APM, and manifest-based auto-launch.

> **7.x is the current Android SDK** (`com.bugsee:bugsee-android:7.1.3`, Gradle plugin `com.bugsee.android.gradle` / `com.bugsee:bugsee-android-gradle-plugin:4.0.5`, re-verified 2026-08-25 on Maven Central and the Gradle Plugin Portal) and the default for new and existing apps. It is plugin-based with a new API. If you are maintaining an app still pinned to the 6.x line, use the legacy [6.x skill](../bugsee-android-sdk-6x/SKILL.md) instead; when upgrading from 6.x, follow the [migration guide](https://docs.bugsee.com/sdk/android/migration/).

## Invoke This Skill When

- User asks to "add Bugsee to Android" or "set up Bugsee" / bug reporting / crash reporting / video recording in an Android app (Kotlin or Java).
- User mentions `bugsee-android`, `com.bugsee:bugsee-android`, or the **Bugsee Gradle plugin** (`com.bugsee.android.gradle`).
- User mentions **APM**, transactions, or spans (`Bugsee.startTransaction`, `startSpan`).
- User mentions **extension modules** (`bugsee-android-okhttp`, `bugsee-android-ktor-2`, `bugsee-android-ktor-3`, `bugsee-android-cronet`, `bugsee-android-compose`, `bugsee-android-feedback`, `bugsee-android-ndk`, `bugsee-android-leak`).
- User asks for **manifest auto-launch** / no Application subclass.
- User asks about **Compose secure modifier**, `Modifier.bugseeSecure()`, Ktor / Cronet / `HttpEngine` integration, **WebSocket** capture, **`FLAG_SECURE`**, or `Bugsee.getStatus()`.
- User mentions detection providers: `DetectAndReportMainThreadMisuse`, `DetectAndReportExit*`, `DetectAndReportEarlyCrash`.

For an app explicitly pinned to the **6.x** line (or when the user asks for "6.x" / the "legacy" SDK), switch to the [6.x skill](../bugsee-android-sdk-6x/SKILL.md).

> **Always verify against** [docs.bugsee.com/sdk/android/installation/](https://docs.bugsee.com/sdk/android/installation/) before implementing. Prefer Maven/Plugin Portal versions over the installation page's placeholders when they disagree.

---

## Phase 1: Detect

```bash
# Build system + language
ls build.gradle build.gradle.kts settings.gradle settings.gradle.kts 2>/dev/null
ls app/build.gradle app/build.gradle.kts 2>/dev/null
find app/src/main -name "*.kt" 2>/dev/null | head -3
find app/src/main -name "*.java" 2>/dev/null | head -3

# SDK versions — 7.x requires minSdk >= 21, AGP >= 8.6, Gradle >= 8.7
grep -E 'minSdk|targetSdk|compileSdk' app/build.gradle app/build.gradle.kts 2>/dev/null | head -6
grep -E 'com\.android\.tools\.build:gradle|agp|AGP' build.gradle build.gradle.kts settings.gradle settings.gradle.kts gradle/libs.versions.toml 2>/dev/null | head -5
cat gradle/wrapper/gradle-wrapper.properties 2>/dev/null | grep distributionUrl

# Existing Bugsee install
grep -ri "bugsee" app/build.gradle app/build.gradle.kts build.gradle build.gradle.kts settings.gradle* 2>/dev/null | head -10

# HTTP clients in use (decides which extension modules to add)
grep -rE 'com\.squareup\.okhttp3|io\.ktor:ktor-client|org\.chromium\.net|okhttp3\.OkHttpClient|HttpClient\(' app/build.gradle* app/src 2>/dev/null | head -10

# Compose?
grep -rE 'androidx\.compose|@Composable' app/build.gradle* app/src 2>/dev/null | head -5

# Native / NDK?
ls app/src/main/cpp app/src/main/jni 2>/dev/null
grep -rE 'ndk|externalNativeBuild|cmake' app/build.gradle* 2>/dev/null | head -5

# Application subclass + manifest (for programmatic-vs-auto-launch decision)
grep -r "extends Application\|: Application()" app/src/main --include="*.java" --include="*.kt" 2>/dev/null | head -3
grep -E 'android:name=|<application' app/src/main/AndroidManifest.xml 2>/dev/null | head -5
```

Decision table:

| Signal | Action |
|---|---|
| AGP < 8.6 or Gradle < 8.7 | **Stop** — tell user to upgrade before applying the Bugsee Gradle plugin. |
| `minSdk` < 21 | **Stop** — 7.x requires `minSdk = 21`. |
| `build.gradle.kts` present | Use Kotlin DSL snippets below. |
| `build.gradle` (Groovy) | Use Groovy snippets. |
| OkHttp 3/4 detected | Plugin auto-installs `bugsee-android-okhttp`; no manual wiring. WebSocket via `OkHttpClient.newWebSocket(...)` is captured automatically (plugin **4.0.3+**). |
| OkHttp 2 only | Not supported in 7.x — user must migrate to OkHttp 3/4 or use the [6.x skill](../bugsee-android-sdk-6x/SKILL.md). |
| Ktor 2.x / 3.x detected | Plugin auto-installs extension, but user must `install(BugseeKtor2Plugin)` / `install(BugseeKtor3Plugin.Plugin)` on every `HttpClient`. WebSocket on the OkHttp engine is automatic; on CIO, route calls through the `bugseeWebSocket` helper (plugin 4.0.3+). |
| Cronet detected | Plugin auto-installs extension, but user must wrap every engine with `BugseeCronet.instrument(engine)`. |
| Compose detected | Plugin auto-installs `bugsee-android-compose` + Kotlin compiler plugin (secure modifier + input capture). |
| Native / NDK code | Enable `ndk { enabled.set(true) }` — pulls `bugsee-android-ndk`. |
| Feedback UI wanted | `feedback.set(true)` — **not** auto-installed from other dependencies. |
| No Application subclass | Prefer manifest auto-launch (Phase 3 Option A). |

---

## Phase 2: Install

### Step 1 — Apply the Bugsee Gradle plugin (mandatory)

7.x requires the plugin. Without it, APM, main-thread misuse detection, log capture rewrites, OkHttp injection, and Compose secure redaction do not work. Pin plugin **4.0.5** (current Plugin Portal / Maven release). Plugin 4.x pairs with SDK 7.x; do not mix with plugin 3.x / SDK 6.x.

**Kotlin DSL (`app/build.gradle.kts`):**

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.bugsee.android.gradle") version "4.0.5"
}

bugsee {
    appToken("<your-app-token>")
    // In-app feedback messenger UI (default: off) — plugin pulls bugsee-android-feedback
    // feedback.set(true)
    // Native crash capture + NDK symbol upload (default: off) — plugin pulls bugsee-android-ndk
    // ndk {
    //     enabled.set(true)
    //     // forceDebugSymbolsUpload.set(true)    // re-upload even when the build UUID hasn't changed
    // }
    // Memory & thread leak detection (default: off) — plugin pulls bugsee-android-leak
    // leak { enabled.set(true) }
    // instrumentation {
    //     mainThreadMisuse.set(false)   // disable any individual hook if needed
    //     ktor.set(false)                // suppress auto-install of an extension
    // }
}
```

**Groovy (`app/build.gradle`):**

```groovy
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'com.bugsee.android.gradle' version '4.0.5'
}

bugsee {
    appToken '<your-app-token>'
    // ndk { enabled.set true }
}
```

Ensure plugin resolution in `settings.gradle[.kts]`:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
        google()
    }
}
```

With the plugin applied, the core `com.bugsee:bugsee-android` artifact is auto-pulled (bounded to the same MAJOR.MINOR series as the plugin's `sdk-min-version`). Dependency-driven extensions (OkHttp, Ktor 2/3, Cronet, Compose) are auto-installed when the matching library is in the graph. Feedback, NDK, and leak are **not** dependency-driven — enable them with the DSL toggles above.

### Step 2 — Pin the core SDK (recommended)

An explicit `com.bugsee:bugsee-android` declaration wins over auto-pull and locks the runtime. Pin **7.1.3**. Avoid `+`.

**Kotlin DSL:**

```kotlin
dependencies {
    implementation("com.bugsee:bugsee-android:7.1.3")
    // Only if you did not use feedback.set(true) / ndk { enabled.set(true) } and need a manual pin:
    // implementation("com.bugsee:bugsee-android-feedback:7.1.3")
    // implementation("com.bugsee:bugsee-android-ndk:7.1.3")
}
```

**Groovy:**

```groovy
dependencies {
    implementation 'com.bugsee:bugsee-android:7.1.3'
}
```

To suppress auto-install of a dependency-driven extension, set the corresponding flag in `bugsee { instrumentation { ... } }` (e.g. `cronet.set(false)`). To opt out of core auto-pull entirely, `sdkAutoLoad.set(false)`.

> Current stable releases (2026-08-25): SDK `com.bugsee:bugsee-android:7.1.3` and Gradle plugin `4.0.5`. The plugin tracks its own 4.x line, separate from the SDK.

---

## Phase 3: Initialize

Pick one path.

### Option A — Manifest auto-launch (recommended)

No Application subclass needed, no `Bugsee.launch(...)` call. Add the token as `<meta-data>` under `<application>`:

```xml
<application ...>
    <meta-data
        android:name="com.bugsee.app-token"
        android:value="@string/bugsee_app_token" />
</application>
```

The SDK launches automatically at process start via `BugseeInitProvider`. Any option can be set via manifest metadata using the `com.bugsee.option.<group>.<name>` key; enums take the value name as a string (e.g. `"High"` for `FrameRate.High`).

```xml
<meta-data android:name="com.bugsee.option.capture.breadcrumbs" android:value="true" />
<meta-data android:name="com.bugsee.option.config.duration"     android:value="120" />
<meta-data android:name="com.bugsee.option.capture.video.adaptive" android:value="true" />
```

### Option B — Programmatic launch

Use when the token is fetched at runtime or options need dynamic values. Context is auto-discovered. If the token is also in the manifest, disable auto-init with `com.bugsee.auto-init` = `false`.

**Kotlin:**

```kotlin
import com.bugsee.library.Bugsee
import com.bugsee.library.contracts.options.Options

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        val options = hashMapOf<String, java.io.Serializable>(
            Options.Duration to 60,
            Options.CaptureVideoAdaptive to true
        )
        Bugsee.launch(this, "<your-app-token>", options)
    }
}
```

Shortest form (context resolved automatically): `Bugsee.launch("<your-app-token>")`.

**Java:**

```java
import com.bugsee.library.Bugsee;
import com.bugsee.library.contracts.options.Options;

public class MyApplication extends Application {
    @Override public void onCreate() {
        super.onCreate();
        HashMap<String, Serializable> options = new HashMap<>();
        options.put(Options.Duration, 60);
        options.put(Options.CaptureVideoAdaptive, true);
        Bugsee.launch(this, "<your-app-token>", options);
    }
}
```

A `Map` passed to `launch(...)` is a **full override**, not a merge with manifest metadata: any key left out of the map falls back to its registered default, not to the manifest value. The 6.x `LaunchOptions` builder is removed.

---

## Phase 4: Configure (Optional)

All configuration flows through either (a) manifest `<meta-data>` entries, or (b) the `Map<String, Serializable>` passed to `Bugsee.launch(...)`. Keys live on `com.bugsee.library.contracts.options.Options`. Canonical table: [configuration](https://docs.bugsee.com/sdk/android/configuration/).

Common toggles (including 7.1.x setup-relevant options):

| `Options` constant | Manifest key | Default | Description |
|---|---|---|---|
| `Duration` | `com.bugsee.option.config.duration` | `60` | Video ring buffer duration (seconds). |
| `WifiOnlyUpload` | `com.bugsee.option.config.wifi-only-upload` | `false` | Restrict uploads to Wi-Fi. |
| `DetectAndReportHang` | `com.bugsee.option.detect.hang` | `false` | Main-thread hang detection. |
| `DetectAndReportMainThreadMisuse` | `com.bugsee.option.detect.main_thread_misuse` | `false` | Flag I/O / network / DB / `SharedPreferences` on main thread. Requires plugin `mainThreadMisuse` instrumentation. |
| `DetectAndReportExit` | `com.bugsee.option.detect.exit` | `true` | Master switch for `ApplicationExitInfo`-based exit reports (7.1 default). |
| `CaptureVideoFrameRate` | `com.bugsee.option.capture.video.frame-rate` | `High` | `Low` / `Medium` / `High`. |
| `CaptureVideoAdaptive` | `com.bugsee.option.capture.video.adaptive` | `false` | **7.1.0.** Skip capturing new frames while nothing on screen has been redrawn (at least one frame per second). Lowers CPU/battery on idle screens. |
| `CaptureNetworkOnLaunch` | `com.bugsee.option.capture.network.on-launch` | `false` | **7.1.0.** Subscribe the network provider during `launch()` instead of when capture starts, so startup requests are not missed. |
| `CaptureRespectFlagSecure` | `com.bugsee.option.capture.respect-flag-secure` | `true` | **7.1.0.** Honor `WindowManager.LayoutParams.FLAG_SECURE`: those windows are blanked in video and screenshots, excluded from the view hierarchy, and their input is dropped. An app that sets `FLAG_SECURE` broadly will see empty recordings after upgrading — set to `false` to restore pre-7.1.0 behavior. |
| `ReportingTriggerByShake` | `com.bugsee.option.reporting.triggers.shake` | `true` | Shake-to-report gesture. |
| `PerformanceMonitoring` | `com.bugsee.option.performance.enabled` | `true` | APM master switch. |
| `PerformanceSampleRate` | `com.bugsee.option.performance.sample-rate` | `0.01` | Standalone-upload probability per transaction (capture is unaffected). |

### NDK native crashes (`bugsee-android-ndk`)

Native crash detection is **not** in the core artifact. Enable it with the plugin DSL (`ndk { enabled.set(true) }`) or `implementation("com.bugsee:bugsee-android-ndk:7.1.3")`. The programmatic constant is `NdkOptions.DetectAndReport` (`com.bugsee.library.ndk.contracts.options.NdkOptions`), **not** the removed `Options.DetectAndReportCrashNdk`. Manifest key `com.bugsee.option.detect.crash-ndk` is unchanged; default is `true` once the module is on the classpath. See [native crashes](https://docs.bugsee.com/sdk/android/issue-detection/native-crashes/).

### WebSocket

OkHttp `newWebSocket(...)` traffic (connection lifecycle, frames, close/error) is captured automatically when the OkHttp extension is installed (Gradle plugin **4.0.3+**, included in 4.0.5). Ktor on the OkHttp engine is automatic; on CIO, route calls through the `bugseeWebSocket` helper. `NetworkEventStage.WebSocket` is the stage value for custom events.

### `FLAG_SECURE`

Windows that set `FLAG_SECURE` (including payment/DRM surfaces that set it for you) are protected by default since 7.1.0 via `CaptureRespectFlagSecure`. See [privacy / video](https://docs.bugsee.com/sdk/android/privacy/video/).

### `Bugsee.getStatus()`

```kotlin
import com.bugsee.library.contracts.lifecycle.BugseeStatus

when (Bugsee.getStatus()) {
    BugseeStatus.Launched -> { /* running */ }
    BugseeStatus.Launching, BugseeStatus.Stopping -> { /* in transition */ }
    BugseeStatus.Stopped -> { /* not running */ }
}
```

`Bugsee.getLaunched()` remains the boolean shortcut (`true` between a successful launch/auto-init and the next `stop()`). See [lifecycle](https://docs.bugsee.com/sdk/android/lifecycle/).

Network client wiring (required for non-OkHttp clients):

- **OkHttp 3/4** — transparent; plugin injects `BugseeOkHttpInterceptor` into every `OkHttpClient.Builder.build()`. No code changes.
- **Ktor 2** — `install(BugseeKtor2Plugin)` on every `HttpClient { ... }`.
- **Ktor 3** — `install(BugseeKtor3Plugin.Plugin)` on every `HttpClient { ... }`.
- **Cronet** — `val engine = BugseeCronet.instrument(CronetEngine.Builder(ctx).build())` for every engine.

Delete all 6.x manual wiring (`Bugsee.addNetworkLoggingToOkHttpBuilder(...)`, `addNetworkLoggingToKtorHttpClient(...)`, etc.) — those methods no longer exist and the code will not compile.

Full reference: [configuration](https://docs.bugsee.com/sdk/android/configuration/) · [network](https://docs.bugsee.com/sdk/android/network/) · [gradle-plugin](https://docs.bugsee.com/sdk/android/gradle-plugin/).

---

## Verification

Build and run. On launch, the Bugsee floating report button appears. Confirm end-to-end delivery:

```kotlin
// Programmatic trigger
Bugsee.showReportDialog()

// Or test crash capture (remove after verifying)
throw RuntimeException("Bugsee 7.x smoke test")

// Or logged exception
Bugsee.logException(IllegalStateException("test"))
```

Check the Bugsee dashboard for the incoming report. If APM is enabled, hit a few screens and HTTP endpoints, then inspect the Performance tab for `ui.load`, `ui.display`, `http.client`, `db.*`, and `file.*` spans.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/android/installation/)
- [Configuration](https://docs.bugsee.com/sdk/android/configuration/)
- [NDK options](https://docs.bugsee.com/sdk/android/configuration/ndk/)
- [Gradle plugin](https://docs.bugsee.com/sdk/android/gradle-plugin/)
- [Network events](https://docs.bugsee.com/sdk/android/network/)
- [Issue detection](https://docs.bugsee.com/sdk/android/issue-detection/)
- [Native crashes](https://docs.bugsee.com/sdk/android/issue-detection/native-crashes/)
- [Privacy / video (`FLAG_SECURE`)](https://docs.bugsee.com/sdk/android/privacy/video/)
- [Lifecycle / `getStatus`](https://docs.bugsee.com/sdk/android/lifecycle/)
- [Performance / APM](https://docs.bugsee.com/sdk/android/performance/)
- [Migration from 6.x](https://docs.bugsee.com/sdk/android/migration/)
