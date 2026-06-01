---
name: bugsee-ios-sdk
description: Full Bugsee SDK setup for iOS. Use when asked to add Bugsee to iOS, install Bugsee via CocoaPods or SPM, or set up bug reporting, crash reporting, and video recording for iOS applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > iOS SDK

# Bugsee iOS SDK

Opinionated wizard that scans your iOS project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, and console logs.

## Invoke This Skill When

- User asks to "add Bugsee to iOS" or "set up Bugsee" in an iOS/iPadOS app
- User wants bug reporting, crash reporting, video recording, or network monitoring in iOS
- User mentions Bugsee CocoaPods, Bugsee SPM, or Bugsee for Swift/Objective-C

> **Note:** Always verify against [docs.bugsee.com/sdk/ios/installation/](https://docs.bugsee.com/sdk/ios/installation/) before implementing.

---

## Phase 1: Detect

Run these commands to understand the project:

```bash
# Detect Xcode project
ls *.xcodeproj *.xcworkspace 2>/dev/null

# Detect Swift vs Objective-C
find . -name "*.swift" -not -path "*/Pods/*" -not -path "*/.build/*" 2>/dev/null | head -5
find . -name "*.m" -not -path "*/Pods/*" 2>/dev/null | head -5

# Detect dependency manager
ls Podfile Package.swift Cartfile 2>/dev/null

# Check for existing Bugsee
grep -ri bugsee Podfile Package.swift Cartfile 2>/dev/null | head -5

# Find AppDelegate
find . -name "AppDelegate.swift" -o -name "AppDelegate.m" 2>/dev/null | head -3

# Detect SwiftUI App lifecycle (no AppDelegate)
grep -r "@main" --include="*.swift" 2>/dev/null | head -3
```

| Question | Impact |
|----------|--------|
| `Podfile` exists? | Use CocoaPods installation |
| `Package.swift` exists or SPM? | Use Swift Package Manager |
| `Cartfile` exists? | Use Carthage installation |
| No dependency manager? | Use manual framework installation |
| Swift files found? | Show Swift init code |
| Objective-C files found? | Show Objective-C init code |
| `AppDelegate` found? | Add `Bugsee.launch()` there |
| SwiftUI `@main` lifecycle? | See [SwiftUI docs](https://docs.bugsee.com/sdk/ios/swiftui/) |
| Already has Bugsee? | Skip install, check initialization |

---

## Phase 2: Install

Choose one method based on detected dependency manager:

### CocoaPods

Add to `Podfile`:

```ruby
pod 'Bugsee'
```

Then run:

```bash
pod install
```

### Swift Package Manager

Add the package in Xcode:
1. File > Add Package Dependencies
2. Enter URL: `https://github.com/bugsee/spm`
3. Select version rule and add to target

### Carthage

Add to `Cartfile`:

```
binary "https://docs.bugsee.com/sdk/ios/Bugsee.json"
```

Then run:

```bash
carthage update --use-xcframeworks
```

Drag `Bugsee.xcframework` from `Carthage/Build` into your target's "Frameworks, Libraries, and Embedded Content".

### Manual

1. Download from [https://download.bugsee.com/sdk/ios/dynamic/Bugsee-stable.xcframework.zip](https://download.bugsee.com/sdk/ios/dynamic/Bugsee-stable.xcframework.zip)
2. Extract and drag `Bugsee.xcframework` into your Xcode project
3. Ensure "Embed & Sign" is selected in target settings

---

## Phase 3: Initialize

Add Bugsee launch to your AppDelegate:

**Swift:**

```swift
import Bugsee

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // ...other initialization code

    Bugsee.launch(token: "<your_app_token>")

    return true
}
```

**Objective-C:**

```objectivec
@import Bugsee;

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // ...other initialization code

    [Bugsee launchWithToken:@"<your_app_token>"];

    return YES;
}
```

> Replace `<your_app_token>` with the token from your Bugsee dashboard.

> Since v6.0.0 the Bugsee iOS SDK supports the simulator; crash capture is excluded. For full functionality, use a real device.

---

## Phase 4: Configure (Optional)

Launch with options for customization:

**Swift:**

```swift
let options = BugseeOptions()
options.shakeToReport = true
options.reportPrioritySelector = true
options.maxRecordingTime = 60
Bugsee.launch(token: "<your_app_token>", options: options)
```

**Objective-C:**

```objectivec
BugseeOptions *options = [[BugseeOptions alloc] init];
options.shakeToReport = YES;
options.reportPrioritySelector = YES;
options.maxRecordingTime = 60;
[Bugsee launchWithToken:@"<your_app_token>" options:options];
```

Common options:

| Option | Default | Description |
|--------|---------|-------------|
| `videoEnabled` | `true` | Enable video recording |
| `crashReport` | `true` | Catch and report crashes |
| `monitorNetwork` | `true` | Capture network traffic |
| `captureLogs` | `true` | Capture console logs |
| `maxRecordingTime` | `60` | Max recording duration (seconds) |
| `shakeToReport` | `false` | Shake device to trigger report |
| `screenshotEnabled` | `true` | Attach screenshot to report |
| `wifiOnlyUpload` | `false` | Upload only on WiFi |

Full options: [docs.bugsee.com/sdk/ios/configuration/](https://docs.bugsee.com/sdk/ios/configuration/)

---

## Verification

After setup, build and run the app on a device. You should see a Bugsee floating button overlay. Tap it to file a test bug report.

Check the Bugsee dashboard for the incoming report.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/ios/installation/)
- [Configuration](https://docs.bugsee.com/sdk/ios/configuration/)
- [SwiftUI](https://docs.bugsee.com/sdk/ios/swiftui/)
- [Custom data](https://docs.bugsee.com/sdk/ios/custom/)
- [Console logs](https://docs.bugsee.com/sdk/ios/logs/)
- [Privacy](https://docs.bugsee.com/sdk/ios/privacy/overview/)
- [Crash symbolication](https://docs.bugsee.com/sdk/ios/symbolication/)
- [Manual invocation](https://docs.bugsee.com/sdk/ios/manual/)
- [Release notes](https://docs.bugsee.com/sdk/ios/release-notes/)
