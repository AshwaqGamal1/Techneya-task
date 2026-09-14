# Wikipedia mobile — save an article to a reading list

End-to-end Appium test in Java 17 (Appium Java client 9, TestNG, AssertJ), written
against the Wikipedia app. Android is the primary target; iOS locators are present in
the same page objects via `@iOSXCUITFindBy`.

## Prerequisites

- JDK 17+, Maven
- Node 18+ and Appium 2: `npm i -g appium`
- Android: `appium driver install uiautomator2`, an emulator or device on `adb devices`,
  and the Wikipedia app installed (or point `android.app.path` at an APK)
- iOS: `appium driver install xcuitest`, Xcode, a simulator, and a `.app`/`.ipa` build

Start the server, then run:

```bash
appium

mvn clean test                                   # Android, defaults from config.properties
mvn clean test -Dandroid.device.name=Pixel_7 -Dandroid.platform.version=14
mvn clean test -Dplatform=ios -Dios.device.name="iPhone 15" -Dios.platform.version=17.5
mvn clean test -Dandroid.app.path=/path/to/wikipedia.apk
```

Nothing needs editing before the first run — every capability resolves system property →
`config.properties` → fail fast with a message naming the missing key.

## Structure

```
config/    Config, Platform          – run configuration, one resolution rule
driver/    DriverFactory             – capabilities live here and nowhere else
           DriverManager             – ThreadLocal session, safe cleanup
pages/     BasePage                  – waits, taps, scrolling, back navigation
           OnboardingPage, ExploreFeedPage, SearchPage, ArticlePage,
           AddToReadingListDialog, ReadingListsPage, ReadingListDetailPage
support/   BaseTest                  – session per test method
           ScreenshotOnFailureListener – png + page source on failure
           TestData                  – unique reading list names
tests/     SaveArticleToReadingListTest
```

## Decisions worth explaining

**The test reads as the user's journey.** Locators, waits and gestures are all in page
objects; the test method is nine lines of intent plus assertions. Someone who has never
used Appium can review it, and an app redesign changes page objects rather than tests.

**Page objects return page objects.** `openSearch()` returns a `SearchPage`,
`openResult()` returns an `ArticlePage`. The type system then stops a test from calling
`saveAndChooseList()` before an article is open.

**No `Thread.sleep`, anywhere.** Every interaction goes through an explicit condition.
A sleep tuned on a laptop is still too short on a loaded CI machine — the fix for a
flaky tap is a better condition, not a longer wait. Implicit waits are set to zero
because mixing them with explicit waits produces timeouts nobody can predict.

**Each page declares a `uniqueElement()`.** When a screen fails to load, the failure says
"ReadingListsPage did not load within PT20S" rather than a bare `NoSuchElementException`
40 lines deep.

**The reading list name is unique per run.** A fixed name would pass on a device where
the list already exists even if creation were broken, and would fail on the second run
when the app rejects a duplicate. `noReset=false` also gives each session a clean app.

**Session per test method, torn down in an `alwaysRun` hook.** A failed test still
releases the device; a leaked session blocks the next run on the same emulator.

**Locator strategy.** Resource ids where the app provides stable ones, accessibility ids
otherwise. No XPath over the full tree — it is slow on mobile and breaks whenever the
hierarchy shifts. Where the article body is a WebView, the test deliberately stays in
the native context and asserts on native chrome; the scenario is about saving, not about
rendering.

**Failure diagnostics.** A screenshot plus the page source on failure answers most
"passes locally, fails on CI" questions: the image shows what was on screen, the XML
shows whether the element was there under a different identifier.

## Honest caveats

- **I could not execute this suite here** — this environment has no emulator, no Appium
  server and no access to Maven Central, so the code is unrun. On a real setup expect to
  re-verify a handful of locators in Appium Inspector before the first green run.
- **Locators are version-sensitive.** The Wikipedia app ships often and its resource ids
  change; the ones here reflect recent Android builds (`org.wikipedia:id/…`). They are
  all in one field block per page, which is the point of the structure — fixing a
  renamed id is a one-line edit.
- **The iOS locators are the weaker half.** They are modelled on the app's accessibility
  labels but have not been confirmed against a simulator build. The class chains for
  cell titles in particular should be replaced with real accessibility identifiers once
  someone has Inspector open.
- **Save is one tap plus a snackbar action.** If the app version under test opens the
  list picker directly instead, `AddToReadingListDialog.openFromSnackbarIfNeeded()` is
  already tolerant of the snackbar being absent.

## What I would add next

- Run the same scenario on both platforms in CI via a TestNG parameter, on a device
  cloud (BrowserStack/Sauce) — the driver layer already takes its target from config.
- Allure reporting with the failure screenshot attached to the step that failed.
- Sibling scenarios that share these page objects: removing an article from a list,
  adding to an existing list, offline availability of a saved article, saving the same
  article twice.
- An API-level or `adb`-level setup shortcut so tests that are *not* about list creation
  do not have to create one through the UI.
