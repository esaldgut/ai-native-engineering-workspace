# apple-auth — iOS auth security, crypto & testing

6 skills for building and verifying an iOS authentication subsystem. Provider-agnostic — Cognito,
Firebase, and Auth0 appear as examples, but the patterns apply to any OAuth/OIDC provider. The
crypto guidance was verified with extra rigor against `developer.apple.com`, RFCs, and OWASP, since
wrong security advice is the worst failure mode.

## Cryptography & hardening

| Skill | What it does |
|-------|--------------|
| [`swift-post-quantum-security-ios26`](swift-post-quantum-security-ios26/SKILL.md) | iOS 26 post-quantum crypto + transport hardening — PQ-TLS 1.3 (automatic), HPKE with XWing/ML-KEM/ML-DSA, certificate pinning. **Constant-time comparison uses `HMAC.isValidAuthenticationCode` — `CryptoKit.timingSafeEqual` does not exist.** |
| [`swift-auth-security-checklist`](swift-auth-security-checklist/SKILL.md) | Defense-in-depth across storage/transport/lifecycle/UX — Keychain protection levels, `ASWebAuthenticationSession` for OAuth (per RFC 8252, not WKWebView), background masking |

## Security & performance testing

| Skill | What it does |
|-------|--------------|
| [`swift-auth-security-audit-suite`](swift-auth-security-audit-suite/SKILL.md) | A test suite that thinks like an attacker — token-leakage scans, ATS validation, injection, anti-enumeration (provider-agnostic) |
| [`swift-auth-performance-benchmarks`](swift-auth-performance-benchmarks/SKILL.md) | Auth performance benchmarks with `ContinuousClock` — token-refresh latency at p50/p95, cold-start, Keychain r/w, refresh coalescence |

## Test architecture

| Skill | What it does |
|-------|--------------|
| [`swift-clean-architecture-test-patterns`](swift-clean-architecture-test-patterns/SKILL.md) | Layered tests for MVVM + Clean — pure unit (Core), contract (Domain), state (ViewModel); `Mirror`-based negative-invariant tests |
| [`swift-testing-framework-conventions-mvvm`](swift-testing-framework-conventions-mvvm/SKILL.md) | When to use Swift Testing vs XCTest — XCTest stays ONLY for `XCUIApplication` UI + `XCTMetric` perf; everything else is Swift Testing |

---

**Freshness:** re-check after **WWDC** or any **CryptoKit/Security release**, or by **2026-12-01**.
Run `/skill-pattern-freshness-audit apple-auth`.
