---
name: swift-feature-scaffold-mvvm-clean-arch
description: >-
  Scaffold one concrete feature end-to-end across Clean Architecture layers — SwiftUI View,
  @Observable @MainActor ViewModel with an explicit State enum, Sendable Repository protocol +
  implementation, Codable+Sendable DTO, and a Mapper — plus the cross-cutting UI rules: anchor a
  multi-step sheet with .sheet(item:) + presentationDetents, and handle cold start with a three-way
  branch (success / recoverable error / terminal auth error). Covers which layer gets @MainActor vs
  nonisolated, why .sheet(isPresented:) can't anchor a wizard step, and respecting Task cancellation
  on retry. Use when adding a new feature module (Feed, Profile, Checkout) and you want the file set,
  isolation, and error/loading states laid out correctly the first time.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — Observable() macro"
      url: "https://developer.apple.com/documentation/observation/observable()"
      version: "iOS 17.0"
    - source: "Apple Developer — presentationDetents(_:)"
      url: "https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)"
      version: "iOS 16.0"
    - source: "Apple Developer — task(id:priority:_:) modifier"
      url: "https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:)"
      version: "iOS 15.0"
    - source: "Swift Evolution SE-0466 — Control default actor isolation (Swift 6.2)"
      url: "https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md"
      version: "Swift 6.2"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (SwiftUI presentation/observation changes), or Swift 6.3 toolchain"
    or_date: "2026-12-01"
  decay_risk: low
  status: current
---

# Feature scaffold (MVVM + Clean Architecture)

A feature module composes the layered architecture into one vertical slice: a SwiftUI `View`, an
`@Observable @MainActor` `ViewModel`, a `Sendable` `Repository` protocol with its implementation,
a `Codable & Sendable` DTO, and a `Mapper` from DTO to Domain. This skill is **opinionated** about
that file set (a Repository between ViewModel and network is one of several valid choices), but the
isolation and UI rules below are Apple-canonical. It builds on the module-level scaffold and uses
[`Observable()`](https://developer.apple.com/documentation/observation/observable()),
[`presentationDetents(_:)`](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)),
and the [`task(id:priority:_:)`](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:))
modifier.

## When to invoke

- You're **adding a new feature** (Feed, Profile, Checkout) and want the View/ViewModel/Repository/
  DTO/Mapper set with the right isolation per file.
- You're building a **multi-step sheet/wizard** and need it to anchor to the current step.
- You're writing the **cold-start load** for a screen and need to branch success / retryable /
  terminal-auth.

**Announce on invoke:** "Using `swift-feature-scaffold-mvvm-clean-arch` to lay out the feature's layers, sheet anchoring, and cold-start states."

Do **not** reach for this for a throwaway screen with no remote data — a single SwiftUI `View` with
local `@State` doesn't need a Repository or Mapper.

## The canonical APIs (verified)

| API | Signature (verified) | Use |
|---|---|---|
| `@Observable` | macro (iOS 17+); adds `Observable` conformance only | ViewModel observation |
| `@MainActor` | global-actor attribute | ViewModel + View isolation |
| `.sheet(item:onDismiss:content:)` | `func sheet<Item, Content>(item: Binding<Item?>, onDismiss: (() -> Void)? = nil, @ViewBuilder content: @escaping (Item) -> Content) -> some View` where `Item : Identifiable` | Anchor a sheet to a value/step |
| `.presentationDetents(_:)` | `func presentationDetents(_ detents: Set<PresentationDetent>) -> some View` | Partial/full sheet heights |
| `.presentationDragIndicator(_:)` | `func presentationDragIndicator(_ visibility: Visibility) -> some View` | Grabber on the sheet |
| `.presentationBackgroundInteraction(_:)` | `func presentationBackgroundInteraction(_:) -> some View` | Let content behind a partial sheet stay tappable |
| `.task(id:priority:_:)` | re-runs the async task when `id` changes; auto-cancels on disappear | Cold-start load + reload |
| `Result` | `enum Result<Success, Failure>` | Optional carrier for the 3-branch outcome |

## The rules (load-bearing)

### 1. Isolation per layer: View & ViewModel are MainActor; Repository & DTO are nonisolated/Sendable

A SwiftUI `View`'s `body` is MainActor, so the `View` and its `@Observable` ViewModel are MainActor.
The `Repository` protocol is `Sendable` and nonisolated (its conformer may be an `actor`). DTOs are
`Sendable` value types. With per-target `defaultIsolation` (SE-0466) you don't annotate every type —
the Feature target is MainActor by default, the Data/Domain targets nonisolated.

### 2. `@Observable` does not add `@MainActor` — declare both

```swift
@Observable @MainActor
public final class FeatureViewModel { /* ... */ }
```

### 3. Model the screen as an explicit `State` enum, not scattered booleans

A single `enum State { case idle, loading, loaded([Item]), error(APIError) }` makes the View a clean
`switch` and makes impossible states unrepresentable (no `isLoading && error != nil`).

### 4. Anchor a wizard with `.sheet(item:)`, never `.sheet(isPresented:)`

