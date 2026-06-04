---
name: compose-clean-architecture-module-scaffold
description: >-
  Scaffold a single Kotlin file into the correct Clean Architecture layer (UI / Domain / Data) the
  canonical Android way — immutable domain data class, repository interface in Domain + impl in Data,
  a screen-level @HiltViewModel exposing uiState: StateFlow<UiState> via stateIn +
  SharingStarted.WhileSubscribed(5_000), and a Composable collecting it with
  collectAsStateWithLifecycle(). Enforces the load-bearing rules (no Context in the ViewModel, DTO
  ≠ domain entity, lifecycle-aware collection) and the Hilt 1.3.0 artifact rename. Use when adding a
  type to an existing Compose feature; for a whole new feature use compose-feature-scaffold.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Android — Architecture recommendations (ViewModel at screen level, collectAsStateWithLifecycle Strongly Recommended, repository even with single source, WhileSubscribed(5000))"
      url: "https://developer.android.com/topic/architecture/recommendations"
      version: "Compose BOM 2026.05.00"
    - source: "Android — Hilt 1.3.0 release notes (hiltViewModel() moved to androidx.hilt:hilt-lifecycle-viewmodel-compose)"
      url: "https://developer.android.com/jetpack/androidx/releases/hilt"
      version: "Hilt 1.3.0 (2025-09-10)"
    - source: "Android — Use Hilt with other Jetpack libraries (@HiltViewModel, hiltViewModel())"
      url: "https://developer.android.com/training/dependency-injection/hilt-jetpack"
      version: "Hilt 1.3.0"
    - source: "Kotlin — kotlinx.coroutines stateIn operator"
      url: "https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html"
      version: "kotlinx-coroutines 1.10+"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Google I/O (May) + Compose BOM major"
    or_date: "2026-12-01"
  decay_risk: high
  status: current
---

# Compose Clean Architecture per-file scaffold (Kotlin)

The official [Android architecture guide](https://developer.android.com/topic/architecture/recommendations)
codifies a three-layer Clean Architecture — **UI / Domain / Data** — and elevates several conventions
from "common practice" to **Strongly Recommended**. This skill places one new Kotlin file into exactly
one layer with those rules baked in, so an extension to an existing feature stays consistent instead of
drifting into "Clean Architecture in name only."

## When to invoke

- You're adding a single type to an existing Compose feature: a new domain entity, a repository method,
  a `RepositoryImpl`, a screen-level `ViewModel`, or a `Composable` screen.
- You see a `ViewModel` taking `Context`, a Composable using `collectAsState()`, or a DTO doubling as a
  domain entity — fix it to the canonical form below.
- For a *whole new feature* (all layers + Hilt module + nav entry at once), use
  `compose-feature-scaffold` instead — this skill is the per-file case.

**Announce on invoke:** "Using `compose-clean-architecture-module-scaffold` to place this file in its
correct Clean Architecture layer per the Android architecture recommendations."

## The layer rules (load-bearing)

| Layer | What lands here | Non-negotiable rule |
|---|---|---|
| **Domain** | immutable entity (`data class`), repository **interface**, optional UseCase | pure Kotlin, **zero** platform/framework imports |
| **Data** | `RepositoryImpl`, DTOs (`@Serializable`/Room), DTO→entity mapper, data sources | the *only* layer that touches network/DB; maps DTO → domain |
| **UI** | screen-level `@HiltViewModel`, `UiState`, `@Composable` screen | observes via `collectAsStateWithLifecycle()`; driven by the Domain **interface**, never the impl |

1. **Screen-level ViewModels only.** *"Do not use ViewModels in reusable pieces of UI"* — one
   `ViewModel` per Composable **screen**, not per button or list row. Reusable UI takes plain state +
   lambdas as parameters.
2. **`collectAsStateWithLifecycle()`, not `collectAsState()`.** Strongly Recommended. Plain
   `collectAsState()` keeps the upstream Flow active when the Composable is off-screen (backgrounded /
   navigated away), wasting work and battery. Only fall back to `collectAsState()` in `commonMain` /
   platform-agnostic code where `androidx.lifecycle:lifecycle-runtime-compose` is unavailable.
3. **No `Context` in the ViewModel constructor; no `AndroidViewModel`.** The guide says *"Do not use
   AndroidViewModel."* If you need resources, inject a tested abstraction (a `ResourceProvider`
   interface), never `Context` or `Application`.
4. **`SharingStarted.WhileSubscribed(5_000)`** is the canonical `stateIn` policy — it survives config
   change and brief backgrounding (the 5 s window) without leaking the upstream collection forever.
   Not `Eagerly`, not `Lazily`.
5. **DTO ≠ domain entity.** The domain `data class` is immutable and annotation-free. DTOs live in
   `data/` with `@Serializable` / Room / Moshi annotations. A mapper in the repository impl translates
   DTO → entity. Conflating them is the most common Clean Architecture failure.

## Hilt 1.3.0 artifact (breaking rename — verify before wiring)

