---
name: bugsee-android-sdk-7x
description: Full Bugsee 7.x (beta) setup for Android. Use when the user explicitly asks for Bugsee Android 7.x / beta, or wants the next-gen features (APM, extension modules, auto-init, new detection providers). For stable production setup, use the 6.x skill instead.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Android SDK 7.x (Beta)

# Bugsee Android SDK 7.x (Beta)

Opinionated wizard that scans the Android project and wires up Bugsee 7.x — core SDK, Gradle plugin, extension modules for network clients / Compose / feedback, APM, and manifest-based auto-launch.

> **Beta warning.** 7.x is in active beta. Option keys, module names, and internal APIs may still change. For production apps shipping today, prefer the stable [6.x skill](../bugsee-android-sdk/SKILL.md). Continue only if the user explicitly asks for 7.x / beta, or needs APM, Ktor/Cronet capture, manifest auto-launch, the Gradle plugin, or individual `bugsee-android-*` artifacts.

## Invoke This Skill When

- User explicitly says "Bugsee 7.x", "beta", "next-gen Bugsee", or `7.0.0-beta*`.
- User mentions **APM**, transactions, or spans (`Bugsee.startTransaction`, `startSpan`).
- User mentions **extension modules** (`bugsee-android-okhttp`, `bugsee-android-ktor-2`, `bugsee-android-ktor-3`, `bugsee-android-cronet`, `bugsee-android-compose`, `bugsee-android-feedback`).
- User asks for **manifest auto-launch** / no Application subclass.
- User mentions the **Bugsee Gradle plugin** (`com.bugsee.android.gradle`).
- User asks about **Compose secure modifier**, `Modifier.bugseeSecure()`, Ktor / Cronet / `HttpEngine` integration.
- User mentions new detection providers: `DetectAndReportMainThreadMisuse`, `DetectAndReportExit*`, `DetectAndReportEarlyCrash`.

For generic "add Bugsee to Android" / "set up bug reporting" with no 7.x signal, switch to the 6.x skill.

> **Always verify against** [docs.bugsee.com/sdk/android/v7/installation/](https://docs.bugsee.com/sdk/android/v7/installation/) before implementing. 7.x docs live under `/sdk/android/v7/`.

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
| OkHttp 3/4 detected | Plugin auto-installs `bugsee-android-okhttp`; no manual wiring. |
| OkHttp 2 only | Not supported in 7.x — user must migrate to OkHttp 3/4 or drop to 6.x. |
| Ktor 2.x / 3.x detected | Plugin auto-installs extension, but user must `install(BugseeKtor2Plugin)` / `install(BugseeKtor3Plugin.Plugin)` on every `HttpClient`. |
| Cronet detected | Plugin auto-installs extension, but user must wrap every engine with `BugseeCronet.instrument(engine)`. |
| Compose detected | Plugin auto-installs `bugsee-android-compose` + Kotlin compiler plugin (secure modifier + input capture). |
| Feedback UI wanted | Manually add `bugsee-android-feedback` — **not** auto-installed. |
| No Application subclass | Prefer manifest auto-launch (Phase 3 Option A). |

---

## Phase 2: Install

### Step 1 — Apply the Bugsee Gradle plugin (mandatory)

7.x requires the plugin. Without it, APM, main-thread misuse detection, log capture rewrites, OkHttp injection, and Compose secure redaction do not work.

**Kotlin DSL (`app/build.gradle.kts`):**

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.bugsee.android.gradle") version "4.0.0-beta12"
}

bugsee {
    appToken("<your-app-token>")
    // ndk {
    //     enabled.set(true)                       // upload NDK symbols for native crash symbolication
    //     // forceDebugSymbolsUpload.set(true)    // re-upload even when the build UUID hasn't changed
    // }
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
    id 'com.bugsee.android.gradle' version '4.0.0-beta12'
}

bugsee {
    appToken '<your-app-token>'
    // ndk {
    //     enabled.set true
    // }
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

### Step 2 — Add the core artifact

**Kotlin DSL:**

```kotlin
dependencies {
    implementation("com.bugsee:bugsee-android:7.0.0-beta12")
    // Feedback UI is NOT auto-installed. Add it if the user wants the feedback flow:
    // implementation("com.bugsee:bugsee-android-feedback:7.0.0-beta12")
}
```

**Groovy:**

```groovy
dependencies {
    implementation 'com.bugsee:bugsee-android:7.0.0-beta12'
    // implementation 'com.bugsee:bugsee-android-feedback:7.0.0-beta12'
}
```

The plugin auto-adds `bugsee-android-okhttp`, `bugsee-android-ktor-2`, `bugsee-android-ktor-3`, `bugsee-android-cronet`, and `bugsee-android-compose` when it detects the matching dependency in the graph. To suppress any of these, set the corresponding flag in the `bugsee { instrumentation { ... } }` block (e.g. `cronet.set(false)`).

> At the time of writing, the latest betas are the SDK `com.bugsee:bugsee-android:7.0.0-beta12` and the Gradle plugin `com.bugsee.android.gradle` version `4.0.0-beta12` (the plugin tracks its own 4.x line, separate from the SDK). Pin to concrete versions; avoid `+`.

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
<meta-data android:name="com.bugsee.option.detect.crash-ndk"   android:value="true" />
<meta-data android:name="com.bugsee.option.capture.breadcrumbs" android:value="true" />
<meta-data android:name="com.bugsee.option.config.duration"     android:value="120" />
```

### Option B — Programmatic launch

Use when the token is fetched at runtime or options need dynamic values. Context is auto-discovered.

**Kotlin:**

```kotlin
import com.bugsee.library.Bugsee
import com.bugsee.library.contracts.options.Options

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        val options = hashMapOf<String, java.io.Serializable>(
            Options.Duration to 60,
            Options.DetectAndReportCrashNdk to true
        )
        Bugsee.launch(this, "<your-app-token>", options)
    }
}
```

**Java:**

```java
import com.bugsee.library.Bugsee;
import com.bugsee.library.contracts.options.Options;