`.sheet(isPresented: Bool)` can only say "a sheet is up" — it can't carry *which* step. Drive a
wizard with `.sheet(item: $step)` where `Step: Identifiable`, so changing the bound value swaps the
step in place. Add `presentationDetents` for partial height; on iOS 26 the sheet adopts Liquid Glass
automatically when a partial detent is set.

### 5. Cold start is a three-way branch, and retry respects cancellation

The initial `.task` must distinguish: (a) success, (b) a recoverable error (offline/timeout → show
Retry), (c) a terminal error (auth invalid → route to sign-out). Any retry loop must check
`Task.isCancelled` (or rely on `.task`'s auto-cancel on disappear) so navigating away mid-load
doesn't leak work.

```swift
func load() async {
    state = .loading
    do { state = .loaded(try await repo.list()) }
    catch is CancellationError { /* navigated away — do nothing */ }
    catch {
        let mapped = APIError.mapping(error)
        if case .unauthorized = mapped { onAuthInvalid() }    // terminal branch
        else { state = .error(mapped) }                        // recoverable branch
    }
}
```

## Canonical example

```swift
// Domain
public protocol ItemRepository: Sendable { func list() async throws -> [Item] }

// Data — DTO + Mapper + impl
struct ItemDTO: Codable, Sendable { let id: String; let title: String }
enum ItemMapper { static func toDomain(_ dto: ItemDTO) -> Item { Item(id: UUID(uuidString: dto.id) ?? UUID(), title: dto.title) } }
public actor ItemRepositoryImpl: ItemRepository {
    private let client: HTTPClient
    public init(client: HTTPClient) { self.client = client }
    public func list() async throws -> [Item] {
        let dtos: [ItemDTO] = try await client.get("/items")
        return dtos.map(ItemMapper.toDomain)
    }
}

// Feature — ViewModel + View
@Observable @MainActor
public final class ItemListViewModel {
    public enum State { case idle, loading, loaded([Item]), error(APIError) }
    public private(set) var state: State = .idle
    private let repo: any ItemRepository
    private let onAuthInvalid: () -> Void
    public init(repo: any ItemRepository, onAuthInvalid: @escaping () -> Void) {
        self.repo = repo; self.onAuthInvalid = onAuthInvalid
    }
    public func load() async {
        state = .loading
        do { state = .loaded(try await repo.list()) }
        catch is CancellationError { }
        catch {
            let e = APIError.mapping(error)
            if case .unauthorized = e { onAuthInvalid() } else { state = .error(e) }
        }
    }
}

public struct ItemListView: View {
    @State private var vm: ItemListViewModel
    public init(vm: ItemListViewModel) { _vm = State(initialValue: vm) }
    public var body: some View {
        Group {
            switch vm.state {
            case .idle, .loading:      ProgressView()
            case .loaded(let items):   List(items) { Text($0.title) }
            case .error(let e):        ContentUnavailableView { Text(e.errorDescription ?? "Error") }
                                           actions: { Button("Retry") { Task { await vm.load() } } }
            }
        }
        .task { await vm.load() }                       // auto-cancels on disappear
    }
}
```

## Decision aid: when NOT to / trade-offs

- **Repository indirection is a choice.** For a trivial read-only screen, a ViewModel calling the
  client directly is defensible. The Repository earns its keep when you need caching, mocking for
  tests, or multiple backends behind one Domain contract.
- **Don't put auth-redirect logic in the View.** The terminal branch belongs in the ViewModel (via
  the injected `onAuthInvalid` closure), so the navigation root stays the source of truth.
- **`presentationBackgroundInteraction(.enabled(upThrough:))`** only makes sense for a persistent
  partial sheet (a player/inspector); a modal wizard wants the default (background disabled).

## Related skills

- `global-skills/apple/swift-clean-architecture-module-scaffold/SKILL.md` — the per-target isolation
  this feature's files assume.
- `global-skills/apple/swift-localizederr-enum-patterns-ios26/SKILL.md` — the `APIError` powering the
  `.error` state and the cold-start mapping.
- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — glass on the sheet/CTAs;
  honor "no glass on glass."
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-checks observation/presentation APIs.

## Sources

- [Observable() macro](https://developer.apple.com/documentation/observation/observable()) · [sheet(item:onDismiss:content:)](https://developer.apple.com/documentation/swiftui/view/sheet(item:ondismiss:content:)) · [presentationDetents(_:)](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)) · [presentationBackgroundInteraction(_:)](https://developer.apple.com/documentation/swiftui/view/presentationbackgroundinteraction(_:)) · [task(id:priority:_:)](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:))
- [SE-0466 default actor isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`Observable`, `presentationDetents`,
`task(id:)` signatures confirmed live) + SE-0466. `.sheet(item:)` confirmed as the step-anchoring form
vs `.sheet(isPresented:)`.
**Re-check after:** WWDC26 / Swift 6.3, or by 2026-12-01. **Decay risk:** low (MVVM layering + sheet
detents are stable since iOS 16–17).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
