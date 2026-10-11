---
name: micorlov-e2e-mobile
description: Mobile E2E/UI testing for Android and iOS apps (native Kotlin/Java/Compose, Swift/SwiftUI, and Unity games). Detects the project, picks the right framework itself (Espresso, XCUITest, Maestro, Appium, or AltTester for Unity), selects a real device first, builds and installs a dev build, runs the tests, and writes a report. Use when writing, running, or fixing mobile UI tests, Espresso/XCUITest/Maestro/Appium/AltTester suites, flaky mobile tests, or "test this flow on my phone".
metadata:
  origin: micorlov
  modeled_on: ecc/e2e-testing
---

# Mobile E2E Testing

Patterns and a run workflow for stable mobile UI test suites on Android and iOS.
Work through Steps 1-7 in order. Load a `references/*.md` file only for the framework you picked.

## Ground Rules

- **Real device first.** Physical device if connected; emulator/simulator only when none is.
- **Development builds only.** Never install or test a release/production-signed build unless the user asks.
- **Ask before side effects:** installing tools, uninstalling an app (wipes its data), or adding test IDs to app code.
- **No production backends, real payments, or real user accounts.** Use test accounts from the project's fixtures.
- **Artifacts go to `<project>/e2e-artifacts/`**, and that folder goes in `.gitignore`.

## Step 1: Detect the Project

Run from the project root. The commands use `find` instead of shell globs, because in zsh an
unmatched glob (`ls build.gradle*`) aborts with "no matches found".

```bash
find . -maxdepth 2 -name "build.gradle*" -not -path "./Library/*" | head -1   # Android (Gradle)
grep -rl "androidx.compose" --include="*.gradle*" --include="*.toml" . 2>/dev/null | head -1  # Compose
find . -maxdepth 2 \( -name "*.xcodeproj" -o -name "*.xcworkspace" -o -name Package.swift \) | head -1  # iOS
test -f ProjectSettings/ProjectVersion.txt && cat ProjectSettings/ProjectVersion.txt  # Unity (+ editor version)
grep -s "com.alttester" Packages/manifest.json                            # AltTester SDK present
find . -path "*/src/androidTest/*" \( -name "*.kt" -o -name "*.java" \) \
  -not -path "*/node_modules/*" -not -path "./Library/*" | head -1          # Espresso suite
find . -maxdepth 3 -type d -name "*UITests" | head -1                     # XCUITest target
find . -maxdepth 1 -type d -name .maestro                                 # Maestro flows dir
grep -rl "^appId:" --include="*.yaml" --exclude-dir=node_modules --exclude-dir=Library . 2>/dev/null | head -1  # Maestro
find . -maxdepth 1 -name "wdio.conf.*"; grep -s -E '"(appium|webdriverio)"' package.json  # Appium/WDIO
```

| Signal | Meaning |
|--------|---------|
| `build.gradle(.kts)` + `AndroidManifest.xml` | Native Android app |
| `androidx.compose` dependency | Jetpack Compose UI (use `testTag`) |
| `*.xcodeproj` / `*.xcworkspace` | Native iOS app |
| `ProjectSettings/ProjectVersion.txt` + `Assets/` | Unity game (may target both platforms) |
| `src/androidTest/` with Espresso deps | Existing Espresso suite |
| `*UITests` target | Existing XCUITest suite |
| `.maestro/` or YAML with `appId:` | Existing Maestro flows |
| `wdio.conf.ts` / `appium` in `package.json` | Existing Appium suite |

## Step 2: Pick the Framework

Go top to bottom and stop at the first match:

```text
1. A suite already exists for this platform?      → Use it. Don't add a second framework.
2. Unity game?                                    → AltTester + Appium      (references/unity-alttester.md)
                                                    Maestro only for a launch/no-crash smoke.
3. Test touches system UI, other apps, permission
   dialogs, notifications, or is a quick black-box
   smoke of an installed build?                   → Maestro                 (references/maestro.md)
                                                    (iOS real device: use XCUITest instead; see note)
4. One suite must run on Android AND iOS, or the
   app has WebViews/hybrid screens, or it runs on
   BrowserStack/Sauce?                            → Appium + WebdriverIO    (references/appium.md)
5. Native Android, in-app flows                   → Espresso / Compose test (references/espresso.md)
6. Native iOS, in-app flows                       → XCUITest                (references/xcuitest.md)
```

