# CLAUDE.md — iOS reference project

> Project-governance file for a native iOS app (Swift 6 / iOS 26), published as a sanitized
> reference. Real identifiers (app name, bundle id, backend endpoints, payment vendor) are
> replaced with generic placeholders (`MyApp`, `com.example.myapp`); the engineering content —
> architecture conventions, concurrency rules, the lesson-capture discipline — is preserved as-is.
> This is how a Claude Code project is steered, not a tutorial.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**MyApp** — iOS/iPadOS native app. A social + marketplace iOS app.
- **Bundle ID:** `com.example.myapp`
- **SDK Target:** iOS 26.4+ / iPadOS 26.4+
- **Language:** Swift 6.3 (Strict Concurrency enabled)
- **Architecture:** MVVM + Repository Pattern + Clean Architecture
- **Project generation:** XcodeGen (`project.yml` → `.xcodeproj`)
- **Documentation:** See `docs/` for architecture decisions and research.

## Build & Test Commands

```bash
# Regenerate xcodeproj after changing project.yml
xcodegen generate

# Build
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build

# Run all tests (unit + UI)
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' test

# Lint & Format
swiftformat .
swiftlint lint

# Archive for App Store
xcodebuild archive \
  -project MyApp.xcodeproj -scheme MyApp \
  -archivePath ./build/MyApp.xcarchive \
  -destination 'generic/platform=iOS'

# Upload to App Store Connect
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath ./build/export \
  -allowProvisioningUpdates
```

## Module Structure

```
MyApp/
├── App/                    # @main entry point
├── Core/                   # Infrastructure (nonisolated/Sendable)
│   ├── Keychain/           # KeychainManager (deleteAll service-scoped)
│   ├── Network/            # GraphQLClient (Apollo), WebSocketClient, CertificatePinning
│   ├── Auth/               # CognitoAuthService, AppleAuthService, AuthInterceptor, InputValidator, AuthRetryPolicy, NetworkMonitor, FirstLaunchCleanup, OAuthStateGenerator, PKCEFlowState/Store, ConstantTimeCompare
│   └── Cache/              # PlayerPool, MediaPrefetcher (Nuke)
├── Domain/                 # Pure models (Sendable) + repository protocols
├── Data/                   # Repository implementations + DTOs + Mappers
├── Generated/              # Apollo codegen (nonisolated module)
├── Features/               # Views + ViewModels per feature (Onboarding, Auth, Feed, Marketplace, Chat)
└── Presentation/           # Global UI: DesignSystem + AppRouter
```

Full structure: `docs/04-module-structure.md`

## MCP Servers (7 connected)

| Server | Tools | Purpose |
|---|---|---|
| **xcode** (Apple native) | `BuildProject`, `RunAllTests`, `RenderPreview`, `DocumentationSearch`, `ExecuteSnippet` | SwiftUI previews, Apple docs search, Swift REPL |
| **XcodeBuildMCP** (getsentry) | 59 tools: build, test, devices, logs | Headless builds, test execution, device management |
| **apple-docs** | `search_apple_docs`, `get_apple_doc_content`, `search_wwdc_videos`, `get_sample_code` | Verify patterns against official Apple documentation |
| **mobile-mcp** | Screenshots, UI tap/swipe/type, app install/launch | Simulator automation and UI testing |
| **appsync** (awslabs) | AppSync resource CRUD, resolver inspection | Schema introspection, resolver analysis |
| **graphql** | `introspect-schema`, `query-graphql` | Validate GraphQL operations against schema |
| **figma** (official) | `get_design_context`, `get_screenshot`, `generate_figma_design` | Extract Figma designs, write designs back |

Setup: `claude mcp list` to verify. Config in `~/.claude.json`.

## Networking Stack

- **GraphQL:** Apollo iOS 2.x (Swift 6 safe, codegen, normalized cache)
- **Subscriptions:** URLSessionWebSocketTask (AppSync proprietary WebSocket protocol)
- **Auth:** Cognito direct HTTPS (email/password) + Hosted UI OAuth (Apple/Google/Facebook) + PKCE
- **Images:** Nuke 13 (LazyImage SwiftUI, disk+memory cache)
- **Video:** AVPlayer + HLS native (CloudFront CDN, adaptive bitrate)
- **No Amplify.** Rejected: no Swift 6 support, data races, 15-30MB binary impact.