public class MyApplication extends Application {
    @Override public void onCreate() {
        super.onCreate();
        HashMap<String, Serializable> options = new HashMap<>();
        options.put(Options.Duration, 60);
        options.put(Options.DetectAndReportCrashNdk, true);
        Bugsee.launch(this, "<your-app-token>", options);
    }
}
```

Options passed to `launch(...)` override manifest metadata. The 6.x `LaunchOptions` builder is removed. There is no shortest-form `Bugsee.launch(token)` — pass context (or use manifest auto-launch).

---

## Phase 4: Configure (Optional)

All configuration flows through either (a) manifest `<meta-data>` entries, or (b) the `Map<String, Serializable>` passed to `Bugsee.launch(...)`. Keys live on `com.bugsee.library.contracts.options.Options`.

Common toggles:

| `Options` constant | Manifest key | Default | Description |
|---|---|---|---|
| `Duration` | `com.bugsee.option.config.duration` | `60` | Video ring buffer duration (seconds). |
| `WifiOnlyUpload` | `com.bugsee.option.config.wifi-only-upload` | `false` | Restrict uploads to Wi-Fi. |
| `DetectAndReportCrashNdk` | `com.bugsee.option.detect.crash-ndk` | `false` | Enable Breakpad native crash capture. |
| `DetectAndReportHang` | `com.bugsee.option.detect.hang` | `false` | Main-thread hang detection. |
| `DetectAndReportMainThreadMisuse` | `com.bugsee.option.detect.main_thread_misuse` | `false` | Flag I/O / network / DB / `SharedPreferences` on main thread. Requires plugin `mainThreadMisuse` instrumentation. |
| `DetectAndReportExit` | `com.bugsee.option.detect.exit` | `false` | Master switch for `ApplicationExitInfo`-based exit reports. |
| `CaptureVideoFrameRate` | `com.bugsee.option.capture.video.frame-rate` | `High` | `Low` / `Medium` / `High`. |
| `ReportingTriggerByShake` | `com.bugsee.option.reporting.triggers.shake` | `true` | Shake-to-report gesture. |
| `PerformanceMonitoring` | `com.bugsee.option.performance.enabled` | `true` | APM master switch. |
| `PerformanceSampleRate` | `com.bugsee.option.performance.sample-rate` | `0.01` | Standalone-upload probability per transaction (capture is unaffected). |

Network client wiring (required for non-OkHttp clients):

- **OkHttp 3/4** — transparent; plugin injects `BugseeOkHttpInterceptor` into every `OkHttpClient.Builder.build()`. No code changes.
- **Ktor 2** — `install(BugseeKtor2Plugin)` on every `HttpClient { ... }`.
- **Ktor 3** — `install(BugseeKtor3Plugin.Plugin)` on every `HttpClient { ... }`.
- **Cronet** — `val engine = BugseeCronet.instrument(CronetEngine.Builder(ctx).build())` for every engine.

Delete all 6.x manual wiring (`Bugsee.addNetworkLoggingToOkHttpBuilder(...)`, `addNetworkLoggingToKtorHttpClient(...)`, etc.) — those methods no longer exist and the code will not compile.

Full reference: [configuration](https://docs.bugsee.com/sdk/android/v7/configuration/) · [network](https://docs.bugsee.com/sdk/android/v7/network/) · [gradle-plugin](https://docs.bugsee.com/sdk/android/v7/gradle-plugin/).

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

- [Installation (7.x)](https://docs.bugsee.com/sdk/android/v7/installation/)
- [Configuration (7.x)](https://docs.bugsee.com/sdk/android/v7/configuration/)
- [Gradle plugin (7.x)](https://docs.bugsee.com/sdk/android/v7/gradle-plugin/)
- [Network events (7.x)](https://docs.bugsee.com/sdk/android/v7/network/)
- [Detection (7.x)](https://docs.bugsee.com/sdk/android/v7/detection/)
- [Performance / APM (7.x)](https://docs.bugsee.com/sdk/android/v7/performance/)
- [Migration from 6.x](https://docs.bugsee.com/sdk/android/v7/migration/)
