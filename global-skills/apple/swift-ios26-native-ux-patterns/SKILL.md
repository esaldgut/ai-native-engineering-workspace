---
name: swift-ios26-native-ux-patterns
description: >-
  Adopt the iOS 26 native UX cluster in SwiftUI — the collapsible floating tab bar (via the Tab(...)
  declaration + TabBarMinimizeBehavior), the bottom accessory "mini-player" slot
  (tabViewBottomAccessory + TabViewBottomAccessoryPlacement .inline/.expanded), scroll-edge glass
  (scrollEdgeEffectStyle), immersive background extension (backgroundExtensionEffect), the zoom
  navigation transition (navigationTransition(.zoom) + matchedTransitionSource), and the @Animatable
  macro for Shapes. Documents each iOS gate (some iOS 18, most iOS 26) and the legacy-API fallback.
  Use when wiring the interactive chrome of an iOS 26 app — tab bar, mini-player, hero transitions,
  immersive headers — the modern way instead of legacy .tabItem / hand-rolled animatableData.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — tabViewBottomAccessory(content:) + TabViewBottomAccessoryPlacement"
      url: "https://developer.apple.com/documentation/swiftui/view/tabviewbottomaccessory(content:)"
      version: "iOS 26.0"
    - source: "Apple Developer — backgroundExtensionEffect() / safeAreaBar / scrollEdgeEffectStyle"
      url: "https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect()"
      version: "iOS 26.0"
    - source: "Apple Developer — navigationTransition zoom(sourceID:in:) + matchedTransitionSource(id:in:)"
      url: "https://developer.apple.com/documentation/swiftui/navigationtransition/zoom(sourceid:in:)"
      version: "iOS 18.0"
    - source: "Apple Developer — Animatable() macro"
      url: "https://developer.apple.com/documentation/SwiftUI/Animatable()"
      version: "WWDC25 / iOS 26 SDK"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual SwiftUI UX surface change), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# iOS 26 native UX patterns (SwiftUI)

