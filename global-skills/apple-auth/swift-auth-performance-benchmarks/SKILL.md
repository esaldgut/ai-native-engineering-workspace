---
name: swift-auth-performance-benchmarks
description: >-
  Write performance benchmarks for an iOS auth subsystem using ContinuousClock — measure token-refresh
  latency at p50/p95 with hand-rolled percentiles (Swift Testing ships no percentile helper), verify
  cold-start Keychain-read + JWT-decode stays under budget, and prove concurrent-refresh coalescence
  costs ~1x not Nx. Uses ContinuousClock.measure for sync blocks and the now/duration form for async,
  with first-sample warmup discarding. Use when adding or reviewing latency budgets for auth.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — ContinuousClock (monotonic, high-resolution)"
      url: "https://developer.apple.com/documentation/swift/continuousclock"
      version: "iOS 16.0+"
    - source: "Apple Developer — Clock.measure(_:) (sync) + measure(isolation:_:) (async)"
      url: "https://developer.apple.com/documentation/swift/clock/measure(_:)"
      version: "iOS 16.0+"
    - source: "Apple Developer — Duration (.milliseconds, comparison)"
      url: "https://developer.apple.com/documentation/swift/duration"
      version: "iOS 16.0+"
    - source: "Apple Developer — Swift Testing @Test"
      url: "https://developer.apple.com/documentation/testing/suite(_:_:)"
      version: "Swift 6.0 / Xcode 16.0"
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

# Auth performance benchmarks (ContinuousClock + Swift Testing)

Auth latency is felt: a slow token refresh stalls every gated request, and a slow cold-start auth
check delays first paint. This skill measures those paths with `ContinuousClock` — the monotonic,
high-resolution clock that (unlike `Date()`) is immune to NTP/user clock changes — and computes
percentiles by hand, because **Swift Testing ships no percentile helper**. Budgets are configurable per
environment; the examples use realistic OAuth/OIDC envelopes.

## When to invoke

- You're setting or enforcing latency budgets for token refresh, cold-start auth, or Keychain access.
- You want a CI guard that catches a regression (a synchronous Keychain call sneaking onto the hot
  path, a refresh that lost its coalescing).
- You need to prove concurrent-refresh coalescence is real — that 16 callers cost ~1× a single
  refresh, not 16×.

**Announce on invoke:** "Using `swift-auth-performance-benchmarks` to add ContinuousClock p50/p95 latency tests for the auth subsystem."

Do **not** reach for XCTest here. `ContinuousClock` keeps these in Swift Testing; the only reason to
fall back to XCTest is when you specifically need an Apple-shipped `XCTMetric` probe (CPU/memory
counters), which is a different measurement.

## The canonical APIs (verified)

| API | Form (verified) | Use |
|---|---|---|
| `ContinuousClock` | `let clock = ContinuousClock()` | Monotonic stopwatch; iOS 16+. Not affected by system clock changes. |
| `Clock.measure(_:)` | `func measure(_ work: () throws -> Void) rethrows -> Duration` | Time a **synchronous** block. |
| async measure | `await clock.measure { await … }` (`measure(isolation:_:)`) | Time an **async** block; or use the `now` / `duration(to:)` form below. |
| `clock.now` + `duration(to:)` | `let start = clock.now; …; let d = start.duration(to: clock.now)` | Explicit, async-safe interval; unambiguous across overloads. |
| `Duration` | `.milliseconds(50)`, `<` | Express and compare budgets. |
| percentile | *(hand-rolled — sort + index)* | Swift Testing has **no** built-in percentile. |

## The rules

### 1. Hand-roll percentiles — there is no built-in

Collect N samples (100 for p50/p95; 1000 for a stable p99), `sort()`, then index:
`p50 = samples[count/2]`, `p95 = samples[Int(Double(count - 1) * 0.95)]`. Don't assume an
`XCTMeasureOptions`-style statistic exists in Swift Testing — it doesn't.

### 2. Discard warmup samples

First-run effects (class load, code-sign cache, lazy `URLSession` setup) inflate the first few
measurements. Drop the first 3–5 samples before computing percentiles, or the p50 is a lie.

### 3. `ContinuousClock`, not `Date()`

`Date()` can jump backward on NTP sync, producing negative or absurd intervals. `ContinuousClock` is
monotonic — the only correct primitive for a benchmark.

### 4. Budgets are configurable, and network-tier-aware

p50 < 1 s / p95 < 3 s is a reasonable **LTE** envelope for OAuth refresh; on Wi-Fi expect p50 < 400 ms /
p95 < 1 s. Cold-start Keychain read + JWT decode should be < 50 ms (Keychain reads are ~1–5 ms; decode
is sub-ms for tokens < 4 KB). Read the budget from config so CI, dev, and device can differ.

