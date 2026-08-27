---
name: bugsee-dotnet-sdk
description: Full Bugsee SDK setup for .NET MAUI. Use when asked to add Bugsee to .NET, MAUI, install Bugsee NuGet, or set up bug reporting, crash reporting, and video recording for .NET mobile applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > .NET SDK

# Bugsee .NET SDK

Opinionated wizard that guides you through complete Bugsee setup in .NET MAUI — bug reporting with video, crash reporting, network monitoring, and console logs for iOS and Android.

## Invoke This Skill When

- User asks to "add Bugsee to .NET" or "set up Bugsee" in a MAUI app
- User wants bug reporting, crash reporting, or video recording in .NET MAUI
- User mentions Bugsee NuGet, Bugsee MAUI, or `BugseePlugin` in .NET context

> **Note:** Bugsee .NET SDK supports .NET 6 and up on iOS, Android, and MAUI. Always verify against [docs.bugsee.com/sdk/dotnet/installation/](https://docs.bugsee.com/sdk/dotnet/installation/).

---

## Phase 1: Detect

```bash
# Detect .NET project
ls *.csproj *.sln 2>/dev/null

# Check target frameworks (MAUI, iOS, Android)
grep -E 'TargetFramework|UseMaui' *.csproj 2>/dev/null | head -5

# Check for existing Bugsee
grep -ri bugsee *.csproj 2>/dev/null | head -5

# Find platform entry points
find . -name "AppDelegate.cs" -o -name "MainApplication.cs" -o -name "MauiProgram.cs" 2>/dev/null | head -5
```

| Question | Impact |
|----------|--------|
| MAUI project? | Use MAUI-specific initialization |
| Pure iOS .NET? | Use iOS AppDelegate pattern |
| Pure Android .NET? | Use Android Application pattern |
| Already has Bugsee NuGet? | Skip install |

---

## Phase 2: Install

Install via NuGet:

```bash
dotnet add package Bugsee --version 6.0.3
```

Or in Visual Studio: right-click Dependencies > Manage NuGet Packages > search "Bugsee" > Install **6.0.3**.

Current NuGet latest (re-verified 2026-08-25): **Bugsee 6.0.3**. Pin it; `dotnet add package Bugsee` without `--version` is acceptable only after confirming that still resolves to 6.0.3.

---

## Phase 3: Initialize

### iOS (AppDelegate)

```csharp
using System.Collections.Generic;

namespace YourNameSpace
{
    [Register("AppDelegate")]
    public class AppDelegate : MauiUIApplicationDelegate
    {
        protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();

        public override bool FinishedLaunching(UIApplication application, NSDictionary launchOptions)
        {
            BugseePlugin.Bugsee.Launch("<your_app_token>");
            return base.FinishedLaunching(application, launchOptions);
        }
    }
}
```

### Android (MainApplication)

```csharp
using System.Collections.Generic;

namespace YourNameSpace
{
    [Application]
    public class MainApplication : MauiApplication
    {
        public MainApplication(IntPtr handle, JniHandleOwnership ownership)
            : base(handle, ownership) { }

        protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();

        public override void OnCreate()
        {
            base.OnCreate();
            BugseePlugin.Bugsee.Launch(this, "<your_app_token>");
        }
    }
}
```

> Replace `<your_app_token>` with the token from your Bugsee dashboard.

> iOS/iPadOS: Since v6.0.0 the Bugsee iOS SDK supports the simulator; crash capture is excluded. For full functionality, use a real device.

---

## Phase 4: Configure (Optional)

Use platform-specific options:

### iOS

```csharp
var options = new BugseePlugin.IOSLaunchOptions();
options.ShakeToReport = true;
options.ReportPrioritySelector = true;
BugseePlugin.Bugsee.Launch("<your_app_token>", options);
```

### Android

```csharp
var options = new BugseePlugin.AndroidLaunchOptions();
options.ShakeToTrigger = true;
options.VideoEnabled = true;
BugseePlugin.Bugsee.Launch(this, "<your_app_token>", options);
```

Full options: [docs.bugsee.com/sdk/dotnet/configuration/](https://docs.bugsee.com/sdk/dotnet/configuration/)

---

## Verification

Build and run on a device. The Bugsee floating button should appear. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/dotnet/installation/)
- [Configuration](https://docs.bugsee.com/sdk/dotnet/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/dotnet/custom/)
- [Console logs](https://docs.bugsee.com/sdk/dotnet/logs/)
- [Privacy](https://docs.bugsee.com/sdk/dotnet/privacy/overview/)
- [Network capture](https://docs.bugsee.com/sdk/dotnet/network/)
- [Crash symbolication](https://docs.bugsee.com/sdk/dotnet/symbolication/)
- [Manual invocation](https://docs.bugsee.com/sdk/dotnet/manual/)
- [Release notes](https://docs.bugsee.com/sdk/dotnet/release-notes/)
