# 09 — Auth module implementation (iOS reference)

> A production authentication module for a native iOS app — email/password + federated
> (Apple / Google / Facebook) over Cognito Hosted UI with PKCE, OAuth `state` CSRF protection,
> atomic flow persistence, constant-time token comparison, and first-launch Keychain hygiene.
> Published as a sanitized reference: endpoints and deeplink scheme are placeholders
> (`app://`, `auth.example.com`); the security engineering and the 102-test suite are intact.

Complete authentication module for MyApp iOS — email/password + federated
(Apple, Google, Facebook) via Cognito Hosted UI.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│ Features/Onboarding/                                                 │
│   OnboardingView — 4-page Liquid Glass, Figma PDF illustrations      │
│                    Adaptive: iPhone/iPad, portrait/landscape          │
│                    Light/dark logo (Asset Catalog luminosity)         │
├─────────────────────────────────────────────────────────────────────┤
│ Features/Auth/                                                       │
│   LoginView → AuthViewModel → AuthRepository                        │
│   SSOButtonsView → WebAuthenticationSession → CognitoFederationService│
│   MainTabView → AppRouter (deep links)                               │
├─────────────────────────────────────────────────────────────────────┤
│ Core/Auth/                                                           │
│   CognitoAuthService     — email/password (USER_PASSWORD_AUTH)       │
│   CognitoFederationService — OAuth code exchange + PKCE + revocation │
│   IdentityPoolService    — AWS credentials (S3, Pinpoint)            │
│   AuthTokenStore         — Keychain persistence                      │
│   AuthInterceptor        — Auto-refresh + header injection           │
│   AppleAuthService       — Credential state monitoring               │
│   PKCEGenerator          — RFC 7636 code verifier/challenge          │
│   AppConfig              — xcconfig → Info.plist → runtime           │
│   AuthError              — LocalizedError with Cognito mapping       │
│   FederationProvider     — Apple/Google/Facebook enum                 │
├─────────────────────────────────────────────────────────────────────┤
│ Domain/                                                              │
│   User                   — nonisolated struct: Sendable              │
│   AWSCredentials         — temporary AWS credentials                 │
│   AuthRepositoryProtocol — interface (signIn, federation, signOut)    │
├─────────────────────────────────────────────────────────────────────┤
│ Data/                                                                │
│   AuthRepository         — orchestrates Core services                │
│   CognitoResponseDTOs    — InitiateAuth response (CodingKeys)        │
│   OAuthTokenResponseDTO  — /oauth2/token response (snake_case)       │
└─────────────────────────────────────────────────────────────────────┘
```

## Files (31 production + 18 test)

### Core/Auth/ (17 files)
| File | Type | Responsibility |
|---|---|---|
| `AppConfig.swift` | `nonisolated enum` | Reads xcconfig values from Info.plist |
| `AuthError.swift` | `nonisolated enum: LocalizedError` | Unified error + timeout/noInternet/serverUnavailable + isRetryable + URLError mapping + anti-enumeration + `invalidOAuthState` |
| `AuthInterceptor.swift` | `nonisolated final class: Sendable` | Auto-refresh + header injection + graceful token degradation in soft window |
| `AuthRetryPolicy.swift` | `nonisolated struct: Sendable` | Exponential backoff (3 attempts, 1s base, 2x, 8s max) + 25% jitter |
| `AuthTokenStore.swift` | `nonisolated final class: Sendable` | Keychain persistence (JSON blob, atomic) + `wipeAll()` for first-launch reset |
| `AppleAuthService.swift` | `nonisolated final class: Sendable` | Credential revocation monitoring only |
| `CognitoAuthService.swift` | `nonisolated final class: Sendable` | Direct HTTPS + timeout 15s/30s + URLError catch + HTTP 5xx + retry on refresh |
| `CognitoFederationService.swift` | `nonisolated final class: Sendable` | OAuth + URLComponents fix + code validation + timeout 30s + `state:` param |
| `CognitoTokens.swift` | `nonisolated struct: Codable` | Token model + JWT decode (manual Base64URL) |
| `ConstantTimeCompare.swift` | `nonisolated enum` | XOR-accumulate constant-time equality for tokens (no `timingSafeEqual` in iOS 26 CryptoKit) |
| `FederationProvider.swift` | `nonisolated enum: CaseIterable, Codable` | Apple/Google/Facebook provider names (Codable for `PKCEFlowState` persistence) |
| `FirstLaunchCleanup.swift` | `nonisolated enum` | Wipes Keychain on first launch after install/reinstall (closure-injected wipe + UserDefaults flag) |
| `IdentityPoolService.swift` | `nonisolated final class: Sendable` | GetId + GetCredentialsForIdentity |
| `InputValidator.swift` | `nonisolated enum` | Email/password/name/code validation + sanitization |
| `NetworkMonitor.swift` | `actor` | NWPathMonitor + isConnected + connectivityUpdates() AsyncStream |
| `OAuthStateGenerator.swift` | `nonisolated enum` | 32B Base64URL CSRF state for OAuth |
| `PKCEFlowState.swift` | `nonisolated struct: Codable` + `final nonisolated class: Sendable` | Atomic OAuth flow snapshot (verifier+state+provider+createdAt) with 10-min TTL Keychain store |
| `PKCEGenerator.swift` | `nonisolated enum` | SHA256 code_challenge + random code_verifier |

### Domain/ (3 files)
| File | Type |
|---|---|
| `User.swift` | `nonisolated struct: Sendable, Identifiable` |
| `AWSCredentials.swift` | `nonisolated struct: Sendable` |
| `AuthRepositoryProtocol.swift` | `nonisolated protocol: Sendable` |

### Data/ (3 files)
| File | Type |
|---|---|
| `AuthRepository.swift` | `nonisolated final class: Sendable` |
| `CognitoResponseDTOs.swift` | `nonisolated struct: Decodable` (CodingKeys PascalCase) |
| `OAuthTokenResponseDTO.swift` | `nonisolated struct: Decodable` (CodingKeys snake_case) |

### Features/ (4 files)
| File | Type |
|---|---|
| `AuthViewModel.swift` | `@MainActor @Observable final class` |
| `LoginView.swift` | `struct: View` (Liquid Glass adaptive: portrait/landscape, iPhone/iPad, light/dark) |
| `SSOButtonsView.swift` | `struct: View` (GlassEffectContainer + `.glass` buttons, WebAuthenticationSession) |
| `MainTabView.swift` | `struct: View` (5-tab placeholder) |

### Presentation/ (1 file)
| File | Type |
|---|---|
| `AppRouter.swift` | `@MainActor @Observable` + `DeepLink` enum + `AppTab` enum |

### Config/ (3 files)
| File | Purpose |
|---|---|
| `Base.xcconfig` | Shared settings |
| `Dev.xcconfig` | Dev environment (Cognito <region>) |
| `Prod.xcconfig` | Production environment (placeholders) |

## Auth Flows

### Email/Password
```
LoginView → AuthViewModel.signIn() → AuthRepository.signIn()
  → CognitoAuthService.signIn() → POST cognito-idp.<region>.amazonaws.com
  → CognitoTokens → AuthTokenStore.save() → Keychain
  → isAuthenticated = true → MainTabView