Before writing any test, state the choice and the reason in one line, e.g.
`Framework: Espresso (native Compose app, in-app login flow, no existing suite)`.

**Why this order:** Espresso and XCUITest run inside the app's process and sync with the UI thread,
so they are the fastest and least flaky for in-app flows. Maestro is the fastest to write and
handles system UI, but it is black-box. Appium is the most portable but also the slowest and
most flake-prone, so use it only when you need its portability.

**Maestro + iOS note:** Maestro targets iOS Simulators. For a physical iPhone, prefer XCUITest
(native) or Appium's XCUITest driver.

## Step 3: Pick the Device

### Android

Use the `adb` on PATH. If there isn't one, use the project's SDK: Android Studio's
(`$ANDROID_HOME/platform-tools/adb`, usually `~/Library/Android/sdk/platform-tools/adb`), or for Unity,
the SDK of the editor version in `ProjectSettings/ProjectVersion.txt`
(`/Applications/Unity/Hub/Editor/<version>/PlaybackEngines/AndroidPlayer/SDK/platform-tools/adb`).

```bash
ADB=$(command -v adb || echo "$HOME/Library/Android/sdk/platform-tools/adb")
"$ADB" devices -l
```

The examples below write `adb`; substitute `"$ADB"` when adb isn't on PATH.

- Use the first serial that does **not** start with `emulator-` and whose state is `device`.
- `unauthorized` → stop and ask the user to accept the USB-debugging prompt on the phone.
- `offline` or no physical device → say so and ask the user to plug the phone in. Use an
  emulator only if the user confirms there is no physical device.
- Always target the device explicitly: `adb -s <serial>`, `ANDROID_SERIAL=<serial>`,
  `maestro --device <serial>`, Appium `appium:udid`.
- Record the model for the report: `adb -s <serial> shell getprop ro.product.model`.
- If an emulator you started has been idle for more than 10 minutes, kill it: `adb -s <emulator-serial> emu kill`.

### iOS

```bash
xcrun devicectl list devices          # physical iPhones/iPads (Xcode 15+)
xcrun simctl list devices booted      # booted simulators
```

- Prefer a connected physical device (needs signing with a development team).
- Otherwise use a booted Simulator, or boot one: `xcrun simctl boot "<name>"`.
- Always pass the target explicitly: `-destination 'id=<UDID>'`, `appium:udid`, `maestro --device <UDID>`.

## Step 4: Tooling Check

| Framework | Check | Install (ask first) |
|-----------|-------|---------------------|
| Espresso | `./gradlew --version`, `adb version` | Comes with Gradle/Android SDK |
| XCUITest | `xcodebuild -version` | Xcode (user installs) |
| Maestro | `maestro --version` | `curl -Ls "https://get.maestro.mobile.dev" \| bash` (needs Java 17+) |
| Appium | `appium --version`, `appium driver list --installed` | `npm i -g appium` then `appium driver install uiautomator2` / `appium driver install xcuitest` |
| AltTester | `python3 -c "import alttester"`, AltTester Desktop running | `pip install AltTester-Driver Appium-Python-Client pytest` |

If something is missing: show the exact install command, **ask**, and install only after the user says yes.

## Step 5: Build and Install a Dev Build

Use the project's own build command (check its README, `scripts/`, or CI config first). Defaults:

```bash
# Android (debug app + test APK)
./gradlew assembleDebug assembleDebugAndroidTest
adb -s <serial> install -r app/build/outputs/apk/debug/app-debug.apk
```

```bash
# iOS (build once, test many times)
xcodebuild build-for-testing -scheme <App> -destination 'id=<UDID>' \
  -derivedDataPath build/DerivedData
```

- Unity: build a **development** build with AltTester instrumentation through the project's build
  script (see `references/unity-alttester.md`). Never ship an AltTester-instrumented build.
