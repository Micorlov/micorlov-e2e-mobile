# Appium + WebdriverIO (TypeScript), Android + iOS

One suite for both platforms, WebView/hybrid screens, and device farms. It's out-of-process
and over HTTP, so it's slower and more flake-prone than Espresso/XCUITest. Use it when you need
that portability.

## File Organization

```text
tests/e2e/
├── screens/                 Screen Objects (cross-platform)
│   ├── BaseScreen.ts
│   ├── LoginScreen.ts
│   └── SearchScreen.ts
├── specs/
│   ├── login.spec.ts
│   └── search.spec.ts
├── fixtures/
│   └── accounts.ts          read-only test data
wdio.conf.ts
tsconfig.json
```

## Install (ask first)

```bash
npm i -D @wdio/cli @wdio/local-runner @wdio/mocha-framework @wdio/spec-reporter \
  @wdio/junit-reporter @wdio/appium-service appium typescript
npx appium driver install uiautomator2      # Android
npx appium driver install xcuitest          # iOS (macOS only)
npx appium driver doctor uiautomator2       # checks ANDROID_HOME, JDK, adb
```

Or scaffold interactively with `npm init wdio@latest .` (choose "Mobile - native app").

## `wdio.conf.ts`

```typescript
const platform = process.env.PLATFORM ?? 'android'

const androidCaps = {
  platformName: 'Android',
  'appium:automationName': 'UiAutomator2',
  'appium:udid': process.env.ANDROID_SERIAL,          // real device serial, never emulator by default
  'appium:app': process.env.APP_PATH ?? './app/build/outputs/apk/debug/app-debug.apk',
  'appium:autoGrantPermissions': true,
  'appium:noReset': false,
  'appium:newCommandTimeout': 120,
}

const iosCaps = {
  platformName: 'iOS',
  'appium:automationName': 'XCUITest',
  'appium:udid': process.env.IOS_UDID,
  'appium:app': process.env.APP_PATH,                 // .app (simulator) or .ipa (device)
  'appium:xcodeOrgId': process.env.TEAM_ID,           // physical device only
  'appium:xcodeSigningId': 'Apple Development',
  'appium:autoAcceptAlerts': true,
  'appium:noReset': false,
}

export const config: WebdriverIO.Config = {
  runner: 'local',
  specs: ['./tests/e2e/specs/**/*.spec.ts'],
  maxInstances: 1,
  capabilities: [platform === 'ios' ? iosCaps : androidCaps],
  services: [['appium', { logPath: './e2e-artifacts/appium/' }]],
  framework: 'mocha',
  mochaOpts: { timeout: 120_000 },
  reporters: ['spec', ['junit', { outputDir: './e2e-artifacts/appium/' }]],
  waitforTimeout: 10_000,
  connectionRetryCount: 2,

  afterTest: async (test, _context, { passed }) => {
    if (passed) return
    const name = test.title.replace(/\W+/g, '_')
    await browser.saveScreenshot(`./e2e-artifacts/screenshots/${name}.png`)
  },
}
```

## Cross-Platform Selectors

```typescript
// tests/e2e/screens/BaseScreen.ts
export const byTestId = (id: string): string =>
  driver.isAndroid ? `id=${id}` : `~${id}`
```

