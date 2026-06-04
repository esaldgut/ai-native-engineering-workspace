---
name: android-testing-patterns
description: >-
  Pick the right test type per Clean Architecture layer the canonical Android way — domain/UseCase/
  repository as JVM unit tests (JUnit5 via the mannodermaus android-junit5 plugin + MockK with coEvery/
  coVerify), ViewModel StateFlow emission sequences with Turbine's test {} + awaitItem(), and Composables
  as instrumented JUnit4 tests using createComposeRule() + onNodeWithText(). Enforces runTest (NOT
  runBlockingTest), Dispatchers.setMain for viewModelScope, and useUnmergedTree for granular semantics
  assertions. Use when writing tests for any Android/Compose layer.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Android — Test your Compose layout (createComposeRule, onNodeWithText, ui-test-junit4, useUnmergedTree)"
      url: "https://developer.android.com/develop/ui/compose/testing"
      version: "Compose BOM 2026.05.00"
    - source: "Android — Testing Kotlin flows (runTest, StandardTestDispatcher, Dispatchers.setMain)"
      url: "https://developer.android.com/kotlin/flow/test"
      version: "kotlinx-coroutines-test 1.10+"
    - source: "Turbine (Cash App) — test {} + awaitItem() + cancelAndIgnoreRemainingEvents(), StateFlow support"
      url: "https://github.com/cashapp/turbine"
      version: "Turbine 1.2.1 (2025-06-11)"
    - source: "MockK — coEvery / coVerify for suspend functions"
      url: "https://mockk.io/"
      version: "MockK 1.14.x"
    - source: "mannodermaus/android-junit5 — JUnit5 on Android, AGP ≥ 8.2"
      url: "https://github.com/mannodermaus/android-junit5"
      version: "plugin 1.14.0.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Google I/O (May) + Compose BOM major"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Android testing patterns (per-layer test type)

