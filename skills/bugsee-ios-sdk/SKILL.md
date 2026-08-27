---
name: bugsee-ios-sdk
description: Full Bugsee SDK setup for iOS. Use when asked to add Bugsee to iOS, install Bugsee via SPM, CocoaPods, or Carthage, or set up bug reporting, crash reporting, video recording, APM, or build size analysis for iOS applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > iOS SDK

# Bugsee iOS SDK

Opinionated wizard that scans your iOS project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, console logs, APM, and optional build size analysis.

Current stable pin (re-verified 2026-08-25 against the package registries; docs.bugsee.com release notes currently stop at 6.3.0):

- **SPM** tag `6.3.2` at `https://github.com/bugsee/spm`
- **CocoaPods** trunk pod `Bugsee` `6.3.2`
- **Carthage** binary catalog also lists `6.3.2`

> **Note:** Always verify against [docs.bugsee.com/sdk/ios/installation/](https://docs.bugsee.com/sdk/ios/installation/) before implementing. Prefer registry tags over the release-notes page when they disagree.

---

## Invoke This Skill When

- User asks to "add Bugsee to iOS" or "set up Bugsee" in an iOS/iPadOS app
- User wants bug reporting, crash reporting, video recording, network monitoring, APM, or build size analysis in iOS
- User mentions Bugsee CocoaPods, Bugsee SPM, Bugsee Carthage, or Bugsee for Swift/Objective-C

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
| `Package.swift` exists or SPM? | Prefer Swift Package Manager (recommended going forward) |
| `Cartfile` exists? | Use Carthage installation |
| `Podfile` exists? | CocoaPods still works for existing projects; see the CocoaPods note below |
| No dependency manager? | Use manual framework installation |
| Swift files found? | Show Swift init code |
| Objective-C files found? | Show Objective-C init code |
| `AppDelegate` found? | Add `Bugsee.launch()` there |
| SwiftUI `@main` lifecycle? | See [SwiftUI docs](https://docs.bugsee.com/sdk/ios/swiftui/) |
| Already has Bugsee? | Skip install, check initialization |

---

## Phase 2: Install

Choose one method based on the detected dependency manager. **Prefer SPM or Carthage for new installs.** CocoaPods trunk publishing is ending in August 2026; keep the CocoaPods path for existing `Podfile` projects (6.3.2 is already on trunk).

### Swift Package Manager (recommended)

Add the package in Xcode:
1. File > Add Package Dependencies
2. Enter URL: `https://github.com/bugsee/spm`
3. Select version **6.3.2** (latest tag) and add it to the app target

### Carthage

Add to `Cartfile`:

```
binary "https://download.bugsee.com/sdk/ios/dynamic/Bugsee.json"
```

Then run:

```bash
carthage update --use-xcframeworks
```

Drag `Bugsee.xcframework` from `Carthage/Build` into your target's "Frameworks, Libraries, and Embedded Content".

### CocoaPods (existing Podfile projects)

CocoaPods trunk is shutting down publishing in August 2026. Prefer SPM or Carthage for new integrations. If the project already uses CocoaPods, add to `Podfile`:

```ruby
pod 'Bugsee', '6.3.2'
```

Then run:

```bash
pod install
pod update Bugsee   # install alone does not guarantee the latest trunk version
```

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
options.captureVideoAdaptive = true          // 6.3.0 — lower FPS on idle UI
options.performanceMonitoring = true         // 6.2.0 — APM (default YES)
options.sanitizeNetworkData = true           // 6.1.3 — PII redaction (default YES)
Bugsee.launch(token: "<your_app_token>", options: options)
```

**Objective-C:**

```objectivec
BugseeOptions *options = [[BugseeOptions alloc] init];
options.shakeToReport = YES;
options.reportPrioritySelector = YES;
options.maxRecordingTime = 60;
options.captureVideoAdaptive = YES;
options.performanceMonitoring = YES;
options.sanitizeNetworkData = YES;
[Bugsee launchWithToken:@"<your_app_token>" options:options];
```

Dictionary-key equivalents (same names as on the [configuration](https://docs.bugsee.com/sdk/ios/configuration/) page): `BugseeCaptureVideoAdaptiveKey` (`"CaptureVideoAdaptive"`), `BugseePerformanceMonitoringKey` (`performanceMonitoring`), `BugseeSanitizeNetworkDataKey` (`sanitizeNetworkData`). The configuration page may lag the 6.3.2 headers; the keys above are in the current SDK.

Common options:

| Option | Default | Description |
|--------|---------|-------------|
| `videoEnabled` | `true` | Enable video recording |
| `crashReport` | `true` | Catch and report crashes |
| `monitorNetwork` | `true` | Capture network traffic |
| `sanitizeNetworkData` | `true` | Auto-redact known PII keys from captured network events (6.1.3). Applies only when no custom network filter is set. |
| `captureLogs` | `true` | Capture console logs |
| `maxRecordingTime` | `60` | Max recording duration (seconds) |
| `shakeToReport` | `false` | Shake device to trigger report |
| `screenshotEnabled` | `true` | Attach screenshot to report |
| `wifiOnlyUpload` | `false` | Upload only on WiFi |
| `captureVideoAdaptive` | `false` | Opt-in adaptive screen capture (6.3.0): ~1 fps on a static screen, ramps to `maxFramerate` on visual activity |
| `performanceMonitoring` | `true` | APM master switch (6.2.0). When `false`, `startTransaction` / `startSpan` return a shared no-op |

Full options: [docs.bugsee.com/sdk/ios/configuration/](https://docs.bugsee.com/sdk/ios/configuration/)

### APM (`startTransaction` / `startSpan`) — 6.2.0+

Auto-instruments NSURLSession HTTP, NSData file I/O, NSFileManager operations, UIViewController lifecycle, and app startup. Transactions are attached to every report. SwiftUI auto-instrumentation is not in 6.3.2 yet.

**Swift:**

```swift
let tx = Bugsee.startTransaction(name: "Checkout", operation: "ui.checkout")
let span = Bugsee.startSpan(operation: "db.query", description: "SELECT cart")
// ... work ...
span.finish()
tx.finish()   // no-arg finish implies SpanStatus.ok
```

**Objective-C:**

```objectivec
id<BGSTransaction> tx = [Bugsee startTransactionWithName:@"Checkout" operation:@"ui.checkout"];
id<BGSSpan> span = [Bugsee startSpanWithOperation:@"db.query" description:@"SELECT cart"];
[span finish];
[tx finishWithStatus:BGSSpanStatusOK];
```

When `performanceMonitoring` is `NO`, both APIs return a shared no-op singleton — no null checks needed. `startSpan` parents to the current active span on the calling thread (pthread TLS; it does not auto-propagate across `dispatch_async`).

### BugseeAgent size analysis — 6.1.5+

The same `BugseeAgent` post-action that uploads dSYMs also registers a build record (version, configuration, VCS context, timings, Mach-O `LC_UUID`). Size analysis is opt-in.

1. Add a Run Script **Archive** post-action (Product → Scheme → Edit Scheme → Archive → Post-actions). Set **Provide build settings from** to the app target.
2. Script body (CocoaPods / manual — locates `BugseeAgent` in the project):

   ```bash
   SCRIPT_SRC=$(find "$PROJECT_DIR" -name 'BugseeAgent' | head -1)
   if [ ! "$SCRIPT_SRC" ]; then
       echo "Error: BugseeAgent not found"
       exit 1
   fi
   python3 "$SCRIPT_SRC" <APP_TOKEN> >> /tmp/BugseeAgent.txt
   ```

   For **SPM**, replace the `find` line with:

   ```bash
   SCRIPT_SRC="$BUILD_DIR/../SourcePackages/checkouts/spm/Tools/BugseeAgent"
   ```

3. `BUGSEE_BUILD_INFO_ENABLED` is **ON by default** — do not set it. To also upload the IPA for download/install size, per-category breakdown, and per-file diffs, add scheme Environment Variables:

   ```text
   BUGSEE_SIZE_ANALYSIS_ENABLED=1
   # optional:
   # BUGSEE_CHUNKED_UPLOAD=1
   # BUGSEE_SIZE_CHECK_ENABLED=1
   # BUGSEE_SIZE_CHECK_WARNING_PCT=5.0
   # BUGSEE_SIZE_CHECK_FAIL_PCT=10.0
   ```

Full env-var reference: [build size analysis](https://docs.bugsee.com/sdk/ios/build-size-analysis/) · [symbolication](https://docs.bugsee.com/sdk/ios/symbolication/)

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
- [Build size analysis](https://docs.bugsee.com/sdk/ios/build-size-analysis/)
- [Manual invocation](https://docs.bugsee.com/sdk/ios/manual/)
- [Release notes](https://docs.bugsee.com/sdk/ios/release-notes/)