- Android `id=login_button` matches `resource-id` (UiAutomator2 adds the app package if you leave it
  out; use `id=com.example.app:id/login_button` if it doesn't). For Compose, set `testTagsAsResourceId = true`.
- iOS `~login.submit` matches `accessibilityIdentifier`.
- If both platforms use the same IDs, `~id` with Android `contentDescription` also works, but
  it pollutes TalkBack. Prefer resource-id on Android.
- Avoid XPath because it's slow on both platforms. Use `-ios predicate string:` / `-ios class chain:` or
  `android=new UiSelector()...` when IDs are missing.

## Screen Object

```typescript
import { byTestId } from './BaseScreen'

export class SearchScreen {
  private get input() { return $(byTestId('search_input')) }
  private get results() { return $$(byTestId('result_card')) }
  private get emptyState() { return $(byTestId('no_results')) }

  async search(query: string): Promise<void> {
    await this.input.waitForDisplayed()
    await this.input.setValue(query)
    if (await driver.isKeyboardShown()) await driver.hideKeyboard()
    await $(byTestId('search_submit')).click()
  }

  async resultCount(): Promise<number> {
    await $(byTestId('result_card')).waitForDisplayed({ timeout: 10_000 }).catch(() => undefined)
    return (await this.results).length
  }

  async isEmptyStateShown(): Promise<boolean> {
    return this.emptyState.waitForDisplayed({ timeout: 10_000 }).then(() => true, () => false)
  }
}
```

## Test Structure

```typescript
import { expect } from '@wdio/globals'
import { SearchScreen } from '../screens/SearchScreen'

describe('Search', () => {
  const screen = new SearchScreen()

  it('shows results for a keyword', async () => {
    await screen.search('test')
    expect(await screen.resultCount()).toBeGreaterThan(0)
  })

  it('shows empty state for no matches', async () => {
    await screen.search('xyznonexistent123')
    expect(await screen.isEmptyStateShown()).toBe(true)
  })
})
```

## WebViews / Hybrid

```typescript
await driver.waitUntil(async () => (await driver.getContexts()).length > 1)
const contexts = (await driver.getContexts()).map(String)
const webview = contexts.find(c => c.startsWith('WEBVIEW'))
if (!webview) throw new Error(`No WebView context. Found: ${contexts.join(', ')}`)
await driver.switchContext(webview)
await $('[data-testid="checkout"]').click()      // CSS selectors work inside the WebView
await driver.switchContext('NATIVE_APP')
```

The debug build must call `WebView.setWebContentsDebuggingEnabled(true)` (Android) or set
`isInspectable = true` (iOS 16.4+). Android also needs a matching Chromedriver. Start Appium
with `--allow-insecure=uiautomator2:chromedriver_autodownload`.

## Gestures

| Need | Android | iOS |
|------|---------|-----|
| Scroll | `driver.execute('mobile: scrollGesture', { left: 100, top: 300, width: 800, height: 1200, direction: 'down', percent: 0.8 })` | `driver.execute('mobile: scroll', { direction: 'down' })` |
| Swipe | `mobile: swipeGesture` | `mobile: swipe` |
| Long press | `mobile: longClickGesture` | `mobile: touchAndHold` |
| Back | `driver.back()` | tap the nav bar back button |
| Deep link | `driver.execute('mobile: deepLink', { url, package })` | `driver.execute('mobile: deepLink', { url, bundleId })` |

## Run

```bash
ANDROID_SERIAL=<serial> npx wdio run wdio.conf.ts
ANDROID_SERIAL=<serial> npx wdio run wdio.conf.ts --spec tests/e2e/specs/search.spec.ts
PLATFORM=ios IOS_UDID=<UDID> APP_PATH=build/App.app npx wdio run wdio.conf.ts
```

Use Appium Inspector (desktop app) against the running session to explore the element tree.

## Appium-Specific Flakiness

| Problem | Fix |
|---------|-----|
| `StaleElementReferenceError` | Re-query through Screen Object getters; don't cache elements |
| Session start timeouts on a real device | Raise `appium:uiautomator2ServerLaunchTimeout` / `appium:wdaLaunchTimeout`; reinstall drivers |
| iOS real device: WDA fails to build | Check signing (`xcodeOrgId`, `xcodeSigningId`, `updatedWDABundleId`) |
| Taps miss after navigation | `waitForDisplayed` on the next screen's anchor element first |
| Keyboard covers elements | `driver.hideKeyboard()` |
| Slow XPath | Replace with id / accessibility id / predicate |
