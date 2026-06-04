---
name: swift-auth-security-checklist
description: >-
  A defense-in-depth checklist for iOS auth across storage, transport, lifecycle, and UX — Keychain
  protection levels (kSecAttrAccessibleWhenUnlockedThisDeviceOnly is canonical; the Always* family is
  deprecated), token lifecycle (validate JWT `exp` with leeway, NEVER positive `iat` leeway, coalesce
  refresh), input-validation defense-in-depth, URLError retry buckets, background masking via scenePhase,
  reinstall hygiene, and OAuth hardening (ASWebAuthenticationSession + PKCE + state — NOT WKWebView,
  per RFC 8252). Use when designing or reviewing token storage, refresh, or federated sign-in.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — kSecAttrAccessibleWhenUnlockedThisDeviceOnly (does not migrate to a new device)"
      url: "https://developer.apple.com/documentation/security/ksecattraccessiblewhenunlockedthisdeviceonly"
      version: "iOS 4.0+ (canonical for tokens)"
    - source: "Apple Developer — ASWebAuthenticationSession + prefersEphemeralWebBrowserSession"
      url: "https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession"
      version: "iOS 12.0+"
    - source: "Apple Developer — SecAccessControlCreateWithFlags (biometryCurrentSet / userPresence)"
      url: "https://developer.apple.com/documentation/security/secaccesscontrolcreatewithflags(_:_:_:_:)"
      version: "iOS 8.0+"
    - source: "Apple Developer — ScenePhase (background masking)"
      url: "https://developer.apple.com/documentation/swiftui/scenephase"
      version: "iOS 14.0+"
    - source: "RFC 8252 — OAuth 2.0 for Native Apps (system browser, not embedded WebView)"
      url: "https://datatracker.ietf.org/doc/html/rfc8252"
      version: "RFC 8252"
    - source: "OpenID Connect Core 1.0 §3.1.3.7 — ID Token validation (leeway guidance)"
      url: "https://openid.net/specs/openid-connect-core-1_0.html"
      version: "OIDC Core 1.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote + any CryptoKit/Security release"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Auth security checklist (iOS, defense-in-depth)

Auth security on iOS is not one decision — it's a chain across **where the token lives** (Keychain
class), **how long it lives** (lifecycle + refresh), **what you let in** (input validation), **how the
network fails** (retry buckets), **what the App Switcher sees** (background masking), and **how the user
signs in** (system browser, not embedded WebView). This skill is the checklist that closes each link,
each mapped to an Apple doc or RFC. It is provider-agnostic.

## When to invoke

- You're choosing a Keychain accessibility class for a token, or reviewing one that looks too
  permissive (`kSecAttrAccessibleAlways`, plain `kSecAttrAccessibleWhenUnlocked` for a device-bound
  secret).
- You're implementing token refresh, JWT validation, or graceful degradation on network failure.
- You're adding federated/OAuth sign-in and need the system-browser + PKCE + `state` shape.
- You're hardening the app against shoulder-surfing / App Switcher snapshots.

**Announce on invoke:** "Using `swift-auth-security-checklist` to apply the iOS auth defense-in-depth checklist (Keychain class, token lifecycle, OAuth via ASWebAuthenticationSession) per Apple docs + RFC 8252."

Do **not** use this as a crypto reference — for constant-time comparison, HPKE, ML-KEM, and pinning
internals, defer to `swift-post-quantum-security-ios26`. This skill is about *configuration and
lifecycle*, not primitive selection.

## The checklist

