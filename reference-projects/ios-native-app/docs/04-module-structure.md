# 04 — Estructura de módulos (iOS reference)

> Estructura de módulos de una app iOS nativa (Swift 6 / iOS 26) bajo Clean Architecture +
> MVVM + Repository, publicada como referencia sanitizada. Los nombres de dominio de negocio se
> reemplazaron por genéricos (`Post`, `Listing`, `Booking`); las reglas de capas, el actor
> isolation por capa y los patrones de concurrencia se conservan tal cual.

Estructura de módulos de una app iOS nativa (social + marketplace), resultado del contraste
entre la propuesta inicial y la estructura optimizada, ajustada con los hallazgos de la
investigación técnica (02-tech-research.md) y las decisiones de arquitectura
(03-architecture-decision.md).

## Estructura Completa

```
MyApp/
├── App/
│   └── MyAppApp.swift                     # @main, dependencias raíz
│
├── Core/                                   # Capa transversal, nonisolated/Sendable
│   ├── Keychain/
│   │   └── KeychainManager.swift           # ✅ Existente (nonisolated Sendable)
│   ├── Network/
│   │   ├── GraphQLClient.swift             # Apollo NetworkTransport + auth interceptor
│   │   ├── WebSocketClient.swift           # AppSync subscriptions (URLSessionWebSocketTask)
│   │   └── CertificatePinning.swift        # URLSession delegate trust evaluation
│   ├── Auth/                               # 13 archivos — auth, seguridad, validación, resiliencia
│   │   ├── AuthError.swift                 # Error unificado + timeout/noInternet + isRetryable + anti-enumeración
│   │   ├── AuthInterceptor.swift           # Auto-refresh + degradación graceful en ventana soft
│   │   ├── AuthRetryPolicy.swift           # Exponential backoff + jitter (3 intentos, 1s base)
│   │   ├── AuthTokenStore.swift            # Keychain token lifecycle (JSON blob atómico)
│   │   ├── CognitoAuthService.swift        # HTTPS directo + timeout 15s/30s + URLError + retry refresh
│   │   ├── CognitoFederationService.swift  # OAuth + URLComponents (percent-encoding seguro)
│   │   ├── InputValidator.swift            # Email/password/nombre/código validación + sanitización
│   │   ├── NetworkMonitor.swift            # Actor NWPathMonitor + connectivityUpdates() AsyncStream
│   │   ├── AppleAuthService.swift          # Credential revocation monitoring
│   │   ├── PKCEGenerator.swift             # SHA256 code_challenge + random code_verifier
│   │   ├── FederationProvider.swift         # Apple/Google/Facebook enum
│   │   ├── IdentityPoolService.swift       # GetId + GetCredentialsForIdentity
│   │   └── AppConfig.swift                 # xcconfig → Info.plist reader
│   └── Cache/
│       ├── PlayerPool.swift                # Pool de AVPlayer reutilizables
│       └── MediaPrefetcher.swift           # Nuke prefetch + AVPlayer preload
│
├── Domain/                                 # Modelos puros Sendable + protocolos
│   ├── Models/
│   │   ├── User.swift
│   │   ├── Post.swift
│   │   ├── Listing.swift
│   │   ├── Conversation.swift
│   │   └── Booking.swift
│   └── Repositories/                       # Protocolos (abstracciones)
│       ├── AuthRepositoryProtocol.swift
│       ├── PostRepositoryProtocol.swift
│       ├── MarketplaceRepositoryProtocol.swift
│       └── ChatRepositoryProtocol.swift
│
├── Data/                                   # Implementaciones concretas
│   ├── Repositories/
│   │   ├── AuthRepository.swift
│   │   ├── PostRepository.swift
│   │   ├── MarketplaceRepository.swift
│   │   └── ChatRepository.swift
│   ├── DTOs/                               # Codable wire format (GraphQL responses)
│   └── Mappers/                            # DTO → Domain Model conversions
│
├── Generated/                              # Apollo codegen output
│   └── GraphQL/                            # Módulo separado: nonisolated isolation
│                                           # (workaround Apollo issue #3601)
│
├── Features/                               # Views + ViewModels por feature
│   ├── Onboarding/
│   │   └── Views/
│   │       └── OnboardingView.swift        # 4-page Liquid Glass onboarding
│   │                                       # Figma illustrations (PDF vector)
│   │                                       # Adaptive: iPhone/iPad, portrait/landscape
│   ├── Auth/
│   │   ├── Views/
│   │   │   ├── LoginView.swift
│   │   │   └── SSOButtonsView.swift
│   │   └── ViewModels/
│   │       └── AuthViewModel.swift
│   ├── Feed/
│   │   ├── Views/
│   │   │   ├── FeedView.swift              # ScrollView infinito
│   │   │   ├── PostCard.swift              # Celda reutilizable
│   │   │   └── VideoPostPlayer.swift       # AVPlayer embebido
│   │   ├── ViewModels/
│   │   │   └── FeedViewModel.swift
│   │   └── Prefetch/
│   │       └── FeedPrefetcher.swift        # Look-ahead de medios
│   ├── Marketplace/
│   │   ├── Views/
│   │   │   ├── MarketplaceView.swift
│   │   │   ├── ListingCard.swift
│   │   │   └── ListingDetailView.swift
│   │   └── ViewModels/
│   │       └── MarketplaceViewModel.swift
│   ├── Chat/
│   ├── Profile/
│   └── Explore/
│
└── Presentation/                           # Componentes UI globales (no features)
    ├── DesignSystem/
    │   ├── Typography.swift
    │   ├── Colors.swift
    │   └── LiquidGlassModifiers.swift      # iOS 26 Liquid Glass
    └── Navigation/
        └── AppRouter.swift                 # NavigationPath centralizado + deep links
```

