---
name: apple-anti-patterns
description: |
  Append-only registry of Apple-framework anti-patterns, each paired with the canonical
  alternative and a citation to Apple's documented guidance. Consulted at sprint Phase 0
  sweep and per-PR micro-sweep BEFORE proposing architecture or wiring. Never delete
  entries; append exceptions if a context-specific reversal is later justified.
version: "1.0.0"
color: red
freshness:
  verified_against:
    - source: "Apple Developer Documentation — Observation framework (AP-5 @Observable over ObservableObject)"
      url: "https://developer.apple.com/documentation/observation"
      version: "iOS 17+"
    - source: "Apple HIG — Loading and progress (AP-2 disable controls on unavailable conditions)"
      url: "https://developer.apple.com/design/human-interface-guidelines/loading"
      version: "iOS 26"
    - source: "WWDC25 Session 266 — Explore Concurrency in SwiftUI (AP-1 .task AsyncSequence consumer)"
      url: "https://developer.apple.com/videos/play/wwdc2025/266/"
      version: "WWDC25"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual Apple API surface change)"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Apple Framework Anti-Pattern Registry

Anti-patterns this project has encountered (or narrowly avoided), each paired
with its canonical alternative and a citation to Apple's documented guidance.
This is the **anti-pattern side** of Phase 0 / per-PR canonical sweeps
(Lessons #56 + #58 give the canonical-pattern side; Lesson #60 introduces this
registry as the matching anti-pattern side).

## When to Consult

**Required (BEFORE coding):**

1. **Sprint kickoff Phase 0 sweep (L56).** For each Apple framework the sprint
   touches, grep this file by framework name. List the registered anti-patterns
   in the sprint plan along with the chosen canonical alternative.
2. **Per-PR micro-sweep (L58).** For each PR that introduces a new integration
   (consumer of an actor, subscriber to an AsyncStream, adopter of a property
   wrapper, embedder of a SwiftUI modifier chain), grep this file by the
   framework being integrated. The PR description states which anti-patterns
   were considered and how they were avoided.
3. **`/capture-lessons` propagation.** When a new anti-pattern is identified
   in PR review or post-merge, append the entry here as part of the same
   capture run that adds the originating lesson to memory.

**Optional:** Code-review checks; refactor planning where the diff touches an
Apple framework integration.

```bash
# Quick lookup by framework:
grep -i "<framework>" .claude/skills/apple-anti-patterns/SKILL.md
# Examples:
# grep -i "asyncstream\|swiftui" .claude/skills/apple-anti-patterns/SKILL.md
# grep -i "observable\|combine" .claude/skills/apple-anti-patterns/SKILL.md
```

## Format

Each entry:

```markdown
### AP-N: <one-line anti-pattern, imperative DON'T form>

**Frameworks:** <list>
**Origin:** Lesson #M (link / commit / PR)
**Status:** active | exception (with rationale)

**Don't:**
\`\`\`swift
// short code example showing the WRONG pattern
\`\`\`

**Do (canonical):**
\`\`\`swift
// short code example showing the CORRECT pattern
\`\`\`

**Why:** <root cause — Apple docs / WWDC session reference>

**Where it bites:** <file types / contexts where the anti-pattern silently
compiles but breaks at runtime, in tests, or in App Store review>
```

Numbering is monotonic (AP-1, AP-2, ...). Never reuse a number. If an entry is
later found to be wrong (false anti-pattern), mark `Status: exception` with a
dated rationale and keep the entry — the historical record matters.

---

## Active Entries

### AP-1: Don't store a `Task<Void, Never>?` on an `@Observable` ViewModel for an AsyncStream subscription

**Frameworks:** SwiftUI (Observation), Concurrency, Network, AsyncSequence consumers in general
**Origin:** Lesson #58 (Sprint 2 PR #2 — NetworkMonitor wiring)
**Status:** active

