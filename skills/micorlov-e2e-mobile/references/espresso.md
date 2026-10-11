# Espresso + Compose UI Test (Android native)

In-process instrumentation tests. They sync with the main thread, so they're fast and stable.
Use them for in-app flows of native Kotlin/Java apps (Views and Jetpack Compose).

## File Organization

```text
app/src/androidTest/java/com/example/app/
├── screens/                 Screen Objects
│   ├── LoginScreen.kt
│   └── SearchScreen.kt
├── flows/
│   ├── LoginTest.kt
│   └── SearchTest.kt
├── fixtures/
│   └── TestAccounts.kt      read-only test data
└── util/
    └── OkHttpIdlingResource.kt
```

## Gradle Setup (`app/build.gradle.kts`)

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        testInstrumentationRunnerArguments["clearPackageData"] = "true"
    }
    testOptions {
        execution = "ANDROIDX_TEST_ORCHESTRATOR"   // each test in its own process
        animationsDisabled = true
    }
}

dependencies {
    androidTestImplementation("androidx.test.ext:junit:<latest>")
    androidTestImplementation("androidx.test:runner:<latest>")
    androidTestImplementation("androidx.test:rules:<latest>")
    androidTestImplementation("androidx.test.espresso:espresso-core:<latest>")
    androidTestImplementation("androidx.test.espresso:espresso-intents:<latest>")
    androidTestImplementation("androidx.test.espresso:espresso-idling-resource:<latest>")
    androidTestUtil("androidx.test:orchestrator:<latest>")
    // Compose
    androidTestImplementation(platform("androidx.compose:compose-bom:<latest>"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

Check the project's version catalog (`gradle/libs.versions.toml`) and match its versions
rather than adding new ones. Look up current versions in the AndroidX release notes.

## Screen Object: Views

```kotlin
class LoginScreen {
    private val email = withId(R.id.email_input)
    private val password = withId(R.id.password_input)
    private val submit = withId(R.id.login_button)
    private val error = withId(R.id.error_text)

    fun login(user: String, pass: String): LoginScreen {
        onView(email).perform(replaceText(user))
        onView(password).perform(replaceText(pass), closeSoftKeyboard())
        onView(submit).perform(click())
        return this
    }

    fun assertError(message: String) {
        onView(error).check(matches(allOf(isDisplayed(), withText(message))))
    }
}
```

## Screen Object: Compose

```kotlin
class SearchScreen(private val rule: ComposeTestRule) {
    fun search(query: String): SearchScreen {
        rule.onNodeWithTag("search_input").performTextInput(query)
        rule.onNodeWithTag("search_submit").performClick()
        rule.waitUntil(timeoutMillis = 5_000) {
            rule.onAllNodesWithTag("result_card").fetchSemanticsNodes().isNotEmpty()
        }
        return this
    }

    fun resultCount(): Int =
        rule.onAllNodesWithTag("result_card").fetchSemanticsNodes().size

    fun assertEmptyState() {
        rule.onNodeWithTag("no_results").assertIsDisplayed()
    }
}
```

## Test Structure

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class SearchTest {
    @get:Rule(order = 0)
    val permissions: GrantPermissionRule =
        GrantPermissionRule.grant(android.Manifest.permission.POST_NOTIFICATIONS)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    private val screen get() = SearchScreen(composeRule)

    @Test
    fun searchByKeyword_showsResults() {
        screen.search("test")
        assertTrue(screen.resultCount() > 0)
        composeRule.onAllNodesWithTag("result_card")[0].assertTextContains("test", ignoreCase = true)
    }

    @Test
    fun searchWithNoMatches_showsEmptyState() {
        screen.search("xyznonexistent123")
        screen.assertEmptyState()
    }
}
```

For View-based activities, use `@get:Rule val activityRule = ActivityScenarioRule(MainActivity::class.java)`.

## Waiting for Background Work

Espresso waits for the main looper and AsyncTask only. Register an IdlingResource for anything else
(network, coroutines on IO dispatchers, custom executors):

```kotlin
@Before fun registerIdling() { IdlingRegistry.getInstance().register(OkHttpIdlingResource.create("okhttp", client)) }
@After fun unregisterIdling() { IdlingRegistry.getInstance().resources.forEach { IdlingRegistry.getInstance().unregister(it) } }
```

(`OkHttpIdlingResource` comes from `com.jakewharton.espresso:okhttp3-idling-resource`.)
For Compose, use `composeRule.waitUntil { ... }` or `waitUntilExactlyOneExists(hasTestTag("..."))`.
Never use `Thread.sleep`.

## Run on the Real Device

```bash
# whole suite on a specific device
ANDROID_SERIAL=<serial> ./gradlew connectedDebugAndroidTest

# single class / single method
ANDROID_SERIAL=<serial> ./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.example.app.flows.SearchTest#searchByKeyword_showsResults

# without Gradle (APKs already installed)
adb -s <serial> shell am instrument -w -r \
  -e class com.example.app.flows.SearchTest \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
```

`./gradlew connected...AndroidTest` uninstalls the app and the test APK after the run. If the user's
app data on the phone matters, install with `adb install -r` and run through `am instrument` instead.

## Artifacts

| What | Where |
|------|-------|
| HTML report | `app/build/reports/androidTests/connected/` |
| JUnit XML | `app/build/outputs/androidTest-results/connected/` |
| Copy to | `e2e-artifacts/espresso/` |

Screenshot on failure with a TestWatcher:

```kotlin
class ScreenshotOnFailure : TestWatcher() {
    override fun failed(e: Throwable?, description: Description) {
        val name = "${description.className}-${description.methodName}"
        Screenshot.capture().setName(name).process()   // androidx.test.runner.screenshot
    }
}
```

Or from the host after a failure: `adb -s <serial> exec-out screencap -p > e2e-artifacts/screenshots/<test>.png`.

## Espresso-Specific Flakiness

| Problem | Fix |
|---------|-----|
| `PerformException: ... at least 90 percent displayed` | `scrollTo()` first, or `nestedScrollTo()` for NestedScrollView |
| `NoMatchingViewException` right after navigation | Missing IdlingResource; the screen isn't loaded yet |
| `AmbiguousViewMatcherException` | Narrow with `allOf(withId(..), isDescendantOfA(withId(..)))` |
| RecyclerView items | `RecyclerViewActions.actionOnItemAtPosition<VH>(0, click())` (espresso-contrib) |
| Compose node not found | Missing `testTag`, or use `useUnmergedTree = true` for child nodes |
| Intents to other apps | `espresso-intents`: `intending(hasAction(...)).respondWith(...)` instead of leaving the app |