`hiltViewModel()` moved out of the navigation artifact in **Hilt 1.3.0** (2025-09-10):

- **Default (host-lifecycle-scoped):** `androidx.hilt:hilt-lifecycle-viewmodel-compose` — package
  `androidx.hilt.lifecycle.viewmodel.compose`. Use this when you do **not** need nav-back-stack scoping
  (no transitive dependency on `androidx.navigation`).
- **Nav-back-stack-scoped:** `androidx.hilt:hilt-navigation-compose` — still valid; use
  `hiltViewModel(navBackStackEntry)` when the ViewModel must survive sibling navigation within a graph.

Older scaffolds that always pull `hilt-navigation-compose` for a non-nav screen are stale.

## Canonical example

```kotlin
// ---------- Domain ----------
// domain/model/MyItem.kt — immutable, no framework imports
data class MyItem(val id: String, val title: String)

// domain/repository/MyRepository.kt — interface lives in Domain
interface MyRepository {
    suspend fun fetchItem(id: String): MyItem
}

// ---------- Data ----------
// data/repository/MyRepositoryImpl.kt — impl + DTO→domain mapping
class MyRepositoryImpl @Inject constructor(
    private val api: MyApi,            // Ktor/Retrofit service returning a DTO
    private val mapper: MyItemMapper,  // DTO -> domain entity
) : MyRepository {
    override suspend fun fetchItem(id: String): MyItem =
        mapper.toDomain(api.getItem(id))
}

// ---------- UI ----------
// ui/MyUiState.kt
sealed interface MyUiState {
    data object Loading : MyUiState
    data class Loaded(val item: MyItem) : MyUiState
    data class Error(val message: String) : MyUiState
}

// ui/MyFeatureViewModel.kt — screen-level, NO Context, driven by the interface
@HiltViewModel
class MyFeatureViewModel @Inject constructor(
    private val repo: MyRepository,
) : ViewModel() {
    val uiState: StateFlow<MyUiState> =
        flow { emit(repo.fetchItem("42")) }
            .map<MyItem, MyUiState> { MyUiState.Loaded(it) }
            .catch { emit(MyUiState.Error(it.message.orEmpty())) }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), MyUiState.Loading)
}

// ui/MyFeatureScreen.kt — collectAsStateWithLifecycle, not collectAsState
@Composable
fun MyFeatureScreen(
    viewModel: MyFeatureViewModel = hiltViewModel(), // hilt-lifecycle-viewmodel-compose
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (val s = state) {
        MyUiState.Loading   -> CircularProgressIndicator()
        is MyUiState.Loaded -> Text(s.item.title)
        is MyUiState.Error  -> Text("Error: ${s.message}")
    }
}
```

## Decision aids

- **`UiState` file vs inline?** A separate `sealed interface MyUiState` file is greppable and reusable
  by the test; inlining it in the ViewModel is fine for a trivial 2-state screen. Default to a separate
  file once there are ≥3 states.
- **Ktor vs Retrofit for `MyApi`?** Retrofit is Android-only and familiar; **Ktor** is the choice if any
  of this logic may later move to a KMP `commonMain` module (see `kmp-shared-extraction`). Pick Ktor when
  cross-platform is on the roadmap.
- **UseCase layer?** Optional. Add a `domain/usecase/` class only when logic spans multiple repositories
  or has real branching; a ViewModel calling a single repository method directly is acceptable.

## Related skills

- `global-skills/android/compose-feature-scaffold/SKILL.md` — the whole-feature version (adds Hilt
  module + navigation entry).
- `global-skills/android/android-testing-patterns/SKILL.md` — the JUnit5 + Turbine + MockK test for the
  `uiState` emission sequence this scaffold produces.
- `global-skills/android/kmp-shared-extraction/SKILL.md` — when the Domain layer here should move to a
  shared KMP `commonMain` module.

## Sources

- [Architecture recommendations](https://developer.android.com/topic/architecture/recommendations) ·
  [UI layer / state holders](https://developer.android.com/topic/architecture/ui-layer/stateholders) ·
  [Data layer](https://developer.android.com/topic/architecture/data-layer)
- [Hilt with other Jetpack libraries](https://developer.android.com/training/dependency-injection/hilt-jetpack) ·
  [Hilt release notes (1.3.0 artifact rename)](https://developer.android.com/jetpack/androidx/releases/hilt)
- [collectAsStateWithLifecycle (androidx.lifecycle.compose)](https://developer.android.com/reference/kotlin/androidx/lifecycle/compose/package-summary) ·
  [stateIn](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html)

---

**Last verified:** 2026-06-03 against developer.android.com (architecture recommendations live; Hilt
1.3.0 `hilt-lifecycle-viewmodel-compose` artifact confirmed) + Compose BOM 2026.05.00.
**Re-check after:** Google I/O 2026 + next Compose BOM major, or by 2026-12-01. **Decay risk:** high
(Hilt artifact coordinates and lifecycle APIs churn).
**Found a drift?** Run `/skill-pattern-freshness-audit android`.
