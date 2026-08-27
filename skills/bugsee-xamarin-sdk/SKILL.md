---
name: bugsee-xamarin-sdk
description: Full Bugsee SDK setup for Xamarin. Use when asked to add Bugsee to Xamarin, install Bugsee NuGet for Xamarin, or set up bug reporting, crash reporting, and video recording for Xamarin applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Xamarin SDK

# Bugsee Xamarin SDK

Opinionated wizard that guides you through complete Bugsee setup in Xamarin — bug reporting with video, crash reporting, network monitoring, and console logs for iOS, Android, and Xamarin.Forms.

## Invoke This Skill When

- User asks to "add Bugsee to Xamarin" or "set up Bugsee" in a Xamarin app
- User wants bug reporting, crash reporting, or video recording in Xamarin
- User mentions Bugsee NuGet for Xamarin, Bugsee Xamarin.Forms, or `BugseePlugin` in Xamarin context

> **Note:** Always verify against [docs.bugsee.com/sdk/xamarin/installation/](https://docs.bugsee.com/sdk/xamarin/installation/).

---

## Phase 1: Detect

```bash
# Detect Xamarin project
ls *.csproj *.sln 2>/dev/null

# Check for Xamarin references
grep -E 'Xamarin|MonoAndroid|Xamarin.iOS' *.csproj 2>/dev/null | head -5

# Check for Xamarin.Forms
grep -i "Xamarin.Forms" *.csproj 2>/dev/null | head -3

# Check for existing Bugsee
grep -ri bugsee *.csproj 2>/dev/null | head -5

# Find entry points
find . -name "AppDelegate.cs" -o -name "MainApplication.cs" -o -name "App.xaml.cs" 2>/dev/null | head -5
```

| Question | Impact |
|----------|--------|
| Xamarin.Forms project? | Add Bugsee in platform-specific projects |
| Xamarin.iOS only? | Use iOS AppDelegate pattern |
| Xamarin.Android only? | Use Android Application pattern |
| Already has Bugsee? | Skip install |

---

## Phase 2: Install

Install via NuGet in each platform project:

In Visual Studio: right-click project > Add > Add NuGet Packages > search "Bugsee" > Add Package.

Or via CLI:

```bash
nuget install Bugsee -Version 6.0.3
```

Current NuGet latest (re-verified 2026-08-25): **Bugsee 6.0.3** (same package as .NET / MAUI).

> Install the Bugsee package in each platform-specific project (iOS and Android), not the shared/PCL project.

---

## Phase 3: Initialize

### iOS (AppDelegate.cs)

```csharp
using BugseePlugin;

[Register("AppDelegate")]
public partial class AppDelegate : UIApplicationDelegate
{
    public override bool FinishedLaunching(UIApplication application, NSDictionary launchOptions)
    {
        Bugsee.Launch("<your_app_token>");
        return base.FinishedLaunching(application, launchOptions);
    }
}
```

### Android (Application subclass)

```csharp
using BugseePlugin;

public class MyApplication : Application
{
    protected override void OnCreate()
    {
        base.OnCreate();
        Bugsee.Launch(this, "<your_app_token>");
    }
}
```

### Xamarin.Forms

Initialize in each platform project as shown above. Then interact with Bugsee from shared code by adding the NuGet to the Forms project too.

> Replace `<your_app_token>` with the token from your Bugsee dashboard.

---

## Phase 4: Configure (Optional)

Use platform-specific options:

### iOS

```csharp
var options = new IOSLaunchOptions();
options.ShakeToReport = true;
options.ReportPrioritySelector = true;
Bugsee.Launch("<your_app_token>", options);
```

### Android

```csharp
var options = new AndroidLaunchOptions();
options.ShakeToTrigger = true;
options.VideoEnabled = true;
Bugsee.Launch(this, "<your_app_token>", options);
```

Full options: [docs.bugsee.com/sdk/xamarin/configuration/](https://docs.bugsee.com/sdk/xamarin/configuration/)

---

## Verification

Build and run on a device. The Bugsee floating button should appear. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/xamarin/installation/)
- [Configuration](https://docs.bugsee.com/sdk/xamarin/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/xamarin/custom/)
- [Console logs](https://docs.bugsee.com/sdk/xamarin/logs/)
- [Privacy](https://docs.bugsee.com/sdk/xamarin/privacy/overview/)
- [Network capture](https://docs.bugsee.com/sdk/xamarin/network/)
- [Crash symbolication](https://docs.bugsee.com/sdk/xamarin/symbolication/)
- [Manual invocation](https://docs.bugsee.com/sdk/xamarin/manual/)
- [Release notes](https://docs.bugsee.com/sdk/xamarin/release-notes/)
