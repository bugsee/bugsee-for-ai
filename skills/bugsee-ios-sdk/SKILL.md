---
name: bugsee-ios-sdk
description: Full Bugsee SDK setup for iOS, tvOS and visionOS (7.x — the current SDK). Use when asked to add Bugsee to iOS, install Bugsee via SPM, migrate a Bugsee 6.x integration to 7.x, or set up bug reporting, crash reporting, video recording, APM, or build size analysis for iOS applications. For legacy 6.x apps, use bugsee-ios-sdk-6x.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > iOS SDK

# Bugsee iOS SDK

Opinionated wizard that scans an iOS project and either wires up Bugsee 7.x from scratch or migrates an existing 6.x integration — core SDK via Swift Package Manager, the separate Feedback package, breadcrumbs, blackout, and the new option-key family.

> **7.x is the current iOS SDK** (GA since 7.0.0). Latest release: `7.0.0` on `https://github.com/bugsee/spm` and `https://github.com/bugsee/feedback-spm` (re-verified 2026-10-31 against the SPM tags). If you are maintaining an app that stays on the 6.x line, use the legacy [6.x skill](../bugsee-ios-sdk-6x/SKILL.md) instead; when upgrading from 6.x, follow the [migration guide](https://docs.bugsee.com/sdk/ios/migration/).

## Invoke This Skill When

- User asks to add Bugsee to an iOS, tvOS or visionOS app, or to set up bug reporting / crash reporting with Bugsee on iOS.
- User asks to upgrade or migrate an existing Bugsee 6.x integration to 7.x.
- User mentions **breadcrumbs** (`Bugsee.addBreadcrumb`, `setBreadcrumbFilter`, `getExchangeFactory`).
- User mentions **blackout** (`startBlackout`, `endBlackout`, `isBlackout`).
- User mentions **notification relay** (`Bugsee.notify`).
- User mentions the **report handler** (`Bugsee.setReportHandler`, `BugseeReportHandler`), the **Crash Report Extension** (`BugseeCrashExtension`), or the separate **`BugseeFeedback`** package.
- User mentions the `BugseeOption*` key family, `setUserIdentifier`, `showReportDialog`, `addSecureView`, or `addSecureRectangle`.
- User needs **tvOS** or **visionOS** support (7.x only).

Switch to the [6.x skill](../bugsee-ios-sdk-6x/SKILL.md) only when the user wants to stay on 6.x (for example, an app that must keep supporting iOS 12–14 or keep CocoaPods).

> **Always verify against** [docs.bugsee.com/sdk/ios/installation/](https://docs.bugsee.com/sdk/ios/installation/) before implementing. 7.x docs live under `/sdk/ios/`; the 6.x docs moved to `/sdk/ios/v6/`.

---

## Phase 1: Detect

```bash
# Xcode project / workspace
ls *.xcodeproj *.xcworkspace 2>/dev/null

# Swift vs Objective-C
find . -name "*.swift" -not -path "*/Pods/*" -not -path "*/.build/*" 2>/dev/null | head -5
find . -name "*.m" -not -path "*/Pods/*" 2>/dev/null | head -5

# Dependency manager — 7.x ships via SPM
ls Podfile Package.swift Cartfile 2>/dev/null

# Existing Bugsee install (and which major version) — also Xcode-managed SPM packages
grep -rin bugsee Podfile Podfile.lock Package.swift Package.resolved Cartfile \
  *.xcodeproj/project.pbxproj *.xcworkspace/xcshareddata/swiftpm/Package.resolved \
  *.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved 2>/dev/null | head -10

# Deployment target — 7.x requires iOS 15+
grep -r "IPHONEOS_DEPLOYMENT_TARGET" *.xcodeproj/project.pbxproj 2>/dev/null | sort -u | head -5

# Entry point
find . -name "AppDelegate.swift" -o -name "AppDelegate.m" 2>/dev/null | head -3
grep -r "@main" --include="*.swift" 2>/dev/null | head -3

# 6.x call sites that will not compile against 7.x
grep -rnE "showReportController|showFeedbackController|setDefaultFeedbackGreeting|didReceiveNewFeedback|setView:.*asHidden|setView\(.*asHidden|addSecureRect\b|removeSecureRect\b|removeAllSecureRects|getAllSecureRects|isViewHidden|registerEvent|traceKey|registerNetworkEvent|removeNetworkEventFilter|setEmail|getEmail|clearEmail|getDeviceId|logAssert|hideKeyboard|testExceptionCrash|testSignalCrash|BugseeExceptionLoggingOptions|BugseeMergingRulesSkipFramesKey|logException.*options:|logError:[^;]*labels:|logError\(error:.*labels:|setDefaultCrashPriority|setDefaultErrorPriority|setDefaultBugPriority|Bugsee\.pause|Bugsee\.resume|\[Bugsee pause\]|\[Bugsee resume\]" \
  --include="*.swift" --include="*.m" --include="*.mm" . 2>/dev/null | grep -v "/Pods/" | head -40

# Removed option keys and properties (build errors), and their raw strings (silently ignored)
grep -rnE "BugseeKillDetectionKey|BugseeEnableMachExceptionsKey|BugseeCaptureVideoAdaptiveKey|\.killDetection|\.enableMachExceptions|\.captureVideoAdaptive|\"CaptureVideoAdaptive\"|\"BugseeEnableMachExceptions\"" \
  --include="*.swift" --include="*.m" --include="*.mm" . 2>/dev/null | grep -v "/Pods/" | head -20

# Raw 6.x option strings — still translated in 7.x, but worth moving to the BugseeOption* constants
grep -rnE '"(BugseeAppLaunchCrashDetectionKey|BugseeDefaultBugPriority|BugseeDefaultCrashPriority|BugseeDefaultErrorPriority|BugseeEnableMachExceptions|BugseeEnableOnDeviceSymbolication|BugseeKillDetectionKey|BugseeReportPrioritySelector|BugseeStyle|BuildTarget|BuildType|CaptureAVPlayer|CaptureLogs|CaptureOSLogs|CaptureVideoAdaptive|CrashReport|DataEncryption|DetectAppExit|FrameRate|MaxDataSize|MaxFrameRate|MaxRecordingTime|MinFrameRate|MonitorBluetoothStatus|MonitorDiskSpace|MonitorNetwork|MonitorWebSocket|PerformanceAdaptiveSampling|PerformanceMonitoring|PerformanceSampleRate|ReportDescriptionRequired|ReportEmailRequired|ReportLabelsEnabled|ReportLabelsRequired|ReportSummaryRequired|SanitizeNetworkData|ScreenshotEnabled|ScreenshotToReport|ShakeToReport|StatusBarInfo|VideoEnabled|VideoScale|ViewHierarchyEnabled|WifiOnlyUpload|bodySizeLimit)"' \
  --include="*.swift" --include="*.m" --include="*.mm" . 2>/dev/null | grep -v "/Pods/" | head -20
```

| Question | Impact |
|----------|--------|
| Deployment target below iOS 15? | **Stop and ask the user.** 7.x needs iOS 15, and raising the minimum drops every user on older iOS versions — a product decision, not yours. Raise it only with their approval. If the app must keep iOS 12–14, switch to [`bugsee-ios-sdk-6x`](../bugsee-ios-sdk-6x/SKILL.md) |
| `Podfile` / `Cartfile` only? | **Stop and ask the user.** CocoaPods and Carthage are not supported in 7.x: with their approval, add an SPM dependency (or the direct-download XCFramework) and remove the pod / Carthage entry. If they must stay on CocoaPods or Carthage, switch to [`bugsee-ios-sdk-6x`](../bugsee-ios-sdk-6x/SKILL.md) |
| Existing Bugsee 6.x found? | This is a migration, not a fresh install — go to Phase 2b after Phase 2 |
| Removed option keys found? | Replace them (see the "Removed" table in Phase 2b); their raw strings are ignored silently |
| Raw 6.x option strings found? | They keep working (7.x translates them); swap them for `BugseeOption*` constants when convenient |
| Options never set explicitly? | Check the changed defaults in Phase 2b — shake/screenshot triggers, exit and early-crash detection, adaptive video, crash handler type, max data size |
| Feedback call sites found? | Feedback needs the separate `BugseeFeedback` package |
| Swift files found? | Show Swift init code |
| Objective-C files found? | Show Objective-C init code |
| SwiftUI `@main` lifecycle? | See [SwiftUI docs](https://docs.bugsee.com/sdk/ios/swiftui/) |

> **Swift-only rename to watch for.** 6.x misspelled one Swift label: `Bugsee.logUnhandledException(exception:comlletion:)`. In 7.x it is `logUnhandledException(exception:completion:)`. Objective-C callers are unaffected, and the grep above will not catch it — check Swift call sites of `logUnhandledException` by hand.

---

## Phase 2: Install

7.x is distributed through Swift Package Manager, from the same repository 6.x used:

```
https://github.com/bugsee/spm
```

Which file you edit depends on how the project is laid out.

**`Package.swift` exists** — edit it directly. Add the dependency and wire the product into the target:

```swift
dependencies: [
    .package(url: "https://github.com/bugsee/spm", from: "7.0.0"),
],
targets: [
    .target(
        name: "YourApp",
        dependencies: [
            // Package identity is "spm" — the last path component of the URL, not "bugsee".
            .product(name: "Bugsee", package: "spm"),
        ]
    ),
]
```

Use `from:` (Xcode: **Up to Next Major Version**). Only a pre-release version (`-betaN`) needs `exact:`, because SwiftPM excludes pre-releases from version ranges. The package must also declare `platforms: [.iOS(.v15)]` or resolution fails.

Products: `Bugsee` (app target), `BugseeSwiftUI` (app target, instead of `Bugsee`, for `bugseeProtect()`), `BugseeCrashExtension` (a Crash Reporter Extension target only, iOS 27+).

**`.xcodeproj` with no package manifest** — the reference lives in `<project>.xcodeproj/project.pbxproj` as an `XCRemoteSwiftPackageReference` plus an `XCSwiftPackageProductDependency` wired into the target's `packageProductDependencies`. Hand-editing that file corrupts projects. Do not attempt it: ask the user to add the package once through Xcode (**File → Add Package Dependencies…**, the URL above, dependency rule **Up to Next Major Version** from 7.0.0), then continue from Phase 3. Confirm it landed before moving on:

```bash
grep -c "XCRemoteSwiftPackageReference" *.xcodeproj/project.pbxproj
grep -rn "bugsee/spm" *.xcodeproj/project.pbxproj *.xcworkspace/xcshareddata/swiftpm/Package.resolved 2>/dev/null | head
```

> **Do not guess a pre-release.** Pin the latest 7.x release unless the user asks for a specific version. If an existing `Package.swift` or `Package.resolved` already references `bugsee/spm`, change the rule in place rather than adding a second dependency on the same repository.

Requirements: iOS 15+, tvOS 15+, visionOS 1+.

For builds that cannot use SwiftPM (Unity, hand-embedded frameworks): download `https://download.bugsee.com/sdk/ios/spm/Bugsee-<version>.zip`, embed `Bugsee.xcframework` (Embed & Sign), and keep its `Tools/` folder (`BugseeAgent`, `BugseeClean`) inside the folder that contains the `.xcodeproj`.

### Optional: in-app feedback

Feedback is no longer in the core framework. It ships as a separate Swift package, `BugseeFeedback`, published with every core release at `https://github.com/bugsee/feedback-spm`. Add it only if the app uses in-app feedback / chat, with the **Exact Version** rule at the same version as the core package — each `BugseeFeedback` release requires that core version exactly. Its package identity is `feedback-spm`: `.product(name: "BugseeFeedback", package: "feedback-spm")`.

---

## Phase 2b: Migrate 6.x call sites

Skip this phase for a fresh install. Run it whenever Phase 1 found existing Bugsee 6.x code.

Work in this order — it is not arbitrary:

1. **Check the defaults that changed.** These are silent. Unless the app sets them, 7.x
   turns on shake-to-report (`BugseeOptionReportingTriggerByShake`), turns off
   screenshot-to-report (`BugseeOptionReportingTriggerByScreenshot`), turns on exit and
   early-crash detection (`BugseeOptionDetectAndReportExit`,
   `BugseeOptionDetectAndReportEarlyCrash`) and adaptive video
   (`BugseeOptionCaptureVideoAdaptive`), uses the `MachAndBSD` crash handler
   (`BugseeOptionCaptureCrashHandlerType`) and raises `BugseeOptionConfigMaxDataSize` from 50
   to 150 MB. Ask the user which 6.x behaviors to keep and pin those
   ([§3](https://docs.bugsee.com/sdk/ios/migration/#3-defaults-that-changed)).
2. **Build.** The removed keys and APIs surface as compiler errors, so let the build drive
   the rest. Two things do not: the `BugseeDelegate` methods' new Swift names (a Swift
   implementation with the 6.x name compiles and is never called — rename by hand,
   [§4.6](https://docs.bugsee.com/sdk/ios/migration/#46-bugseedelegate-methods-have-new-swift-names)),
   and lifecycle event raw values if the app forwards them as numbers.
3. **Apply the renames** in the table below.
4. **Rewire feedback** if the app uses it — see *Optional: in-app feedback* above.
5. **Move option keys to the `BugseeOption*` constants.** The 6.x `Bugsee*Key` constants and
   raw strings still work in 7.x, so this is not urgent, but options new in 7.x exist only
   under the new names ([option tables](https://docs.bugsee.com/sdk/ios/migration/#2-option-keys)).
6. **Re-check `pause` / `resume` by hand.** `startBlackout` / `endBlackout` is the closest
   match but is narrower: it suppresses only what describes the screen, while logs, network
   events and traces keep being recorded. If the app used `pause` to stop *all* capture,
   `[Bugsee stop:]` is the honest replacement.

An iOS 15 deployment target is a prerequisite for all of the above. If the app is below it, ask the user before raising it (see Phase 1); if they must keep the older target, stay on 6.x with [`bugsee-ios-sdk-6x`](../bugsee-ios-sdk-6x/SKILL.md).


**Renamed.** A plain search and replace does most of these.

| 6.x | 7.x |
| --- | --- |
| `Bugsee.appearance()` | `Bugsee.getAppearance()` |
| `Bugsee.activeSpan()` | `Bugsee.getActiveSpan()` |
| `registerEvent:` | `event:` |
| `registerEvent:withParams:` | `event:params:` (params now optional) |
| `traceKey:withValue:` | `trace:value:` |
| `registerNetworkEvent:` | `addNetworkEvent:` |
| `registerNetworkEvent:needsToBeFiltered:` | `addNetworkEvent:requiresFiltering:` |
| `removeNetworkEventFilter` | `setNetworkEventFilter:nil` |
| `showReportController*` | `showReportDialog*` |
| `setView:asHidden:` | `addSecureView:` / `removeSecureView:` |
| `addSecureRect:` | `addSecureRectangle:` |
| `removeSecureRect:` | `removeSecureRectangle:` |
| `removeAllSecureRects` | `removeAllSecureRectangles` |
| `getAllSecureRects` | `getAllSecureRectangles` |
| `setEmail:` / `getEmail` / `clearEmail` | `setUserIdentifier:` / `getUserIdentifier` / `clearUserIdentifier` |
| `clearAllAttribute` | `clearAllAttributes` |
| `testExceptionCrash` / `testSignalCrash` | `testCrash` |
| `uploadWithSummary:…:includeVideo:` | `uploadWithSummary:description:severity:labels:` |
| `logError:labels:includeVideo:` / `logError:labels:` | `logError:`; labels via `addLabels:` on the report in `bugseeAttachmentsForReport:` |
| `logException:options:completion:` | `logException:completion:` / `logException:domain:skipFrames:completion:` |
| `logException:reason:options:completion:` | `logException:reason:completion:` / `logException:reason:domain:skipFrames:completion:` |
| `BugseeExceptionLoggingOptions` (`exceptionDomain`, `mergingRules[BugseeMergingRulesSkipFramesKey]`, `labels`) | `domain:` / `skipFrames:` parameters of `logException`; labels via `addLabels:` on the report in `bugseeAttachmentsForReport:` (applies to every error report, not one call); `includeVideo` has no replacement |
| Swift `Bugsee.launch(token:options:)` _(dictionary)_ | Swift `Bugsee.launch(token:dictionaryOptions:)` |
| Swift `Bugsee.relaunch(options:)` _(dictionary)_ | Swift `Bugsee.relaunch(dictionaryOptions:)` |
| Swift `Bugsee.uploadReport(_:)` | Swift `Bugsee.upload(_:)` |
| Swift `Bugsee.deleteCollectedData(completion:)` | Swift `Bugsee.deleteCollectedDataOnDevice(completion:)`. The Objective-C selector `deleteCollectedDataOnDevice:` is unchanged and still deletes everything. Do not switch to the `includingIntermediate:` forms: they keep pending crashes (and, without `includingBundles:`, the reports waiting for upload) |
| Swift `logUnhandledException(exception:comlletion:)` | Swift `logUnhandledException(exception:completion:)` |
| CocoaPods / Carthage | Swift Package Manager or the direct-download XCFramework |
| `showFeedbackController` | `[BugseeFeedback.shared showFeedbackUI]` |
| `setDefaultFeedbackGreeting:` | `[BugseeFeedback.shared setGreeting:]` |
| `bugsee:didReceiveNewFeedback:` | `BugseeFeedbackListener` |

**Removed with no direct replacement.**

| 6.x | Notes |
| --- | --- |
| `pause` / `resume` | Closest is `startBlackout` / `endBlackout`, which is narrower — see [§4.2](https://docs.bugsee.com/sdk/ios/migration/#42-pause-and-resume-are-gone) |
| `getDeviceId` | — |
| `accessToken` | — |
| `isViewHidden:` | — |
| `hideKeyboard:` | — |
| `setInternalSecureRectangles:` | — |
| `logAssert:withLocation:` | Use `logException:reason:completion:` |
| `log:level:timestamp:` | `log:level:requiresFiltering:` takes the slot but does something different — it opts the message into the log filter |
| `setDefaultCrashPriority:` | `BugseeOptionReportingDefaultCrashPriority` at launch |
| `setDefaultErrorPriority:` | `BugseeOptionReportingDefaultErrorPriority` at launch |
| `setDefaultBugPriority:` | `BugseeOptionReportingDefaultBugPriority` at launch |
| `BugseeAttachmentOverrideLabels` | — |
| `BugseeOptions.captureVideoAdaptive` / `BugseeCaptureVideoAdaptiveKey` | `BugseeOptionCaptureVideoAdaptive` key |
| `BugseeLifecycleEventStarted` / `Resumed` / `Paused` | — (raw values of the remaining events changed) |
| `BugseeEnableMachExceptionsKey` / `BugseeOptions.enableMachExceptions` | `BugseeOptionCaptureCrashHandlerType` / `crashHandlerType`: 6.x NO → `BugseeCrashHandlerTypeBSD`, YES → `BugseeCrashHandlerTypeMach`; the 7.x default is `BugseeCrashHandlerTypeMachAndBSD`. The old key, also as a plain string, is ignored with a console warning. New in 7.x: `BugseeOptionCaptureCrashSignals` / `crashSignals` chooses which fatal signals are intercepted; leave it unset, since the default intercepts every signal (the Xamarin and .NET SDKs leave out `SIGSEGV` and `SIGBUS`), and an explicit value is used as given |
| `BugseeKillDetectionKey` / `BugseeOptions.killDetection` | Out-of-memory kills are reported through `BugseeOptionDetectAndReportExit`, on by default |


Full detail, with before/after code for each: [migration guide](https://docs.bugsee.com/sdk/ios/migration/).

---

## Phase 3: Initialize

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

> Replace `<your_app_token>` with the token from the Bugsee dashboard.

> Simulator slices are built without the native crash reporter. Run on a real device to exercise crash capture.

Capture is brought up off the main thread, and a launch from `didFinishLaunching` waits for the app to become active, so `launch` returns before the SDK is recording. The `started:` block only says the launch was accepted; to know recording is live, observe `Bugsee.sharedInstance()?.status` against `BugseeStatusLaunched` or implement `bugsee(didChangeStatus:)`.

### Feedback wiring (only if `BugseeFeedback` was added)

**Swift:**

```swift
import BugseeFeedback

BugseeFeedback.register()          // optional (the package also registers itself on load); must come before launch
Bugsee.launch(token: "<your_app_token>")

// Present the chat
BugseeFeedback.shared.showFeedbackUI()
```

**Objective-C:**

```objectivec
@import BugseeFeedback;

[BugseeFeedback register];         // optional (the package also registers itself on load); must come before launch
[Bugsee launchWithToken:@"<your_app_token>"];

[BugseeFeedback.shared showFeedbackUI];
```

---

## Phase 4: Configure (Optional)

Options are passed as a `BugseeOptions` object, as a dictionary keyed by the `BugseeOption*` constants, or both.

**Swift:**

```swift
let options = BugseeOptions.default()
options.shakeToReport = true
options.maxRecordingTime = 60

// Options with no BugseeOptions property are set by key:
options.updateLaunch([
    BugseeOptionCaptureBreadcrumbs: true,
    BugseeOptionDetectAndReportHang: true,
])

Bugsee.launch(token: "<your_app_token>", options: options)
```

**Objective-C:**

```objectivec
BugseeOptions *options = [BugseeOptions defaultOptions];
options.shakeToReport = YES;
options.maxRecordingTime = 60;

[options updateLaunchOptions:@{
    BugseeOptionCaptureBreadcrumbs  : @YES,
    BugseeOptionDetectAndReportHang : @YES,
}];

[Bugsee launchWithToken:@"<your_app_token>" options:options];
```

> **Use the `BugseeOption*` constants.** 7.x still translates the 6.x constants and raw strings (`@"ShakeToReport"`), but options new in 7.x exist only under the new names, and an unknown key is ignored without a warning.

Commonly used option keys:

| Key | Default | Description |
|-----|---------|-------------|
| `BugseeOptionCaptureVideo` | `YES` | Video recording. |
| `BugseeOptionCaptureLogs` | `YES` | Console log capture. |
| `BugseeOptionCaptureNetwork` | `YES` | Network traffic capture. |
| `BugseeOptionCaptureBreadcrumbs` | `NO` | Automatic breadcrumb collection (new in 7.x). |
| `BugseeOptionDetectAndReportCrash` | `YES` | Crash capture. |
| `BugseeOptionReportingTriggerByShake` | `YES` | Shake-to-report gesture. Was `NO` in 6.x. |
| `BugseeOptionReportingTriggerByScreenshot` | `NO` | Screenshot-to-report trigger. Was `YES` in 6.x. |
| `BugseeOptionConfigDuration` | `60` | Max recording duration, seconds. |
| `BugseeOptionConfigWifiOnlyUpload` | `NO` | Restrict uploads to Wi-Fi. |
| `BugseeOptionPerformanceMonitoring` | `YES` | APM master switch. |

Full list: [configuration](https://docs.bugsee.com/sdk/ios/configuration/) · [6.x → 7.x key map](https://docs.bugsee.com/sdk/ios/migration/#2-option-keys).

### New 7.x APIs worth wiring

```swift
// Suppress everything that describes the screen during a sensitive flow.
// Logs, network events and traces keep being captured.
Bugsee.startBlackout()
Bugsee.endBlackout()

// Identify the user (replaces 6.x setEmail).
Bugsee.setUserIdentifier("user@example.com")

// Custom breadcrumb.
let crumb = Bugsee.getExchangeFactory().createBreadcrumb()
crumb.category = "checkout"
crumb.message  = "Coupon applied"
crumb.type     = "user"
Bugsee.addBreadcrumb(crumb)

// Lightweight notification straight to the messaging integrations —
// no video, logs or events attached.
Bugsee.notify(title: "Checkout failed", body: "Provider returned 502")
```

> A network filter does **not** reach breadcrumbs. If `setNetworkEventFilter` drops or rewrites a URL, repeat the rule in `setBreadcrumbFilter` or it will still appear in the breadcrumb trail.

### BugseeAgent build info and size analysis

The same `BugseeAgent` post-action that uploads dSYMs also registers a build record (version, configuration, VCS context, timings, Mach-O `LC_UUID`). Size analysis is opt-in.

1. Add a Run Script **Archive** post-action (Product → Scheme → Edit Scheme → Archive → Post-actions). Set **Provide build settings from** to the app target.
2. Script body for the direct-download XCFramework (locates `BugseeAgent` in the archive's `Tools/` folder, kept inside the project folder):

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

Build and run on a real device. Trigger a report end to end:

```swift
// Programmatic trigger
Bugsee.showReportDialog()

// Or test crash capture (remove after verifying)
Bugsee.testCrash()

// Or a logged error
Bugsee.logError(error: NSError(domain: "smoke", code: 1))
```

Check the Bugsee dashboard for the incoming report. If breadcrumbs are enabled, navigate a few screens first and confirm the trail appears on the report.

## Debug Symbols

A Release build strips symbols, so crashes arrive as raw addresses until dSYMs are uploaded. Confirm `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` for the Release configuration first — without it there are no dSYMs to upload at all.

Two shapes, both driven by the [Bugsee CLI](../bugsee-cli/SKILL.md):

- **Scheme post-action** — the SDK's `BugseeAgent` script runs `bugsee-cli xcode post-action` from Archive → Post-actions. Uploads dSYMs *and* registers the build (build info, dependencies, timings, optional size analysis). Gated to Release + Archive by default.
- **Run Script build phase** — `bugsee-cli xcode upload-dsyms` (CLI 0.7.7+) uploads dSYMs alone, with no build registration, so it is safe on every build. A genuine failure fails the build on purpose; `--no-fail --no-background` is the usual CI combination.

> **Xcode 15+:** `ENABLE_USER_SCRIPT_SANDBOXING` defaults to `YES` and blocks a build phase from reading the dSYM folder. Set it to `NO` on the target, or declare the folder in the phase's input file lists.

Full workflow, including manual upload of App Store Connect dSYMs: [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md).

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/ios/installation/)
- [Migration from 6.x](https://docs.bugsee.com/sdk/ios/migration/)
- [Configuration](https://docs.bugsee.com/sdk/ios/configuration/)
- [SwiftUI](https://docs.bugsee.com/sdk/ios/swiftui/)
- [Report handler](https://docs.bugsee.com/sdk/ios/report-handler/)
- [Privacy](https://docs.bugsee.com/sdk/ios/privacy/overview/)
- [Crash symbolication](https://docs.bugsee.com/sdk/ios/symbolication/)
- [Bug reporting](https://docs.bugsee.com/sdk/ios/manual/bug-reporting/)
- [Crash Report Extension](https://docs.bugsee.com/sdk/ios/issue-detection/crash-report-extension/)
- [Build size analysis](https://docs.bugsee.com/sdk/ios/build-size-analysis/)
- [Release notes](https://docs.bugsee.com/sdk/ios/release-notes/)
- [6.x docs](https://docs.bugsee.com/sdk/ios/v6/installation/) · [6.x skill](../bugsee-ios-sdk-6x/SKILL.md)
