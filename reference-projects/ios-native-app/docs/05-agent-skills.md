# 05 — Agent skills reference (iOS reference)

> The custom Claude Code agent skills that scaffold and guard a native iOS app (Swift 6 /
> iOS 26), published as a sanitized reference. Brand-named skills are renamed generically
> (`app-feature`, `app-auth`, …); the dependency graph, the compiler-verified Swift 6 lessons,
> and the WWDC citations are preserved verbatim — that engineering is the point.

23 custom Claude Code agent skills for the MyApp iOS project.
Located in `.claude/skills/`.

## Tier 1: Foundation

| Skill | Invocación | Función |
|---|---|---|
| `swift-module` | `/swift-module Auth` | Scaffold archivos con actor isolation correcto por capa |
| `graphql-codegen` | `/graphql-codegen` | Apollo codegen con workaround nonisolated (issue #3601) |
| `appsync-operation` | `/appsync-operation getAllActivePosts` | Crear .graphql + repository + DTO + mapper |

## Tier 2: Feature Development

| Skill | Invocación | Función |
|---|---|---|
| `app-feature` | `/app-feature Feed` | Scaffold feature completa (View + ViewModel + Repository) |
| `app-auth` | `/app-auth` | Cognito + Sign in with Apple + passkey upgrade |
| `app-websocket` | `/app-websocket onNewMessage` | AppSync subscriptions con URLSessionWebSocketTask |

## Tier 3: Quality & DevOps

| Skill | Invocación | Función |
|---|---|---|
| `app-test` | `/app-test KeychainManager` | Unit tests (Swift Testing) o UI tests (XCTest) |
| `app-security` | auto-invocado | Review seguridad al tocar auth/network/keychain |
| `project-sync` | `/project-sync` | xcodegen + swiftformat + swiftlint + build verify |

## Tier 4: iOS 26 Native Patterns

| Skill | Invocación | Función |
|---|---|---|
| `liquid-glass` | `/liquid-glass` | Liquid Glass UI patterns (.glassEffect, GlassEffectContainer) |
| `foundation-models` | `/foundation-models` | On-device AI (LanguageModelSession, @Generable, Tool) |
| `apple-security-patterns` | `/apple-security-patterns` | Post-quantum CryptoKit, TLS 1.3, passkeys, pinning |

## Tier 5: UX/UI Patterns

| Skill | Invocación | Función |
|---|---|---|
| `adaptive-layout` | `/adaptive-layout` | iPhone vs iPad layouts (NavigationSplitView, size classes, grids) |
| `ios26-ux-patterns` | `/ios26-ux-patterns` | Tab collapse, zoom transitions, safe area bars, mini-player, autoplay |

## Tier 6: Industry Patterns (Swift 6.3 / iOS 26.4 native)

| Skill | Invocación | Función |
|---|---|---|
| `social-feed-patterns` | `/social-feed-patterns` | Infinite scroll, cursor pagination, video autoplay, AVPlayer pool, prefetch, skeleton loading, optimistic updates, offline cache |
| `chat-patterns` | `/chat-patterns` | Message bubbles, status indicators, typing, swipe-to-reply, expandable input bar, scroll-to-bottom, reactions, local-first sending |
| `marketplace-patterns` | `/marketplace-patterns` | Product cards with glass, gallery, booking wizard, guest count, date picker, filters, a third-party payment provider integration |
| `social-graph-patterns` | `/social-graph-patterns` | Profile page, stats bar, follow/connect states, posts grid, onboarding flow, notification center, block/report |

## Tier 7: Auth Testing

| Skill | Invocación | Función |
|---|---|---|
| `auth-test-suite` | `/auth-test-suite CognitoAuthService` | Unit tests per layer, ViewModel state tests, fixtures (JWT, Cognito, Apple), mocks |
| `auth-security-audit` | `/auth-security-audit` | Token leakage, insecure storage, ATS, injection/XSS, boundary, concurrency, lifecycle |
| `auth-performance-test` | `/auth-performance-test` | Token refresh latency (p50<1s), cold start (<50ms), Keychain r/w (<10ms), coalescence |

## Dependency Graph

```
swift-module ─────┬──→ app-feature ──→ app-test
graphql-codegen ──┤        │
appsync-operation ┘        ├──→ liquid-glass
                           ├──→ adaptive-layout
                           ├──→ ios26-ux-patterns
                           ├──→ social-feed-patterns
                           ├──→ chat-patterns
                           ├──→ marketplace-patterns
                           └──→ social-graph-patterns
                       app-auth ──→ app-security ──→ apple-security-patterns
                           │              │
                           ▼              ▼
                       auth-test-suite ←→ auth-security-audit
                           │                    │
                           └──→ auth-performance-test ←┘
                       app-websocket ──→ chat-patterns
                       foundation-models (independent)
                       project-sync (independent)
```

## Design Rules (all skills)

1. Core/Data services: `nonisolated final class: Sendable` (SE-0449). Domain structs `Sendable` by inference. **Do NOT use SE-0478 typealias — not available in Xcode 26.4.**
2. Features/, Presentation/: `@MainActor` default (no override)
3. No Amplify SDK anywhere
4. SwiftFormat post-generation on all created Swift files
5. Doc comments: `///` with `- Parameter:`, `- Returns:`, `- Throws:` (Apple DocC)
6. Views must follow Liquid Glass patterns (iOS 26) — see `liquid-glass`
7. Views must be adaptive iPhone/iPad — see `adaptive-layout`
8. Views must use iOS 26 UX patterns — see `ios26-ux-patterns`
9. Security code triggers `app-security` auto-review

## Compiler-Verified Lessons (applied to all skills)

1. **SE-0478 broken**: `private typealias DefaultIsolation = nonisolated` does NOT compile in Xcode 26.4. Use `nonisolated` on type declarations (SE-0449).
2. **DTOs**: External API JSON with PascalCase keys → use `CodingKeys` mapping to camelCase Swift properties.
3. **Test imports**: `@testable import` does NOT transitively import frameworks. Add explicit `import Foundation`, `import AuthenticationServices`, etc.
4. **Actor closures**: Closures used inside `Task {}` in actors require `@escaping`.
5. **Mock protocols**: Non-async functions satisfy async protocol requirements. Remove `async` from mocks that don't use `await`.
6. **Unused params**: Use `_` external labels in mock protocol conformance.
7. **SwiftLint disable**: No inline text after rule name (`-- comment` breaks). Use a separate comment line above.
8. **Nesting**: `CodingKeys` inside nested struct = 3 levels → extract inner types to top level.
9. **Force unwrap**: Use `guard let` + `preconditionFailure` instead of `!`.
10. **Compile incrementally**: Build after each phase, not in bulk.
11. **EVERY non-UI type needs `nonisolated`**: structs, enums, protocols, classes in Core/Domain/Data. The compiler applies `@MainActor` to ALL types. Without `nonisolated`, `.shared` singletons become `@MainActor`, causing "call to main actor-isolated initializer" warnings.
12. **SwiftLint CLI ≠ Xcode warnings**: CLI may show 0 while Xcode shows 19. Always verify with `xcodebuild clean && xcodebuild build 2>&1 | grep "warning:"`.
13. **Extensions do NOT inherit `nonisolated`**: A `nonisolated struct Foo` does NOT make `extension Foo { ... }` nonisolated. Mark extensions as `nonisolated extension Foo { ... }`.
14. **SignInWithAppleButton constraints**: Internal `width <= 375`, `height <= 64`. Use `frame(maxWidth: 375, maxHeight: 60)`.
15. **ASAuthorizationError.unknown (Code=1000)**: Silently ignore alongside `.canceled` and `.notInteractive` — fires in simulator without Apple ID, causes re-render loops if errorMessage is set.
16. **Error enums need `Equatable`**: Swift Testing `#expect(throws:)` macro requires the error type to conform to `Equatable`. Add conformance to all error enums.
17. **Test functions using `await` must be `async throws`**: Missing `async` causes "async call in a function that does not support concurrency" — won't compile.
18. **`INFOPLIST_KEY_*` only works with Apple-predefined keys**: Custom keys need a custom `Info.plist` with `$(VARIABLE)` syntax. Flow: xcconfig → Info.plist → `Bundle.main.infoDictionary`.
19. **`WebAuthenticationSession` is a SwiftUI cross-import overlay**: Only available when both `AuthenticationServices` AND `SwiftUI` are imported. Keep OAuth flow in View layer, pass code+verifier to ViewModel.
20. **`.onOpenURL` goes on View, not Scene**: `WindowGroup { }.onOpenURL` fails — must be `WindowGroup { Group { }.onOpenURL }`.

## Skills Added After Initial Set

| Skill | Invocation | Tier |
|---|---|---|
| `swift-error-patterns` | `/swift-error-patterns` | Design Patterns — LocalizedError, performAuth, exhaustive switch |
| `cognito-federation` | `/cognito-federation` | Auth — Hosted UI OAuth, PKCE, Identity Pool, token revocation |

## Design Patterns (verified)

1. **Exhaustive switch on enums is correct** regardless of case count. Apple's `URLError.Code` has 30+. Compiler optimizes to O(1). Do NOT replace with dictionaries.
2. **`LocalizedError` over custom `userMessage`** — conforming to `LocalizedError.errorDescription` integrates with `localizedDescription`, SwiftUI `.alert`, and Cocoa error pipeline.
3. **`performAuth()` helper in ViewModels** — extracts repeated try/catch/loading into a single method. Reduces duplication by ~30%.
4. **Error enums: one per domain** — `AuthError`, `FeedError`, `ChatError`. Not a single `AppError`.

## Sources

- Swift 6.2 concurrency: SE-0449, SE-0466 (SE-0478 NOT available in Xcode 26.4)
- Apollo issue #3601: MainActor isolation conflict with generated code
- AppSync WebSocket: https://docs.aws.amazon.com/appsync/latest/devguide/real-time-websocket-client.html
- Liquid Glass: WWDC25 session 323
- Foundation Models: WWDC25 session 286
- Post-quantum crypto: WWDC25 session 314
- Passkeys: WWDC25 session 279
- iPad app design: WWDC25 session 208
- iOS 26 design system: WWDC25 session 356
