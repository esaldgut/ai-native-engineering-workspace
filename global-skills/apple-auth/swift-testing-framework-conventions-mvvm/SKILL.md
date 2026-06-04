---
name: swift-testing-framework-conventions-mvvm
description: >-
  Conventions for testing a Swift MVVM app — when to use Swift Testing vs XCTest (XCTest stays ONLY for
  XCUIApplication UI automation and XCTMetric performance, per Apple WWDC24), import ordering (regular
  imports first, @testable last, and why it's substantive not just style), final-class / actor mock
  idioms with exposed call-counts (the MockK-equivalent), and CaseIterable regression-guard tests that
  keep typed-error siblings in sync. Use when establishing or reviewing test conventions for any MVVM
  iOS module.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — Swift Testing landing (default for new tests)"
      url: "https://developer.apple.com/xcode/swift-testing/"
      version: "Xcode 16.0"
    - source: "Apple Developer — @Suite / @Test macros (Swift 6.0 / Xcode 16.0)"
      url: "https://developer.apple.com/documentation/testing/suite(_:_:)"
      version: "Swift 6.0 / Xcode 16.0"
    - source: "Apple Developer — XCUIApplication (UI automation stays XCTest)"
      url: "https://developer.apple.com/documentation/xctest/xcuiapplication"
      version: "XCTest"
    - source: "Apple Developer — XCTMetric (performance probes stay XCTest)"
      url: "https://developer.apple.com/documentation/xctest/xctmetric"
      version: "XCTest"
    - source: "The Swift Programming Language — Access Control (@testable exposes internal, not private)"
      url: "https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol"
      version: "Swift 6"
    - source: "WWDC24 Session 10179 — Meet Swift Testing (XCTest split is the stated long-term position)"
      url: "https://developer.apple.com/videos/play/wwdc2024/10179/"
      version: "WWDC24"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote + any CryptoKit/Security release"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Swift Testing conventions (MVVM)

Swift Testing is the default for new unit and integration tests since WWDC24. But the migration isn't
total, and the conventions around it (import order, mock shape, the XCTest boundary) matter for a clean,
reviewable MVVM suite. This skill codifies them — including the one fact teams keep getting wrong: **UI
automation and performance metrics stay in XCTest indefinitely**, per Apple's own stated position.

## When to invoke

- You're standing up a test target or reviewing a PR's tests and need the framework choice, import
  ordering, and mock idiom to be consistent.
- Someone tried to write a UI test in Swift Testing (it can't drive `XCUIApplication`) or claimed Swift
  Testing requires iOS 18 (it doesn't).
- You have a typed error whose derived properties drift apart and want a single regression guard.

**Announce on invoke:** "Using `swift-testing-framework-conventions-mvvm` to apply the Swift-Testing-vs-XCTest split, import ordering, and mock conventions per Apple WWDC24."

Do **not** rewrite existing UI tests into Swift Testing — that's the one place XCTest is the *correct*,
not legacy, choice. This skill draws the line; it doesn't erase it.

## The conventions (verified)

| Convention | Rule | Source |
|---|---|---|
| **Default framework** | Swift Testing for unit + integration. Requires Xcode 16+/Swift 6 toolchain; compiles in Swift 5 language mode; back-deploys ~iOS 13 | Apple Swift Testing |
| **XCTest stays for** | `XCUIApplication` (UI automation) **and** `XCTMetric` (perf). Stated long-term, not transitional | WWDC24 #10179 |
| **Import ordering** | regular imports first, `@testable import MyApp` **last** | TSPL Access Control |
| **`@testable` exposes** | `internal` (acts like `public`); **not** `private`/`fileprivate` | TSPL Access Control |
| **Mock shape** | `final class`, exposed `calls`/`stub`; `actor` for awaited services; `@unchecked Sendable` only after audit | Swift 6 concurrency |
| **Test naming** | free-form descriptive funcs (`happyPathTransitions…`), not `testFoo` | Apple-blessed style |
| **`@MainActor`** | on the suite for `@Observable` SUTs | Swift 6 concurrency |

## The rules

### 1. Swift Testing default; XCTest only for UI + `XCTMetric`

Apple's WWDC24 guidance is explicit and long-term: *"continue using XCTest for any tests which use UI
automation APIs like `XCUIApplication` or performance testing APIs like `XCTMetric`, as these are not
supported in Swift Testing."* Everything else (Core, Domain, ViewModel, integration, regression guards)
is Swift Testing.

### 2. Import order is substantive — `@testable` last

Regular imports first, `@testable import MyApp` last. This isn't only style: a `@testable` module's
elevated access can shadow types from non-`@testable` modules, so putting it last makes the intentional
exposure visible and lintable. And `@testable` only surfaces `internal` symbols — it does **not** reach
`private`/`fileprivate`; don't write a test that assumes otherwise.

### 3. Mocks are `final class` with observable call-counts (the MockK-equivalent)

The portable mock idiom: a `final class` conforming to the dependency's protocol, exposing a `calls`
array (or counters) and a `stub`/`Result` you set per test, then asserting on `calls` after acting —
the Swift analogue of a MockK/Mockito verify. For any dependency the SUT `await`s off the main actor,
use an `actor` mock. Reach for `@unchecked Sendable` only after auditing the access is actually safe.

### 4. Regression-guard typed-error siblings with one `CaseIterable` test

When an error has paired derived properties that must stay in sync (`errorMessage` + `category`, or
`errorMessage` + `lastError`), write **one** test that iterates all cases and asserts each derived value
is defined — catching "added a case, forgot the switch." For associated-value enums, supply a manual
`allCasesForTesting` (`CaseIterable` isn't synthesized for them).

### 5. Don't over-constrain deployment for traits

Swift Testing itself back-deploys to ~iOS 13, but some traits (e.g. `.timeLimit`) require higher
availability. Targeting iOS 17+ for the app sidesteps the trait gymnastics and lines up with
`@Observable`.

## Canonical example

```swift
// MyAppTests/Auth/SignInViewModelTests.swift
// Regular imports first; @testable LAST (substantive — shadowing visibility).
import Foundation
import Testing
@testable import MyApp

@MainActor
@Suite("SignInViewModel")
struct SignInViewModelTests {

    // MockK-equivalent: final class, recorded calls, settable stub.
    final class MockSignInUseCase: SignInUseCasing {
        private(set) var calls: [(email: String, password: String)] = []
        var stub: Result<AuthSession, AuthError> = .success(.sample)
        func execute(email: String, password: String) async throws -> AuthSession {
            calls.append((email, password))
            return try stub.get()
        }
    }

    @Test func happyPathCallsUseCaseOnce() async {
        let useCase = MockSignInUseCase()
        let vm = SignInViewModel(useCase: useCase)
        await vm.submit(email: "a@b.com", password: "x")
        #expect(useCase.calls.count == 1)               // verify, MockK-style
        #expect(vm.state.isSuccess)
    }

    @Test func unknownUserSurfacesGenericMessage() async {
        let useCase = MockSignInUseCase()
        useCase.stub = .failure(.userNotFound)
        let vm = SignInViewModel(useCase: useCase)
        await vm.submit(email: "x@y.com", password: "x")
        #expect(vm.errorMessage?.lowercased().contains("not found") == false)  // anti-enumeration
    }
}

// Regression guard for typed-error siblings (Swift Testing, not XCTest).
@Suite("AuthError contract")
struct AuthErrorContractTests {
    @Test func messageAndCategoryDefinedForEveryCase() {
        for c in AuthError.allCasesForTesting {          // manual: associated values → no synthesis
            #expect(!c.errorMessage.isEmpty, "AuthError.\(c) has no errorMessage")
            #expect(c.category != .unknown,  "AuthError.\(c) has no category")
        }
    }
}
```

The one case that stays in XCTest:

```swift
import XCTest   // UI automation — NOT Swift Testing

final class SignInUITests: XCTestCase {
    func testSignInButtonEnablesAfterValidInput() {
        let app = XCUIApplication(); app.launch()        // XCUIApplication is XCTest-only
        // … drive the keyboard, assert button state …
    }
}
```

## Decision aid: Swift Testing or XCTest

- **Unit / Domain / ViewModel / integration / regression guard?** → Swift Testing.
- **Drives `XCUIApplication` (taps, scrolls, navigation)?** → XCTest.
- **Uses an Apple-shipped `XCTMetric` (CPU/memory probe)?** → XCTest.
- **Snapshot test (most libs are XCTest-bound)?** → XCTest.
- **Everything else** → Swift Testing.

## Related skills

- `global-skills/apple-auth/swift-clean-architecture-test-patterns/SKILL.md` — the layered suites these
  conventions format.
- `global-skills/apple-auth/swift-auth-performance-benchmarks/SKILL.md` — how to keep perf tests in Swift
  Testing with `ContinuousClock` and when to drop to `XCTMetric`.
- `global-skills/apple-auth/swift-auth-security-audit-suite/SKILL.md` — applies these conventions to an
  attacker-perspective audit suite.

## Sources

- [Swift Testing (Apple)](https://developer.apple.com/xcode/swift-testing/) · [@Suite / @Test](https://developer.apple.com/documentation/testing/suite(_:_:)) · WWDC24 [10179 Meet Swift Testing](https://developer.apple.com/videos/play/wwdc2024/10179/)
- [XCUIApplication](https://developer.apple.com/documentation/xctest/xcuiapplication) · [XCTMetric](https://developer.apple.com/documentation/xctest/xctmetric) (the XCTest-only surfaces)
- [TSPL — Access Control (`@testable`)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol) · [swift-testing repo](https://github.com/swiftlang/swift-testing)

---

**Last verified:** 2026-06-03 against Apple Swift Testing docs + WWDC24 #10179 (live). Confirmed the
XCTest split (`XCUIApplication` + `XCTMetric` only) is Apple's stated **long-term** position, and that
`@testable` exposes `internal` but not `private`. **`runBlockingTest`-style "Swift Testing replaces
XCTest entirely" is FALSE** and guarded against here.
**Re-check after:** WWDC26 + any CryptoKit/Security release, or by 2026-12-01. **Decay risk:** low.
**Found a drift?** Run `/skill-pattern-freshness-audit apple-auth`.
