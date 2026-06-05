# 03 — Decisión de Arquitectura: Stack de Networking MyApp iOS

Documento de decisión basado en la investigación técnica (02-tech-research.md)
y el contexto del MVP (01-mvp-context.md).

## Decisión: Arquitectura Híbrida

Ningún SDK individual cumple todos los requisitos. La arquitectura combina
las mejores herramientas para cada capa:

```
┌──────────────────────────────────────────────────────┐
│                    MyApp iOS App                      │
│              Swift 6.3 / iOS 26.4                     │
├──────────────────────────────────────────────────────┤
│                                                       │
│  AUTH              Amazon Cognito SDK directo          │
│                    + AuthenticationServices (Apple)    │
│                    + Keychain (token storage)          │
│                                                       │
│  GRAPHQL           Apollo iOS 2.x                     │
│                    Queries + Mutations                 │
│                    Codegen tipado                      │
│                    Normalized cache (SQLite)           │
│                                                       │
│  SUBSCRIPTIONS     URLSessionWebSocketTask             │
│                    Protocolo AppSync custom             │
│                    Chat, real-time updates              │
│                                                       │
│  MEDIA             Nuke 13 (imágenes CloudFront)       │
│                    AVPlayer + HLS (video CloudFront)   │
│                                                       │
│  PUSH              APNS nativo                         │
│                    registerDevice mutation              │
│                    Deep link routing                    │
│                                                       │
│  OFFLINE           Apollo normalized cache              │
│                    + SwiftData (mutations pendientes,   │
│                      borradores, mensajes en cola)     │
│                                                       │
│  SECURITY          NSPinnedDomains (Info.plist)         │
│                    + URLSession delegate pinning        │
│                    TLS 1.3 + HTTP/3 (QUIC)             │
│                    Keychain para tokens                 │
│                                                       │
└──────────────────────────────────────────────────────┘
```

## Justificación por Componente

### Auth: Cognito SDK directo (NO Amplify)

| Razón | Detalle |
|---|---|
| Amplify Auth requiere el framework completo | 15-30 MB de dependencias innecesarias |
| Sign in with Apple nativo | Amplify requiere Hosted UI (web view). `AuthenticationServices` es nativo. |
| Control del token flow | idToken directo a AppSync sin abstracción intermedia |
| Swift 6 safe | Nuestro `KeychainManager` ya es `nonisolated Sendable` |

**Implementación**: `ASAuthorizationController` para Sign in with Apple,
Cognito `InitiateAuth` / `RespondToAuthChallenge` para email/password,
Google/Facebook SDKs nativos para federation.

### GraphQL: Apollo iOS 2.x

| Razón | Detalle |
|---|---|
| Único client compilado en Swift 6 `.v6` | Sendable, async/await, zero data races |
| Codegen tipado | Types Swift generados del schema GraphQL |
| Normalized cache | In-memory + SQLite, consultas cache-first |
| Binary size lean | ~3-8 MB sin dependencias externas |
| Certificate pinning | Vía ApolloURLSession delegate |

**Workaround requerido**: Generated code en target/módulo con
`SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` (issue #3601).

**Custom scalar**: `AWSDateTime` → `String` en codegen config, con extensión
para conversión a `Date`.

### Subscriptions: URLSessionWebSocketTask

| Razón | Detalle |
|---|---|
| AppSync usa protocolo propietario | Ni Apollo ni Amplify lo soportan correctamente en Swift 6 |
| Nativo iOS | Zero dependencias, async/await, certificate pinning |
| Control total | Reconexión, keep-alive, auth refresh bajo nuestro control |

**Implementación**: Client custom que implementa el protocolo AppSync
(connection_init, start, start_ack, data, ka, stop).

### Media: Nuke 13 + AVPlayer

| Razón | Detalle |
|---|---|
| Nuke 13 | Swift 6 safe, LazyImage SwiftUI, cache agresivo, < 2s compile |
| AVPlayer + HLS | Nativo, adaptive bitrate automático, CloudFront compatible |
| AsyncImage descartado | No cachea, causa hitching en scroll, sin pipeline |
| HTTP/3 | URLSession y CloudFront lo soportan. Hasta 15% menos latencia. |

### Offline: Apollo Cache + SwiftData

| Razón | Detalle |
|---|---|
| Apollo normalized cache | Consultas cache-first para feed y marketplace |
| SwiftData | Cola de mutations pendientes, borradores de posts, mensajes en queue |
| No DataStore de Amplify | Complejo con schemas grandes, no Swift 6 safe |

### Seguridad

| Capa | Implementación |
|---|---|
| Certificate pinning (declarativo) | `NSPinnedDomains` en Info.plist — SHA-256 SPKI de CloudFront y AppSync |
| Certificate pinning (runtime) | URLSession delegate con trust evaluation programática |
| Token storage | Keychain vía `KeychainManager` (ya implementado, nonisolated Sendable) |
| TLS | 1.3 por defecto en iOS 26 |
| HTTP/3 | Automático en URLSession + habilitar en CloudFront |
| ATS | Activo (default iOS) |

## Dependencias del Proyecto

| Dependencia | Versión | Propósito | Tamaño |
|---|---|---|---|
| Apollo iOS | 2.0.6+ | GraphQL client + codegen | ~3-8 MB |
| Nuke | 13.0+ | Image loading + cache | ~2 MB |
| NukeUI | 13.0+ | SwiftUI LazyImage | Incluido |
| — | — | Auth, subscriptions, video = nativo | 0 MB |

**Total de dependencias externas: 2** (Apollo + Nuke).
Todo lo demás es nativo (URLSession, AVPlayer, AuthenticationServices, Keychain).

## Riesgos y Mitigaciones

| Riesgo | Mitigación |
|---|---|
| Apollo #3601 (MainActor isolation en generated code) | Generated code en módulo separado con `nonisolated` default |
| AppSync WebSocket protocol changes | Implementar con protocol abstraction para swap futuro |
| Cognito token refresh manual | Implementar refresh interceptor con retry automático |
| Nuke 13 breaking changes | Pinear versión exacta en Package.swift |

## Acción Pendiente en Infraestructura

- [ ] Habilitar HTTP/3 en la distribución CloudFront del CDN
- [ ] Verificar si CloudFront ya sirve HLS para video o configurar MediaConvert
