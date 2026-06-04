---
name: swift-clean-architecture-test-patterns
description: >-
  Layered test patterns for Swift MVVM + Clean Architecture — pure unit tests for Core, contract tests
  for Domain use-cases against repository protocols, @MainActor state-transition tests for @Observable
  ViewModels, Mirror-based negative-invariant tests (assert no stored property leaks a secret/URL
  without enumerating every field), and CaseIterable coherence guards for typed-error siblings
  (errorMessage + category stay in sync). Built with Swift Testing (@Suite/@Test, parameterized, @Tag).
  Use when adding tests for any layer of a Clean Architecture iOS app.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — Swift Testing (@Suite, @Test, parameterized, traits)"
      url: "https://developer.apple.com/documentation/testing"
      version: "Swift 6.0 / Xcode 16.0"
    - source: "Apple Developer — Mirror (reflecting:) / .children (negative-invariant tests)"
      url: "https://developer.apple.com/documentation/swift/mirror"
      version: "iOS 8.0+ (tests only)"
    - source: "Apple Developer — CaseIterable (coherence guards; NOT synthesized for associated-value enums)"
      url: "https://developer.apple.com/documentation/swift/caseiterable"
      version: "Swift"
    - source: "Apple Developer — LocalizedError (typed-error basis)"
      url: "https://developer.apple.com/documentation/foundation/localizederror"
      version: "Foundation"
    - source: "WWDC24 Session 10179 — Meet Swift Testing"
      url: "https://developer.apple.com/videos/play/wwdc2024/10179/"
      version: "WWDC24"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote + any CryptoKit/Security release"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Clean Architecture test patterns (Swift MVVM)

