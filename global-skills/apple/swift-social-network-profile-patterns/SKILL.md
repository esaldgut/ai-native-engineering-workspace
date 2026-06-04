---
name: swift-social-network-profile-patterns
description: >-
  Build social-network profile UI in SwiftUI — a sticky/shrinking header driven by
  onScrollGeometryChange, an immersive cover photo via backgroundExtensionEffect (iOS 26), a moments
  grid (LazyVGrid) with a hero zoom into detail (matchedTransitionSource + navigationTransition.zoom),
  a follow/connection state machine modeled as an @Observable enum (notFollowing/requested/following/
  blocked) with optimistic transitions, and a notification center (List + swipeActions +
  UNUserNotificationCenter). Documents that follow state is domain logic (no Apple API) and that UGC
  apps must offer in-app block/report per App Review. Use when building a profile page, follow button
  state, a moments grid, or a notifications inbox.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — onScrollGeometryChange(for:of:action:)"
      url: "https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:)"
      version: "iOS 18.0"
    - source: "Apple Developer — matchedTransitionSource(id:in:) + navigationTransition zoom"
      url: "https://developer.apple.com/documentation/swiftui/view/matchedtransitionsource(id:in:)"
      version: "iOS 18.0"
    - source: "Apple Developer — backgroundExtensionEffect()"
      url: "https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect()"
      version: "iOS 26.0"
    - source: "Apple Developer — UNUserNotificationCenter"
      url: "https://developer.apple.com/documentation/usernotifications/unusernotificationcenter"
      version: "iOS 10.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote, or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Social network profile patterns (SwiftUI, iOS 26)

