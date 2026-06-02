# 02 — Architecture: Clean + MVVM (paridad ios-reference-app, idiomático Android)

Estructura objetivo. El repo arranca single-module (`:app`) desde el template de Android
Studio; se migra progresivamente a esta estructura modular.

## Estructura de Módulos Gradle (objetivo)

```
android-reference-app/
├── app/                          # entry point: MainActivity, navigation host, DI wiring
│
├── core/                         # módulos transversales (sin UI)
│   ├── core-network/             # Apollo Kotlin client, OkHttp, interceptors, cert pinning
│   ├── core-auth/                # Cognito, token store (EncryptedSharedPreferences), refresh
│   ├── core-data/                # repository impls, DTOs, mappers
│   ├── core-domain/              # modelos (data class), repository interfaces, use cases
│   └── core-ui/                  # design system Compose: theme, components, modifiers
│
├── feature/                      # un módulo por feature (o packages si single-module)
│   ├── feature-auth/
│   ├── feature-feed/
│   ├── feature-marketplace/
│   ├── feature-chat/
│   └── feature-profile/
│
├── shared/                       # (KMP) lógica compartida con ios-reference-app
│   └── src/commonMain/kotlin/    # modelos dominio, use cases, validación
│
└── gradle/libs.versions.toml     # Version Catalog (source of truth)
```

> **Migración pragmática:** empezar single-module con **packages** (`ui/`, `features/`,
> `core/`, `domain/`, `data/`) dentro de `:app`, y extraer a módulos Gradle reales cuando
> el build time o los límites de equipo lo justifiquen. No sobre-modularizar al inicio.

## Reglas de dependencia (Clean Architecture)

```
feature/  → core-domain (interfaces + modelos), core-ui (design system)
core-data → core-domain (implementa interfaces), core-network, core-auth
core-domain → (sin dependencias internas — puro Kotlin)
core-network/core-auth → (sin features)
shared (KMP) → (puro Kotlin común, consumido por core-domain)
```

Regla de oro: **las dependencias apuntan hacia el dominio**. `core-domain` no conoce a nadie.

## Capas y convenciones de concurrencia

| Capa | Tipo | Concurrencia |
|---|---|---|
| **UI (Compose)** | `@Composable` functions | Main thread; `collectAsStateWithLifecycle()` |
| **ViewModel** | `class : ViewModel()` | `viewModelScope`; expone `StateFlow<UiState>` inmutable |
| **UseCase** | `class` con `operator fun invoke()` | `suspend`; sin estado |
| **Repository (interface)** | `interface` en core-domain | `suspend fun` / `Flow<T>` |
| **Repository (impl)** | `class` en core-data | `suspend` + `withContext(Dispatchers.IO)` |
| **Domain model** | `data class` inmutable (`val`) | thread-safe por inmutabilidad |
| **DTO** | `data class` + `@SerialName` | — |

## Patrón UiState (paridad con ios-reference-app error handling)

```kotlin
// Un estado por pantalla, inmutable
sealed interface FeedUiState {
    data object Loading : FeedUiState
    data class Success(val moments: List<Moment>) : FeedUiState
    data class Error(val type: AppError) : FeedUiState   // typed, no string parsing
}

@HiltViewModel
class FeedViewModel @Inject constructor(
    private val getFeed: GetFeedUseCase,
) : ViewModel() {
    private val _uiState = MutableStateFlow<FeedUiState>(FeedUiState.Loading)
    val uiState: StateFlow<FeedUiState> = _uiState.asStateFlow()   // expone inmutable
    // ...
}
```

> **Lección heredada de ios-reference-app (#44-47):** rutear UI por **tipo de error**, no por
> parsing del string localizado. `AppError` es `sealed` → `when` exhaustivo. El mensaje
> de display y el error tipado se mantienen coherentes.

## Errores: sealed por dominio

```kotlin
sealed class AppError {
    data object Network : AppError()
    data object Unauthorized : AppError()
    data class Server(val code: Int) : AppError()
    data class Unknown(val cause: Throwable?) : AppError()
}
```

Uno por dominio (`AuthError`, `FeedError`), no un `Exception` genérico. `when` exhaustivo
sobre el sealed → el compilador obliga a manejar casos nuevos.

## DI con Hilt

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideApolloClient(/* ... */): ApolloClient = /* ... */
}
```

ViewModels con `@HiltViewModel` + `@Inject constructor`. Módulos en `di/` por capa.

## Navegación type-safe

Navigation Compose con destinos tipados (`@Serializable` routes, no strings):

```kotlin
@Serializable data class ProfileRoute(val userId: String)
// navController.navigate(ProfileRoute(userId = "123"))
```

## Testing (paridad ios-reference-app)

| Tipo | Herramienta | Equivalente iOS |
|---|---|---|
| Unit (ViewModel, UseCase) | JUnit5 + MockK + Turbine (Flow) | Swift Testing |
| Repository contract | JUnit5 + fakes | Domain contract tests |
| Compose UI | `createComposeRule()` + semantics | XCTest UI |
| Instrumented | Espresso (cuando aplique) | XCUIApplication |

Turbine para testear `StateFlow`/`Flow` emisiones. MockK para mocks idiomáticos Kotlin.