Each layer of a Clean Architecture app has a distinct test discipline. Testing a use-case like a
ViewModel, or a value object like a repository, blurs boundaries and produces brittle suites. This skill
maps **layer → test style** and adds two high-leverage idioms: **Mirror-based negative invariants** (a
struct may not contain a secret/URL, enforced without listing every field) and **`CaseIterable`
coherence guards** (a typed error's derived siblings stay in sync). All in Swift Testing.

## When to invoke

- You're adding tests and need to know *which* style fits the type: pure unit, contract, state-transition,
  or invariant.
- You have a security-sensitive value type and want a test that breaks if someone later adds a property
  holding a secret or a raw `URL`.
- You have a typed error with paired derived properties (`errorMessage` for UI, `category` for retry)
  and keep hitting "added a case, forgot to extend the switch."

**Announce on invoke:** "Using `swift-clean-architecture-test-patterns` to add layered Swift Testing suites (Core/Domain/ViewModel + Mirror invariants + error coherence guards)."

Do **not** use this for UI automation — `XCUIApplication` is XCTest-only. This skill covers Core,
Domain, and Presentation logic; UI tests live in XCTest (see the conventions skill).

## Layer → test style (verified)

| Layer | Style | Framework | Notes |
|---|---|---|---|
| **Core** (value objects, crypto wrappers) | pure unit, parameterized | Swift Testing | no I/O; `@Tag(.fast)` |
| **Domain** (use-cases, business rules) | contract test vs repository protocol, injected fakes | Swift Testing | async-friendly |
| **Data** (repositories, network) | integration with mocked transports | Swift Testing (+ XCTest only if `XCTMetric` needed) | |
| **Presentation** (`@Observable` ViewModels) | state-transition over `@Observable` props | Swift Testing + `@MainActor` | annotate the suite `@MainActor` |
| **UI** | end-to-end automation | **XCTest** (`XCUIApplication`) | not supported in Swift Testing |

## The rules

### 1. Each layer tests against the boundary below it, not the implementation

Domain use-cases test against the **repository protocol** with an injected fake — never the concrete
network repo. ViewModels test against the **use-case protocol** with an injected mock. This keeps a Data
refactor from breaking Domain tests.

### 2. ViewModel suites are `@MainActor`; mocks are `final class`

`@Observable` ViewModels created on the main actor force `@MainActor` on the suite (Swift 6 strict
concurrency). Test doubles are `final class` (no inheritance friction) exposing call-count / last-arg
properties for verification. Use an `actor` mock for any service the SUT `await`s off the main actor.

### 3. Mirror-based negative invariants — reserve for security-sensitive types

`Mirror(reflecting:).children` lets a test enforce "no stored property of `AuthSession` is a raw `URL`"
or "nothing here is plaintext-secret-shaped" **without enumerating every property** — so adding a field
can't silently bypass the check. This has reflection overhead and skips computed properties: **tests
only, never production**, and gated behind `@Tag(.invariants)` so fast CI can skip it.

### 4. `CaseIterable` coherence guards — and the associated-value caveat

When a typed error carries paired derived properties (`errorMessage`, `category`), one coherence test
iterates all cases and asserts each derived value is non-default. **Caveat:** `CaseIterable` is **not**
synthesized for enums with associated values — provide a static `allCasesForTesting: [Self]` array
manually for those.

### 5. Anti-enumeration belongs in the coherence guard

The same coherence test that checks every error case has a message should also assert none of those
messages leaks a provider-specific "user not found" code — the provider-agnostic anti-enumeration rule,
enforced once at the error boundary.

## Canonical example

```swift
import Testing
@testable import MyApp

// Core — pure, parameterized.
@Suite("Email value object", .tags(.fast))
struct EmailTests {
    @Test(arguments: [("a@b.com", true), ("not-an-email", false), ("", false)])
    func parses(_ input: String, _ valid: Bool) {
        #expect((Email(input) != nil) == valid)
    }
}

// Domain — contract test against the repository PROTOCOL via a fake.
@Suite("SignInUseCase contract")
struct SignInUseCaseTests {
    @Test func returnsSessionOnHappyPath() async throws {
        let useCase = SignInUseCase(repository: FakeAuthRepository(behavior: .success(.sample)))
        let session = try await useCase.execute(email: "a@b.com", password: "x")
        #expect(!session.accessToken.isEmpty)
    }
}

// Presentation — @MainActor state transitions over @Observable props.
@MainActor
@Suite("SignInViewModel states")
struct SignInViewModelTests {
    @Test func idleToLoadingToError() async {
        let vm = SignInViewModel(useCase: FakeFailingSignIn())
        #expect(vm.state == .idle)
        await vm.submit(email: "a@b.com", password: "x")   // .idle -> .loading -> .error
        #expect(vm.state.isError)
        #expect(vm.errorMessage?.isEmpty == false)
    }
}

// Typed-error coherence guard (+ anti-enumeration). Manual allCases for an associated-value enum.
@Suite("AuthError coherence")
struct AuthErrorCoherenceTests {
    @Test func everyCaseHasMessageAndCategoryAndLeaksNothing() {
        for c in AuthError.allCasesForTesting {           // hand-rolled: CaseIterable not synthesized
            #expect(!c.errorMessage.isEmpty, "no errorMessage for \(c)")
            #expect(c.category != .unknown,  "no category for \(c)")
            #expect(!c.errorMessage.lowercased().contains("not found"))  // anti-enumeration
        }
    }
}

// Negative invariant via Mirror — gated, security-sensitive type only.
@Suite("AuthSession invariants", .tags(.invariants))
struct AuthSessionInvariantTests {
    @Test func noStoredPropertyIsARawURL() {
        for child in Mirror(reflecting: AuthSession.sample).children {
            #expect(!(child.value is URL), "AuthSession leaked URL via \(child.label ?? "?")")
        }
    }
}
```

## Decision aid: which test for this type

- **Pure value / no I/O?** → Core unit, parameterized, `.tags(.fast)`.
- **Use-case with a dependency?** → Domain contract test, inject a **protocol** fake.
- **`@Observable` ViewModel?** → `@MainActor` suite, assert state transitions.
- **Security-sensitive struct?** → add a Mirror negative invariant under `.tags(.invariants)`.
- **Typed error with derived siblings?** → one `CaseIterable` coherence guard (manual `allCasesForTesting` if associated values).
- **Tap/scroll/navigation?** → XCTest `XCUIApplication`, not here.

## Related skills

- `global-skills/apple-auth/swift-testing-framework-conventions-mvvm/SKILL.md` — the conventions (import
  order, mock idioms, Swift-Testing-vs-XCTest boundary) these suites follow.
- `global-skills/apple-auth/swift-auth-performance-benchmarks/SKILL.md` — the performance counterpart for
  the Data layer.
- `global-skills/apple-auth/swift-auth-security-audit-suite/SKILL.md` — where the anti-enumeration
  coherence guard is extended into a full attacker-perspective audit.

## Sources

- [Swift Testing](https://developer.apple.com/documentation/testing) · [@Suite](https://developer.apple.com/documentation/testing/suite(_:_:)) · WWDC24 [10179 Meet Swift Testing](https://developer.apple.com/videos/play/wwdc2024/10179/)
- [Mirror](https://developer.apple.com/documentation/swift/mirror) · [CaseIterable](https://developer.apple.com/documentation/swift/caseiterable) · [LocalizedError](https://developer.apple.com/documentation/foundation/localizederror)
- [XCUIApplication](https://developer.apple.com/documentation/xctest/xcuiapplication) (UI tests stay in XCTest)

---

**Last verified:** 2026-06-03 against Swift Testing, `Mirror`, `CaseIterable`, `LocalizedError` (Apple
Developer docs, live). Confirmed `CaseIterable` is **not** synthesized for associated-value enums (hence
manual `allCasesForTesting`) and `XCUIApplication` remains XCTest-only.
**Re-check after:** WWDC26 + any CryptoKit/Security release, or by 2026-12-01. **Decay risk:** low
(Clean Architecture layering + Swift reflection are stable).
**Found a drift?** Run `/skill-pattern-freshness-audit apple-auth`.
