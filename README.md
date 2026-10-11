# micorlov-e2e-mobile

A Claude Code skill and agent for mobile E2E/UI testing on Android and iOS (native Kotlin/Java/Compose,
Swift/SwiftUI, and Unity games). It is the mobile counterpart of ECC's Playwright `e2e-testing` skill.

It detects the project, picks the framework itself, runs on a **real device first**, builds and installs a
dev build, runs the tests, and writes `e2e-artifacts/REPORT.md`.

| Situation | Framework |
|-----------|-----------|
| Suite already in the repo | That one |
| Unity game | AltTester + Appium (Python) |
| System UI, permissions, quick smoke | Maestro |
| One suite for Android + iOS, WebViews, device farms | Appium + WebdriverIO (TypeScript) |
| Native Android, in-app flows | Espresso / Compose test |
| Native iOS, in-app flows | XCUITest |

CI recipes: GitHub Actions, Firebase Test Lab, BrowserStack / Sauce Labs, Maestro Cloud.

## Install

```bash
git clone git@github.com:Micorlov/micorlov-e2e-mobile.git
cp -R micorlov-e2e-mobile/skills/micorlov-e2e-mobile ~/.claude/skills/
cp micorlov-e2e-mobile/agents/mobile-e2e-runner.md ~/.claude/agents/
```

Restart Claude Code, then ask for example: "test the login flow on my phone".

## Layout

```text
skills/micorlov-e2e-mobile/
├── SKILL.md              detection, framework choice, device selection, run workflow, report
└── references/           espresso, xcuitest, maestro, appium, unity-alttester, ci
agents/mobile-e2e-runner.md   subagent that runs the skill end to end
```

## Safety defaults

- Real physical device first; never silently falls back to an emulator.
- Development builds only.
- Asks before installing tools, uninstalling an app (wipes its data), or adding test IDs to app code.
- Never targets production backends or real payments.
