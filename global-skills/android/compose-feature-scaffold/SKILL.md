---
name: compose-feature-scaffold
description: >-
  Bootstrap a complete Compose feature module at once the canonical Android way — domain model +
  repository interface + RepositoryImpl + DTO mapper + screen-level @HiltViewModel + @Composable Screen
  + a Hilt @Module that @Binds the interface + a Navigation Compose entry. Wires @HiltAndroidApp,
  @AndroidEntryPoint on a single MainActivity, @Binds over @Provides for the interface, the Hilt 1.3.0
  hilt-lifecycle-viewmodel-compose artifact, and type-safe @Serializable routes. Use when starting a new
  feature; for a single file in an existing feature use compose-clean-architecture-module-scaffold.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Android — Hilt in multi-module apps (@Module @InstallIn, @Binds interface→impl, @HiltAndroidApp, @AndroidEntryPoint)"
      url: "https://developer.android.com/training/dependency-injection/hilt-multi-module"
      version: "Hilt 1.3.0 (2025-09-10)"
    - source: "Android — Hilt with other Jetpack libraries (@HiltViewModel + hiltViewModel())"
      url: "https://developer.android.com/training/dependency-injection/hilt-jetpack"
      version: "Hilt 1.3.0"
    - source: "Android — Navigation with Compose (NavHost, type-safe @Serializable routes)"
      url: "https://developer.android.com/develop/ui/compose/navigation"
      version: "navigation-compose 2.8+"
    - source: "Android — Architecture recommendations (layered feature, screen-level ViewModel)"
      url: "https://developer.android.com/topic/architecture/recommendations"
      version: "Compose BOM 2026.05.00"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Google I/O (May) + Compose BOM major"
    or_date: "2026-12-01"
  decay_risk: high
  status: current
---

# Compose feature scaffold (Kotlin, Hilt + Navigation)

This is `compose-clean-architecture-module-scaffold` promoted from "one file" to "one feature": it
generates the full UI / Domain / Data stack **plus** the Hilt module and the Navigation Compose entry,
so a new feature module is consistent and wired from the first commit. It composes the same official
guidance and adds the two pieces a single file doesn't need — DI binding and a nav destination.

## When to invoke

- You're starting a **new feature** (Feed, Profile, Checkout) and want the layered tree, the
  `@HiltViewModel`, the `@Binds` module, and the `NavHost` entry created together.
- You're standing up the app shell: `@HiltAndroidApp Application`, the single `@AndroidEntryPoint
  MainActivity`, and the root `NavHost`.

For adding one type to a feature that already exists, use
`compose-clean-architecture-module-scaffold` instead.

**Announce on invoke:** "Using `compose-feature-scaffold` to generate the full Compose feature
(layers + Hilt `@Binds` module + Navigation entry) per Android's Hilt and Navigation guidance."

## Files this scaffold produces

```
feature/myfeature/
├── domain/model/MyItem.kt              # immutable data class
├── domain/repository/MyRepository.kt   # interface
├── data/api/MyApi.kt + MyItemDto.kt    # DTO + service
├── data/mapper/MyItemMapper.kt         # DTO -> domain
├── data/repository/MyRepositoryImpl.kt # @Inject constructor, implements interface
├── di/MyFeatureModule.kt               # @Module @InstallIn, @Binds interface->impl
├── ui/MyUiState.kt                     # sealed interface
├── ui/MyFeatureViewModel.kt            # @HiltViewModel, screen-level
├── ui/MyFeatureScreen.kt               # @Composable, collectAsStateWithLifecycle
└── ui/MyFeatureNav.kt                  # @Serializable route + composable() entry
```

## The wiring rules (load-bearing)

1. **`@Binds`, not `@Provides`, for interface → impl.** Both compile, but `@Binds` generates a
   zero-overhead direct cast in the factory; `@Provides` for a single-impl interface is a review smell.
2. **Exactly one `@AndroidEntryPoint` Activity.** Single-Activity is the canonical Compose pattern; the
   one `MainActivity` carries `@AndroidEntryPoint` and `@HiltAndroidApp` goes on the `Application`.
   Annotating multiple Activities implies the multi-Activity anti-pattern Compose was built to remove.
3. **`hiltViewModel()` artifact follows nav scoping (Hilt 1.3.0).** Host-scoped →
   `androidx.hilt:hilt-lifecycle-viewmodel-compose`. Nav-back-stack-scoped →
   `androidx.hilt:hilt-navigation-compose` + `hiltViewModel(navBackStackEntry)`. The wrong choice gives
   "my state vanished" or "my state leaked across screens" bugs.
4. **KSP, not KAPT, for Hilt.** The feature module's `build.gradle.kts` applies
   `id("com.google.dagger.hilt.android")` + `id("com.google.devtools.ksp")`. KAPT is being phased out and
   is markedly slower.
5. **Type-safe `@Serializable` routes, no inline strings.** Declare a `@Serializable` route object/class
   per destination and reference it at the `NavHost` callsite — never a hardcoded `"myfeature"` string —
   so the call graph is greppable and the route is a compile-time symbol.

## Canonical example

```kotlin
// ---------- di/MyFeatureModule.kt ----------
@Module
@InstallIn(SingletonComponent::class)
abstract class MyFeatureModule {
    @Binds abstract fun bindRepo(impl: MyRepositoryImpl): MyRepository
}

