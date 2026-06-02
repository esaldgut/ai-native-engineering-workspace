# 02 — Investigación Técnica: Networking para The Platform iOS Nativo

Investigación realizada el 2026-03-26 para determinar el stack de networking
óptimo para Swift 6.3, iOS 26.4, AWS AppSync GraphQL, con requisitos de
máximo rendimiento, mínima latencia y seguridad total.

## Candidatos Evaluados

### A. AWS Amplify Swift SDK (v2.54.1)

**Fuente**: https://github.com/aws-amplify/amplify-swift

| Aspecto | Hallazgo | Fuente |
|---|---|---|
| Swift 6 strict concurrency | **NO soportado**. `swift-tools-version: 5.9`, sin `swiftLanguageMode(.v6)`. Solo un target interno habilita strict concurrency. | Package.swift del repo |
| Data races | Crash confirmado en `StorageMultipartUploadSession` — `swift_retain` / `EXC_BAD_ACCESS` afecta ~5-10% de uploads de video. | Issue #4138 (abierto) |
| Subscriptions | Bug de reconexión agresiva: costos AppSync 28x mayores (5k → 144k SubscribeSuccess/día, 100k → 4M ActiveSubscriptions/día). | Issue #4007 |
| Binary size | 15-30 MB (incluye aws-sdk-swift v1.6.71, Smithy runtime, SQLite.swift) | Análisis de dependencias |
| Certificate pinning | No soportado. No hay hook para custom `URLSessionDelegate`. | Búsqueda en issues (4 resultados) |
| Sign in with Apple nativo | No implementado — requiere Hosted UI (web view). Solicitado desde 2022. | Issue #1121 (abierto) |
| Cognito token storage | Keychain automático vía `AWSCognitoAuthPlugin` | Documentación oficial |
| DataStore offline | SQLite local con conflict resolution. Funcional pero complejo con schemas grandes. | Documentación oficial |
| Deployment target mínimo | iOS 13.0 | Package.swift |

### B. Apollo iOS (v2.0.6 stable, v2.1.0-rc-1)

**Fuente**: https://github.com/apollographql/apollo-ios

