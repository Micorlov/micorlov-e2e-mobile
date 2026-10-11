# Unity Games: AltTester + Appium (Python)

Unity renders to one native surface, so Espresso/XCUITest/Appium can't see buttons, sprites or
UI Toolkit elements. **AltTester** puts an SDK in a dev build and exposes the GameObject tree to
a test driver. **Appium** covers the native parts around the game: install/launch, OS permission
dialogs, backgrounding, and system UI.

Language: **Python + pytest**. AltTester has drivers for C#, Python, Java and Robot Framework but
not TypeScript, and the Python Appium client sits next to it cleanly.

## How It Connects

```text
pytest ── AltDriver ──► AltTester Server (AltTester Desktop, host:13000) ◄── instrumented game on device
       └─ Appium ─────► Appium server (host:4723) ──► UiAutomator2 / XCUITest ──► device
```

- Android over USB: forward the port so the game reaches the host:
  `adb -s <serial> reverse tcp:13000 tcp:13000`
- iOS device: set the AltTester Server host to the Mac's LAN IP in the instrumentation settings;
  phone and Mac on the same network.
- Start **AltTester Desktop** (it runs the AltTester Server) before running tests.

## File Organization

```text
tests/unity/
├── conftest.py              Appium + AltDriver fixtures
├── screens/                 Screen Objects per Unity scene/menu
│   ├── main_menu.py
│   └── gameplay.py
├── test_main_menu.py
├── test_gameplay.py
└── requirements.txt         AltTester-Driver, Appium-Python-Client, pytest
Assets/Editor/BuildForTests.cs   instrumented dev build (project's own build script)
```

## Instrumented Dev Build

Prefer the project's existing build script (check README, `Assets/Editor/`, `scripts/`, CI).
If none exists, propose one like this and wait for the user's OK. The AltTester `AltBuilder` API
differs between SDK versions, so check the installed SDK's docs.

```csharp
// Assets/Editor/BuildForTests.cs
using AltTester.AltTesterUnitySDK.Editor;
using UnityEditor;

public static class BuildForTests
{
    public static void Android()
    {
        var scenes = EditorBuildSettingsScene.GetActiveSceneList(EditorBuildSettings.scenes);
        AltBuilder.AddAltTesterInScriptingDefineSymbolsGroup(BuildTargetGroup.Android);
        AltBuilder.InsertAltInScene(scenes[0], new AltInstrumentationSettings());
        try
        {
            BuildPipeline.BuildPlayer(new BuildPlayerOptions
            {
                scenes = scenes,
                locationPathName = "Builds/Android/game-alttester-dev.apk",
                target = BuildTarget.Android,
                options = BuildOptions.Development,
            });
        }
        finally
        {
            AltBuilder.RemoveAltTesterFromScriptingDefineSymbols(BuildTargetGroup.Android);
        }
    }
}
```

```bash
"$UNITY_PATH" -batchmode -quit -projectPath . \
  -executeMethod BuildForTests.Android -logFile e2e-artifacts/logs/unity-build.log
adb -s <serial> install -r Builds/Android/game-alttester-dev.apk
```

**Never ship an AltTester build.** It opens a remote-control socket into the game. Keep it to
`Development` builds with a distinct file name, and make sure release builds strip the define symbol.

## `conftest.py`