// ---------- ui/MyFeatureViewModel.kt (see clean-architecture skill for the body) ----------
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

// ---------- ui/MyFeatureScreen.kt ----------
@Composable
fun MyFeatureScreen(viewModel: MyFeatureViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    // render state...
}

// ---------- ui/MyFeatureNav.kt — type-safe route ----------
@Serializable data object MyFeatureRoute

fun NavGraphBuilder.myFeatureGraph() {
    composable<MyFeatureRoute> { MyFeatureScreen() }
}

// ---------- App shell ----------
@HiltAndroidApp class MyApp : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val nav = rememberNavController()
            NavHost(nav, startDestination = MyFeatureRoute) { myFeatureGraph() }
        }
    }
}
```

## Decision aids

- **Gradle module (`:feature:myfeature`) vs package in `:app`?** Multi-module is canonical for medium+
  codebases (faster incremental builds, clear ownership); a package is fine for a small app. Default to a
  Gradle module once the app has more than a handful of features.
- **Type-safe vs string routes?** Type-safe `@Serializable` (shown above) is the modern recommendation
  (`navigation-compose 2.8+`); the string form `composable("myfeature")` still works if you're on an older
  setup. Prefer type-safe for new code.
- **`@Binds` module abstract class vs `@Provides` object?** Use an `abstract class` + `@Binds` for
  interface→impl; use an `object` + `@Provides` only for types you don't own (e.g. constructing a Retrofit
  or Ktor `HttpClient`).

## Related skills

- `global-skills/android/compose-clean-architecture-module-scaffold/SKILL.md` — per-file version and the
  full layer-rule rationale (ViewModel, `UiState`, mapper).
- `global-skills/android/android-testing-patterns/SKILL.md` — the per-layer test suite for the files this
  scaffold emits.
- `global-skills/android/android-security-checklist/SKILL.md` — harden the new feature's network +
  manifest surface before shipping.

## Sources

- [Hilt for Android](https://developer.android.com/training/dependency-injection/hilt-android) ·
  [Hilt in multi-module apps](https://developer.android.com/training/dependency-injection/hilt-multi-module) ·
  [Hilt with other Jetpack libraries](https://developer.android.com/training/dependency-injection/hilt-jetpack)
- [Navigation with Compose](https://developer.android.com/develop/ui/compose/navigation) ·
  [Architecture recommendations](https://developer.android.com/topic/architecture/recommendations)

---

**Last verified:** 2026-06-03 against developer.android.com (Hilt multi-module `@Binds`, Hilt 1.3.0
artifact, Navigation Compose type-safe routes) + Compose BOM 2026.05.00.
**Re-check after:** Google I/O 2026 + next Compose BOM major, or by 2026-12-01. **Decay risk:** high
(Hilt artifacts + Navigation Compose APIs evolve).
**Found a drift?** Run `/skill-pattern-freshness-audit android`.