```

### Federation (Apple/Google/Facebook) — with CSRF protection
```
SSOButtonsView.startFederation
  → PKCEGenerator.generate() + OAuthStateGenerator.generate()
  → PKCEFlowStore.save(verifier+state+provider+createdAt) → atomic Keychain blob
  → WebAuthenticationSession.authenticate(authorizeURL with &state=...)
  → User authenticates → 4 Lambda triggers run server-side
  → app://signup?code=XXX&state=YYY

SSOButtonsView.processCallback (defer { PKCEFlowStore.shared.clear() })
  → extract code + state from callback
  → PKCEFlowStore.load() → throws on missing/expired (auto-clears)
  → ConstantTimeCompare.equals(returnedState, storedFlow.state)
  → mismatch: AuthError.invalidOAuthState (Spanish: "Validación de seguridad falló")
  → match: completeFederatedSignIn(code, codeVerifier: storedFlow.codeVerifier)
  → CognitoFederationService.exchangeCodeForTokens()
  → POST auth.example.com/oauth2/token (code + code_verifier)
  → CognitoTokens → Keychain → isAuthenticated = true
```

Three independent guarantees:
- **CSRF (RFC 6749 §10.12)** — fresh `state` per flow, validated client-side because `ASWebAuthenticationSession` does NOT validate it.
- **Atomicity** — verifier + state persist as one Keychain blob; crash mid-flow leaves either complete state or nothing, never half-saved.
- **Defense-in-depth** — token exchange uses verifier from the *stored* flow, not the local scope, so a corrupted Keychain fails the exchange instead of silently bypassing the state check.
- **Constant-time validation** — `ConstantTimeCompare.equals` (XOR + reduce) replaces `String.utf8.elementsEqual` to avoid leaking prefix-match length. CryptoKit on iOS 26.4 has no public `timingSafeEqual`; only `HMAC.isValidAuthenticationCode` is timing-safe and MAC-specific.

### Sign Out
```
AuthViewModel.signOut() → AuthRepository.signOut()
  → CognitoFederationService.revokeToken() (server-side revocation)
  → AuthTokenStore.clear() (Keychain)
  → isAuthenticated = false → LoginView
```

### First Launch (Install / Reinstall)
```
MyAppApp.init() → FirstLaunchCleanup.runIfNeeded()
  → if !UserDefaults("hasLaunchedBefore"):
       AuthTokenStore.shared.wipeAll()  (KeychainManager.deleteAll service-scoped)
       UserDefaults("hasLaunchedBefore") = true