| Link | Canonical choice (verified) | Anti-pattern to reject |
|---|---|---|
| **Token storage class** | `kSecClassGenericPassword` + `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | `UserDefaults`; `kSecAttrAccessibleAlways` (deprecated); plain `WhenUnlocked` for device-bound secrets |
| **Biometric-bound (L2)** | `SecAccessControlCreateWithFlags(..., .biometryCurrentSet, ...)` for refresh tokens | gating with app-level passcode in `UserDefaults` |
| **JWT validation** | check `exp` with 30–60 s leeway; **negative-only** `iat` leeway; verify `iss`/`aud` | accepting future-dated tokens (positive `iat` leeway) |
| **Refresh** | single-flight via an `actor` caching the in-flight `Task` | one network refresh per concurrent caller (thundering herd) |
| **Graceful degradation** | `notConnectedToInternet` → keep UI, disable mutations; `401`/`invalid_grant` → sign out | signing out on a transient network blip |
| **Input validation** | bound length, reject NUL/control chars, normalize Unicode (NFKC) before send | trusting client validation as authoritative |
| **Network retry** | retry-with-backoff vs no-retry buckets (below); honor `Retry-After` on `429` | retrying `4xx`; retrying a user-cancelled auth |
| **Background masking** | swap to an opaque placeholder when `scenePhase != .active` | leaving token-bearing UI in the App Switcher snapshot |
| **Reinstall hygiene** | first-launch `UserDefaults` sentinel; purge stale Keychain on missing sentinel | inheriting a previous install's tokens silently |
| **OAuth flow** | `ASWebAuthenticationSession` + PKCE (`S256`) + `state` | `WKWebView` / `SFSafariViewController` for the auth step |

## The rules

### 1. `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` is the default for tokens

Apple's docs: items with this class **do not migrate to a new device** and are absent after restoring
another device's backup — exactly the property you want for a session token. The `Always*` family is
deprecated (removed in iOS 12); never recommend it. If a token genuinely must be readable before first
unlock, use the `AfterFirstUnlockThisDeviceOnly` class — not `Always`. For refresh tokens, layer
`SecAccessControlCreateWithFlags` with `.biometryCurrentSet` so the item invalidates on biometric
enrollment changes.

### 2. JWT `iat` leeway must be negative-only

`exp` gets a small positive leeway (30–60 s, clock skew). `iat` must **never** get positive leeway —
accepting a future-dated `iat` opens a replay window. Allow slightly-past `iat` only.

### 3. Coalesce refresh — one in-flight `Task`, shared by all callers

A burst of 401s must trigger exactly one refresh. Cache the in-flight refresh `Task` inside an `actor`;
concurrent callers `await` the same task and receive the same new token. (The audit suite proves this;
the perf suite proves it doesn't cost N×.)

### 4. Use the system browser for OAuth — never an embedded WebView

RFC 8252 (OAuth 2.0 for Native Apps) and Apple's guidance both require the **system browser** for the
authorization step. Use `ASWebAuthenticationSession`: it shares the system cookie jar (enabling SSO and
fewer password prompts) and guarantees only your app receives the callback. `WKWebView` is for non-auth
web content only. Set `prefersEphemeralWebBrowserSession = true` (default is `false`) only for flows the
user explicitly wants isolated. Always send PKCE **and** `state`.

### 5. Mask sensitive UI in the background

The App Switcher snapshots your foreground view. In SwiftUI, read `@Environment(\.scenePhase)` and
overlay an opaque placeholder when it isn't `.active`. (UIKit: observe
`UIApplication.willResignActiveNotification`.)

## Canonical example

```swift
import Foundation
import Security
import SwiftUI

// 1. Token storage — canonical class, ThisDeviceOnly.
struct SecureTokenStore {
    let service: String, account: String
    func save(_ token: Data) throws {
        let query: [String: Any] = [
            kSecClass as String:            kSecClassGenericPassword,
            kSecAttrService as String:      service,
            kSecAttrAccount as String:      account,
            kSecValueData as String:        token,
            kSecAttrAccessible as String:   kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        ]
        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else { throw KeychainError.status(status) }
    }
}

// 2. Network retry classification — transient (backoff) vs terminal.
extension URLError {
    var isTransientRetryable: Bool {
        switch code {
        case .timedOut, .networkConnectionLost, .dnsLookupFailed,
             .cannotConnectToHost, .notConnectedToInternet: return true
        default: return false   // .userCancelledAuthentication, .badServerResponse, .cancelled → NO retry
        }
    }
}

