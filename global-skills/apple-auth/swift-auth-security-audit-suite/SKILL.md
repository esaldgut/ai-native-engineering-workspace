---
name: swift-auth-security-audit-suite
description: >-
  Build a security-audit test suite for an iOS auth subsystem that thinks like an attacker —
  token-leakage scans (UserDefaults / pasteboard / os_log), ATS validation (no NSAllowsArbitraryLoads),
  injection and boundary inputs, anti-enumeration (no provider-specific "user not found" leaks for ANY
  OAuth/OIDC provider), OAuth `state` CSRF + constant-time comparison, concurrent-refresh single-flight,
  and first-launch Keychain-cleanup idempotency. Built with Swift Testing (@Suite/@Test, Issue.record
  for soft audit findings). Use when hardening or reviewing the auth/network/storage layers.
version: "1.0.0"
freshness:
  verified_against:
    - source: "OWASP MASVS-AUTH (authentication & authorization controls)"
      url: "https://mas.owasp.org/MASVS/07-MASVS-AUTH/"
      version: "MASVS v2"
    - source: "OWASP MASTG-KNOW-0071 — App Transport Security (iOS ATS audit checks)"
      url: "https://mas.owasp.org/MASTG/knowledge/ios/MASVS-NETWORK/MASTG-KNOW-0071/"
      version: "MASTG"
    - source: "Apple Developer — OSLogPrivacy (token redaction: %@ defaults to .private)"
      url: "https://developer.apple.com/documentation/os/oslogprivacy"
      version: "iOS 14.0+"
    - source: "AWS Cognito — Managing user existence error responses (PreventUserExistenceErrors=ENABLED), generalized to any provider"
      url: "https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-managing-errors.html"
      version: "2026-06-03"
    - source: "RFC 9700 — OAuth 2.0 Security Best Current Practice (state / PKCE)"
      url: "https://datatracker.ietf.org/doc/rfc9700/"
      version: "RFC 9700"
    - source: "Apple Developer — Swift Testing @Suite (audit suites)"
      url: "https://developer.apple.com/documentation/testing/suite(_:_:)"
      version: "Swift 6.0 / Xcode 16.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote + any CryptoKit/Security release"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Auth security audit suite (iOS, Swift Testing)

A functional auth flow can still leak secrets, enumerate users, or race itself into a corrupt token
state. This skill builds a **security-audit test suite** that exercises the auth subsystem the way an
attacker would: it scans for stored-secret leaks, validates App Transport Security, throws hostile
inputs at validators, proves the error mapper can't be used for user enumeration, and asserts the
concurrency and install-hygiene invariants. It is **provider-agnostic** — the anti-enumeration rule
holds for Cognito, Firebase, Auth0, Okta, or any OAuth/OIDC backend.

## When to invoke

- You have an auth subsystem (token store, OAuth flow, error mapper) and want a repeatable security
  gate in CI rather than a one-time manual review.
- A review flagged a possible secret leak (a token in a log line, a `UserDefaults` key, a pasteboard
  copy) and you want a regression-locking test.
- You're wiring federated/OAuth sign-in and need to prove `state` is single-use and compared in
  constant time, and that the provider's error codes never reach the UI.

**Announce on invoke:** "Using `swift-auth-security-audit-suite` to add attacker-perspective auth tests (leakage, ATS, anti-enumeration, OAuth state) per OWASP MASVS-AUTH + RFC 9700."

Do **not** treat this suite as a substitute for server-side controls. Client tests are
defense-in-depth; the provider must *also* be configured to suppress user-existence errors
(e.g. Cognito `PreventUserExistenceErrors=ENABLED`). Both layers are required.

## What the suite must cover (verified rationale)