- **On-device AI:** Foundation Models framework (iOS 26) for content tagging, summarization
- **UI Design:** Liquid Glass (iOS 26) — `.glassEffect()`, `GlassEffectContainer`
- **Security:** Post-quantum TLS 1.3 (automatic), CryptoKit HPKE, certificate pinning (NSPinnedDomains), InputValidator, AuthRetryPolicy, anti-enumeration, background snapshot masking, FirstLaunchCleanup (wipe Keychain on install/reinstall), OAuth `state` CSRF protection with atomic `PKCEFlowState`, constant-time token comparison (no `timingSafeEqual` in iOS 26 CryptoKit)
- **No Amplify.** Rejected: no Swift 6 support, data races, 15-30MB binary impact.

Decision rationale: `docs/03-architecture-decision.md`

## Agent Skills

25 custom skills in `.claude/skills/`:

| Skill | Invocation | Purpose |
|---|---|---|
| `swift-module` | `/swift-module Auth` | Scaffold files with correct isolation |
| `graphql-codegen` | `/graphql-codegen` | Apollo codegen with nonisolated workaround |
| `appsync-operation` | `/appsync-operation getName` | Create .graphql + repository + DTO |
| `app-feature` | `/app-feature Feed` | Scaffold complete feature |
| `app-auth` | `/app-auth` | Auth implementation (Cognito + Apple) |
| `app-websocket` | `/app-websocket onNewMessage` | AppSync WebSocket subscriptions |
| `app-test` | `/app-test TypeName` | Generate unit/UI tests |
| `app-security` | auto-triggered | Security review on auth/network code |
| `project-sync` | `/project-sync` | xcodegen + format + lint + build |
| `liquid-glass` | `/liquid-glass` | iOS 26 Liquid Glass UI patterns |
| `foundation-models` | `/foundation-models` | On-device AI (LanguageModelSession) |
| `apple-security-patterns` | `/apple-security-patterns` | Post-quantum crypto + passkeys |
| `adaptive-layout` | `/adaptive-layout` | iPhone vs iPad layouts (split view, grids) |
| `ios26-ux-patterns` | `/ios26-ux-patterns` | Tab collapse, zoom transitions, mini-player |
| `social-feed-patterns` | `/social-feed-patterns` | Infinite scroll, autoplay, player pool, prefetch |
| `chat-patterns` | `/chat-patterns` | Bubbles, status, typing, swipe-reply, input bar |
| `marketplace-patterns` | `/marketplace-patterns` | Cards, gallery, listing flow, filters, payments |
| `social-graph-patterns` | `/social-graph-patterns` | Profile, follow/connect, onboarding, notifications |
| `auth-test-suite` | `/auth-test-suite CognitoAuthService` | Auth unit tests, fixtures, mocks per layer |
| `auth-security-audit` | `/auth-security-audit` | Token leakage, storage, ATS, injection, concurrency |
| `auth-performance-test` | `/auth-performance-test` | Refresh latency, cold start, Keychain perf benchmarks |
| `cognito-federation` | `/cognito-federation` | Hosted UI OAuth (Apple/Google/FB), PKCE, Identity Pool |
| `ui-design-workflow` | `/ui-design-workflow` | Iterative UI design with RenderPreview + HIG + Glass rules |
| `capture-lessons` | `/capture-lessons [PR#]` | Capture lessons from a merged PR and propagate to skills + docs + memory; reminded by Stop hook |

## Architecture Conventions