iOS 26 ships a cluster of UX affordances tied to the redesign: a floating tab bar that collapses on
scroll, a bottom accessory slot above the tab bar (the "mini-player"), scroll-edge glass, immersive
background extension, and the zoom navigation transition. This skill wires them with verified APIs —
[`tabViewBottomAccessory(content:)`](https://developer.apple.com/documentation/swiftui/view/tabviewbottomaccessory(content:)),
[`backgroundExtensionEffect()`](https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect()),
[`navigationTransition(.zoom(...))`](https://developer.apple.com/documentation/swiftui/navigationtransition/zoom(sourceid:in:)),
and the [`@Animatable`](https://developer.apple.com/documentation/SwiftUI/Animatable()) macro — each
with its iOS gate, since the cluster spans iOS 18 and iOS 26.

## When to invoke

- You're wiring the **interactive chrome** of an iOS 26 app: tab bar, a persistent mini-player, an
  immersive header, hero transitions between a grid/list and a detail.
- You see legacy `.tabItem { … }` or hand-written `animatableData` and want the modern equivalents.
- You need a view's background to extend **under** a sidebar/inspector for a seamless look.

**Announce on invoke:** "Using `swift-ios26-native-ux-patterns` for the tab bar collapse, bottom accessory, zoom transition, and @Animatable."

Do **not** reach for this on pre-iOS-26 deployment targets without a fallback plan — most of these
modifiers are iOS 26.0+ and have no backport (see the gate column and fallbacks below).

## The canonical APIs (verified, with gates)

| API | Signature (verified) | iOS gate |
|---|---|---|
| `.tabViewBottomAccessory(content:)` | `func tabViewBottomAccessory<Content>(@ViewBuilder content: () -> Content) -> some View` | 26.0+ |
| `TabViewBottomAccessoryPlacement` | `enum` — `.inline`, `.expanded`; read via `@Environment(\.tabViewBottomAccessoryPlacement)` | 26.0+ |
| `TabBarMinimizeBehavior` / `.tabBarMinimizeBehavior(_:)` | controls collapse-on-scroll of the tab bar | 26.0+ |
| `.searchToolbarBehavior(_:)` | `func searchToolbarBehavior(_ behavior: SearchToolbarBehavior) -> some View` — `.minimize` | 26.0+ |
| `.scrollEdgeEffectStyle(_:for:)` | `func scrollEdgeEffectStyle(_ style: ScrollEdgeEffectStyle?, for edges: Edge.Set) -> some View` — `.hard` / `.soft` | 26.0+ |
| `.backgroundExtensionEffect()` | `@MainActor func backgroundExtensionEffect() -> some View` | 26.0+ |
| `.safeAreaBar(edge:alignment:spacing:content:)` | `func safeAreaBar(edge: HorizontalEdge, alignment: VerticalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content:) -> some View` | 26.0+ |
| `.navigationTransition(_:)` + `.zoom(sourceID:in:)` | `static func zoom(sourceID: some Hashable, in namespace: Namespace.ID) -> ZoomNavigationTransition` | 18.0+ |
| `.matchedTransitionSource(id:in:)` | `func matchedTransitionSource(id: some Hashable, in namespace: Namespace.ID) -> some View` | 18.0+ |
| `@Animatable` / `@AnimatableIgnored` | synthesizes `Animatable.animatableData` for a `Shape`/`ViewModifier` | iOS 26 SDK (WWDC25) |
| `.onScrollVisibilityChange(threshold:_:)` | `func onScrollVisibilityChange(threshold: Double = 0.5, _ action: @escaping (Bool) -> Void) -> some View` | 18.0+ |

## The rules (load-bearing)

### 1. The new `Tab(...)` declaration is the entry point — legacy `.tabItem` opts out

The floating, collapsible iOS 26 tab bar only works when you build tabs with
`Tab("Home", systemImage: "house") { … }`. The legacy `TabView { View().tabItem { … } }` form does
not collapse or host a bottom accessory. Migrate first; this is the precondition for the rest.

### 2. The bottom accessory adapts to its placement — read `tabViewBottomAccessoryPlacement`

`.tabViewBottomAccessory { … }` renders a persistent slot (mini-player). When the tab bar is full
size the accessory sits *above* it; when collapsed it goes *inline*. Switch your content on the
placement so the inline form is compact:

```swift
@Environment(\.tabViewBottomAccessoryPlacement) private var placement
var body: some View {
    switch placement {
    case .inline:   CompactPlayerControls()
    case .expanded: ExpandedPlayer()
    @unknown default: CompactPlayerControls()
    }
}
```

### 3. `safeAreaBar` takes a `HorizontalEdge` — it's a *side* bar, not a top/bottom bar

`safeAreaBar(edge:…)` insets the safe area horizontally (leading/trailing) and extends the scroll-
edge effect. For a top/bottom blurred bar, keep `safeAreaInset(edge:.bottom)` and pair it with
`scrollEdgeEffectStyle(.hard, for: .bottom)`. Don't expect `safeAreaBar(edge: .top)` to compile — its
`edge` is `HorizontalEdge`.

### 4. `backgroundExtensionEffect()` belongs in the detail column, used sparingly

It mirrors+blurs a view's edges into adjacent safe area (under a sidebar/inspector). Apple says apply
it "with discretion" to a *single* background view per screen — it's a performance- and clarity-
sensitive effect, not a decoration to sprinkle.

### 5. Zoom transition needs a matched source id unique per item, shared namespace

Tag the source with `.matchedTransitionSource(id:in:)` and the destination with
`.navigationTransition(.zoom(sourceID:in:))`, both reading one `@Namespace`. The `id` must be unique
per cell — reusing ids across cells breaks the hero animation. This is iOS 18+, and works for
`NavigationStack` pushes and `.sheet`/`.fullScreenCover`.

### 6. `@Animatable` replaces hand-written `animatableData` — delete the manual conformance

On a custom `Shape`/`ViewModifier`, annotate `@Animatable` and SwiftUI synthesizes `animatableData`
from the animatable stored properties (numbers, `Angle`, sizes). Exclude non-animatable fields with
`@AnimatableIgnored`. The macro is a WWDC25 addition usable when building with the iOS 26 SDK; remove
any existing `var animatableData` you wrote by hand.

```swift
@Animatable
struct Wedge: Shape {
    var angle: Angle
    @AnimatableIgnored var clockwise: Bool
    func path(in rect: CGRect) -> Path { /* ... */ }
}
```

## Canonical example

A media app: `Tab(...)` declaration, a placement-aware bottom accessory, the tab bar set to minimize
on scroll, and visibility-driven autoplay for an inline video.

```swift
struct RootView: View {
    @State private var query = ""
    var body: some View {
        TabView {
            Tab("Home", systemImage: "house") { FeedView() }
            Tab("Library", systemImage: "books.vertical") { LibraryView() }
            Tab(role: .search) {
                NavigationStack { SearchResults(query: query) }.searchable(text: $query)
            }
        }
        .tabBarMinimizeBehavior(.onScrollDown)          // collapse on scroll (iOS 26)
        .tabViewBottomAccessory { MiniPlayer() }        // persistent player slot (iOS 26)
        .searchToolbarBehavior(.minimize)               // collapse search to a button (iOS 26)
    }
}

struct AutoplayCell: View {
    @State private var isPlaying = false
    var body: some View {
        VideoSurface(isPlaying: isPlaying)
            .onScrollVisibilityChange(threshold: 0.6) { isPlaying = $0 }   // iOS 18+
    }
}
```

## Decision aid: when NOT to / fallbacks

- **Pre-iOS-26 deployment targets:** gate each modifier with `if #available(iOS 26, *)` and fall back
  to `safeAreaInset` + `.background(.bar)` for the bar, `.tabItem` for tabs (losing collapse), and a
  cross-fade instead of the zoom transition (which is at least iOS 18).
- **`@Animatable` is Shape/ViewModifier only** — it does not apply to `View`. For view-level
  animation, keep using `withAnimation` / `phaseAnimator`.
- **Don't stack glass at the scroll edge yourself** — `scrollEdgeEffectStyle` is the system control;
  a manual `.glassEffect()` at the edge violates "no glass on glass."

## Related skills

- `global-skills/apple/swift-adaptive-layouts-ios26/SKILL.md` — the `TabView`/`NavigationSplitView`
  root these modifiers attach to.
- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — the glass material under
  this chrome; the "no glass on glass" rule applies here too.
- `global-skills/apple/swift-infinite-scroll-video-feed-ios26/SKILL.md` — pairs the zoom transition
  and visibility autoplay with an AVPlayer pool.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify the iOS 26 UX surface each WWDC.

## Sources

- [tabViewBottomAccessory(content:)](https://developer.apple.com/documentation/swiftui/view/tabviewbottomaccessory(content:)) · [TabViewBottomAccessoryPlacement](https://developer.apple.com/documentation/swiftui/tabviewbottomaccessoryplacement) · [searchToolbarBehavior(_:)](https://developer.apple.com/documentation/swiftui/view/searchtoolbarbehavior(_:)) · [scrollEdgeEffectStyle(_:for:)](https://developer.apple.com/documentation/swiftui/view/scrolledgeeffectstyle(_:for:)) · [safeAreaBar(edge:alignment:spacing:content:)](https://developer.apple.com/documentation/swiftui/view/safeareabar(edge:alignment:spacing:content:))
- [backgroundExtensionEffect()](https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect()) · [zoom(sourceID:in:)](https://developer.apple.com/documentation/swiftui/navigationtransition/zoom(sourceid:in:)) · [matchedTransitionSource(id:in:)](https://developer.apple.com/documentation/swiftui/view/matchedtransitionsource(id:in:)) · [Animatable() macro](https://developer.apple.com/documentation/SwiftUI/Animatable()) · [onScrollVisibilityChange(threshold:_:)](https://developer.apple.com/documentation/swiftui/view/onscrollvisibilitychange(threshold:_:))
- WWDC25 [256 What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2025/256/) · [323 Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/)

---

**Last verified:** 2026-06-03 against Apple Developer docs. Corrections applied vs. the research draft:
`safeAreaBar(edge:)` takes a **`HorizontalEdge`** (a side bar, not top/bottom); `tabViewBottomAccessory`
has no `isEnabled:` in its primary form; `TabViewBottomAccessoryPlacement` cases are `.inline`/`.expanded`;
the `@Animatable` macro page is `developer.apple.com/documentation/SwiftUI/Animatable()`.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (this surface churned through the
iOS 26 beta cycle).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
