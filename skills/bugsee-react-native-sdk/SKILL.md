---
name: bugsee-react-native-sdk
description: Full Bugsee SDK setup for React Native. Use when asked to add Bugsee to React Native, install react-native-bugsee, or set up bug reporting, crash reporting, and video recording for React Native applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > React Native SDK

# Bugsee React Native SDK

Opinionated wizard that scans your React Native project and guides you through complete Bugsee setup — bug reporting with video, crash reporting, network monitoring, and console logs on iOS and Android.

## Invoke This Skill When

- User asks to "add Bugsee to React Native" or "set up Bugsee" in a React Native app
- User wants bug reporting, crash reporting, video recording, or network monitoring in React Native
- User mentions `react-native-bugsee` or Bugsee for React Native

> **Note:** Always verify against [docs.bugsee.com/sdk/react_native/installation/](https://docs.bugsee.com/sdk/react_native/installation/) before implementing.

---

## Phase 1: Detect

```bash
# Confirm React Native project
ls package.json 2>/dev/null
grep "react-native" package.json 2>/dev/null | head -3

# Check for existing Bugsee
grep -i bugsee package.json 2>/dev/null

# Detect iOS dependency manager
ls ios/Podfile 2>/dev/null

# Check main app entry
ls App.js App.tsx index.js 2>/dev/null
```

| Question | Impact |
|----------|--------|
| Has `react-native` in package.json? | Confirm RN project |
| Already has `react-native-bugsee`? | Skip install |
| `ios/Podfile` exists? | Run `pod install` after |
| TypeScript (`App.tsx`)? | Show TS examples |

---

## Phase 2: Install

### 1. Install the npm module

```bash
npm install --save react-native-bugsee@6.0.5
```

Current npm latest (re-verified 2026-08-25): **6.0.5**. Pin the version; an unpinned `npm install` can silently resolve an older cache.

### 2. Prepare iOS project

Run CocoaPods:

```bash
cd ios && pod install && cd ..
```

If not using CocoaPods, download the framework from [https://download.bugsee.com/sdk/ios/dynamic/Bugsee-stable.xcframework.zip](https://download.bugsee.com/sdk/ios/dynamic/Bugsee-stable.xcframework.zip) and add it manually to your Xcode project.

### 3. Prepare Android project

After installing the module, perform a Gradle Sync. No additional manual configuration is required for Android.

---

## Phase 3: Initialize

Add Bugsee launch to your main `App.js` or `App.tsx`:

```javascript
import React from 'react';
import Bugsee from 'react-native-bugsee';
import { Platform } from 'react-native';

export default class App extends React.Component {
  constructor(props) {
    super(props);
    this.launchBugsee();
  }

  async launchBugsee() {
    let appToken;

    if (Platform.OS === 'ios') {
      appToken = '<IOS-APP-TOKEN>';
    } else {
      appToken = '<ANDROID-APP-TOKEN>';
    }

    await Bugsee.launch(appToken);
  }

  render() {
    // your app UI
  }
}
```

For functional components:

```javascript
import { useEffect } from 'react';
import Bugsee from 'react-native-bugsee';
import { Platform } from 'react-native';

function App() {
  useEffect(() => {
    const token = Platform.OS === 'ios'
      ? '<IOS-APP-TOKEN>'
      : '<ANDROID-APP-TOKEN>';
    Bugsee.launch(token);
  }, []);

  return (/* your app UI */);
}
```

> Replace `<IOS-APP-TOKEN>` and `<ANDROID-APP-TOKEN>` with tokens from your Bugsee dashboard.

> iOS/iPadOS: Since v6.0.0 the Bugsee iOS SDK supports the simulator; crash capture is excluded. For full functionality, use a real device.

---

## Phase 4: Configure (Optional)

Launch with options:

```javascript
const options = {
  shakeToTrigger: true,
  maxRecordingTime: 60,
  videoEnabled: true,
};

await Bugsee.launch(appToken, options);
```

Full options: [docs.bugsee.com/sdk/react_native/configuration/](https://docs.bugsee.com/sdk/react_native/configuration/)

---

## Verification

Run the app:

```bash
npx react-native run-ios
# or
npx react-native run-android
```

You should see the Bugsee floating button. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Debug Symbols

A React Native crash has **two layers**, and each needs its own symbols. Uploading only one leaves half the trace raw.

**JavaScript — source maps.** The `bugsee-sourcemaps` npm tool is the documented React Native path and still the safe default:

It ships as a `react-native-bugsee` devDependency — **do not install it globally unpinned**:

```bash
npx bugsee-sourcemaps make -t <APP_TOKEN> -p ios -v 1.2.3 ./
```

See [React Native crashes](https://docs.bugsee.com/sdk/react_native/crashes/) and [docs.bugsee.com/tools/sourcemaps](https://docs.bugsee.com/tools/sourcemaps/).

The [Bugsee CLI](../bugsee-cli/SKILL.md) can do it instead — one binary for JS *and* native — but mind the file extension:

```bash
npm i -D @bugsee/cli@0.7.11
# bundle with a .js name: --bundle-output ios/main.js --sourcemap-output ios/main.js.map
npx bugsee-cli sourcemaps inject ios/main.js
npx bugsee-cli debug-files upload ios/main.js.map --type sourcemaps \
    --version 1.4.0 --build 1400
```

> **`inject` only rewrites `.js`, `.cjs`, and `.mjs` files.** React Native's default `main.jsbundle` output is **skipped silently** — it reports `js_injected=0` and exits 0, and the upload then fails because the map carries no debug ID. Either emit the bundle with a `.js` name, or stay on `bugsee-sourcemaps`.

**Native.** iOS needs dSYMs — `bugsee-cli xcode upload-dsyms` from a Run Script build phase (CLI 0.7.7+) is the shape a config plugin can generate via `withXcodeProject`. Android needs the R8/ProGuard mapping, which the Bugsee Android Gradle plugin uploads automatically.

Full workflow: [`bugsee-upload-symbols`](../bugsee-upload-symbols/SKILL.md).

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/react_native/installation/)
- [Configuration](https://docs.bugsee.com/sdk/react_native/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/react_native/custom/)
- [Console logs](https://docs.bugsee.com/sdk/react_native/logs/)
- [Privacy](https://docs.bugsee.com/sdk/react_native/privacy/overview/)
- [Network capture](https://docs.bugsee.com/sdk/react_native/network/)
- [Error boundary](https://docs.bugsee.com/sdk/react_native/errorboundary/)
- [Crash reports](https://docs.bugsee.com/sdk/react_native/crashes/)
- [Release notes](https://docs.bugsee.com/sdk/react_native/release-notes/)