| Audit area | Assertion | Source |
|---|---|---|
| **Token leakage — storage** | No `UserDefaults` key whose name/value looks token-like; tokens live only in Keychain | OWASP MASVS-STORAGE |
| **Token leakage — pasteboard** | `UIPasteboard.general.string` doesn't echo a secret after a copy event | MASVS-STORAGE |
| **Token leakage — logs** | No `os_log` call interpolates a token as `%{public}@`; default `%@` is `.private` | `OSLogPrivacy` |
| **ATS** | `NSAllowsArbitraryLoads` / `…InWebContent` not `true`; no TLS-min below `TLSv1.2` without a justified `NSExceptionDomains` entry | MASTG-KNOW-0071 |
| **Injection / boundary** | Username/redirect validators reject `javascript:` schemes, NUL bytes, U+202E RTL override, over-length input | MASVS-CODE |
| **Anti-enumeration** | The error mapper never surfaces `USER_NOT_FOUND` / `auth/user-not-found` / "does not exist"; collapses to one generic credential error | provider docs (generalized) |
| **OAuth `state` CSRF** | Two parallel flows produce different `state`; a tampered `state` on callback is rejected; comparison is constant-time | RFC 9700 / RFC 6749 §10.12 |
| **Concurrent refresh** | N parallel `accessToken()` calls → exactly one network refresh, all callers get the same token | MASVS-AUTH |
| **First-launch hygiene** | Fresh install (missing `UserDefaults` sentinel) purges stale Keychain items; idempotent across launches | Apple Keychain (survives uninstall) |

## The rules

### 1. Anti-enumeration is provider-agnostic — assert the mapped output, not the raw code

Every provider has a distinguishing "no such user" signal: Cognito `UserNotFoundException`, Firebase
`auth/user-not-found`, Auth0 `invalid_grant` with a user-not-found `description`. The UI must receive
**one** generic case ("Incorrect username or password"). The test iterates the data-layer error enum
and asserts no case's user-facing message contains `"not found"`, `"does not exist"`, `"unknown user"`,
or a raw provider code. (The server must *also* suppress these — e.g. Cognito `PreventUserExistenceErrors=ENABLED`.)

### 2. `state` is required even with PKCE — and compared in constant time

RFC 9700 lets PKCE provide CSRF protection only when the authorization server supports it; otherwise
`state` is **required**. Best practice: always send both. The audit asserts (a) `state` differs across
parallel flows, (b) a tampered callback `state` is rejected, (c) comparison uses
`HMAC.isValidAuthenticationCode` or XOR-accumulate — never `==` on the raw value.

### 3. `Issue.record` for sweep findings; `#expect` for hard invariants

For "scan everything and report all leaks" passes (UserDefaults dump, log grep), use
`Issue.record(...)` so one offending key doesn't abort the sweep. For binary invariants (ATS has no
arbitrary loads; refresh is single-flight) use `#expect`.

### 4. Keychain survives uninstall — the first-launch purge is mandatory and must be idempotent

iOS keeps Keychain items across app uninstall by design. Test that a missing first-launch sentinel
triggers a purge, and that running the hook twice leaves the same state (no crash, no double-delete error).

## Canonical example