The Android test stack stabilized around four pieces: **JUnit5** on the JVM unit tier (via the
[mannodermaus android-junit5](https://github.com/mannodermaus/android-junit5) plugin), **MockK** for
mocking (Kotlin-first, suspend-aware), **[Turbine](https://github.com/cashapp/turbine)** for asserting
`Flow`/`StateFlow` emission *sequences*, and **JUnit4 + Compose Test** for instrumented UI. This skill
maps each Clean Architecture layer to its correct tier and template.

## When to invoke

- You're writing a test for any layer: a domain `data class`, a UseCase, a repository, a `ViewModel`, or
  a `@Composable`.
- You see `runBlockingTest`, `every` used on a `suspend` function, or a ViewModel test that asserts only
  `uiState.value` and misses the Loading → Loaded transition — fix to the canonical form below.

**Announce on invoke:** "Using `android-testing-patterns` to select the per-layer test type (JUnit5 +
MockK + Turbine on the JVM, JUnit4 + Compose Test on device) per developer.android.com."

## The per-layer matrix

| Layer | Tier | Framework |
|---|---|---|
| Domain `data class` / pure logic | unit (JVM) | JUnit5 — no Android, no mocks |
| UseCase / Repository | unit (JVM) | JUnit5 + MockK (`coEvery`) + Turbine (if it returns a `Flow`) |
| ViewModel | unit (JVM) | JUnit5 + MockK + Turbine + `runTest` + `Dispatchers.setMain` |
| Composable (UI) | instrumented (device/emulator) | JUnit4 + `createComposeRule()` + semantics finders |
| End-to-end | instrumented | JUnit4 + Compose semantics (or Espresso for legacy Views) |

**JUnit5 on the JVM tier, JUnit4 on the instrumented Compose tier** is the pragmatic split:
`developer.android.com` examples and the `ComposeTestRule` are JUnit4 `TestRule`s, so JUnit4 stays the
documented default for UI tests, while JUnit5 (via the plugin, AGP ≥ 8.2) is production-ready for unit
tests. The artifacts: `androidx.compose.ui:ui-test-junit4` + `debugImplementation
androidx.compose.ui:ui-test-manifest` for UI; `app.cash.turbine:turbine`, `io.mockk:mockk`,
`org.jetbrains.kotlinx:kotlinx-coroutines-test` for the JVM tier.

## The rules (load-bearing — break them and the test lies)

1. **`runTest { }`, NOT `runBlockingTest`.** `runBlockingTest` (kotlinx-coroutines-test ≤ 1.5) is
   deprecated; `runTest { }` (1.6+) is the current builder. It auto-skips delays and provides a
   `TestScope`.
2. **`coEvery` / `coVerify` for suspend functions, never `every` / `verify`.** Kotlin can't overload
   `every` for a `suspend` return, so MockK provides the `co`-prefixed variants. Using `every` on a
   suspend function compiles but the stub silently never matches the real suspending call. MockK also has
   `coAnswers { }` for computed suspend returns.
3. **`Dispatchers.setMain(testDispatcher)` before constructing a ViewModel.** Without it,
   `viewModelScope.launch { }` throws `IllegalStateException: Module with the Main dispatcher had failed
   to initialize`. Wrap it in a JUnit5 `@BeforeEach`/extension (or a JUnit4 `MainDispatcherRule`) and
   `Dispatchers.resetMain()` after.
4. **Turbine for emission *sequences*; `StateFlow.value` only for the final snapshot.** Asserting
   `vm.uiState.value` catches only the terminal state and misses Loading → Loaded → Error transitions.
   Turbine's `test { }` + `awaitItem()` enforces the full event log. End the block with
   `cancelAndIgnoreRemainingEvents()` (this is the current API — *not* `cancelAndConsumeRemainingEvents`).
5. **`useUnmergedTree = true` for child-node assertions.** The semantics tree merges leaf nodes into
   their parent for accessibility; to assert on a `Text` inside a `Button` you need
   `onNodeWithText("…", useUnmergedTree = true)`, otherwise the test passes/fails for the wrong reason.

## Canonical example

```kotlin
// ---------- ViewModel test: JUnit5 + MockK + Turbine ----------
@ExtendWith(MainDispatcherExtension::class)   // calls Dispatchers.setMain in beforeEach / resetMain in after
class MyFeatureViewModelTest {

    private val repo = mockk<MyRepository>()
    private lateinit var vm: MyFeatureViewModel

    @BeforeEach
    fun setUp() {
        coEvery { repo.fetchItem("42") } returns MyItem("42", "hello")  // co-prefixed for suspend
        vm = MyFeatureViewModel(repo)
    }

    @Test
    fun `emits Loading then Loaded`() = runTest {        // runTest, not runBlockingTest
        vm.uiState.test {                                // Turbine
            assertEquals(MyUiState.Loading, awaitItem())
            assertEquals(MyUiState.Loaded(MyItem("42", "hello")), awaitItem())
            cancelAndIgnoreRemainingEvents()             // current Turbine API
        }
        coVerify(exactly = 1) { repo.fetchItem("42") }   // co-prefixed verify
    }
}

// ---------- Composable test: JUnit4 + Compose Test ----------
class MyFeatureScreenTest {
    @get:Rule val composeTestRule = createComposeRule()  // androidx.compose.ui.test.junit4

    @Test
    fun showsLoadedItem() {
        composeTestRule.setContent { MyFeatureScreen(viewModel = fakeLoadedViewModel()) }
        composeTestRule.onNodeWithText("hello").assertIsDisplayed()
    }
}
```

## Decision aids

- **MockK vs hand-written fakes for the ViewModel test?** Both are valid. MockK is faster to write for
  one-off stubs; a `class FakeRepository : MyRepository` is clearer when many tests share the same canned
  data and you want zero mocking framework in the ViewModel tier. Pick fakes when the fake is reused ≥3×.
- **One dispatcher helper or two?** Provide a single `MainDispatcherExtension` (JUnit5) for the JVM tier
  and add a `MainDispatcherRule` (JUnit4) only if some tests stay on JUnit4. Don't maintain both unless
  the codebase genuinely runs a mixed tier.
- **`StandardTestDispatcher` vs `UnconfinedTestDispatcher`?** `StandardTestDispatcher` (default in
  `runTest`) queues coroutines so you control ordering with `advanceUntilIdle()`;
  `UnconfinedTestDispatcher` runs them eagerly. Use Unconfined when a `stateIn`/`WhileSubscribed` upstream
  must emit immediately for the Turbine assertion.

## Related skills

- `global-skills/android/compose-clean-architecture-module-scaffold/SKILL.md` — the layered code these
  tests target (the `uiState: StateFlow` shape the Turbine test asserts on).
- `global-skills/android/android-security-checklist/SKILL.md` — negative/boundary security tests
  (injection, malformed tokens) complement these functional tests.

## Sources

- [Test your Compose layout](https://developer.android.com/develop/ui/compose/testing) ·
  [Compose testing APIs](https://developer.android.com/develop/ui/compose/testing/apis) ·
  [Testing Kotlin flows on Android](https://developer.android.com/kotlin/flow/test)
- [Turbine](https://github.com/cashapp/turbine) · [MockK](https://mockk.io/) ·
  [mannodermaus/android-junit5](https://github.com/mannodermaus/android-junit5)

---

**Last verified:** 2026-06-03 against developer.android.com (Compose testing + flow testing live),
Turbine 1.2.1 (`cancelAndIgnoreRemainingEvents` confirmed; `cancelAndConsumeRemainingEvents` does not
exist), MockK 1.14.x, android-junit5 plugin 1.14.0.0.
**Re-check after:** Google I/O 2026 + next Compose BOM major, or by 2026-12-01. **Decay risk:** medium.
**Found a drift?** Run `/skill-pattern-freshness-audit android`.