- `adb install -r` keeps app data. If it fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`
  (signature mismatch) or `INSTALL_FAILED_VERSION_DOWNGRADE`, **ask before uninstalling**
  because uninstalling wipes the app's data.

## Step 6: Prepare the Device and Run

### Disable animations (Android), then restore them afterwards

```bash
for s in window_animation_scale transition_animation_scale animator_duration_scale; do
  adb -s <serial> shell settings put global $s 0
done
# ... run tests ...
for s in window_animation_scale transition_animation_scale animator_duration_scale; do
  adb -s <serial> shell settings put global $s 1
done
```

(For Espresso, `testOptions { animationsDisabled = true }` handles this for you.)

### Run commands

| Framework | Command (full details in reference) |
|-----------|------------------------------------|
| Espresso | `ANDROID_SERIAL=<serial> ./gradlew connectedDebugAndroidTest` |
| XCUITest | `xcodebuild test-without-building -scheme <App> -destination 'id=<UDID>' -resultBundlePath e2e-artifacts/ios/Result.xcresult` |
| Maestro | `maestro --device <serial> test .maestro/ --format junit --output e2e-artifacts/maestro/report.xml` |
| Appium | `ANDROID_SERIAL=<serial> npx wdio run wdio.conf.ts` |
| AltTester | `pytest tests/unity --junitxml=e2e-artifacts/unity/report.xml` |

Run a single test or flow first to check the setup, then the whole suite.

## Step 7: Collect Artifacts and Report

```text
e2e-artifacts/
├── REPORT.md
├── screenshots/        failure + key-step screenshots
├── videos/             screen recordings
├── logs/               logcat / device logs
└── <framework>/        JUnit XML, .xcresult, framework reports
```

```bash
adb -s <serial> logcat -c                                   # before the run
adb -s <serial> shell screenrecord /sdcard/e2e.mp4 &        # max 180 s per file
# ... run ...
adb -s <serial> logcat -d > e2e-artifacts/logs/logcat.txt
adb -s <serial> pull /sdcard/e2e.mp4 e2e-artifacts/videos/ && adb -s <serial> shell rm /sdcard/e2e.mp4
adb -s <serial> exec-out screencap -p > e2e-artifacts/screenshots/final.png
xcrun simctl io booted recordVideo e2e-artifacts/videos/ios.mp4   # Simulator; Ctrl+C to stop
```

Add `e2e-artifacts/` to `.gitignore` if it is not there.

### Report Template (`e2e-artifacts/REPORT.md`)

```markdown
# Mobile E2E Test Report

**Date:** YYYY-MM-DD HH:MM
**Duration:** Xm Ys
**Status:** PASSING / FAILING

## Setup
- Platform: Android 15 / iOS 18
- Device: Pixel 8 (serial 1A2B3C) | physical
- Framework: Espresso (reason: native Compose app, in-app flow)
- Build: app/build/outputs/apk/debug/app-debug.apk (debug)

## Summary
- Total: X | Passed: Y (Z%) | Failed: A | Flaky: B | Skipped: C

## Failed Tests

### LoginTest.loginWithValidCredentials
**File:** `app/src/androidTest/java/com/example/LoginTest.kt:42`
**Error:** NoMatchingViewException: id/login_button
**Screenshot:** e2e-artifacts/screenshots/login-failed.png
**Log excerpt:** e2e-artifacts/logs/logcat.txt (line 812, FATAL EXCEPTION)
**Recommended Fix:** [description]

