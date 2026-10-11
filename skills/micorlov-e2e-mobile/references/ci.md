# CI/CD and Cloud Devices

All secrets (API keys, service accounts, test passwords) come from CI secrets. Never put them in
workflow files or test code. Always upload `e2e-artifacts/` with `if: always()`.

## GitHub Actions: Android Emulator (Espresso / Maestro / Appium)

```yaml
# .github/workflows/android-e2e.yml
name: Android E2E
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4

      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - name: Build
        run: ./gradlew assembleDebug assembleDebugAndroidTest

      # Espresso
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          arch: x86_64
          target: google_apis
          disable-animations: true
          script: ./gradlew connectedDebugAndroidTest

      # Maestro (alternative: replace the step above)
      # - run: curl -Ls "https://get.maestro.mobile.dev" | bash
      # - uses: reactivecircus/android-emulator-runner@v2
      #   with:
      #     api-level: 34
      #     arch: x86_64
      #     script: |
      #       adb install -r app/build/outputs/apk/debug/app-debug.apk
      #       $HOME/.maestro/bin/maestro test .maestro/ --format junit --output e2e-artifacts/maestro/report.xml

      - name: Collect reports
        if: always()
        run: |
          mkdir -p e2e-artifacts/espresso
          cp -r app/build/reports/androidTests e2e-artifacts/espresso/ 2>/dev/null || true
          cp -r app/build/outputs/androidTest-results e2e-artifacts/espresso/ 2>/dev/null || true
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: android-e2e-artifacts
          path: e2e-artifacts/
          retention-days: 30
```

## GitHub Actions: iOS Simulator (XCUITest / Maestro / Appium)

```yaml
  ios-e2e:
    runs-on: macos-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
      - name: Pick simulator
        id: sim
        run: |
          UDID=$(xcrun simctl list devices available -j | python3 -c \
            "import json,sys;d=json.load(sys.stdin)['devices'];print(next(x['udid'] for k,v in d.items() if 'iOS' in k for x in v if x['name'].startswith('iPhone')))")
          echo "udid=$UDID" >> "$GITHUB_OUTPUT"
      - name: XCUITest
        run: |
          xcodebuild test -scheme App \
            -destination "id=${{ steps.sim.outputs.udid }}" \
            -resultBundlePath e2e-artifacts/ios/Result.xcresult
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: ios-e2e-artifacts
          path: e2e-artifacts/
```

Simulator names change between runner images, which is why the job picks one by UDID instead of hard-coding a name.

## Firebase Test Lab (real devices in Google's cloud)

```bash
# Espresso
gcloud firebase test android run --type instrumentation \
  --app app/build/outputs/apk/debug/app-debug.apk \
  --test app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk \
  --device model=<model>,version=<api> \
  --use-orchestrator --environment-variables clearPackageData=true \
  --num-flaky-test-attempts 1 \
  --results-dir "e2e-${GITHUB_RUN_ID:-local}"

gcloud firebase test android models list      # pick <model>/<api>

# XCUITest (build for real devices, zip products + .xctestrun)
xcodebuild build-for-testing -scheme App -sdk iphoneos -derivedDataPath build
(cd build/Build/Products && zip -r ../../../ios-tests.zip Debug-iphoneos ./*.xctestrun)
gcloud firebase test ios run --test ios-tests.zip --device model=<model>,version=<ios>
gcloud firebase test ios models list
```

In GitHub Actions, authenticate with `google-github-actions/auth@v2` (Workload Identity
Federation preferred over JSON keys) and `google-github-actions/setup-gcloud@v2`.

## BrowserStack

**Appium (WebdriverIO):** add `@wdio/browserstack-service`:

```typescript
// wdio.browserstack.conf.ts
import { config as base } from './wdio.conf'

export const config: WebdriverIO.Config = {
  ...base,
  user: process.env.BROWSERSTACK_USERNAME,
  key: process.env.BROWSERSTACK_ACCESS_KEY,
  services: [['browserstack', { app: process.env.APP_PATH }]],
  capabilities: [{
    platformName: 'Android',
    'appium:automationName': 'UiAutomator2',
    'bstack:options': { deviceName: 'Google Pixel 8', platformVersion: '14.0', video: true },
  }],
}
```

**Espresso:** upload the app and test APKs, then start a build:

```bash
AUTH="$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY"
APP=$(curl -s -u "$AUTH" -X POST https://api-cloud.browserstack.com/app-automate/espresso/v2/app \
  -F "file=@app/build/outputs/apk/debug/app-debug.apk" | jq -r .app_url)
SUITE=$(curl -s -u "$AUTH" -X POST https://api-cloud.browserstack.com/app-automate/espresso/v2/test-suite \
  -F "file=@app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk" | jq -r .test_suite_url)
curl -s -u "$AUTH" -X POST https://api-cloud.browserstack.com/app-automate/espresso/v2/build \
  -H "Content-Type: application/json" \
  -d "{\"app\":\"$APP\",\"testSuite\":\"$SUITE\",\"devices\":[\"Google Pixel 8-14.0\"]}"
```

BrowserStack also runs XCUITest and Maestro suites through similar App Automate endpoints.
Check their current docs for exact paths.

## Sauce Labs

**Appium (WebdriverIO):** `@wdio/sauce-service`:

```typescript
// wdio.sauce.conf.ts
import { config as base } from './wdio.conf'

export const config: WebdriverIO.Config = {
  ...base,
  user: process.env.SAUCE_USERNAME,
  key: process.env.SAUCE_ACCESS_KEY,
  region: 'us',
  services: ['sauce'],
  capabilities: [{
    platformName: 'Android',
    'appium:automationName': 'UiAutomator2',
    'appium:deviceName': 'Google Pixel.*',          // regex = any matching real device
    'appium:app': 'storage:filename=app-debug.apk',
    'sauce:options': { appiumVersion: 'latest', build: process.env.GITHUB_RUN_ID },
  }],
}
```

**Espresso / XCUITest:** use `saucectl` with `.sauce/config.yml` (`kind: espresso` or `kind: xcuitest`):

```bash
saucectl storage upload app/build/outputs/apk/debug/app-debug.apk
saucectl run
```

## Maestro Cloud

```bash
maestro cloud --api-key "$MAESTRO_CLOUD_API_KEY" --project-id "$MAESTRO_PROJECT_ID" \
  app/build/outputs/apk/debug/app-debug.apk .maestro/
```

GitHub Actions:

```yaml
      - uses: mobile-dev-inc/action-maestro-cloud@v1
        with:
          api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
          project-id: ${{ secrets.MAESTRO_PROJECT_ID }}
          app-file: app/build/outputs/apk/debug/app-debug.apk
          workspace: .maestro
```

## Unity in CI

Build the instrumented dev build with `game-ci/unity-builder` (needs a Unity license secret) using
`buildMethod: BuildForTests.Android`. Then run the AltTester suite on an emulator with the
AltTester Server reachable from the job, and upload `e2e-artifacts/`.

## Choosing Where to Run

| Need | Use |
|------|-----|
| Every PR, fast feedback | GitHub Actions emulator/simulator |
| Real-device coverage, Espresso/XCUITest | Firebase Test Lab |
| Real-device Appium across many models | BrowserStack / Sauce Labs |
| Maestro flows without managing devices | Maestro Cloud |