- **Storage by sensitivity.** Fields that may persist (email, name, preferences, draft state) bind to `@AppStorage` via a centralized key constant (`SignUpDraft.emailKey`). Sensitive fields (password, tokens, secrets) bind to `@State` (view-local) or use `KeychainManager`. The persistence-key enum's exposed members ARE the contract — sensitive fields don't have a key by construction. Verify the negative invariant with a `Mirror(reflecting:)` test.
- **ViewModels**: `@MainActor @Observable final class` — consumed via `@State`, never `@StateObject`/`@ObservedObject`.
- **Core services**: `nonisolated final class: Sendable` (SE-0449). No actor isolation. Exception: `NetworkMonitor` is `actor` (mutable NWPathMonitor state).
- **Domain models**: `struct: Sendable` (value types).
- **Repository protocols**: `protocol: Sendable` with async methods.
- **Concurrency**: `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` project-wide (SE-0466). **EVERY type in Core/, Domain/, Data/ MUST have `nonisolated`** on its declaration — structs, enums, protocols, and classes (SE-0449). Without it, the compiler applies `@MainActor` to everything, causing cascade warnings. SE-0478 (`DefaultIsolation` typealias) is NOT available in Xcode 26.4.
- **Apollo generated code**: Must be in a separate module/target with `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` (Apollo issue #3601).
- **Keychain**: All sensitive data through `KeychainManager.shared`. Never `UserDefaults` for secrets.
- **Unit tests**: Swift Testing framework (`import Testing`, `@Test`), not XCTest.
- **UI tests**: XCTest (`XCUIApplication`).
- **Cold-start async bootstrap.** `App.init` is not async — any launch-time async work (token refresh, feed prefetch, identity-pool warmup, push registration) lives in a ViewModel `bootstrap...() async` method invoked from the WindowGroup root view's `.task { ... }`. The error path has THREE branches, not two: permanent typed error → clear/route, retryable typed error → **keep optimistic state and surface no error from the bootstrap** (the next user-initiated request exercises the in-context retry policy), unknown error → defensive fallback. Logging the user out on a transient launch-time blip is punitive UX for a problem the user neither caused nor perceived.
- **Retry block scope: validation OUTSIDE, transient-failing work INSIDE.** When wrapping an operation in `AuthRetryPolicy.execute` (or any retry/throttle/circuit-breaker wrap), preconditions that produce terminal errors (validation, URL building, percent-encoding, JSON serialization, configuration lookup) live BEFORE the retry block. Re-running terminal failures inside the retry has zero benefit and pays N× backoff latency on inputs that will never succeed. Walk top-down; for each `try` / `guard let`, ask "Does this throw with an error the policy retries on?" If no → move it above the retry block.
- **Backend-driven API changes propagate atomically across layers.** When a method signature in `Core/<X>Service.swift`, `Domain/Repositories/<X>Protocol.swift`, or the corresponding ViewModel changes BECAUSE the server contract changed (Cognito payload, AppSync schema, S3 metadata, payment-provider body, push registration body), flip ALL THREE layers in the same PR. NO `@available(*, deprecated)` shim of the old signature — the two signatures cannot both be correct when only one matches the server. The build break in mocks, repository implementations, and test fixtures IS the discovery mechanism. Internal refactors (renaming private helpers, restructuring init params for a SwiftUI View, moving a model type) MAY use deprecation; backend-driven changes never can.
- **Defense-in-depth for input validation at TWO layers.** User-input fields that reach an external service (Cognito, AppSync, S3, payment provider) run `InputValidator` checks in BOTH the ViewModel (UX rapid feedback, no network round trip on malformed input) AND the Repository (safety net for direct callers that skip the ViewModel — UI tests, scaffolded Views, helper scripts). Both layers consume the same validator function; the algorithm is shared, only the invocation sites duplicate. Fields that never leave the device (UI state, animation params, local preferences) don't need the Repository safety net.
- **Sheet anchoring above the dismiss path.** When a flow has the shape "user opens sub-View A → submits → A dismisses → sub-View B should appear next", attach `.sheet(isPresented:)` for B on the PARENT of both A and B (typically the originating View), never on A itself. SwiftUI tears down a view's modifier chain when the view dismisses; any `.sheet` attached to A dies with A. The single source of truth for B's presentation lives on the shared ViewModel; the parent reads it; either sub-View may write it. Generalizes to `NavigationStack` push-then-pop, full-screen-cover transitions, and any "submit → next step" flow.

## Code Style

- **Indent:** 2 spaces (`.swiftformat`)
- **Trailing commas:** always in multiline collections
- **Import ordering:** regular imports first, `@testable` last (Apple convention)
- **SwiftFormat** pre-build, **SwiftLint** post-compile (build phases in `project.yml`)
- **Doc comments:** `///` with `- Parameter:`, `- Returns:`, `- Throws:` (Apple DocC)
- **`@unknown default`** required on every switch over an Apple SDK or third-party enum. Apple ships new cases in minor SDK updates; the `@unknown default` branch keeps the binary forward-compatible. Project-internal enums (frozen by definition) don't need it.
- **Storage keys centralized.** `@AppStorage` / `UserDefaults` / `Info.plist` keys live in a `nonisolated enum` (e.g., `SignUpDraft.emailKey`), never inline literals. When the key set defines a security boundary (which fields are safe to persist), the enum's members ARE the contract — sensitive fields must NOT have a key, and a `Mirror(reflecting:)` test asserts no forbidden member exists.
- **UI routes by typed enum case, not by parsing the localized string.** When the View needs case-specific UX, the ViewModel exposes both a display `errorMessage: String?` and a typed `lastError: AuthError?` sibling. Pattern-match on `.serverUnavailable`, not `errorMessage.contains("…")`. The two MUST stay coherent across every lifecycle path — entry clears both, catch sets both, signOut clears both. The generic catch branch sets `lastError = .unknown(...)`, never `nil` while `errorMessage` is set. Use `ContentUnavailableView` (iOS 17+) for recurrent/systemic errors; keep inline red text for transient input errors.
- **Localization via `Localizable.xcstrings` from day 1.** `project.yml` already has `LOCALIZATION_PREFERS_STRING_CATALOGS: YES` + `STRING_CATALOG_GENERATE_SYMBOLS: YES`. Every user-facing string is a semantic dot-notation key (`auth.signIn.cta`); SwiftUI auto-localizes `Text("auth.signIn.cta")`, `Button(...)`, `TextField(placeholder, text:)`. NEVER hand-roll a `Strings.swift` enum that wraps `String(localized:)` — the build setting already provides type-safe access via the catalog.
- **Lock typed-error contracts with regression-guard tests.** When a feature consumes a project-level discriminator (`AuthError.isRetryable`, any `var foo: Bool { switch self { case .a, .b: true; default: false } }`), add a test in the CONSUMER's suite that locks in the contract from the consumer's perspective. The test asserts the discriminator's behavior on a specific case the consumer relies on; if a future contributor trims the case from the discriminator's true-set, the consumer's test goes red and forces the change to be visible in PR review instead of silently altering downstream semantics.
- **Hardcoded URLs live in a `private enum` with closure-initialized `static let`.** Never inline `URL(string: "https://…")!` at a callsite. Declare it as `private enum Name { static let value: URL = { guard let url = URL(string: "…") else { preconditionFailure("Invalid …") }; return url }() }` and read `Name.value` at the callsite. Three benefits: no force-unwrap in the diff, a clear panic message in the impossible failure mode, and one declaration site so a future regional redirect / new console / A/B-rolled endpoint touches one line, not the entire View tree. Pattern is already in use at `CognitoConfig.endpoint`.
- **Document the WHY of policy choices INLINE at the callsite.** When a callsite picks one of N syntactically-interchangeable options whose semantic differs by domain context (`AuthRetryPolicy.default` vs `.transientNetworkOnly` vs `.none`; `@AppStorage` vs `@State` vs `KeychainManager`; `@MainActor` vs `nonisolated` vs `actor`; `.task` vs `.task(id:)` vs `.onAppear`), the WHY of the choice goes inline in the method's doc comment in 2–4 sentences answering: WHICH variant, WHY this one, WHAT GOES WRONG if flipped. NOT in commit messages (squash-merge discards body), NOT in PR descriptions (degrade after migrations), NOT in READMEs (drift from code). Doc comments live with the code and travel with every refactor.
- **`os.Logger` privacy markers live in typed helper methods, not at every callsite.** When ≥3 call sites mix PII (`.private`) and semantic categories (`.public`), define typed helpers on a `nonisolated enum` namespace (e.g., `AuthLogger.logSignIn(success:latency:)`, `AuthLogger.logAuthError(_:in:)`). The helpers encapsulate the `.private` / `.public` interpolation decision; the call sites pass typed values. Forgetting `.public` on a category token (loses Console.app filter) and accidentally marking a token as `.public` (production data leak) are both invisible in review at the call site — centralizing collapses both error modes to one place.
- **Framework-only opaque types need a test-only sentinel field for bounded-behavior tests.** When a Core type holds opaque values that lack public initializers (`MXMetricPayload`, `MXDiagnosticPayload`, `XCTMetric`), unit tests cannot construct synthetic values to inject through the production code path. Add a private `sentinelCount` (or equivalent) field that mirrors the production trim/dedup/cap pathway; tests inject via `ingestSentinel(count:)` and verify via `sentinelEntryCount()`. The mirror is the test surface; the real field is the production surface; both share the trim algorithm. Document inline that the field is test-only.

## Pre-Sprint Discipline

Before starting any sprint that adds new UI:

0. **Phase 0 — Plan-time Apple-canonical recovery + anti-pattern registry consultation:** BEFORE entering plan mode, list the Apple frameworks the sprint will touch (SwiftUI, AuthenticationServices, CryptoKit, os.Logger, MetricKit, Foundation, etc.) and do TWO things in parallel:
   - **Canonical side:** for each capability, check `developer.apple.com/documentation/<framework>`, WWDC sessions (last 3 years), the SDK's `swiftinterface`, Apple Sample Code, and existing `project.yml` / xcconfig flags. Bake the findings into the plan as explicit constraints.
   - **Anti-pattern side:** `grep -i "<framework>" .claude/skills/apple-anti-patterns/SKILL.md` to surface registered anti-patterns for that framework. The sprint plan explicitly states which anti-patterns are being avoided and how. New anti-patterns discovered during the sprint get appended via `/capture-lessons` (alongside the originating lesson).

   Each PR description cites the canonical pattern it consumes AND the anti-pattern(s) it's deliberately avoiding. Required for sprints adding new infrastructure, abstractions, helpers, or wrappers; skipped for pure bug-fix or copy-change PRs. The canonical sweep prevents wrapper proposals at plan time; review-time inspection catches them at review time; the anti-pattern registry makes that side an explicit planning input. **The Plan agent's prompt MUST require BOTH the canonical sweep AND the anti-pattern registry grep BEFORE proposing architecture.**

   **Phase 0.5 — Question plan-silent specific details:** at the start of each PR within the sprint, scan the plan for the PR's scope and flag any domain-specific detail the plan does NOT explicitly cover (wire format, name composition, ID encoding, locale rule, currency rule, timezone default, sort order, pagination cursor shape, vendor attribute name). The plan's silence is a SIGNAL — surface the detail in a "Decision not in the plan" section of the PR description, ask the user / plan author during execution if there's any doubt, and lock the chosen format with a regression-guard test + inline doc-comment. Never give plan-silent details by default.
1. **Dedup-before-features:** if ≥2 new Views in the same domain, FIRST extract shared building blocks to `Presentation/DesignSystem/<domain>/` (gradient, input field style, primary button, logo header). Inline duplication multiplies as the surface grows.
2. **String Catalog:** every user-facing string is a key in `Localizable.xcstrings`. No inline literals after the first PR with ≥10 user-facing strings.
3. **Verify project.yml:** before proposing infrastructure, grep `project.yml` and `xcconfig` files for related flags. Apple build settings active often signal features partially wired.
4. **Apple-canonical first:** never wrap an Apple API without checking the SDK's canonical idiom first. Search `developer.apple.com/documentation`, WWDC sessions (last 3 years), the SDK's `swiftinterface`, and Apple Sample Code BEFORE designing a custom abstraction.
5. **`git status` before `git checkout -b`:** unexpected staged changes / untracked files / accidental renames must be resolved before creating a new branch — never build on a corrupted state.

### Per-PR Micro-Sweep

The Phase 0 sprint sweep covers framework existence and headline patterns; per-PR sweeps cover the **how-to-consume** detail at the integration boundary. Required when a PR adds a NEW consumer for an Apple framework (subscriber to an AsyncStream, adopter of a property wrapper, embedder of a SwiftUI modifier chain, override of a protocol-required method, etc.). Spend 5–10 min BEFORE editing code:

```bash
# 1. Apple docs / WWDC for the integration boundary (consumer pattern, lifecycle)
# 2. Anti-pattern registry by framework name
grep -i "<framework>" .claude/skills/apple-anti-patterns/SKILL.md
# 3. Compare planned approach vs canonical / anti-pattern entries
# 4. If mismatch → REVISE THE PLAN before coding; document correction in PR description
```

The canonical example: a sprint plan carried `private var connectivityTask: Task<Void, Never>?` on the ViewModel (a registered anti-pattern); the per-PR sweep caught it before any code shipped, and the corrected approach removed -3 files, +0 stored Task, +0 new ViewModel method.

### UX Optimism Asymmetry

- **Bootstrap / automatic flows:** retryable typed errors → KEEP authenticated state, surface no error from the bootstrap path itself. Punitive UX (logout) for user-not-perceived-cause is wrong.
- **User-initiated actions on visible-error state:** if the View renders a banner / `ContentUnavailableView` / toast announcing the error AND the action's prerequisite is exactly that error condition, DISABLE the control via `.disabled(viewModel.<errorFlag>)` referencing the SAME flag the surface renders against. Optimism for explicit user-initiated requests over a known-broken prerequisite is just a different flavor of punitive UX (wasted tap, redundant error toast competing with banner).

## Git Safety Protocol

Before every `git checkout -b <new-branch>`:

```bash
git status                # any uncommitted/staged changes?
git diff --stat           # what would they touch?
git log -3 --oneline      # are we where we think we are?
git remote -v             # right upstream?
```

If anything looks unexpected, ask the user before proceeding. Destructive git actions (`reset --hard`, `clean -f`, `force push`, `push --force-with-lease` to main) ALWAYS require user confirmation. Corrupted local state can be the user's in-progress work — never assume.

## Tooling

- **XcodeGen** — `project.yml` is source of truth. Never edit `.xcodeproj` manually.
- **SwiftLint** 0.63+ — `.swiftlint.yml`. `trailing_comma` disabled (SwiftFormat handles it).
- **SwiftFormat** 0.60+ — `.swiftformat`. `testable-last`, `organizeDeclarations`, `markTypes`.
- **Apollo Codegen** — Generates typed Swift from GraphQL schema. Custom scalar: `AWSDateTime` → `String`.

## Lesson Capture Pipeline

Every PR merged to `develop` should be followed by a captured-lessons PR if it contains compiler quirks, security findings, or architectural patterns worth preserving. The pipeline is enforced by:

1. **`/capture-lessons` skill** — reads the target PR diff, proposes lessons (gated by user), and propagates them to skills + repo docs + project memory.
2. **`Stop` hook in `.claude/settings.json`** — after every Claude response, prints a yellow warning if the latest feature merge to `develop` (PR > BASELINE_PR=N) has no corresponding `chore(lessons): capture PR#N` commit. Hook is informational, never blocks.
3. **Capture PR title convention** — `chore(lessons): capture PR#<N>`. Squash merges discard the body, so the commit subject is the only durable signal.

The pipeline is designed to be **invoked once per merge** with human gating until 5 successful captures unlock `--auto`. See `.claude/skills/capture-lessons/SKILL.md` for the full workflow.

## Backend Context

- **API:** AWS AppSync (GraphQL) with Lambda resolvers (Go)
- **Auth:** Cognito User Pool. idToken in Authorization header → AppSync validates JWT → Lambda receives identity claims. Hosted UI OAuth on `auth.example.com`.
- **Storage:** S3 + CloudFront CDN (multimedia)
- **Push:** Pinpoint → APNS
- **Payments:** a third-party payment provider (redirect-based checkout). Physical services exempt from IAP (Guideline 3.1.3e).
- **GraphQL schema:** `<backend-repo>/infra/appsync/schema.graphql`

Full context: `docs/01-mvp-context.md`
Auth module: `docs/09-auth-module.md` (31 files, 102 tests, input validation, retry policy, network monitoring, first-launch keychain cleanup, OAuth state CSRF, constant-time validation)
