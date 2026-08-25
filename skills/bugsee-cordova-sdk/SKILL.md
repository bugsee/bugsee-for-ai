---
name: bugsee-cordova-sdk
description: Full Bugsee SDK setup for Cordova. Use when asked to add Bugsee to Cordova, install bugsee cordova plugin, or set up bug reporting, crash reporting, and video recording for Cordova applications.
license: MIT
category: sdk-setup
parent: bugsee-sdk-setup
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, WebFetch, Glob, Grep
---

> [All Skills](../../SKILL_TREE.md) > [SDK Setup](../bugsee-sdk-setup/SKILL.md) > Cordova SDK

# Bugsee Cordova SDK

Opinionated wizard that guides you through complete Bugsee setup in Cordova — bug reporting with video, crash reporting, network monitoring, and console logs for iOS and Android.

## Invoke This Skill When

- User asks to "add Bugsee to Cordova" or "set up Bugsee" in a Cordova app
- User wants bug reporting, crash reporting, or video recording in Cordova
- User mentions `com.bugsee.cordova-plugin`, Bugsee Cordova, or Bugsee for Cordova/PhoneGap

> **Note:** Always verify against [docs.bugsee.com/sdk/cordova/installation/](https://docs.bugsee.com/sdk/cordova/installation/).

---

## Phase 1: Detect

```bash
# Confirm Cordova project
ls config.xml 2>/dev/null
ls www 2>/dev/null

# Check for existing Bugsee plugin
cordova plugin list 2>/dev/null | grep -i bugsee

# Check if device plugin is installed (useful for platform detection)
cordova plugin list 2>/dev/null | grep device

# Check platforms
ls platforms 2>/dev/null
```

| Question | Impact |
|----------|--------|
| `config.xml` exists? | Confirm Cordova project |
| Bugsee plugin installed? | Skip install |
| `cordova-plugin-device` installed? | Needed for platform-based token selection |

---

## Phase 2: Install

```bash
cordova plugin add com.bugsee.cordova-plugin@7.0.1 --save

# Optional: install device plugin for platform detection
cordova plugin add cordova-plugin-device --save
```

Current npm latest (re-verified 2026-08-25): **7.0.1**. Pin it rather than leaving the add unversioned.

---

## Phase 3: Initialize

Initialize Bugsee when the `deviceready` event fires. In your `www/js/index.js` or equivalent:

```javascript
document.addEventListener('deviceready', function() {
    var appToken = (device.platform == "Android")
        ? "<ANDROID_APP_TOKEN>"
        : "<IOS_APP_TOKEN>";

    Bugsee.launch(appToken);
}, false);
```

If you don't use the `cordova-plugin-device`, use a single token:

```javascript
document.addEventListener('deviceready', function() {
    Bugsee.launch("<your_app_token>");
}, false);
```

> Replace the tokens with values from your Bugsee dashboard.

> iOS/iPadOS: Since v6.0.0 the Bugsee iOS SDK supports the simulator; crash capture is excluded. For full functionality, use a real device.

---

## Phase 4: Configure (Optional)

Launch with options:

```javascript
var options = {
    shakeToTrigger: true,
    maxRecordingTime: 60,
    videoEnabled: true
};

Bugsee.launch(appToken, options);
```

Full options: [docs.bugsee.com/sdk/cordova/configuration/](https://docs.bugsee.com/sdk/cordova/configuration/)

---

## Verification

Build and run:

```bash
cordova run android
# or
cordova run ios
```

The Bugsee floating button should appear. Tap it to file a test bug report, then check the Bugsee dashboard.

---

## Documentation Links

- [Installation](https://docs.bugsee.com/sdk/cordova/installation/)
- [Configuration](https://docs.bugsee.com/sdk/cordova/configuration/)
- [Custom data](https://docs.bugsee.com/sdk/cordova/custom/)
- [Console logs](https://docs.bugsee.com/sdk/cordova/logs/)
- [Privacy](https://docs.bugsee.com/sdk/cordova/privacy/overview/)
- [Feedback](https://docs.bugsee.com/sdk/cordova/feedback/)
- [Release notes](https://docs.bugsee.com/sdk/cordova/release-notes/)