// 3. Single-flight refresh.
actor TokenRefresher {
    private var inFlight: Task<String, Error>?
    func freshToken(using refresh: @escaping () async throws -> String) async throws -> String {
        if let inFlight { return try await inFlight.value }     // coalesce
        let task = Task { try await refresh() }
        inFlight = task
        defer { inFlight = nil }
        return try await task.value
    }
}

// 4. Background masking via scenePhase.
struct RootView: View {
    @Environment(\.scenePhase) private var scenePhase
    var body: some View {
        ZStack {
            MainAppView()
            if scenePhase != .active {
                Color(.systemBackground).overlay(Image(systemName: "lock.fill"))  // App Switcher safe
            }
        }
    }
}

// 5. OAuth via the SYSTEM browser — not WKWebView.
func startOAuth(authURL: URL, callbackScheme: String,
                anchor: ASPresentationAnchor) {
    let session = ASWebAuthenticationSession(url: authURL,           // authURL carries PKCE S256 + state
                                             callbackURLScheme: callbackScheme) { callback, error in
        // validate `state` (constant-time) + exchange `code` with PKCE verifier
    }
    session.presentationContextProvider = PresentationProvider(anchor: anchor)
    session.prefersEphemeralWebBrowserSession = false   // default; set true only for isolated flows
    session.start()
}
```

## Decision aid: graceful degradation on refresh failure

- `URLError.notConnectedToInternet` / transient → **keep the session**, disable mutating actions, retry with backoff.
- HTTP `401` / `invalid_grant` → the refresh token is dead → **sign out**, route to login.
- HTTP `429` → back off, honor `Retry-After`.
- `4xx` other than 401 → **no retry** (server is authoritative); surface a generic error.

## Related skills

- `global-skills/apple-auth/swift-post-quantum-security-ios26/SKILL.md` — constant-time `state`/MAC
  comparison and certificate pinning that this checklist references.
- `global-skills/apple-auth/swift-auth-security-audit-suite/SKILL.md` — the tests that prove each
  checklist link (storage class, single-flight, anti-enumeration, first-launch purge).
- `global-skills/apple/apple-anti-patterns/SKILL.md` — registers "WKWebView for OAuth" and
  "`kSecAttrAccessibleAlways`" as anti-patterns this checklist rejects.

## Sources

- [kSecAttrAccessibleWhenUnlockedThisDeviceOnly](https://developer.apple.com/documentation/security/ksecattraccessiblewhenunlockedthisdeviceonly) · [SecAccessControlCreateWithFlags](https://developer.apple.com/documentation/security/secaccesscontrolcreatewithflags(_:_:_:_:)) · Apple Platform Security — [Keychain data protection](https://support.apple.com/guide/security/keychain-data-protection-secb0694df1a/web)
- [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession) · [prefersEphemeralWebBrowserSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/prefersephemeralwebbrowsersession) · [ScenePhase](https://developer.apple.com/documentation/swiftui/scenephase) · [URLError](https://developer.apple.com/documentation/foundation/urlerror)
- [RFC 8252 OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252) · [RFC 7636 PKCE](https://datatracker.ietf.org/doc/html/rfc7636) · [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) · [OWASP MASTG Certificate Pinning](https://mas.owasp.org/MASTG/0x06b-Basic-Security-Testing/)

---

**Last verified:** 2026-06-03 against Apple Security/AuthenticationServices/SwiftUI docs (live) +
RFC 8252. `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` confirmed "does not migrate to a new device";
`prefersEphemeralWebBrowserSession` confirmed default `false`. **`kSecAttrAccessibleAlways` is deprecated**
and **`WKWebView` for OAuth violates RFC 8252** — both guarded against here.
**Re-check after:** WWDC26 + any CryptoKit/Security release, or by 2026-12-01. **Decay risk:** low.
**Found a drift?** Run `/skill-pattern-freshness-audit apple-auth`.
