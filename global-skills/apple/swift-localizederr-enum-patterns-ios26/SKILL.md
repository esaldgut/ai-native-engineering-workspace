---
name: swift-localizederr-enum-patterns-ios26
description: >-
  Model app errors as a Sendable enum that conforms to LocalizedError, switch on it exhaustively,
  and map external/network failures (URLError) into your domain cases instead of string-matching
  error messages. Covers the errorDescription / failureReason / recoverySuggestion contract, when
  @unknown default is required (non-frozen enums imported from C/ObjC), why conforming to bare Error
  loses your localized text, and the Swift 6.2 rule that an error with a nested Error payload must
  also be Sendable to cross actors. Use when designing a feature's error type, mapping a thrown
  URLError into user-facing copy, or replacing a fragile `error.localizedDescription.contains(...)`
  check.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — LocalizedError (protocol)"
      url: "https://developer.apple.com/documentation/foundation/localizederror"
      version: "iOS 8.0+ (current docs 2026-06-03)"
    - source: "Apple Developer — URLError (struct)"
      url: "https://developer.apple.com/documentation/foundation/urlerror"
      version: "iOS 8.0+ (current docs 2026-06-03)"
    - source: "The Swift Programming Language — Switch / @unknown default & @frozen"
      url: "https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements/"
      version: "Swift 6.2"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote, or any Swift 6.3 toolchain (Sendable/typed-throws changes)"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# LocalizedError enum patterns (Swift 6.2 / iOS 26)