## Principios de la Estructura

### Separación de capas

| Capa | Responsabilidad | Actor Isolation |
|---|---|---|
| **Core/** | Infraestructura transversal | `nonisolated` on EVERY type (class, struct, enum, protocol) + `Sendable` |
| **Domain/** | Modelos puros y protocolos de repositorio | `nonisolated` on EVERY type + `Sendable` |
| **Data/** | Implementaciones concretas de repositorios | `nonisolated` on EVERY type (async) |
| **Generated/** | Apollo codegen output | `nonisolated` (módulo separado) |
| **Features/** | Views + ViewModels por feature | `@MainActor` (default del proyecto) |
| **Presentation/** | Componentes UI globales | `@MainActor` (default del proyecto) |
| **App/** | Entry point | `@MainActor` |

### Reglas de dependencia (Clean Architecture)

```
Features/ → Domain/ (protocolos + modelos)
Features/ → Presentation/ (design system + router)
Data/ → Domain/ (implementa protocolos)
Data/ → Core/ (networking, auth, cache)
Data/ → Generated/ (Apollo types)
Core/ → (ninguna dependencia interna)
Domain/ → (ninguna dependencia interna)
Generated/ → (ninguna dependencia interna)
```

### Concurrency patterns por capa

| Capa | Pattern |
|---|---|
| Core/Keychain | `nonisolated final class: Sendable` |
| Core/Network | `nonisolated` on ALL types (class, struct, protocol) |
| Core/Auth | `nonisolated` on ALL types (class, struct, enum, protocol) |
| Core/Cache | `nonisolated`, actor-based para thread safety |
| Domain/Models | `nonisolated struct: Sendable` — MUST be explicit |
| Domain/Repositories | `nonisolated protocol: Sendable` con async methods |
| Data/DTOs | `nonisolated struct: Decodable` with CodingKeys |
| Data/Repositories | `nonisolated final class: Sendable` |
| Features/ViewModels | `@MainActor @Observable final class` |
| Features/Views | `struct: View` (implícitamente @MainActor) |

## Dependencias Externas

| Dependencia | Versión | Ubicación de uso | Tamaño |
|---|---|---|---|
| Apollo iOS | 2.0.6+ | Core/Network, Data/, Generated/ | ~3-8 MB |
| Nuke + NukeUI | 13.0+ | Core/Cache, Features/Feed | ~2 MB |
| — (nativo) | — | Auth, Subscriptions, Video, Push | 0 MB |

## Current State (Auth + Onboarding + Security — initial auth + onboarding implementation complete)

| Layer | Files | Status |
|---|---|---|
| Core/Auth/ | 17 | Complete + **OAuthStateGenerator + PKCEFlowState/Store + ConstantTimeCompare** (CSRF + timing-safe validation) |
| Core/Keychain/ | 1 | Complete (`deleteAll(service:)` added) |
| Domain/Models/ | 2 | User, AWSCredentials |
| Domain/Repositories/ | 1 | AuthRepositoryProtocol |
| Data/Repositories/ | 1 | AuthRepository (defense-in-depth validation) |
| Data/DTOs/ | 2 | Cognito + OAuth response DTOs |
| Features/Onboarding/ | 1 | OnboardingView (4-page Liquid Glass adaptive) |
| Features/Auth/ | 3 | LoginView (inline validation, retry), SSOButtonsView (state CSRF + atomic flow + constant-time), AuthViewModel |
| Features/Main/ | 1 | MainTabView (5-tab placeholder) |
| Presentation/Navigation/ | 1 | AppRouter + DeepLink + AppTab |
| Config/ | 3 | Base, Dev, Prod xcconfig |
| **Total production** | **31** | + Info.plist, MyAppApp.swift (with FirstLaunchCleanup invocation) |
| **Total tests** | **18** | 102 tests in 15 suites |