**Don't:**
```swift
@MainActor @Observable
final class FeatureViewModel {
  var isOffline = false
  private var connectivityTask: Task<Void, Never>?

  func observeConnectivity() {
    connectivityTask = Task { [weak self] in
      for await isConnected in await NetworkMonitor.shared.connectivityUpdates() {
        self?.isOffline = !isConnected
      }
    }
  }
  // Now you must remember to cancel `connectivityTask` somewhere — and
  // there's no SwiftUI hook that fires when @Observable instances die,
  // because @Observable is a class with @State retention semantics.
}
```

**Do (canonical):**
```swift
@MainActor @Observable
final class FeatureViewModel {
  var isOffline = false
  // No stored Task. The View owns the subscription lifetime.
}

struct FeatureView: View {
  @Bindable var viewModel: FeatureViewModel
  var body: some View {
    content
      .task {
        for await isConnected in await NetworkMonitor.shared.connectivityUpdates() {
          viewModel.isOffline = !isConnected
        }
      }  // SwiftUI auto-cancels on disappearance.
  }
}
```

**Why:** WWDC21 ["Discover Concurrency in SwiftUI"](https://developer.apple.com/videos/play/wwdc2021/10019/)
and WWDC25 ["Explore Concurrency in SwiftUI"](https://developer.apple.com/videos/play/wwdc2025/266/)
both show `View.task { for await ... in stream { ... } }` as the canonical
consumer for AsyncSequence in SwiftUI. The `.task` modifier is the only built-in
mechanism that ties the loop's cancellation to view appearance/disappearance.
Storing the Task on the ViewModel transfers cancellation responsibility to your
code and creates orphaned tasks across `@Observable` re-creation (which can
happen during scene reconfigurations, NavigationStack mutations, and tab
switches). The ViewModel-as-`@MainActor` lets you write `viewModel.x = y`
directly inside the loop without `MainActor.run` — no isolation gymnastics
required.

**Where it bites:** Any SwiftUI feature that consumes `AsyncStream` from
`Network.framework`, `EventKit`, `MetricKit`, custom actors, or third-party
async iterators. The ViewModel-stored Task pattern was inherited from the
older `@StateObject` + `Set<AnyCancellable>` Combine idiom; it does NOT
translate to `@Observable` + AsyncStream.

---

### AP-2: Don't dispatch user-initiated submit actions when the View renders a visible-error state contradicting the action's prerequisite

**Frameworks:** SwiftUI, application-level UX
**Origin:** Lesson #59 (Sprint 2 PR #2 — offline banner + login button)
**Status:** active

**Don't:**
```swift
struct LoginView: View {
  var body: some View {
    VStack {
      if viewModel.isOffline {
        OfflineBanner()              // Tells user "you're offline"
      }
      Button("Sign in") {
        Task { await viewModel.signIn(...) }  // Tries anyway — pointless retry
      }
      // Button stays enabled; user taps, gets serverUnavailable 3s later.
    }
  }
}
```

**Do (canonical):**
```swift
struct LoginView: View {
  var body: some View {
    VStack {
      if viewModel.isOffline {
        OfflineBanner()
      }
      Button("Sign in") {
        Task { await viewModel.signIn(...) }
      }
      .disabled(viewModel.isOffline)   // Predicate matches banner's flag.
    }
  }
}
```

**Why:** Apple HIG ["Loading and progress"](https://developer.apple.com/design/human-interface-guidelines/loading)
explicitly: "If a user action depends on a system condition that's unavailable,
disable the control and tell the user why." Lesson #57 mandates optimism for
**automatic** flows (bootstrap, periodic refresh) where there's no user intent
in flight; Lesson #59 inverts the rule for **explicit** actions on a surface
that already shows the error.

**Where it bites:** Any feature with a banner / `ContentUnavailableView` /
toast that announces an error AND a button that would dispatch a request
affected by that same error. Common at: login (offline), checkout (payment
backend down), upload (no network), any "submit" path that the same surface
documents as currently impossible.

---

### AP-3: Don't wrap an Apple-canonical API in a hand-rolled enum/struct/class that duplicates state

**Frameworks:** ALL (SwiftUI, CryptoKit, AuthenticationServices, Foundation, AVFoundation, etc.)
**Origin:** Lesson #51 (Sprint 2 planning) — see also Lesson #38 (CryptoKit)
**Status:** active

**Don't:**
```swift
// String catalog wrapper — duplicates STRING_CATALOG_GENERATE_SYMBOLS
nonisolated enum AuthStrings {
  static let signInTitle = String(localized: "auth.signIn.title", defaultValue: "Iniciar Sesión")
  static let signInCTA   = String(localized: "auth.signIn.cta",   defaultValue: "Iniciar")
  // ...
}

// CryptoKit wrapper for a method that doesn't exist
extension CryptoKit {
  static func timingSafeEqual(_ a: Data, _ b: Data) -> Bool {
    a.timingSafeEqual(b)  // <-- this method does NOT exist in CryptoKit
  }
}
```

**Do (canonical):**
```swift
// Use the catalog directly with Swift symbol generation
Text("auth.signIn.title")            // SwiftUI auto-localizes (LocalizedStringKey)
Button("auth.signIn.cta") { ... }    // Same — no wrapper

// XOR-accumulate idiom for constant-time comparison (Monocypher / OWASP)
static func equals(_ a: String, _ b: String) -> Bool {
  let aBytes = Array(a.utf8), bBytes = Array(b.utf8)
  guard aBytes.count == bBytes.count else { return false }
  var diff: UInt8 = 0
  for i in 0..<aBytes.count { diff |= aBytes[i] ^ bBytes[i] }
  return diff == 0
}
```

**Why:** Apple's documented APIs and build settings (`STRING_CATALOG_GENERATE_SYMBOLS`,
`SwiftUI.Text(_:LocalizedStringKey)`, `Foundation.URLComponents`) ARE the
canonical interface; wrappers create parallel state that drifts over time and
confuses contributors who recognize the canonical idiom but not the local
wrapper. For APIs that Apple does NOT ship (e.g., `CryptoKit.timingSafeEqual`
does NOT exist as of iOS 26.4), use the documented community idiom (XOR
accumulate from Monocypher / OWASP), NOT a custom Swift extension that hides
the algorithm.

Verify Apple ships the canonical API by searching:
- `developer.apple.com/documentation/<framework>`
- WWDC sessions (last 3 years)
- The framework's `.swiftinterface` file (cf. Lesson #38)
- Apple Sample Code: https://developer.apple.com/sample-code/
- Existing build flags in `project.yml` and xcconfig (Lesson #50)

**Where it bites:** Sprint plans authored without a Phase 0 canonical sweep
(Lesson #56). LLM-generated scaffolding that produces "centralization" enums
which were never needed. Refactor PRs that "consolidate" code by introducing
a wrapper instead of consuming an existing build-flag-generated symbol.

---

### AP-4: Don't pre-cache simulator UDIDs in scripts; use `name=` selectors

**Frameworks:** xcodebuild, simctl, CI scripting
**Origin:** Lesson #21 (Sprint 1)
**Status:** active

**Don't:**
```bash
# Hardcoded UDID in script / docs / Makefile
xcodebuild -destination 'platform=iOS Simulator,id=BB41A470-5332-403F-97B4-771FD1C4A626' test
```

**Do (canonical):**
```bash
# Name-based selector survives `simctl erase all` and SDK updates
xcodebuild -destination 'platform=iOS Simulator,name=iPhone 17 Pro' test
```

**Why:** `xcrun simctl erase all` (commonly used to recover from "Busy /
preflight checks failed") regenerates every simulator UDID. Cached UDIDs in
scripts, docs, or commit messages become invalid silently — `xcodebuild`
errors with `Unable to find a destination matching the provided destination
specifier`. Apple's `xcodebuild` docs accept `name=` as a stable selector
that resolves to the current UDID at invocation time.

**Where it bites:** Repository tooling (Makefile, CI scripts), agent-skill
documentation, capture-lessons commit messages with verbatim UDIDs from a
local run.

---

### AP-5: Don't use `@StateObject` / `@ObservedObject` for new ViewModels in iOS 17+

**Frameworks:** SwiftUI, Observation
**Origin:** Project convention (CLAUDE.md Architecture Conventions)
**Status:** active

**Don't:**
```swift
final class FeatureViewModel: ObservableObject {
  @Published var items: [Item] = []
}

struct FeatureView: View {
  @StateObject private var viewModel = FeatureViewModel()
  // ...
}
```

**Do (canonical, iOS 17+):**
```swift
@MainActor @Observable
final class FeatureViewModel {
  var items: [Item] = []
}

struct FeatureView: View {
  @State private var viewModel = FeatureViewModel()
  // ...
}
```

**Why:** Apple introduced the [Observation framework](https://developer.apple.com/documentation/observation)
in iOS 17 to replace the older Combine-based `ObservableObject` /
`@Published` pattern with macro-driven mutation tracking. `@Observable` works
seamlessly with `@State`, `@Bindable`, async/await, and avoids the
`@Published` indirection. The project SDK target is iOS 26.4+ — the older
pattern has zero benefit and adds Combine dependency surface.

**Where it bites:** New feature scaffolding pasted from older tutorials,
LLM-generated code trained on iOS 14–16 patterns, refactors that "modernize"
a class without flipping the wrapper.

---

### AP-6: Don't put validation, URL building, or other terminal preconditions INSIDE the retry block

**Frameworks:** Foundation, application-level retry / throttle / circuit-breaker wraps
**Origin:** Lesson #62 (Sprint 2 PR #3 — `exchangeCodeForTokens` retry wrap)
**Status:** active

**Don't:**
```swift
// WRONG — wraps validation + URL building inside retry; runs N× on terminal failure
func exchangeCodeForTokens(code: String, codeVerifier: String) async throws -> CognitoTokens {
  return try await AuthRetryPolicy.transientNetworkOnly.execute { [self] in
    guard let tokenURL = URL(string: "...") else { throw ... }      // ← runs 3×
    let sanitized = InputValidator.sanitize(code)                   // ← runs 3×
    if case let .invalid = InputValidator.validateOAuthCode(...) {  // ← runs 3×
      throw AuthError.validationFailed(...)
    }
    let request = buildRequest(...)                                 // ← runs 3×
    let (data, _) = try await httpClient.data(for: request)
    return ...
  }
}
```

**Do (canonical):**
```swift
// CORRECT — preconditions outside, retry around the operation only
func exchangeCodeForTokens(code: String, codeVerifier: String) async throws -> CognitoTokens {
  // Terminal — outside retry
  guard let tokenURL = URL(string: "...") else {
    throw AuthError.unknown("Invalid endpoint URL")
  }
  let sanitized = InputValidator.sanitize(code)
  if case let .invalid(errors) = InputValidator.validateOAuthCode(sanitized) {
    throw AuthError.validationFailed(errors)
  }
  let request = buildRequest(...)

  // Transient — inside retry
  return try await AuthRetryPolicy.transientNetworkOnly.execute { [self] in
    let (data, response) = try await httpClient.data(for: request)
    // ... HTTP status check + decode
    return decodedTokens
  }
}
```

**Why:** The retry block defines a precise scope, not a convenience wrapper.
Errors that fall OUTSIDE the policy's predicate (validation failures, URL
construction failures, encoding failures) are terminal — re-running them
produces the same failure with extra latency the user observes. The rule:
walk the body top-down; for each `try` / `guard let`, ask "Does this throw
with an error the policy retries on?" If no → move it above the retry
block.

**Where it bites:** Any retry/throttle/circuit-breaker wrap that includes
preconditions — Cognito calls, Apollo retries, image fetch retries,
payment retries, push registration. Especially insidious because the code
"works" — terminal errors propagate correctly — but inputs that will never
succeed pay 3× back-off latency before surfacing.

---

### AP-7: Don't document the WHY of a callsite's policy choice in commit messages or README — put it in the doc comment

**Frameworks:** Application-level (any callsite that picks among interchangeable options)
**Origin:** Lesson #63 (Sprint 2 PR #3 — per-endpoint retry policy choices)
**Status:** active

**Don't:**
```swift
// Doc comment says nothing about WHY this policy was chosen.
// Future contributor must read commit message (truncated by squash-merge),
// open PR description (may be unreachable), or git blame.
/// Confirms a sign-up with the verification code.
func confirmSignUp(email: String, code: String) async throws {
  try await AuthRetryPolicy.transientNetworkOnly.execute { [self] in
    _ = try await performAction("ConfirmSignUp", body: body)
  }
}
```

**Do (canonical):**
```swift
/// Confirms a sign-up with the verification code sent to the user's email.
///
/// Wrapped in `AuthRetryPolicy.transientNetworkOnly` because the
/// verification code is single-use: if the server processed the
/// confirmation but the response was lost (5xx), retrying with the
/// same code returns `CodeMismatchException`. Pure network failures
/// (timeout, connectionLost, networkError) are safe to retry — the
/// server never received the request.
func confirmSignUp(email: String, code: String) async throws { ... }
```

**Why:** git blame degrades over time (refactors split blame across
commits); GitHub squash-merge discards commit body by default; PR
descriptions become inaccessible after repo migrations or GitHub UI
changes; READMEs drift away from the code they reference. Doc comments
live in the same file as the code and travel with every refactor.
Apple's own SDK doc comments document non-obvious choices inline (see
`URLSession.dataTask(with:completionHandler:)` doc on completion-handler
vs delegate trade-offs).

The doc-comment paragraph answers three questions in 2–4 sentences:
WHICH variant, WHY this one, WHAT GOES WRONG if flipped.

**Where it bites:** Retry policy selection per endpoint, storage scope
selection per field (`@AppStorage` vs `@State` vs `KeychainManager`),
actor isolation deviations from project default (`actor` instead of
`final nonisolated class`), lifecycle modifier choice (`.task` vs
`.task(id:)` vs `.onAppear`), `URLSession` configuration choice. The bar
for requiring inline doc: would a contributor new to the domain stop
and ask "why this and not the other"?

---

### AP-8: Don't open external URLs with `Button { UIApplication.shared.open(url) }` from SwiftUI — use `Link`

**Frameworks:** SwiftUI, UIKit interop
**Origin:** Sprint 2 PR #4 (AWS status page Link). Captured during PR #25 capture run.
**Status:** active

**Don't:**
```swift
// UIKit-style; works but loses canonical SwiftUI integration
Button {
  UIApplication.shared.open(URL(string: "https://status.aws.amazon.com/")!)
} label: {
  Text("Ver estado de servicios AWS")
}
```

**Do (canonical):**
```swift
// Canonical SwiftUI (iOS 17+) — auto-localizes, in-app overlay,
// accessibility integrated
Link(
  "auth.serviceUnavailable.statusLink",
  destination: AWSStatusPageURL.value
)
.font(.footnote)
```

**Why:** SwiftUI's [`Link`](https://developer.apple.com/documentation/swiftui/link)
is the iOS 17+ canonical view for an external URL. It:

1. Accepts a `LocalizedStringKey` label that auto-localizes via the
   project's `Localizable.xcstrings` (Lesson #49). The Button-with-Text
   form needs an explicit `String(localized:)` or a hand-rolled key.
2. Routes `http`/`https` URLs to an in-app `SFSafariViewController`-style
   overlay (no app switch — keeps the user in the app context).
   `UIApplication.shared.open(_:)` for an HTTP URL switches to Safari.
3. Integrates accessibility, Dynamic Type, and dark/light appearance via
   SwiftUI's environment, with no extra wiring.
4. `UIApplication.shared` is `@MainActor`-isolated; calling it from a
   non-`@MainActor` SwiftUI Action closure produces a warning under
   Swift 6 strict concurrency. `Link` has no such isolation concern —
   it's a value type that defers opening to the framework.

For `mailto:`, `tel:`, or other custom schemes that don't have an
in-app overlay, `Link` is still preferred (iOS routes them through
`UIApplication.openURL(_:)` internally, same as the manual form, but
the SwiftUI integration benefits persist). The only legitimate
`UIApplication.shared.open(_:)` callsite is in non-View code (a router,
a notification handler, an `@AppDelegate` callback) where SwiftUI is
not available.

**Where it bites:** New Views that need a "learn more" affordance,
"contact support" mailto links, "rate on App Store" deep links,
external documentation, or any link out of the app. LLM-generated
SwiftUI Views from older tutorials often paste `UIApplication.shared.open(URL(string: "...")!)`
because the pattern predates SwiftUI's `Link`. Force-unwrap inside is a
second bite (Lesson #64 + AP-9 — use the `private enum` URL pattern instead).

---

### AP-9: Don't force-unwrap a hardcoded `URL(string: "...")!` — wrap the literal in a `private enum` with a closure-initialized `static let`

**Frameworks:** Foundation, SwiftUI (anywhere a `URL` constant is read at the callsite)
**Origin:** Lesson #64 (Sprint 2 PR #4 — `AWSStatusPageURL` pattern)
**Status:** active

**Don't:**
```swift
// At the callsite — force-unwrap, no panic context, no centralization
Link(
  "auth.serviceUnavailable.statusLink",
  destination: URL(string: "https://status.aws.amazon.com/")!
)
```

**Do (canonical):**
```swift
// Single declaration site for the URL; closure init with explicit panic
private enum AWSStatusPageURL {
  static let value: URL = {
    guard let url = URL(string: "https://status.aws.amazon.com/") else {
      preconditionFailure("Invalid AWS status page URL literal")
    }
    return url
  }()
}

// Callsite reads the named constant — refactor-safe, panic-safe
Link("auth.serviceUnavailable.statusLink", destination: AWSStatusPageURL.value)
```

**Why:** Three problems with the force-unwrap:

1. **No panic context.** `URL(string: "...")!` on an unparseable literal
   produces `fatal error: unexpectedly found nil` with no semantic clue.
   The `guard let / preconditionFailure` shape produces a clear panic
   message ("Invalid AWS status page URL literal") that points at the
   intent of the literal.
2. **No centralization.** A future migration (regional dashboard, new
   console URL, A/B-rolled endpoint) has to chase every `URL(string:)!`
   in the View tree. The `private enum` puts the URL at one declaration
   site so the migration touches one line, not the entire surface.
3. **Lint/review friction.** Force-unwrap is the convention violation
   hardest to catch in review because "the URL string is obviously
   correct". The closure form documents the invariant ("URL is constant
   by construction; failure means the binary is corrupt") in the diff
   itself.

The pattern was already in the codebase (`CognitoConfig.endpoint:18-23`
uses it for the Cognito service endpoint) — Lesson #64 formalized the
rule so new URL literals adopt it without rediscovering it.

**Where it bites:** Status page URLs, privacy-policy URLs, App Store
review URLs, mailto: support links, well-known endpoint constants,
deep-link prefixes. LLM-generated code and older tutorials default to
`URL(string: "...")!` because it's two tokens shorter than the closure
form; the savings are not worth the loss of context and centralization.

---

### AP-10: Don't add `@available(*, deprecated)` shims when the API change is driven by a backend contract change

**Frameworks:** Application-level (Core / Domain / Presentation method signatures driven by an external system)
**Origin:** Lesson #65 (Sprint 2 PR #5a — `signUp` signature change for OIDC claims)
**Status:** active

**Don't:**
```swift
// CognitoAuthService.swift — keeping the old shape "for compatibility"
final nonisolated class CognitoAuthService: Sendable {
  // New signature, correct
  func signUp(email: String, password: String, givenName: String, familyName: String) async throws { ... }

  // OLD signature, deprecated — DON'T DO THIS
  @available(*, deprecated, renamed: "signUp(email:password:givenName:familyName:)")
  func signUp(email: String, password: String, name: String) async throws {
    // Tries to be "compatible" by splitting `name` on space
    let parts = name.split(separator: " ", maxSplits: 1)
    let given = String(parts.first ?? "")
    let family = parts.count > 1 ? String(parts.last ?? "") : ""
    try await signUp(email: email, password: password, givenName: given, familyName: family)
  }
}
```

**Do (canonical):**
```swift
// Flip atomically. The build break IS the discovery mechanism.
final nonisolated class CognitoAuthService: Sendable {
  func signUp(email: String, password: String, givenName: String, familyName: String) async throws {
    // Backend wants three OIDC name claims; client is the only source for email/password sign-up.
    // ... see CognitoAuthService.swift for the full body.
  }
  // No deprecated overload. Tests, mocks, and call sites are all updated in the same PR.
}
```

**Why:** When a backend contract changes (Cognito payload, AppSync schema,
GraphQL operation shape, S3 upload metadata, MercadoPago request body),
keeping the old client signature alive under `@available(*, deprecated)`
is dangerous, NOT prudent:

1. **Both signatures cannot be correct.** The backend already speaks
   only one shape. The deprecated overload is silently wrong from the
   moment it ships — every caller that hasn't migrated produces malformed
   payloads at the server boundary. The "smooth transition" is an
   illusion: the new shape works, the old one breaks, and you don't see
   the breakage until users in production report missing data.
2. **It hides the cost of the migration.** The whole point of an atomic
   refactor is that the compile errors enumerate every callsite. Hiding
   them behind a working overload defers the work and accumulates debt.
3. **The shim's "compatibility" code is itself a guess.** The example
   above guesses that a full name like "Erick Aldama Gutiérrez" splits
   into given="Erick" and family="Aldama Gutiérrez". For Spanish names
   with two family names (Mexico, Spain, much of Latin America), that's
   wrong — the user has `apellido paterno` + `apellido materno`, and
   the given name itself may be compound (María José, Juan Carlos). A
   "compatibility" shim encodes one parsing guess as the migration
   semantics; L65 + L68 say question the guess instead.

The legitimate use of `@available(*, deprecated)` is client-side
refactors (renaming a public helper, restructuring a SwiftUI View's
init params, moving a model type to a different module) where both
signatures CAN produce correct behavior simultaneously and the
deprecation gives downstream consumers (other modules, dependent
packages) time to migrate. Backend-driven contract changes don't have
that property.

**Where it bites:** Any PR that changes a method signature in
`Core/<X>Service.swift`, `Domain/Repositories/<X>Protocol.swift`, or
the corresponding ViewModel because the server payload changed.
Pattern signal: PR description contains phrases like "for backwards
compatibility", "smooth migration", "give consumers time to update",
"both signatures are supported" — when those phrases appear next to
a Cognito / AppSync / S3 / MercadoPago / push API contract change,
the PR violates AP-10. Flip atomically; the build break is the
discovery tool.

---

### AP-11: Don't surface a generic error toast for every failure in an enumerable-by-existence endpoint

**Frameworks:** Application-level (any ViewModel method whose backend can distinguish "account exists" from "doesn't exist" in user-observable timing or response shape)
**Origin:** Lesson #44 refinement (Sprint 2 PR #5b — `startPasswordReset` anti-enumeration). NOT a new lesson — this entry pairs the existing typed-error-routing rule (L44) with its concrete anti-pattern shape for endpoints that leak identity through error surfacing.
**Status:** active

**Don't:**
```swift
// Naive: catches everything, surfaces everything. The presence /
// absence of `errorMessage` after submit leaks account existence.
func startPasswordReset(email: String) async {
  do {
    try await repository.forgotPassword(email: email)
    needsPasswordResetConfirmation = true
  } catch {
    // Attacker submits "victim@target.com" — gets "no encontramos esa
    // cuenta" toast. Attacker submits "anyone@gmail.com" — gets
    // "Te enviamos un código" / advance. Email enumeration confirmed
    // via UX surface, no SMTP probing needed.
    errorMessage = error.localizedDescription
  }
}
```

**Do (canonical):**
```swift
func startPasswordReset(email: String) async {
  let sanitizedEmail = InputValidator.sanitize(email).lowercased()
  // Terminal format error — safe to surface, NOT an identity signal.
  if case let .invalid(errors) = InputValidator.validateEmail(sanitizedEmail) {
    errorMessage = errors.first?.errorDescription
    return
  }
  do {
    try await repository.forgotPassword(email: sanitizedEmail)
  } catch let error as AuthError where error.isRetryable {
    // Transient — safe to surface, user can retry the same input.
    errorMessage = error.errorDescription
    lastError = error
    canRetry = true
    lastFailedAction = { [weak self] in await self?.startPasswordReset(email: email) }
    return                                           // does NOT advance
  } catch {
    // Identity-related (.invalidCredentials disguised UserNotFoundException,
    // .invalidParameter, .unknown) — SILENCED. Attacker can't probe.
  }
  passwordResetEmail = sanitizedEmail
  needsPasswordResetConfirmation = true              // ALWAYS advance after format-valid email
}
```

**Why:** OWASP Authentication Cheat Sheet identifies password reset flows as
the canonical user-enumeration vector. The naive shape — "catch every error,
show it to the user" — leaks the same information that a SMTP-level user
enumeration attack would, just through the UX surface instead of the SMTP
response. Apple's own security guidance for sign-in
([WWDC22 Session 10118](https://developer.apple.com/videos/play/wwdc2022/10118/))
recommends "uniform timing and UI for failed and successful identity probes
in authentication flows" — the same principle applies to recovery flows.
`AuthError.from(cognitoType:)` already disguises `UserNotFoundException` as
`.invalidCredentials` to deny SMTP-level enumeration; AP-11 closes the
matching UX-level leak.

The asymmetry — surface transient errors, silence identity-related errors —
is the typed-error routing of Lesson #44 applied to a specific
enumeration-prone surface. Two regression-guard tests (Lesson #61) lock
both sides:

- "startPasswordReset still advances on .invalidCredentials (anti-enumeration)"
- "startPasswordReset surfaces transient errors and does NOT advance"

**Where it bites:** Any flow whose backend can answer "account exists?"
(forgot password, friend search, invite by email, profile lookup by handle).
LLM-generated SwiftUI ViewModels default to the naive shape because
"show errors to the user" is the default tutorial advice. The pattern is
correct in most flows; it is wrong in this specific class. Pair this entry
with Lesson #44 refinement (silence-vs-surface bidirectional asymmetry)
when reviewing ANY new "enter email / continue" surface.

---

## Adding a New Entry

When `/capture-lessons` produces a lesson whose root cause is a misuse of an
Apple framework boundary (not just a defensive bug fix), append a new AP-N
entry here as part of the same capture commit. The lesson's `**Why:**` section
becomes the entry's `**Why:**`; the lesson's `**Do instead:**` becomes
`**Do (canonical):**`; add the explicit `**Don't:**` snippet that motivated
the lesson, the framework list, and the bites-where context.

The lesson itself documents WHEN to apply the rule; the AP entry documents
WHAT the wrong pattern looks like in code, side-by-side with the right one.
Both are needed: the lesson is the rule, the AP entry is the diff.

## Related skills / docs

- `capture-lessons` skill — the pipeline that adds entries here
- `swift-module` / `swift-feature-scaffold` skills — Phase 0 / planning gates that consult this file
- `ui-design-workflow` skill — pre-sprint discipline that consults this file
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — keeps the cited Apple APIs current

---

**Last verified:** 2026-06-03 against Apple Developer docs (Observation framework), Apple HIG
(Loading and progress), and WWDC25 #266 (Explore Concurrency in SwiftUI).
**Re-check after:** WWDC26 keynote, or by 2026-12-01. **Decay risk:** medium.
**Found a drift?** Run `/skill-pattern-freshness-audit meta`.
