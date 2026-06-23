---
name: bugsee-android-sdk-6x
description: Bugsee SDK setup for legacy Android 6.x apps. Use only when maintaining an app already on bugsee-android 6.x, or when the user explicitly asks for the 6.x line. For new apps and the current SDK, use bugsee-android-sdk (7.x).
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Android SDK 6.x (Legacy)

# Bugsee Android SDK (6.x, Legacy)

Opinionated wizard for the **6.x** line of the Bugsee Android SDK — bug reporting with video, crash reporting, network monitoring, and console logs, using the classic `Bugsee.launch(...)` API (no Gradle plugin).

> **Legacy.** 7.x is the current Android SDK and the default for new apps — use [`bugsee-android-sdk`](../bugsee-android-sdk/SKILL.md) instead. Use this 6.x skill **only** to maintain an app already pinned to `com.bugsee:bugsee-android` 6.x, or when the user explicitly asks for the 6.x line. 7.x is a different, plugin-based SDK with a new API; see the [migration guide](https://docs.bugsee.com/sdk/android/migration/) when upgrading.

## Invoke This Skill When

- The user is maintaining an existing app already on `com.bugsee:bugsee-android` 6.x
- The user explicitly asks for Bugsee Android "6.x" / the "legacy" / "old" SDK
- The user's project pins a 6.x version (`com.bugsee:bugsee-android:6.x.y`) and they want to keep it

For a fresh "add Bugsee to Android" with no 6.x signal, use the current [`bugsee-android-sdk`](../bugsee-android-sdk/SKILL.md) (7.x) skill instead.

> **Note:** Always verify against [docs.bugsee.com/sdk/android/v6/installation/](https://docs.bugsee.com/sdk/android/v6/installation/) before implementing. 6.x docs live under `/sdk/android/v6/`.

---

## Phase 1: Detect

Run these commands to understand the project before making any changes:

```bash
# Detect project structure and build system
ls build.gradle build.gradle.kts settings.gradle settings.gradle.kts 2>/dev/null

# Check app-level build file (Groovy vs KTS)
ls app/build.gradle app/build.gradle.kts 2>/dev/null

# Detect Kotlin vs Java
find app/src/main -name "*.kt" 2>/dev/null | head -3
find app/src/main -name "*.java" 2>/dev/null | head -3

# Check minSdk, targetSdk, compileSdk
grep -E 'minSdk|targetSdk|compileSdk|minSdkVersion|targetSdkVersion|compileSdkVersion' app/build.gradle app/build.gradle.kts 2>/dev/null | head -6

# Check for existing Bugsee
grep -ri bugsee app/build.gradle app/build.gradle.kts 2>/dev/null | head -5

# Find existing Application subclass
grep -r "extends Application\|: Application()" app/src/main --include="*.java" --include="*.kt" 2>/dev/null | head -3

# Check AndroidManifest for android:name on <application>
grep -E 'android:name=' app/src/main/AndroidManifest.xml 2>/dev/null | head -3
```

Use the results to answer:

| Question | Impact |
|----------|--------|
| `build.gradle.kts` present? | Use Kotlin DSL syntax |
| `build.gradle` (Groovy) present? | Use Groovy syntax |
| Kotlin files found? | Show Kotlin init code first |
| Java files found? | Show Java init code first |
| Existing `Application` subclass? | Add `Bugsee.launch()` there |
| No `Application` subclass? | Create one, register in manifest |
| Already has Bugsee dependency? | Skip install, check initialization |

---

## Phase 2: Install

Add the Bugsee dependency to the app module's build file.

**Groovy (`app/build.gradle`):**

```gradle
dependencies {
    implementation 'com.bugsee:bugsee-android:6.0.4'
}
```

**Kotlin DSL (`app/build.gradle.kts`):**

```kotlin
dependencies {
    implementation("com.bugsee:bugsee-android:6.0.4")
}
```

> `6.0.4` is the latest 6.x release — pin to it (or another 6.x version from the [6.x release notes](https://docs.bugsee.com/sdk/android/v6/release-notes/)). **Do not** use `+`: that resolves to 7.x, which is the plugin-based SDK with a different API — switch to the [`bugsee-android-sdk`](../bugsee-android-sdk/SKILL.md) (7.x) skill for that.

If your `compileSdkVersion` is below 29 and you get `android:foregroundServiceType not found`, set `compileSdkVersion` to 29 or higher.

---

## Phase 3: Initialize

### Step 1 — Ensure an Application subclass exists

If no `Application` subclass exists, create one:

**Kotlin:**

```kotlin
import android.app.Application
import com.bugsee.library.Bugsee

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Bugsee.launch(this, "<your_app_token>")
    }
}
```

**Java:**

```java
import android.app.Application;
import com.bugsee.library.Bugsee;

public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        Bugsee.launch(this, "<your_app_token>");
    }
}
```

### Step 2 — Register in AndroidManifest.xml

If you created a new `Application` subclass, add `android:name` to the `<application>` tag:

```xml
<application
    android:name=".MyApplication">
    <!--...-->
</application>
```

> Replace `<your_app_token>` with the token from your Bugsee dashboard.

---

## Phase 4: Configure (Optional)

Launch with options for customization:

**Kotlin:**

```kotlin
val options = hashMapOf<String, Any>(
    Bugsee.Option.MaxRecordingTime to 60,
    Bugsee.Option.ShakeToTrigger to false,
    Bugsee.Option.VideoEnabled to true
)
Bugsee.launch(this, "<your_app_token>", options)
```

**Java:**

```java
HashMap<String, Object> options = new HashMap<>();
options.put(Bugsee.Option.MaxRecordingTime, 60);
options.put(Bugsee.Option.ShakeToTrigger, false);
options.put(Bugsee.Option.VideoEnabled, true);
Bugsee.launch(this, "<your_app_token>", options);
```

Common options:

| Option | Default | Description |
|--------|---------|-------------|
| `VideoEnabled` | `true` | Enable video recording |
| `CrashReport` | `true` | Catch and report crashes |
| `MonitorNetwork` | `true` | Capture network traffic |
| `CaptureLogs` | `true` | Capture console logs |
| `MaxRecordingTime` | `60` | Max recording duration (seconds) |
| `ShakeToTrigger` | `false` | Shake device to trigger report |
| `ScreenshotEnabled` | `true` | Attach screenshot to report |
| `WifiOnlyUpload` | `false` | Upload only on WiFi |

Full options: [docs.bugsee.com/sdk/android/v6/configuration/](https://docs.bugsee.com/sdk/android/v6/configuration/)

---

## Verification

After setup, build and run the app. You should see a Bugsee floating button overlay. Tap it to file a test bug report.

To verify programmatically, add a test exception after initialization:

**Kotlin:**

```kotlin
Bugsee.launch(this, "<your_app_token>")
// Test: remove after verifying
throw RuntimeException("Bugsee test crash")
```

Check the Bugsee dashboard for the incoming report.

---

## Documentation Links

- [Installation (6.x)](https://docs.bugsee.com/sdk/android/v6/installation/)
- [Configuration (6.x)](https://docs.bugsee.com/sdk/android/v6/configuration/)
- [Custom data (6.x)](https://docs.bugsee.com/sdk/android/v6/custom/)
- [Network events (6.x)](https://docs.bugsee.com/sdk/android/v6/network/)
- [Console logs (6.x)](https://docs.bugsee.com/sdk/android/v6/logs/)
- [Privacy (6.x)](https://docs.bugsee.com/sdk/android/v6/privacy/overview/)
- [Manual invocation (6.x)](https://docs.bugsee.com/sdk/android/v6/manual/)
- [Release notes (6.x)](https://docs.bugsee.com/sdk/android/v6/release-notes/)
- [Migrate to 7.x](https://docs.bugsee.com/sdk/android/migration/)