A profile screen is composition over SwiftUI primitives plus a domain **state machine** for follow
status. The load-bearing pieces are
[`onScrollGeometryChange`](https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:))
for the shrinking header, [`backgroundExtensionEffect()`](https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect())
for the immersive cover (iOS 26), a `LazyVGrid` moments grid with a hero zoom via
[`matchedTransitionSource(id:in:)`](https://developer.apple.com/documentation/swiftui/view/matchedtransitionsource(id:in:))
+ `navigationTransition(.zoom(...))`, and [`UNUserNotificationCenter`](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)
behind the notifications inbox. Follow state has **no Apple API** — it's a domain enum you drive with
`@Observable`. This skill provides a reference state machine but marks it advisory.

## When to invoke

- You're building a **profile page** (sticky/shrinking header, stats bar, cover photo), a **moments
  grid** with zoom-to-detail, a **follow/connect button** with multiple states, or a **notifications
  inbox**.

**Announce on invoke:** "Using `swift-social-network-profile-patterns` for the sticky header, moments grid hero zoom, follow state machine, and notifications inbox."

Do **not** treat the follow state machine here as Apple-prescribed — it's a reference shape. Match it
to your backend's real states (some products have "close friends", "muted", "restricted", etc.).

## The canonical APIs (verified)

| API | Signature / form (verified) | iOS gate | Use |
|---|---|---|---|
| `.onScrollGeometryChange(for:of:action:)` | `func onScrollGeometryChange<T: Equatable>(for type: T.Type, of transform: (ScrollGeometry) -> T, action: (T, T) -> Void) -> some View` | 18.0+ | Shrink header on scroll |
| `LazyVStack(pinnedViews:)` | `pinnedViews: [.sectionHeaders]` | 14.0+ | Sticky stats bar header |
| `LazyVGrid` + `GridItem` | `GridItem(.flexible())` × 3 | 14.0+ | Moments grid |
| `.matchedTransitionSource(id:in:)` | `func matchedTransitionSource(id: some Hashable, in namespace: Namespace.ID) -> some View` | 18.0+ | Grid cell → detail hero |
| `.navigationTransition(_:)` + `.zoom(sourceID:in:)` | `static func zoom(sourceID: some Hashable, in namespace: Namespace.ID) -> ZoomNavigationTransition` | 18.0+ | The zoom into detail |
| `.backgroundExtensionEffect()` | `@MainActor func backgroundExtensionEffect() -> some View` | 26.0+ | Immersive cover into safe area |
| `@Observable` | macro | 17.0+ | Follow state machine |
| `List` + `.swipeActions` | `swipeActions(edge:allowsFullSwipe:content:)` | 15.0+ | Notifications mark-read/delete |
| `UNUserNotificationCenter` | `class`; `.current()`, `requestAuthorization(options:)` | 10.0+ | System notifications |

## The rules (load-bearing)

### 1. Shrink the header from `onScrollGeometryChange`, transforming to a *small* type

Don't react to every offset on the whole view — transform the `ScrollGeometry` to a compact
`Equatable` (a `CGFloat` header height or a `Bool` collapsed flag) so only the header invalidates:

```swift
.onScrollGeometryChange(for: CGFloat.self) { geo in
    max(120, 280 - geo.contentOffset.y)          // derived header height
} action: { _, newHeight in headerHeight = newHeight }
```

Note: if multiple scroll views exist in the hierarchy, only the first invokes the closure (Apple logs
a runtime issue) — scope it to the profile's single `ScrollView`.

### 2. Sticky stats bar only pins inside a `LazyVStack`

`pinnedViews: [.sectionHeaders]` requires the lazy stack; a plain `VStack` won't pin. Put the stats
bar in a `Section(header:)`.

### 3. Hero zoom needs a unique source id per cell, one shared namespace

Tag each moments cell with `.matchedTransitionSource(id: moment.id, in: ns)` and the destination with
`.navigationTransition(.zoom(sourceID: moment.id, in: ns))`. Reusing an id across cells breaks the
animation. iOS 18+.

### 4. Immersive cover via `backgroundExtensionEffect()` — sparingly, behind content

Apply it to the cover image so it extends under the nav bar / sidebar. Apple says use it with
discretion on a *single* background view; don't stack it.

### 5. Follow state is a domain state machine — model it as an `@Observable` enum, transition optimistically

There is no Apple follow API. Model the states explicitly and flip optimistically on tap, reverting on
server failure:

```swift
enum FollowState: Sendable { case notFollowing, requested, following, blocked }

@Observable @MainActor
final class FollowModel {
    private(set) var state: FollowState
    private let api: any SocialAPI
    init(state: FollowState, api: any SocialAPI) { self.state = state; self.api = api }
    func toggle(userID: UserID) async {
        let previous = state
        state = (state == .following) ? .notFollowing : .requested     // optimistic
        do { state = try await api.setFollow(userID, follow: previous != .following) }
        catch { state = previous }                                      // revert on failure
    }
}
```

### 6. UGC apps must offer in-app block/report

App Review (Guideline 1.2 for user-generated content) requires an in-app mechanism to block users
and report content. Wire it into the profile overflow menu and the notifications inbox; it's a
shipping requirement, not a nicety.

## Canonical example

```swift
struct ProfileView: View {
    let user: User
    @State private var headerHeight: CGFloat = 280
    @Namespace private var heroNS

    var body: some View {
        ScrollView {
            LazyVStack(pinnedViews: [.sectionHeaders]) {
                Section {
                    LazyVGrid(columns: Array(repeating: GridItem(.flexible(), spacing: 2), count: 3),
                              spacing: 2) {
                        ForEach(user.moments) { moment in
                            NavigationLink(value: moment) {
                                MomentCell(moment: moment)
                                    .matchedTransitionSource(id: moment.id, in: heroNS)
                            }
                        }
                    }
                } header: {
                    StatsBar(user: user).background(.bar)            // sticky
                }
            }
        }
        .onScrollGeometryChange(for: CGFloat.self) { max(120, 280 - $0.contentOffset.y) }
            action: { _, h in headerHeight = h }
        .navigationDestination(for: Moment.self) { moment in
            MomentDetail(moment: moment)
                .navigationTransition(.zoom(sourceID: moment.id, in: heroNS))
        }
        .background {
            CoverImage(url: user.coverURL)
                .frame(height: headerHeight)
                .backgroundExtensionEffect()                          // iOS 26 immersive cover
        }
    }
}
```

## Decision aid: when NOT to / trade-offs

- **The `FollowState` enum is advisory.** Map it to your backend's actual states; a four-case enum is
  a starting point, not a contract.
- **`backgroundExtensionEffect()` is iOS 26+** — on earlier targets fall back to a plain image
  background under `.ignoresSafeArea()` (no edge mirroring).
- **Don't over-animate the header.** Respect Reduce Motion; a snapping header on every scroll tick is
  an accessibility problem.

## Related skills

- `global-skills/apple/swift-ios26-native-ux-patterns/SKILL.md` — the zoom transition and
  `backgroundExtensionEffect()` shared here.
- `global-skills/apple/swift-clean-architecture-module-scaffold/SKILL.md` — the `Sendable SocialAPI`
  and `@MainActor` follow model isolation.
- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — glass on the follow CTA;
  no glass-on-glass over the cover.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify scroll/transition APIs each WWDC.

## Sources

- [onScrollGeometryChange(for:of:action:)](https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:)) · [matchedTransitionSource(id:in:)](https://developer.apple.com/documentation/swiftui/view/matchedtransitionsource(id:in:)) · [zoom(sourceID:in:)](https://developer.apple.com/documentation/swiftui/navigationtransition/zoom(sourceid:in:)) · [backgroundExtensionEffect()](https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect()) · [UNUserNotificationCenter](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)
- WWDC23 [Discover Observation in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10149/) · WWDC24 [What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2024/10144/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`onScrollGeometryChange` iOS 18,
`matchedTransitionSource` iOS 18, `backgroundExtensionEffect` iOS 26, `UNUserNotificationCenter` iOS 10
confirmed live). Follow state confirmed to have **no** Apple API — it's modeled as a domain enum.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (scroll-geometry + transition APIs
are iOS 18-new; `backgroundExtensionEffect` is iOS 26-new).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