The canonical way to surface a failure to a user on Apple platforms is a typed `enum` that conforms
to [`LocalizedError`](https://developer.apple.com/documentation/foundation/localizederror) (a
`Foundation` protocol since iOS 8). `LocalizedError` is the bridge that makes
`error.localizedDescription` return *your* copy rather than a debug type name, and exposes
`failureReason` / `recoverySuggestion` to system presenters. This skill pairs that with exhaustive
`switch` handling and a mapping function that turns
[`URLError`](https://developer.apple.com/documentation/foundation/urlerror) into your domain cases —
never a string match on the message.

## When to invoke

- You're designing a feature's **error type** and want it to render correctly in alerts.
- A network call throws a `URLError` and you need to translate `.notConnectedToInternet` / `.timedOut`
  / `.userAuthenticationRequired` into user-facing copy and a retry decision.
- You see a fragile `error.localizedDescription.contains("...")` or `(error as NSError).code == ...`
  branch and want to replace it with a typed `switch`.

**Announce on invoke:** "Using `swift-localizederr-enum-patterns-ios26` to model a Sendable LocalizedError enum and map URLError exhaustively."

Do **not** reach for this when you simply need to `throw` and propagate without showing the user a
message — a transient internal error doesn't need the full `LocalizedError` ceremony.

## The canonical APIs (verified)

| API | Signature (verified) | Use |
|---|---|---|
| `LocalizedError` | `protocol LocalizedError : Error` | The conformance that powers user-facing copy |
| `errorDescription` | `var errorDescription: String? { get }` | The headline shown to the user (return non-nil) |
| `failureReason` | `var failureReason: String? { get }` | Why it failed |
| `recoverySuggestion` | `var recoverySuggestion: String? { get }` | What the user can do |
| `URLError` | `struct URLError` (domain `NSURLErrorDomain`) | The Foundation networking error you map from |
| `URLError.Code` | `.notConnectedToInternet`, `.timedOut`, `.cancelled`, `.userAuthenticationRequired`, … | The case you switch on |
| `RecoverableError` | `protocol RecoverableError : Error` | Optional — for in-place recovery options (rare on iOS) |

## The rules (load-bearing)

### 1. Conform to `LocalizedError`, not bare `Error`, or you lose your copy

An enum that conforms only to `Error` makes `.localizedDescription` return the **type name string**,
not your computed message. Conform to `LocalizedError` and implement `errorDescription`:

```swift
enum APIError: LocalizedError { case offline
    var errorDescription: String? { String(localized: "You're offline.") }   // shown to user
}
```

### 2. `errorDescription` is `String?` — return non-nil, never an empty string

Returning `nil` (or `""`) hands the user the system fallback (an opaque domain+code). Provide real,
localized text via `String(localized:)` (iOS 15+; gate older deployment targets).

### 3. Make the error `Sendable` — including any nested `Error` payload

Under Swift 6.2's actor model, an error that crosses an actor boundary must be `Sendable`. If a case
carries an underlying `Error`, that nested value must itself be `Sendable`:

```swift
enum APIError: LocalizedError, Sendable {
    case unknown(underlying: any Error & Sendable)   // payload is Sendable too
}
```

### 4. Map `URLError` by switching on `.code` — never on the message string

Cast the caught error to `URLError` and switch on `urlError.code`. Do **not** branch on
`error.localizedDescription.contains(...)` — message strings are localized and change between OS
versions, so a string match silently breaks.

### 5. `@unknown default` is for non-frozen enums you don't own

When you `switch` over an enum imported from C/Obj-C (an `NS_ENUM`) or any non-`@frozen` enum from a
library you don't control, add `@unknown default` so a future case compiles with a warning instead
of a crash. For a pure Swift enum in **your own** module, a normal exhaustive switch (no
`@unknown default`) is what you want — adding it there suppresses the helpful "you missed a case"
error when you grow the enum.

```swift
switch urlError.code {
case .notConnectedToInternet, .dataNotAllowed: return .offline
case .timedOut:                                return .timeout
case .userAuthenticationRequired:              return .unauthorized
@unknown default:                              return .unknown(underlying: urlError)  // imported enum
}
```

## Canonical example

```swift
public enum APIError: LocalizedError, Sendable {
    case unauthorized
    case rateLimited(retryAfter: TimeInterval)
    case offline
    case timeout
    case server(statusCode: Int)
    case unknown(underlying: any Error & Sendable)

    public var errorDescription: String? {
        switch self {                                  // exhaustive, no @unknown default — it's ours
        case .unauthorized:        return String(localized: "You're signed out.")
        case .rateLimited:         return String(localized: "Please slow down and try again.")
        case .offline:             return String(localized: "You're offline.")
        case .timeout:             return String(localized: "The request timed out.")
        case .server(let code):    return String(localized: "Server error (\(code)).")
        case .unknown(let e):      return e.localizedDescription
        }
    }

    public var recoverySuggestion: String? {
        switch self {
        case .offline, .timeout:   return String(localized: "Check your connection and retry.")
        case .unauthorized:        return String(localized: "Sign in again to continue.")
        default:                   return nil
        }
    }

    /// External → domain. Switch on the typed code, never the message.
    public static func mapping(_ error: Error) -> APIError {
        if let urlError = error as? URLError {
            switch urlError.code {                     // imported enum → @unknown default
            case .notConnectedToInternet, .dataNotAllowed: return .offline
            case .timedOut:                                 return .timeout
            case .userAuthenticationRequired:               return .unauthorized
            @unknown default:                               return .unknown(underlying: urlError)
            }
        }
        return .unknown(underlying: error as? (any Error & Sendable) ?? URLError(.unknown))
    }
}
```

## Decision aid: when NOT to / trade-offs

- **`RecoverableError`** lets the system offer recovery *options* (buttons) and attempt them — it's
  common on macOS, rare on iOS. Add it only if you genuinely present recovery choices; otherwise
  `recoverySuggestion` text is enough.
- **Don't over-model.** One feature, one error enum. A giant app-wide error enum forces unrelated
  features to switch over cases they can't produce.
- **`String(localized:)` is iOS 15+.** On older deployment targets fall back to
  `NSLocalizedString(_:comment:)`.

## Related skills

- `global-skills/apple/swift-feature-scaffold-mvvm-clean-arch/SKILL.md` — consumes this enum in the
  ViewModel's `case error(APIError)` state and the cold-start three-branch handler.
- `global-skills/apple/apple-anti-patterns/SKILL.md` — registers "string-match an error message"
  as the anti-pattern this skill replaces.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-checks `LocalizedError` /
  `URLError` and any typed-throws evolution.

## Sources

- [LocalizedError](https://developer.apple.com/documentation/foundation/localizederror) · [URLError](https://developer.apple.com/documentation/foundation/urlerror) · [URLError.Code](https://developer.apple.com/documentation/foundation/urlerror/code) · [RecoverableError](https://developer.apple.com/documentation/foundation/recoverableerror) · [CustomNSError](https://developer.apple.com/documentation/foundation/customnserror)
- [The Swift Programming Language — Switch statement (`@unknown default`)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements/) · [Attributes (`@frozen`)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`LocalizedError : Error`, `URLError` confirmed
live) + The Swift Programming Language. `@unknown default` confirmed required only for non-frozen
imported enums, not for your own Swift enums.
**Re-check after:** WWDC26 / Swift 6.3, or by 2026-12-01. **Decay risk:** low (LocalizedError is a
decades-stable Foundation protocol).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