```swift
import Testing
import Foundation
@testable import MyApp   // @testable import last

@Suite("Auth security audit")
struct AuthSecurityAuditTests {

    @Test func noTokenLikeKeysInUserDefaults() {
        for (key, value) in UserDefaults.standard.dictionaryRepresentation()
        where key.lowercased().contains("token") || key.lowercased().contains("secret") {
            Issue.record("Secret-like UserDefaults key leaked: \(key) = \(value)")  // soft: report all
        }
    }

    @Test func atsHasNoArbitraryLoads() {
        let ats = Bundle.main.object(forInfoDictionaryKey: "NSAppTransportSecurity") as? [String: Any] ?? [:]
        #expect(ats["NSAllowsArbitraryLoads"] as? Bool != true)
        #expect(ats["NSAllowsArbitraryLoadsInWebContent"] as? Bool != true)
    }

    @Test(arguments: ["javascript:alert(1)", "user\u{0000}name", "admin\u{202E}", String(repeating: "a", count: 5_000)])
    func validatorRejectsHostileInput(_ input: String) {
        #expect(UsernameValidator.isValid(input) == false)
    }

    // Anti-enumeration — provider-agnostic. No raw provider code reaches the UI.
    @Test func errorMapperNeverLeaksUserExistence() {
        for mapped in AuthError.allCasesForTesting.map(AuthErrorPresenter.message(for:)) {
            let m = mapped.lowercased()
            #expect(!m.contains("not found"))
            #expect(!m.contains("does not exist"))
            #expect(!m.contains("user_not_found"))
        }
    }

    @Test func oauthStateIsUniqueAndConstantTimeComparable() {
        let a = OAuthState.generate(), b = OAuthState.generate()
        #expect(a.value != b.value)                                  // single-use, unguessable
        #expect(a.matches(a.value))                                  // constant-time equal
        #expect(!a.matches(b.value))                                 // tampered → reject
    }

    @Test func concurrentRefreshIsSingleFlight() async {
        let session = MockAuthSession()
        await withTaskGroup(of: String?.self) { group in
            for _ in 0..<16 { group.addTask { try? await session.accessToken() } }
            var tokens: [String] = []
            for await t in group { if let t { tokens.append(t) } }
            #expect(Set(tokens).count == 1)            // all callers got the SAME new token
            #expect(session.refreshCallCount == 1)     // only ONE refresh hit the network
        }
    }

    @Test func firstLaunchPurgeIsIdempotent() {
        FirstLaunchHygiene.reset()                      // simulate fresh install
        FirstLaunchHygiene.purgeStaleKeychainIfNeeded() // purges
        FirstLaunchHygiene.purgeStaleKeychainIfNeeded() // no-op, no crash
        #expect(FirstLaunchHygiene.didComplete)
    }
}
```

## Decision aid: CI gate vs runtime check

- **CI gate (default):** run the whole suite on every PR. ATS, anti-enumeration, and leakage scans
  belong here — they're cheap and deterministic.
- **DEBUG runtime assertion:** the UserDefaults/pasteboard scan can also fire on launch in DEBUG to
  catch leaks introduced between test runs; never ship it in RELEASE.
- **UI-driven pasteboard test:** the in-process `UIPasteboard` check is enough for logic; only escalate
  to an `XCUIApplication` keyboard-driven test if a real copy button is the surface under audit.

## Related skills

- `global-skills/apple-auth/swift-auth-security-checklist/SKILL.md` — the defense-in-depth checklist
  these tests verify.
- `global-skills/apple-auth/swift-post-quantum-security-ios26/SKILL.md` — the constant-time comparison
  the `state` tests rely on (`HMAC.isValidAuthenticationCode`).
- `global-skills/apple-auth/swift-testing-framework-conventions-mvvm/SKILL.md` — Swift Testing
  conventions (`@Suite`, import order, `Issue.record` vs `#expect`).

## Sources

- [OWASP MASVS-AUTH](https://mas.owasp.org/MASVS/07-MASVS-AUTH/) · [MASTG-KNOW-0071 ATS](https://mas.owasp.org/MASTG/knowledge/ios/MASVS-NETWORK/MASTG-KNOW-0071/) · [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OSLogPrivacy](https://developer.apple.com/documentation/os/oslogprivacy) · [UIPasteboard](https://developer.apple.com/documentation/uikit/uipasteboard) · [Swift Testing @Suite](https://developer.apple.com/documentation/testing/suite(_:_:))
- [AWS Cognito — Managing user existence error responses](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-managing-errors.html) (one provider example of the generic rule)
- [RFC 9700 OAuth 2.0 Security BCP](https://datatracker.ietf.org/doc/rfc9700/) · [RFC 7636 PKCE](https://datatracker.ietf.org/doc/html/rfc7636)

---

**Last verified:** 2026-06-03 against OWASP MASVS-AUTH / MASTG, `OSLogPrivacy`, Cognito
`PreventUserExistenceErrors` (exact name + `ENABLED`/`LEGACY` behavior confirmed), and RFC 9700.
The anti-enumeration rule was verified to generalize across Cognito/Firebase/Auth0.
**Re-check after:** WWDC26 + any CryptoKit/Security release, or by 2026-12-01. **Decay risk:** low
(RFC/OWASP-grounded; the testing API surface is stable).
**Found a drift?** Run `/skill-pattern-freshness-audit apple-auth`.