```
iOS Keychain items survive uninstall on the same device. UserDefaults DOES clear on
uninstall. Combining a `UserDefaults` flag with a Keychain wipe gives every fresh install
a clean slate (revoked tokens, deleted accounts, signed-out-elsewhere states are honored
on relaunch). The wipe is filtered strictly by `kSecAttrService = bundle id` and treats
`errSecItemNotFound` as success — idempotent across crashes.

### Deep Links
```
app://signin → AppRouter → signOut
app://payment/<id> → switch to Marketplace tab
app://post/<id> → switch to Feed tab
app://listing/<id> → switch to Marketplace tab
app://profile/<username> → switch to Profile tab
```

## Test Coverage (102 tests, 15 suites)

| Suite | Tests | Coverage |
|---|---|---|
| CognitoAuthServiceTests | 5 | Sign in, headers, error mapping (parameterized), refresh |
| CognitoFederationServiceTests | 3 | Authorize URL contains state, verbatim, S256 + other params |
| CognitoTokensTests | 8 | Properties, JWT decode, malformed tokens, Codable |
| ConstantTimeCompareTests | 12 | Identical/empty equality, length mismatch, mismatch in any byte position, real 32-byte state, multibyte UTF-8 |
| KeychainManagerTests | 6 | CRUD, protection level audit, `deleteAll` service-scoped |
| AuthTokenStoreTests | 8 | Token persistence, Apple credential cache |
| AuthInterceptorTests | 3 | Valid token, header injection, refresh trigger, soft-window degradation |
| AuthRetryPolicyTests | 5 | First-attempt success, retry-then-succeed, exhaustion, non-retryable, none policy |
| AuthViewModelTests | 5 | State transitions, sign in/out, sign up |
| InputValidatorTests | ~30 | Email/password/name/OAuth code/verification code, sanitization |
| FirstLaunchCleanupTests | 4 | First launch wipes + sets flag, subsequent skip, idempotent, `deleteAll` not-found = success |
| OAuthStateGeneratorTests | 7 | Non-empty, uniqueness, Base64URL charset, no padding, 43-char, 100-iter entropy |
| PKCEFlowStoreTests | 7 | Round-trip, notFound, clear, overwrite, expired auto-clear, isExpired boundary |
| (Fixtures) | — | JWTFactory, CognitoResponseFactory, TestTags |
| (Mocks) | — | MockHTTPClient, MockAuthRepository |

## Configuration

```
Dev.xcconfig → Info.plist $(VARIABLE) → AppConfig.infoPlistValue(for:) → CognitoConfig
```

Values: COGNITO_REGION, COGNITO_USER_POOL_ID, COGNITO_CLIENT_ID, COGNITO_DOMAIN,
COGNITO_IDENTITY_POOL_ID, APPSYNC_ENDPOINT, CLOUDFRONT_DOMAIN

## Lessons Learned

Key lessons from Auth + Onboarding + Security + first sprint + timing-safe fix:
1. SE-0478 broken — use `nonisolated` on every type (not typealias)
2. EVERY non-UI type needs `nonisolated` — structs, enums, protocols, classes
3. Extensions don't inherit `nonisolated` — use `nonisolated extension`
4. WebAuthenticationSession is SwiftUI cross-import — keep in View layer
5. SignInWithAppleButton constraints: maxWidth 375, maxHeight 60
6. ASAuthorizationError.unknown must be silently ignored
7. INFOPLIST_KEY_* only works for Apple-predefined keys
8. LocalizedError over custom userMessage
9. performAuth() helper eliminates try/catch duplication
10. URLComponents (not string concat) for form-urlencoded
11. Foundation.pow is @MainActor — use manual loop in nonisolated context
12. Anti-enumeration: UserNotFoundException → invalidCredentials — same message as wrong password
13. Keychain survives uninstall — wipe on first launch via UserDefaults flag
14. Closure injection (`wipe: () -> Void`) > full service injection for tests
15. UserDefaults(suiteName: UUID) per test for isolation
16. No force-unwrap on `UserDefaults(suiteName:)!` — guard with `.standard` fallback
17. `simctl erase all` invalidates simulator UDIDs — prefer `name=` over `id=`
18. **OAuth `state` parameter is mandatory and the client must validate it**
19. **Bundle PKCE verifier + state into atomic `PKCEFlowState` Keychain blob with TTL**
20. **Token exchange uses verifier from STORED flow, not local scope (defense-in-depth)**
21. **No `CryptoKit.timingSafeEqual` in iOS 26 — implement XOR + reduce manually**
22. **SwiftFormat strip-implicit-sendable removes redundant `Sendable` — trust compiler inference**

Full list: the project's lesson log
