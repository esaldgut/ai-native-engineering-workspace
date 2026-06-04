---
name: swift-clean-architecture-module-scaffold
description: >-
  Scaffold a multi-module Swift app (MVVM + Clean layers) with correct actor isolation per layer
  under Swift 6.2 — nonisolated value types & protocols in the Domain, MainActor ViewModels &
  Views in the Feature layer — using SE-0466 per-target defaultIsolation(MainActor.self) instead
  of annotating every type. Covers the SE-0449 nonisolated escape hatch for cross-module protocol
  conformance, when to choose an actor vs a nonisolated class for repositories, and why @Observable
  does NOT add @MainActor or Sendable for you. Use when standing up a new SwiftPM package graph or
  deciding which layer owns which isolation, and you want the compiler — not convention — to enforce
  the boundaries.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Swift Evolution SE-0466 — Control default actor isolation (Implemented, Swift 6.2)"
      url: "https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md"
      version: "Swift 6.2"
    - source: "Swift Evolution SE-0449 — Allow nonisolated to prevent global actor inference (Implemented, Swift 6.1)"
      url: "https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md"
      version: "Swift 6.1"
    - source: "Apple Developer — SwiftSetting.defaultIsolation(_:_:)"
      url: "https://developer.apple.com/documentation/packagedescription/swiftsetting/defaultisolation(_:_:)"
      version: "Swift 6.2"
    - source: "Apple Developer — Observable() macro"
      url: "https://developer.apple.com/documentation/observation/observable()"
      version: "iOS 17.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual Swift concurrency surface change), or any Swift 6.3 toolchain"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Clean Architecture module scaffold (Swift 6.2 actor isolation)

