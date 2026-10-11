# XCUITest (iOS native)

Apple's UI testing framework. Tests run in a separate runner process and drive the app through
accessibility. Use it for in-app flows of UIKit and SwiftUI apps, and for any test on a physical iPhone.

## File Organization

```text
AppUITests/
├── Screens/                 Screen Objects
│   ├── LoginScreen.swift
│   └── SearchScreen.swift
├── Flows/
│   ├── LoginTests.swift
│   └── SearchTests.swift
├── Support/
│   ├── XCUIApplication+Launch.swift
│   └── TestAccounts.swift
└── AppUITests.xctestplan
```

## Stable Identifiers

```swift
// SwiftUI
TextField("Email", text: $email).accessibilityIdentifier("login.email")
Button("Log in", action: login).accessibilityIdentifier("login.submit")

// UIKit
loginButton.accessibilityIdentifier = "login.submit"
```

Use `screen.element` naming (`login.submit`, `search.result`) so identifiers stay unique across screens.

## Launch Helper (known state on every launch)

```swift
extension XCUIApplication {
    static func launchForTesting(resetState: Bool = true) -> XCUIApplication {
        let app = XCUIApplication()
        app.launchArguments += ["-uiTesting"]
        if resetState { app.launchArguments += ["-resetState"] }
        app.launchEnvironment["API_BASE_URL"] = "https://staging.example.com"
        app.launch()
        return app
    }
}
```

The app checks `ProcessInfo.processInfo.arguments.contains("-resetState")` in debug builds to
clear UserDefaults/Keychain and disable animations (`UIView.setAnimationsEnabled(false)`).
Propose this hook to the user if it's missing; don't add it without asking.

## Screen Object

```swift
struct SearchScreen {
    let app: XCUIApplication

    private var field: XCUIElement { app.textFields["search.input"] }
    private var results: XCUIElementQuery { app.cells.matching(identifier: "search.result") }
    private var emptyState: XCUIElement { app.staticTexts["search.empty"] }

    @discardableResult
    func search(_ query: String) -> Self {
        XCTAssertTrue(field.waitForExistence(timeout: 5))
        field.tap()
        field.typeText(query + "\n")
        return self
    }

    func resultCount(timeout: TimeInterval = 5) -> Int {
        _ = results.firstMatch.waitForExistence(timeout: timeout)
        return results.count
    }

    func assertEmptyState(file: StaticString = #filePath, line: UInt = #line) {
        XCTAssertTrue(emptyState.waitForExistence(timeout: 5), file: file, line: line)
    }
}
```

## Test Structure

```swift
final class SearchTests: XCTestCase {
    private var app: XCUIApplication!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = .launchForTesting()
    }

    override func tearDownWithError() throws {
        if let run = testRun, run.failureCount > 0 {
            let shot = XCTAttachment(screenshot: app.screenshot())
            shot.name = "failure-\(name)"
            shot.lifetime = .keepAlways
            add(shot)
        }
    }

    func testSearchByKeywordShowsResults() {
        let screen = SearchScreen(app: app).search("test")
        XCTAssertGreaterThan(screen.resultCount(), 0)
    }

    func testSearchWithNoMatchesShowsEmptyState() {
        SearchScreen(app: app).search("xyznonexistent123").assertEmptyState()
    }
}
```

## System Alerts (permissions)

```swift
addUIInterruptionMonitor(withDescription: "System alert") { alert in
    for label in ["Allow", "Allow While Using App", "OK"] where alert.buttons[label].exists {
        alert.buttons[label].tap(); return true
    }
    return false
}
app.tap()   // the monitor fires on the next interaction
```

The monitor's checks are on English button labels; use the device's language or reset permissions on
the Simulator instead: `xcrun simctl privacy <UDID> grant location <bundle-id>`.

## Run

```bash
# physical device or simulator by UDID
xcodebuild build-for-testing -scheme App -destination 'id=<UDID>' -derivedDataPath build/DerivedData
xcodebuild test-without-building -scheme App -destination 'id=<UDID>' \
  -derivedDataPath build/DerivedData \
  -resultBundlePath e2e-artifacts/ios/Result.xcresult

# single test
xcodebuild test-without-building ... -only-testing:AppUITests/SearchTests/testSearchByKeywordShowsResults

# retry and repeat
xcodebuild test-without-building ... -retry-tests-on-failure -test-iterations 3
xcodebuild test-without-building ... -test-iterations 10 -run-tests-until-failure
```

Physical devices need a development team and a provisioning profile for both the app and the
UITests runner target (`DEVELOPMENT_TEAM=<id>` and automatic signing). Delete an existing
`Result.xcresult` before re-running; xcodebuild refuses to overwrite it.

## Artifacts

```bash
xcrun xcresulttool get test-results summary --path e2e-artifacts/ios/Result.xcresult   # Xcode 16+
xcrun xcresulttool export attachments --path e2e-artifacts/ios/Result.xcresult \
  --output-path e2e-artifacts/screenshots/                                            # Xcode 16+
open e2e-artifacts/ios/Result.xcresult        # full report in Xcode
```

For JUnit XML in CI, convert the bundle with `xcresultparser` or `xcbeautify --report junit`.

## XCUITest-Specific Flakiness

| Problem | Fix |
|---------|-----|
| Element exists but `tap()` fails | Check `isHittable`; scroll with `swipeUp()` until `isHittable` |
| `typeText` drops characters | Tap the field first and wait for `hasKeyboardFocus` (via `value(forKey:)`) |
| Wrong element matched | Query by identifier, not label; avoid `firstMatch` on broad queries |
| Animations | `-resetState` hook disables them; or wait for the destination element |
| Keyboard covers button | `app.keyboards.buttons["Return"].tap()` or tap outside the field |
| Slow first launch on device | `waitForExistence(timeout: 15)` only on the first screen |