| Aspecto | Hallazgo | Fuente |
|---|---|---|
| Swift 6 strict concurrency | **Compilado en `.v6` language mode**. `swift-tools-version: 6.1`, `swiftLanguageModes: [.v6, .v5]`. Tipos Sendable, APIs async/await. | Package.swift del repo |
| MainActor default isolation | Issue conocido: generated code conflicta con `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. Workaround: generated code en target con isolation `nonisolated`. | Issue #3601 (abierto) |
| AppSync subscriptions | **NO soportado nativo**. Apollo usa `graphql-transport-ws`, AppSync usa protocolo propietario. | Documentación Apollo |
| Bridge oficial AWS | `aws-appsync-apollo-extensions-swift` v1.0.5 — **solo soporta Apollo 1.x**, no 2.x. Tiene thread-safety crash en `AuthTokenAuthorizer` (issue #46). | GitHub aws-amplify |
| Codegen con directivas AppSync | Directivas `@aws_cognito_user_pools`, `@aws_auth` son server-side → ignoradas correctamente por codegen. | Documentación Apollo |
| Custom scalars (`AWSDateTime`) | Requiere mapping manual en configuración de codegen (e.g., `AWSDateTime` → `String` o `Date` wrapper). | Documentación Apollo |
| Normalized cache | In-memory + SQLite opcional. Read-through cache, no offline-first queue. | Documentación Apollo |
| Certificate pinning | Sí, vía `ApolloURLSession` protocol / delegate injection (2.1.0-rc). | Changelog v2.1.0-rc-1 |
| Binary size | ~3-8 MB, zero dependencias externas en 2.x | Análisis de dependencias |
| Deployment target mínimo | iOS 15.0 | Package.swift |

### C. URLSession + Custom Client

| Aspecto | Hallazgo | Fuente |
|---|---|---|
| GraphQL client | No existe un cliente lightweight mantenido para AppSync en Swift. | Búsqueda GitHub/web |
| WebSocket (subscriptions) | `URLSessionWebSocketTask` disponible desde iOS 13. Soporta `async/await`. Puede implementar protocolo AppSync. | Apple docs |
| Certificate pinning | Control total vía `URLSessionDelegate` | Apple docs |
| Complejidad | Alto — requiere implementar codegen, cache, subscriptions protocol desde cero. | Evaluación técnica |

## Protocolo WebSocket de AppSync

**Fuente**: https://docs.aws.amazon.com/appsync/latest/devguide/real-time-websocket-client.html

AppSync usa un protocolo **propietario** basado en `graphql-ws` (subscriptions-transport-ws)
con extensiones AppSync-specific:

```
1. WebSocket handshake → wss:// con auth base64 en query string
2. connection_init → client envía
3. connection_ack → server responde con connectionTimeoutMs
4. ka (keep-alive) → server envía periódicamente
5. start → client registra subscription con extensions.authorization
6. start_ack → server confirma (AppSync-specific)
7. data → server pushea eventos
8. stop → client cancela subscription
```

**Diferencia clave**: El mensaje `start` incluye `extensions.authorization` y las
credenciales auth se pasan durante el handshake (base64 en URL). `start_ack` es
AppSync-specific, no existe en graphql-transport-ws estándar.

## Media Loading

### Imágenes — Nuke 13

**Fuente**: https://github.com/kean/Nuke

| Aspecto | Hallazgo |
|---|---|
| Swift 6 | Requiere Swift 6.2 / Xcode 26. Fully compatible. |
| SwiftUI | `LazyImage` vía NukeUI |
| Cache | LRU memory + aggressive LRU disk + HTTP disk cache nativo |
| Formatos | Progressive JPEG, WebP, HEIF, GIF |
| Video corto | Módulo `NukeVideo` para decode/playback |
| Compile time | < 2 segundos |

### Video — AVPlayer + HLS

- `AVPlayer` soporta HLS (`.m3u8`) nativamente
- Adaptive bitrate automático — cambia calidad según red
- SwiftUI: `VideoPlayer(player:)` de AVKit
- CloudFront sirve HLS eficientemente como CDN
- Para reels/stories: preload con `AVQueuePlayer`

### AsyncImage nativo — **DESCARTADO**

- No cachea imágenes — re-descarga en cada aparición
- Causa animation hitching en listas con scroll
- Sin disk cache, sin progressive loading, sin pipeline de procesamiento

## Seguridad de Red

### Certificate Pinning

| Método | Descripción | Fuente |
|---|---|---|
| **NSPinnedDomains** (Info.plist) | Declarativo. SHA-256 del SPKI. iOS 14+. Zero code. | Apple News ID g9ejcf8y |
| **URLSession delegate** | Programático. `urlSession(_:didReceive:completionHandler:)`. Control total. | Apple docs |
| **TrustKit** (v3.0.1) | Library con reporting de violaciones. Swizzling o delegate explícito. | GitHub datatheorem |

**Recomendación**: `NSPinnedDomains` como baseline + URLSession delegate para validación runtime.

### HTTP/3 (QUIC)

| Componente | Soporte | Fuente |
|---|---|---|
| URLSession iOS 15+ | **Habilitado por defecto**. Descubrimiento automático vía Alt-Svc. | Apple TN3102 |
| CloudFront | **Soportado** en todos los edge locations. Sin costo adicional. | AWS Blog |
| Beneficio | Hasta 10% mejora en TTFB, 15% en page load. Multiplexing sin head-of-line blocking. | AWS Blog |

**Acción requerida**: Habilitar HTTP/3 en la distribución CloudFront.

### TLS

- iOS 26 usa TLS 1.3 por defecto
- App Transport Security (ATS) aplica estándar en todos los SDKs

## URLSessionWebSocketTask vs Network.framework

| Factor | URLSessionWebSocketTask | Network.framework |
|---|---|---|
| API | Simple send/receive con async/await | Low-level, state machine |
| TLS control | Vía URLAuthenticationChallenge | Vía sec_protocol_options |
| Certificate pinning | Sí | Sí |
| HTTP proxy | Automático | Manual |
| Background | Limitado | Mejor soporte |

**Recomendación para AppSync**: `URLSessionWebSocketTask` — API más simple, proxy automático,
pinning integrado. Network.framework solo si necesitamos persistencia en background.
