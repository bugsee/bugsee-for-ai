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
