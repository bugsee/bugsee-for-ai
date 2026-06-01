---
name: bugsee-android-sdk
description: Full Bugsee SDK setup for Android. Use when asked to add Bugsee to Android, install bugsee-android, or set up bug reporting, crash reporting, and video recording for Android applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Android SDK

# Bugsee Android SDK

Opinionated wizard that scans your Android project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, and console logs.

## Invoke This Skill When

- User asks to "add Bugsee to Android" or "set up Bugsee" in an Android app
- User wants bug reporting, crash reporting, video recording, or network monitoring in Android
- User mentions `bugsee-android`, `com.bugsee:bugsee-android`, or Bugsee for Kotlin/Java Android

> **Note:** Always verify against [docs.bugsee.com/sdk/android/installation/](https://docs.bugsee.com/sdk/android/installation/) before implementing.

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
    implementation 'com.bugsee:bugsee-android:+'
}
```

**Kotlin DSL (`app/build.gradle.kts`):**

```kotlin
dependencies {
    implementation("com.bugsee:bugsee-android:+")
}
```

> The `+` fetches the latest version. Pin to a specific version from [release notes](https://docs.bugsee.com/sdk/android/release-notes/) for production stability.

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

Full options: [docs.bugsee.com/sdk/android/configuration/](https://docs.bugsee.com/sdk/android/configuration/)

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

- [Installation](https://docs.bugsee.com/sdk/android/installation/)
- [Configuration](https://docs.bugsee.com/sdk/android/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/android/custom/)
- [Network events](https://docs.bugsee.com/sdk/android/network/)
- [Console logs](https://docs.bugsee.com/sdk/android/logs/)
- [Privacy](https://docs.bugsee.com/sdk/android/privacy/overview/)
- [Manual invocation](https://docs.bugsee.com/sdk/android/manual/)
- [Release notes](https://docs.bugsee.com/sdk/android/release-notes/)