```python
import os
import pytest
from alttester import AltDriver
from appium import webdriver
from appium.options.android import UiAutomator2Options

APPIUM_URL = os.environ.get("APPIUM_URL", "http://127.0.0.1:4723")
ALT_HOST = os.environ.get("ALT_HOST", "127.0.0.1")
ALT_PORT = int(os.environ.get("ALT_PORT", "13000"))


@pytest.fixture(scope="session")
def appium_driver():
    options = UiAutomator2Options()
    options.udid = os.environ["ANDROID_SERIAL"]                 # real device serial
    options.app_package = os.environ.get("APP_PACKAGE", "com.example.game")
    options.app_activity = os.environ.get("APP_ACTIVITY", "com.unity3d.player.UnityPlayerActivity")
    options.no_reset = True
    options.auto_grant_permissions = True
    driver = webdriver.Remote(APPIUM_URL, options=options)
    yield driver
    driver.quit()


@pytest.fixture(scope="session")
def alt_driver(appium_driver):
    alt = AltDriver(host=ALT_HOST, port=ALT_PORT, timeout=60)
    yield alt
    alt.stop()


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    report = outcome.get_result()
    driver = item.funcargs.get("appium_driver")
    if report.when == "call" and report.failed and driver:
        os.makedirs("e2e-artifacts/screenshots", exist_ok=True)
        driver.save_screenshot(f"e2e-artifacts/screenshots/{item.name}.png")
```

Unity 2023.1+ may use `com.unity3d.player.UnityPlayerGameActivity`. Check `AndroidManifest.xml`
in the exported project, or run `adb shell cmd package resolve-activity --brief <package>`.

## Screen Object (scene)

```python
# tests/unity/screens/main_menu.py
from alttester import By


class MainMenu:
    SCENE = "MainMenu"

    def __init__(self, alt):
        self.alt = alt

    def wait_loaded(self):
        self.alt.wait_for_current_scene_to_be(self.SCENE, timeout=30)
        self.alt.wait_for_object(By.NAME, "PlayButton", timeout=10)
        return self

    def start_game(self):
        self.alt.find_object(By.NAME, "PlayButton").tap()
        self.alt.wait_for_current_scene_to_be("Gameplay", timeout=30)

    def coins_text(self) -> str:
        return self.alt.find_object(By.PATH, "//Canvas/TopBar/CoinsLabel").get_text()
```

## Test Structure

```python
# tests/unity/test_main_menu.py
from screens.main_menu import MainMenu


def test_start_game_loads_gameplay(alt_driver):
    menu = MainMenu(alt_driver).wait_loaded()
    menu.start_game()
    assert alt_driver.get_current_scene() == "Gameplay"


def test_app_survives_background_and_resume(appium_driver, alt_driver):
    MainMenu(alt_driver).wait_loaded()
    appium_driver.background_app(3)              # native: send app to background for 3 s
    MainMenu(alt_driver).wait_loaded()           # game resumed on the same scene
```

## Selectors

| Strategy | Use for |
|----------|---------|
| `By.NAME` | GameObject names. Ask devs to keep UI names unique and stable |
| `By.PATH` | `//Canvas/Panel/Button` when names repeat |
| `By.ID` | AltId component values (most stable; add AltId to key objects) |
| `By.TEXT` | Visible text on Text/TMP components (avoid for localized games) |
| `By.TAG` / `By.COMPONENT` | Grouped lookups (`find_objects`) |

If key objects lack stable names/AltIds, propose the change and wait for the user's OK.

## Run

```bash
adb -s <serial> reverse tcp:13000 tcp:13000
npx appium &                                                  # or a running Appium server
ANDROID_SERIAL=<serial> pytest tests/unity -v \
  --junitxml=e2e-artifacts/unity/report.xml
```

Quick smoke without AltTester (launch, no crash): a Maestro flow (see `maestro.md`) with
`launchApp`, `extendedWaitUntil` on the first native-visible element, and
`assertNotVisible: ".*isn't responding.*"`.

## Unity-Specific Flakiness

| Problem | Fix |
|---------|-----|
| `AltDriver` can't connect | AltTester Desktop not running, or missing `adb reverse` |
| Object found but tap does nothing | It's off-screen or under an overlay; wait for the panel's open animation (`wait_for_object` on an end-state object) |
| Scene load timing | `wait_for_current_scene_to_be`, never sleeps |
| Frame-rate dependent logic | Assert on game state, not timing; change time scale only when the test is about it |
| Loading screens / ads | Disable ads and remote config in the dev build through a define symbol |