### 5. Prove coalescence by wall-clock AND call-count

A coalescence test must assert **both**: the fan-out's wall-clock ≈ 1× single-refresh latency (not N×),
**and** the network mock observed exactly 1 call. Either alone can pass while the bug hides.

### 6. Keep RSS measurement out of unit tests

Process-RSS probes (`task_info` / `TASK_VM_INFO`) are brittle in unit tests. For a static check use
`MemoryLayout<T>.size` ("the token model is < 256 bytes"); for real memory profiling use Instruments.

## Canonical example

```swift
import Testing
import Foundation
@testable import MyApp

@Suite("Auth performance")
struct AuthPerformanceTests {

    /// Hand-rolled percentile — Swift Testing has none built in.
    private func percentile(_ p: Double, of samples: [Duration]) -> Duration {
        let sorted = samples.sorted()
        return sorted[Int(Double(sorted.count - 1) * p)]
    }

    @Test func tokenRefreshLatencyMeetsBudget() async throws {
        let clock = ContinuousClock()
        let session = AuthSession()
        var samples: [Duration] = []
        for i in 0..<105 {
            let start = clock.now
            _ = try? await session.refresh()
            let elapsed = start.duration(to: clock.now)   // async-safe interval
            if i >= 5 { samples.append(elapsed) }          // discard 5-sample warmup
        }
        #expect(percentile(0.50, of: samples) < .milliseconds(1000))
        #expect(percentile(0.95, of: samples) < .milliseconds(3000))
    }

    @Test func coldStartKeychainAndDecodeUnderBudget() {
        let clock = ContinuousClock()
        let elapsed = clock.measure {                       // synchronous block
            _ = SecureTokenStore(service: "app", account: "user").readBlocking()
            _ = JWTDecoder.decode(Self.sampleJWT)
        }
        #expect(elapsed < .milliseconds(50))
    }

    @Test func concurrentRefreshIsCoalesced() async {
        let clock = ContinuousClock()
        let session = AuthSession()
        let elapsed = await clock.measure {                 // async measure
            await withTaskGroup(of: Void.self) { group in
                for _ in 0..<16 { group.addTask { _ = try? await session.refresh() } }
            }
        }
        #expect(elapsed < .milliseconds(1500))              // ~1x, NOT 16x
        #expect(session.networkCallCount == 1)              // exactly one network refresh
    }

    @Test func tokenModelStaysSmall() {
        #expect(MemoryLayout<AuthSession.Token>.size < 256) // static check, not RSS
    }
}
```

## Decision aid: where to put a benchmark

- **Latency of an async auth call** → Swift Testing + `ContinuousClock`, percentiles over ≥100 samples.
- **Static size invariant** → `MemoryLayout<T>.size`, no clock needed.
- **CPU/memory counters over a flow** → XCTest `XCTMetric` (the one place to leave Swift Testing).
- **Coalescence** → wall-clock budget **and** mock call-count, together.

## Related skills

- `global-skills/apple-auth/swift-auth-security-audit-suite/SKILL.md` — the *correctness* counterpart:
  it asserts coalescence is single-flight; this skill asserts it's also *fast*.
- `global-skills/apple-auth/swift-testing-framework-conventions-mvvm/SKILL.md` — when to stay in Swift
  Testing vs fall back to XCTest's `XCTMetric`.
- `global-skills/apple-auth/swift-auth-security-checklist/SKILL.md` — the single-flight refresh design
  whose latency this measures.

## Sources

- [ContinuousClock](https://developer.apple.com/documentation/swift/continuousclock) · [Clock.measure(_:)](https://developer.apple.com/documentation/swift/clock/measure(_:)) · [Clock.measure(isolation:_:)](https://developer.apple.com/documentation/swift/clock/measure(isolation:_:)) · [Duration](https://developer.apple.com/documentation/swift/duration)
- [Swift Testing](https://developer.apple.com/documentation/testing) · WWDC24 [10179 Meet Swift Testing](https://developer.apple.com/videos/play/wwdc2024/10179/)
- [MemoryLayout](https://developer.apple.com/documentation/swift/memorylayout) · XCTest [performance metrics (XCTMetric)](https://developer.apple.com/documentation/xctest/xctmetric) (when to fall back)

---

**Last verified:** 2026-06-03 against Swift `ContinuousClock` / `Clock.measure` / `Duration` (Apple
Developer docs, live; sync `measure` and the async `measure(isolation:_:)` overload both confirmed).
**Swift Testing ships no percentile helper** — percentiles are hand-rolled here.
**Re-check after:** WWDC26 + any CryptoKit/Security release, or by 2026-12-01. **Decay risk:** low
(`ContinuousClock` is a stable Swift primitive).
**Found a drift?** Run `/skill-pattern-freshness-audit apple-auth`.
