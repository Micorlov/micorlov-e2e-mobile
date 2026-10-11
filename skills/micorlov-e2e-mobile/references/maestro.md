# Maestro (Android + iOS Simulator)

Black-box YAML flows run against an installed build. Fastest to write, waits for the UI to settle
automatically, and can drive system UI (permission dialogs, notifications, settings).

## File Organization

```text
.maestro/
├── config.yaml
├── flows/
│   ├── login.yaml
│   ├── search.yaml
│   └── smoke_launch.yaml
├── subflows/                Screen Objects as reusable subflows
│   ├── do_login.yaml
│   └── open_search.yaml
└── scripts/
    └── test_data.js         read-only test data (env vars or JS)
```

## `config.yaml`

```yaml
flows:
  - flows/*
includeTags: []
excludeTags:
  - flaky
executionOrder:
  continueOnFailure: true
  flowsOrder:
    - smoke_launch
```

## Subflow (Screen Object)

`subflows/do_login.yaml`:

```yaml
appId: ${APP_ID}
---
- tapOn:
    id: "email_input"
- inputText: ${EMAIL}
- tapOn:
    id: "password_input"
- inputText: ${PASSWORD}
- hideKeyboard
- tapOn:
    id: "login_button"
```

## Flow

`flows/search.yaml`:

```yaml
appId: com.example.app
name: Search by keyword
tags:
  - search
env:
  EMAIL: qa+maestro@example.com
---
- launchApp:
    clearState: true
    permissions:
      notifications: allow
- runFlow:
    file: ../subflows/do_login.yaml
    env:
      APP_ID: com.example.app
      PASSWORD: ${TEST_PASSWORD}       # pass with -e TEST_PASSWORD=..., never commit it
- tapOn:
    id: "search_input"
- inputText: "test"
- pressKey: Enter
- extendedWaitUntil:
    visible:
      id: "result_card"
    timeout: 10000
- assertVisible:
    id: "result_card"
- takeScreenshot: e2e-artifacts/screenshots/search-results
```

Empty-state case in the same file style:

```yaml
- inputText: "xyznonexistent123"
- pressKey: Enter
- assertVisible:
    id: "no_results"
- assertNotVisible:
    id: "result_card"
```

## Selectors

- `id:` matches Android `resource-id` and iOS `accessibilityIdentifier`.
  Compose `testTag` needs `testTagsAsResourceId = true` (see SKILL.md).
- `text:` is a regex on visible text, so escape special characters.
- `index:`, `below:`, `childOf:` disambiguate repeated elements.
- `point: "50%,90%"` is a last resort.

Inspect the hierarchy: `maestro --device <serial> hierarchy`, or interactively with `maestro studio`.

## Useful Commands

| Need | YAML |
|------|------|
| Wait for animations | `- waitForAnimationToEnd` |
| Scroll to element | `- scrollUntilVisible: { element: { id: "footer" }, direction: DOWN }` |
| Conditional dialog | `- runFlow: { when: { visible: "Allow" }, commands: [ { tapOn: "Allow" } ] }` |
| Retry a block | `- retry: { maxRetries: 2, commands: [ ... ] }` |
| Deep link | `- openLink: myapp://search?q=test` |
| Back button | `- back` |
| JS logic | `- runScript: scripts/test_data.js` then use `${output.someValue}` |
| Stop the app | `- stopApp` |

## Run

```bash
# Android real device
maestro --device <serial> test .maestro/flows/search.yaml

# whole suite + JUnit report + per-flow artifacts
maestro --device <serial> test .maestro/ \
  --format junit --output e2e-artifacts/maestro/report.xml \
  --test-output-dir e2e-artifacts/maestro

# tags and secrets
maestro --device <serial> test .maestro/ --include-tags search --exclude-tags flaky \
  -e TEST_PASSWORD="$TEST_PASSWORD"

# iOS Simulator
maestro --device <SIM-UDID> test .maestro/
```

Without `--device`, Maestro picks a device itself and may pick an emulator. Always pass it.

**iOS:** Maestro drives iOS **Simulators**. For a physical iPhone, use XCUITest or Appium.
**Install the build first:** `adb -s <serial> install -r ...` or `xcrun simctl install <UDID> App.app`.
Maestro doesn't build the app.

## Artifacts

- `--test-output-dir` holds screenshots, logs and command metadata for each flow.
- `takeScreenshot: <path>` saves `<path>.png`.
- Screen recording: `- startRecording: e2e-artifacts/videos/search` and `- stopRecording`.

## Maestro-Specific Flakiness

| Problem | Fix |
|---------|-----|
| Text match fails on dynamic copy | Use `id:` or a regex: `text: "Results \\(\\d+\\)"` |
| Taps on the wrong one of several matches | Add `index:` or a relative selector (`below:`) |
| Permission dialog appears sometimes | `launchApp.permissions` or a conditional `runFlow` |
| Keyboard covers button | `- hideKeyboard` before tapping |
| Slow backend | `extendedWaitUntil` with a timeout, not `- wait` |
| Leftover login state | `launchApp: { clearState: true }` |
