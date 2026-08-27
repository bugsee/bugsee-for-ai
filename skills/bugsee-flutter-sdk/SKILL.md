---
name: bugsee-flutter-sdk
description: Full Bugsee SDK setup for Flutter. Use when asked to add Bugsee to Flutter, install bugsee_flutter, or set up bug reporting, crash reporting, and video recording for Flutter applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Flutter SDK

# Bugsee Flutter SDK

Opinionated wizard that scans your Flutter project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, and console logs on iOS and Android.

## Invoke This Skill When

- User asks to "add Bugsee to Flutter" or "set up Bugsee" in a Flutter app
- User wants bug reporting, crash reporting, video recording, or network monitoring in Flutter
- User mentions `bugsee_flutter`, Bugsee for Dart, or Bugsee Flutter plugin

> **Note:** Always verify against [docs.bugsee.com/sdk/flutter/installation/](https://docs.bugsee.com/sdk/flutter/installation/) before implementing. Install from **pub.dev** (`bugsee_flutter`). Do not add a git dependency on `github.com/bugsee/flutter-bugsee` — that repo is not the current package source.

---

## Phase 1: Detect

```bash
# Confirm Flutter project
ls pubspec.yaml 2>/dev/null

# Check for existing Bugsee
grep -i bugsee pubspec.yaml 2>/dev/null

# Detect current Flutter/Dart setup
grep -E 'sdk:|flutter:' pubspec.yaml 2>/dev/null | head -5

# Check for iOS CocoaPods setup
ls ios/Podfile 2>/dev/null

# Check main.dart entry point
ls lib/main.dart 2>/dev/null
cat lib/main.dart 2>/dev/null | head -30
```

| Question | Impact |
|----------|--------|
| `pubspec.yaml` exists? | Confirm Flutter project |
| Already has `bugsee_flutter`? | Skip install |
| `main.dart` structure? | Determine where to add Bugsee launch |

---

## Phase 2: Install

Add to `pubspec.yaml`:

```yaml
dependencies:
  bugsee_flutter: ^9.0.4
```

Current pub.dev latest (re-verified 2026-08-25): **9.0.4**. Docs still show `^8.0.0` — pin from the registry, not the installation page.

Then run:

```bash
flutter pub get
```

For iOS, also run:

```bash
cd ios && pod install && cd ..
```

---

## Phase 3: Initialize

Bugsee must be initialized before the rest of the app to properly intercept unhandled exceptions. Replace the contents of `lib/main.dart`:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:bugsee_flutter/bugsee.dart';

String getApplicationToken() {
  return Platform.isAndroid
      ? '<android-app-token>'
      : (Platform.isIOS ? '<ios-app-token>' : '');
}

Future<void> launchBugsee(
    void Function(bool isBugseeLaunched) appRunner) async {
  await Bugsee.launch(getApplicationToken(), appRunCallback: appRunner);
}

Future<void> main() async {
  // Required to let Bugsee intercept network requests
  HttpOverrides.global = Bugsee.defaultHttpOverrides;

  await launchBugsee((bool isBugseeLaunched) async {
    runApp(const MyApp());
  });
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      home: Scaffold(body: Center(child: Text('Hello Bugsee!'))),
    );
  }
}
```

> Replace `<android-app-token>` and `<ios-app-token>` with tokens from your Bugsee dashboard. It is common to use different app tokens for iOS and Android.

**Key points:**
- `HttpOverrides.global = Bugsee.defaultHttpOverrides;` is required to intercept network requests
- `Bugsee.launch()` with `appRunCallback` ensures all errors in the app are captured in the proper zone

---

## Phase 4: Configure (Optional)

Launch with platform-specific options:

```dart
import 'package:bugsee_flutter/bugsee.dart';

BugseeLaunchOptions? createLaunchOptions() {
  if (Platform.isAndroid) {
    var options = AndroidLaunchOptions();
    options.videoEnabled = true;
    options.monitorNetwork = true;
    return options;
  }
  if (Platform.isIOS) {
    var options = IOSLaunchOptions();
    options.videoEnabled = true;
    options.monitorNetwork = true;
    return options;
  }
  return null;
}

await Bugsee.launch(getApplicationToken(),
    appRunCallback: appRunner, launchOptions: createLaunchOptions());
```

Common options (both platforms):

| Option | Default | Description |
|--------|---------|-------------|
| `videoEnabled` | `true` | Enable video recording |
| `crashReport` | `true` | Catch and report crashes |
| `monitorNetwork` | `true` | Capture network traffic |
| `captureLogs` | `true` | Capture console logs |
| `maxRecordingTime` | `60` | Max recording duration (seconds) |
| `wifiOnlyUpload` | `false` | Upload only on WiFi |

Full options: [docs.bugsee.com/sdk/flutter/configuration/](https://docs.bugsee.com/sdk/flutter/configuration/)

---

## Verification

Build and run on a device or simulator:

```bash
flutter run
```

You should see the Bugsee floating button. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/flutter/installation/)
- [Configuration](https://docs.bugsee.com/sdk/flutter/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/flutter/custom/)
- [Console logs](https://docs.bugsee.com/sdk/flutter/logs/)
- [Privacy](https://docs.bugsee.com/sdk/flutter/privacy/overview/)
- [Network capture](https://docs.bugsee.com/sdk/flutter/network/)
- [Crash symbolication](https://docs.bugsee.com/sdk/flutter/symbolication/)
- [Manual invocation](https://docs.bugsee.com/sdk/flutter/manual/)
- [Release notes](https://docs.bugsee.com/sdk/flutter/release-notes/)