## Artifacts
- Framework report: e2e-artifacts/<framework>/
- Screenshots: e2e-artifacts/screenshots/
- Videos: e2e-artifacts/videos/
- Logs: e2e-artifacts/logs/
```

## Shared Patterns (all frameworks)

### Screen Object Model

This is the mobile version of the Page Object Model: one class (or Maestro subflow) per screen. It
exposes user actions (`login(user, pass)`) and queries (`isErrorShown()`). Tests never touch raw
selectors. Each framework reference has a concrete example.

### Selector Priority

1. **Stable test IDs:** Android `resource-id` / Compose `testTag`, iOS `accessibilityIdentifier`,
   Unity AltTester object name/`AltId`.
2. Accessibility label / `content-desc`. Fine to use, but it changes with copy and localization.
3. Visible text. Only for asserting copy.
4. XPath or coordinates. Last resort; flag it in the report as fragile.

If a needed element has no stable ID, **propose** the exact one-line change to the app code
(e.g. `Modifier.testTag("login_button")`, `.accessibilityIdentifier("loginButton")`) and wait
for the user's OK before editing app code.

Compose + Appium/Maestro: `testTag` shows up as `resource-id` only when you set
`Modifier.semantics { testTagsAsResourceId = true }` on a root composable.

### Test Isolation

- Each test starts from a known state: Android Test Orchestrator + `clearPackageData`,
  XCUITest `launchArguments` to reset state, Maestro `launchApp: { clearState: true }`,
  Appium `appium:noReset: false`.
- Don't let tests depend on each other's order or on leftover data.
- Treat test data as immutable: build fresh fixtures per test instead of mutating shared objects.

### Waiting

Never use fixed sleeps (`Thread.sleep`, `sleep()`, `browser.pause`). Use the framework's sync:
Espresso IdlingResources / `composeRule.waitUntil`, XCUITest `waitForExistence(timeout:)`,
Maestro `extendedWaitUntil`, WDIO `waitForDisplayed`, AltTester `wait_for_object`.

## Flaky Tests

### Quarantine

| Framework | How |
|-----------|-----|
| Espresso | `@Ignore("Flaky - Issue #123")` or `@FlakyTest` + exclude via `-Pandroid.testInstrumentationRunnerArguments.notAnnotation=androidx.test.filters.FlakyTest` |
| XCUITest | `try XCTSkipIf(true, "Flaky - Issue #123")`, or disable in the test plan |
| Maestro | `tags: [flaky]` in the flow header, run with `--exclude-tags flaky` |
| Appium/WDIO | `it.skip('... (Flaky - Issue #123)', ...)` |
| AltTester/pytest | `@pytest.mark.skip(reason="Flaky - Issue #123")` |

### Identify flakiness (run 10x)

```bash
for i in $(seq 1 10); do ANDROID_SERIAL=<serial> ./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.example.LoginTest || echo "FAIL $i"; done
xcodebuild test-without-building ... -only-testing:AppUITests/LoginTests -test-iterations 10 -run-tests-until-failure
for i in $(seq 1 10); do maestro --device <serial> test .maestro/login.yaml || echo "FAIL $i"; done
```

### Common Causes & Fixes

| Cause | Symptom | Fix |
|-------|---------|-----|
| Animations | Element found but click misses | Disable animations (Step 6); Maestro `waitForAnimationToEnd` |
| Background work not tracked | Assert runs before data loads | Espresso IdlingResource; wait for a loaded-state element, not a timer |
| Soft keyboard covers button | Tap lands on keyboard | Hide the keyboard (`closeSoftKeyboard()`, Maestro `hideKeyboard`, `driver.hideKeyboard()`) |
| Runtime permission dialog | Test stuck on first run | Grant up front: `GrantPermissionRule`, `adb shell pm grant`, Appium `autoGrantPermissions`, XCUITest `addUIInterruptionMonitor` |
| Network variance | Random timeouts | Point the debug build at a mock/staging server; avoid live third-party calls |
| Leftover state | Passes alone, fails in suite | Clear state per test (see Isolation) |
| Device-specific layout | Fails on small screens only | Scroll to the element (`scrollTo()`, `scrollUntilVisible`) instead of assuming it's visible |

## CI/CD

See `references/ci.md`: GitHub Actions with an Android emulator / iOS Simulator, Firebase Test Lab,
BrowserStack / Sauce Labs, and Maestro Cloud.

## Reference Index

| File | Load when |
|------|-----------|
| `references/espresso.md` | Android native in-app tests (Views or Compose) |
| `references/xcuitest.md` | iOS native in-app tests (UIKit or SwiftUI) |
| `references/maestro.md` | Black-box flows, system UI, quick smoke, Android or iOS Simulator |
| `references/appium.md` | Cross-platform suite, WebViews, device farms (WebdriverIO + TypeScript) |
| `references/unity-alttester.md` | Unity games (AltTester + Appium, Python) |
| `references/ci.md` | Setting up CI or cloud device runs |
