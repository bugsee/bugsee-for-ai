---
name: bugsee-unity-sdk
description: Full Bugsee SDK setup for Unity. Use when asked to add Bugsee to Unity, install Bugsee Unity package, or set up bug reporting, crash reporting, and video recording for Unity games.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Unity SDK

# Bugsee Unity SDK

Opinionated wizard that guides you through complete Bugsee setup in Unity — bug reporting with video, crash reporting, and console logs for iOS and Android games.

## Invoke This Skill When

- User asks to "add Bugsee to Unity" or "set up Bugsee" in a Unity project
- User wants bug reporting, crash reporting, or video recording in a Unity game
- User mentions Bugsee Unity, BugseeLauncher, or Bugsee for C# Unity

> **Note:** Always verify against [docs.bugsee.com/sdk/unity/installation/](https://docs.bugsee.com/sdk/unity/installation/) before implementing.

---

## Phase 1: Detect

```bash
# Confirm Unity project
ls Assets ProjectSettings 2>/dev/null

# Check for existing Bugsee
find Assets -name "Bugsee*" -o -name "bugsee*" 2>/dev/null | head -5

# Check existing plugins
ls Assets/Plugins/Bugsee 2>/dev/null

# Check platform settings
grep -r "AndroidManifest" Assets/Plugins 2>/dev/null | head -3
```

| Question | Impact |
|----------|--------|
| Unity project structure found? | Confirm Unity project |
| `Assets/Plugins/Bugsee` exists? | Old version installed, remove first |
| Building for Android? | May need foreground service permissions |

---

## Phase 2: Install

1. Download the latest [Bugsee.unitypackage](https://download.bugsee.com/sdk/unity/BugseeUnity-stable.unitypackage) (or see [all versions](https://docs.bugsee.com/sdk/unity/versions/)). Source repo: [github.com/bugsee/bugsee-unity](https://github.com/bugsee/bugsee-unity).

2. Import the package in Unity: Assets > Import Package > Custom Package

3. If upgrading, remove the `Assets/Plugins/Bugsee` folder first

---

## Phase 3: Initialize

### Option A: Scene-based setup (recommended)

1. Create an empty GameObject in your scene
2. Add the `BugseeLauncher` script component to it
3. Enter your Android and/or iOS app tokens in the inspector fields

### Option B: Code-based setup

If launching from code (e.g., on a user action), create a dedicated GameObject first:

```csharp
using BugseePlugin;

// Create or reference a persistent GameObject
GameObject bugseeObject = new GameObject("bgs_gameObject");
bugseeObject.AddComponent<Bugsee>();

// Launch Bugsee
Bugsee.Launch("<your_app_token>");
```

> The `bgs_gameObject` is required for communication between managed code and the native SDK. Do not remove or rename it.

### Android permissions

For Android 9+ with `VideoMode.V2`, add to `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
```

---

## Phase 4: Configure (Optional)

Create a partial class for custom options:

```csharp
namespace BugseePlugin
{
    public partial class BugseeLauncher
    {
        static BugseeLauncher()
        {
            AndroidOptionsHandler = GetAndroidOptions;
            IosOptionsHandler = GetIosOptions;
        }

        private static AndroidLaunchOptions GetAndroidOptions()
        {
            return new AndroidLaunchOptions()
            {
                VideoEnabled = true,
                MonitorNetwork = true
            };
        }

        private static IOSLaunchOptions GetIosOptions()
        {
            return new IOSLaunchOptions()
            {
                VideoEnabled = true,
                MonitorNetwork = true
            };
        }
    }
}
```

Full options: [docs.bugsee.com/sdk/unity/configuration/](https://docs.bugsee.com/sdk/unity/configuration/)

---

## Verification

Build and run on a device. The Bugsee floating button should appear. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/unity/installation/)
- [Configuration](https://docs.bugsee.com/sdk/unity/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/unity/custom/)
- [Console logs](https://docs.bugsee.com/sdk/unity/logs/)
- [Privacy](https://docs.bugsee.com/sdk/unity/privacy/overview/)
- [Crash reports](https://docs.bugsee.com/sdk/unity/crashes/)
- [Manual invocation](https://docs.bugsee.com/sdk/unity/manual/)
- [Release notes](https://docs.bugsee.com/sdk/unity/release-notes/)