A layered Swift app (MVVM over Clean Architecture) maps each concern onto a SwiftPM target, and
each target declares its **default actor isolation** so the compiler enforces the boundary instead
of leaving it to code review. Swift 6.2 shipped this control via
[SE-0466 "Control default actor isolation"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
(the `defaultIsolation(_:_:)` SwiftSetting) and Swift 6.1 shipped the escape hatch via
[SE-0449 "Allow `nonisolated` to prevent global actor inference"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md).
This skill is **opinionated**: a four-layer topology (`Domain` / `Data` / `Feature` / app) is one
of several valid shapes — but the isolation rules below hold for any topology.

## When to invoke

- You're creating a **new SwiftPM package graph** and need to decide which target is MainActor by
  default and which is nonisolated.
- You're hitting "Sending main actor-isolated value to nonisolated context" or "Call to main
  actor-isolated initializer in a synchronous nonisolated context" while wiring layers together.
- You're deciding **actor vs nonisolated class** for a repository or service.

**Announce on invoke:** "Using `swift-clean-architecture-module-scaffold` to set per-target default isolation and the SE-0449 escape hatches."

Do **not** reach for this for a single-target app with no module boundaries — there, a top-level
`-default-isolation MainActor` flag plus `Sendable` discipline is enough; you don't need the package
graph this skill prescribes.

## The canonical APIs (verified)

| API | Signature (verified) | Layer it lives in |
|---|---|---|
| `defaultIsolation(_:_:)` | `static func defaultIsolation(_ globalActor: MainActor.Type?, _ condition: BuildSettingCondition? = nil) -> SwiftSetting` | `Package.swift` per target |
| `-default-isolation` | compiler flag, values `MainActor` or `nonisolated` (the CLI equivalent of the SwiftSetting) | build settings |
| `nonisolated` | keyword — applicable to struct/class/enum/protocol/extension/stored property (SE-0449) | Domain protocols, repos |
| `@Observable` | macro (iOS 17+); conforms a type to `Observable`. Adds **neither** `@MainActor` **nor** `Sendable` | Feature ViewModels |
| `@MainActor` | global-actor attribute (the only valid actor for `defaultIsolation`) | Feature layer default |
| `Sendable` | marker protocol — the contract for crossing actor boundaries | Domain models, DTOs |

> The **only** valid arguments to `defaultIsolation(_:_:)` are `MainActor.self` and `nil` (where
> `nil` means nonisolated). No other global actor is accepted (SE-0466).

## The rules (load-bearing)

### 1. Isolation is a per-target decision, set once in `Package.swift`

App targets and SwiftPM packages **do not inherit each other's** default isolation — set it
explicitly on every target. Domain/Data go nonisolated; the Feature/UI target goes MainActor.

```swift
.target(name: "Domain", swiftSettings: [.defaultIsolation(nil)]),          // nonisolated
.target(name: "Data",   swiftSettings: [.defaultIsolation(nil)]),          // nonisolated
.target(name: "Feature", dependencies: ["Domain"],
        swiftSettings: [.defaultIsolation(MainActor.self)]),               // MainActor by default
```

### 2. Domain models are `Sendable` value types; protocols are `Sendable` + nonisolated

The Domain layer is the wire-independent core. Its models cross actors, so they must be `Sendable`.
Its repository protocols are `Sendable` so either an `actor` or a `nonisolated` class can conform.

```swift
public struct Item: Sendable, Identifiable {   // crosses actors freely
    public let id: UUID
    public var title: String
}
public protocol ItemRepository: Sendable {     // conformers pick their own isolation
    func fetch(id: UUID) async throws -> Item
}
```

### 3. `@Observable` adds neither `@MainActor` nor `Sendable` — you declare them

The `Observable()` macro only synthesizes observation tracking. For a ViewModel you want both
attributes, written explicitly:

```swift
@Observable @MainActor
public final class ItemViewModel { /* ... */ }
```

### 4. Cross-module protocol conformance is the #1 migration pain — SE-0449 is the fix

When a MainActor-default Feature type must conform to a nonisolated Domain protocol, mark the
conforming members `nonisolated` (or the whole conformance) so they don't drag MainActor up into
the Domain contract:

```swift
extension SomeService: ItemRepository {
    public nonisolated func fetch(id: UUID) async throws -> Item { /* off the main actor */ }
}
```

### 5. Repository: `actor` for mutable shared state, `nonisolated class` for stateless I/O

Choose an **`actor`** when the repository owns mutable state that must be serialized (an in-memory
cache, a token, a dedupe map). Choose a **`nonisolated final class`** (or struct) when it's a
stateless wrapper over an already-thread-safe client (`URLSession`, a generated GraphQL client).
Don't reach for an actor just to "be safe" — it adds suspension points the happy path pays for.

## Canonical example

A three-target slice: nonisolated Domain, an `actor` repository in Data, a MainActor ViewModel in
Feature.

```swift
// Domain (defaultIsolation: nil)
public struct Item: Sendable, Identifiable { public let id: UUID; public var title: String }
public protocol ItemRepository: Sendable {
    func list() async throws -> [Item]
}

// Data (defaultIsolation: nil) — actor because it caches
public actor ItemRepositoryImpl: ItemRepository {
    private var cache: [Item]?
    private let client: HTTPClient
    public init(client: HTTPClient) { self.client = client }
    public func list() async throws -> [Item] {
        if let cache { return cache }
        let items = try await client.get("/items")
        cache = items
        return items
    }
}

// Feature (defaultIsolation: MainActor.self) — no per-type @MainActor needed
@Observable
public final class ItemViewModel {            // MainActor by default, from the target setting
    public private(set) var items: [Item] = []
    private let repo: any ItemRepository
    public init(repo: any ItemRepository) { self.repo = repo }
    public func load() async { items = (try? await repo.list()) ?? [] }
}
```

## Decision aid: when NOT to apply this

- **SE-0478 (per-file default isolation typealias) is NOT shipped** — it went back for revision and
  is still in pitch. Do **not** rely on a file-level `typealias DefaultIsolation = MainActor`; use
  the per-target SwiftSetting above, which is the accepted (SE-0466) mechanism.
- **Generated code (e.g., GraphQL/Apollo types) in a MainActor-default module** will inherit
  MainActor and break when handed to a nonisolated repo. Keep generated types in a nonisolated
  target, or use the generator's own "emit nonisolated" flag — see the related Apollo skill.
- **A pure SwiftUI prototype** doesn't need the package split; isolation ceremony there is friction.

## Related skills

- `global-skills/apple/swift-feature-scaffold-mvvm-clean-arch/SKILL.md` — applies this layering to a
  single concrete feature (View/ViewModel/Repository/DTO/Mapper).
- `global-skills/apple/swift-localizederr-enum-patterns-ios26/SKILL.md` — the `Sendable` typed-error
  enum that flows up through these layers.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verifies SE-0466/0449/0478 status
  after each Swift toolchain.

## Sources

- [SE-0466 Control default actor isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md) · [SE-0449 nonisolated for global actor inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md)
- [SwiftSetting.defaultIsolation(_:_:)](https://developer.apple.com/documentation/packagedescription/swiftsetting/defaultisolation(_:_:)) · [Observable() macro](https://developer.apple.com/documentation/observation/observable()) · [MainActor](https://developer.apple.com/documentation/swift/mainactor) · [Sendable](https://developer.apple.com/documentation/swift/sendable)

---

**Last verified:** 2026-06-03 against Swift 6.2 (SE-0466 Implemented, SE-0449 Implemented; `defaultIsolation`
accepts only `MainActor.self` / `nil`, confirmed live). SE-0478 confirmed **not** accepted — still in
pitch, so the per-file typealias trick is excluded here.
**Re-check after:** WWDC26 / Swift 6.3, or by 2026-12-01. **Decay risk:** low (Clean Architecture layering
is stable; the SE proposals are accepted and unlikely to reverse).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
