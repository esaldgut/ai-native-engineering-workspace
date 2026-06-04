---
name: swift-adaptive-layouts-ios26
description: >-
  Build one SwiftUI app that adapts across iPhone and iPad — NavigationSplitView as the adaptive
  root (two/three columns that collapse to a stack on iPhone), size-class routing with
  @Environment(\.horizontalSizeClass), and the iOS 26 floating/collapsible tab bar via the Tab(...)
  declaration. Covers when to pick NavigationSplitView vs NavigationStack vs TabView, controlling
  columns with NavigationSplitViewVisibility / NavigationSplitViewColumn, the .balanced style, and
  the rule never to nest a split view inside a NavigationStack. Use when a screen or app must look
  right on both compact (iPhone) and regular (iPad) widths instead of shipping two layouts.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — NavigationSplitView (struct)"
      url: "https://developer.apple.com/documentation/swiftui/navigationsplitview"
      version: "iOS 16.0+ (verified current 2026-06-03)"
    - source: "Apple Developer — TabRole / TabRole.search"
      url: "https://developer.apple.com/documentation/swiftui/tabrole"
      version: "iOS 18.0"
    - source: "WWDC25 Session 323 — Build a SwiftUI app with the new design"
      url: "https://developer.apple.com/videos/play/wwdc2025/323/"
      version: "WWDC25"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual SwiftUI navigation/tab surface change), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Adaptive layouts (iPhone + iPad, iOS 26 SwiftUI)

One adaptive SwiftUI app beats two hand-tuned layouts. The Apple-blessed adaptive root is
[`NavigationSplitView`](https://developer.apple.com/documentation/swiftui/navigationsplitview): it
"presents views in two or three columns" and **automatically collapses to a single-column stack** on
iPhone, in Slide Over, and on watchOS/tvOS. For strictly stack-based flows you stay on
`NavigationStack`; for top-level sections you use `TabView` — which on iOS 26 gains the floating,
collapsible tab bar. This skill routes between them by **size class** and wires the iOS 18+
`Tab(role: .search)` search tab.

## When to invoke

- A screen or app must render correctly on **both iPhone (compact) and iPad (regular)** widths.
- You're choosing between `NavigationSplitView`, `NavigationStack`, and `TabView` for the root.
- You need to **show/hide columns** programmatically or pick which column wins when collapsed.

**Announce on invoke:** "Using `swift-adaptive-layouts-ios26` to route the root by size class with NavigationSplitView and the iOS 26 tab bar."

Do **not** reach for this for a single-purpose, stack-only flow (onboarding, a settings drill-down).
A plain `NavigationStack` is the right tool there; a split view would add empty columns on iPad.

## The canonical APIs (verified)

| API | Signature / form (verified) | iOS gate | Use |
|---|---|---|---|
| `NavigationSplitView` | `init(sidebar:detail:)`, `init(sidebar:content:detail:)`, `init(columnVisibility:sidebar:detail:)`, `init(preferredCompactColumn:sidebar:detail:)` | 16.0+ | Adaptive 2/3-column root |
| `NavigationSplitViewVisibility` | `struct` — `.automatic`, `.all`, `.doubleColumn`, `.detailOnly` | 16.0+ | Programmatic column show/hide |
| `NavigationSplitViewColumn` | `struct` — `.sidebar`, `.content`, `.detail` | 16.0+ | Which column wins when collapsed |
| `.navigationSplitViewStyle(_:)` | `.automatic` / `.balanced` / `.prominentDetail` | 16.0+ | Column emphasis |
| `.navigationSplitViewColumnWidth(min:ideal:max:)` | sets a column's sizing | 16.0+ | Tune sidebar width |
| `@Environment(\.horizontalSizeClass)` | `UserInterfaceSizeClass?` (`.compact` / `.regular`) | 13.0+ | Branch iPhone vs iPad |
| `NavigationStack` | `init(path:root:)` | 16.0+ | Stack-only flows / inside a column |
| `Tab(_:systemImage:content:)` / `Tab(role:)` | the new tab declaration (enables iOS 26 collapse) | 18.0+ | Top-level sections + search tab |
| `TabRole.search` | `static var search: TabRole` | 18.0+ | Bind the system search tab |

## The rules (load-bearing)

### 1. `NavigationSplitView` is the adaptive root; it collapses itself on iPhone

You don't write the iPhone collapse — the split view does it. Embed a `NavigationStack` **inside a
column** (typically the detail column) so links within that column push normally:

```swift
NavigationSplitView {
    List(sections, selection: $selection) { Text($0.title) }     // sidebar
} detail: {
    NavigationStack { DetailView(section: selection) }           // stack lives in the column
}
```

### 2. Never nest a `NavigationSplitView` inside a `NavigationStack`

They each own navigation. Wrapping a split view in a stack (or vice versa) double-manages the
back-stack and breaks state restoration. A split view is a **root**, not a destination.

### 3. Route by `horizontalSizeClass` when you want different *structures*, not just widths

A split view already adapts. Branch on `horizontalSizeClass` only when the compact experience is a
genuinely different structure (e.g., a `TabView` of stacks on iPhone vs a sidebar on iPad):

```swift
@Environment(\.horizontalSizeClass) private var sizeClass
var body: some View {
    if sizeClass == .regular { iPadSplitView } else { iPhoneTabView }
}
```

### 4. Control collapse with `NavigationSplitViewColumn`, visibility with `NavigationSplitViewVisibility`

When collapsed, SwiftUI picks the top column automatically; override with
`init(preferredCompactColumn:)`. To force a column open/closed on regular width, bind a
`NavigationSplitViewVisibility` (e.g., `.detailOnly` hides the sidebar). The split view **ignores
visibility while collapsed** — it's a regular-width control.

### 5. Use the new `Tab(...)` declaration to get the iOS 26 floating tab bar

The legacy `TabView { View().tabItem { … } }` form does **not** participate in the iOS 26 collapse-on-
scroll tab bar. Adopt `Tab("Home", systemImage: "house") { … }` and add a `Tab(role: .search)` for
the search tab (iOS 18+; the floating chrome is the iOS 26 visual layer).

## Canonical example

```swift
struct ContentView: View {
    @Environment(\.horizontalSizeClass) private var sizeClass
    @State private var selection: Section? = .home
    @State private var visibility: NavigationSplitViewVisibility = .automatic
    @State private var query = ""

    var body: some View {
        if sizeClass == .regular {
            NavigationSplitView(columnVisibility: $visibility) {
                List(Section.allCases, id: \.self, selection: $selection) { section in
                    Label(section.title, systemImage: section.icon)
                }
            } detail: {
                NavigationStack { DetailView(section: selection) }
            }
            .navigationSplitViewStyle(.balanced)            // trim detail width when sidebar shows
        } else {
            TabView {
                Tab("Home", systemImage: "house") {
                    NavigationStack { DetailView(section: .home) }
                }
                Tab("Library", systemImage: "books.vertical") {
                    NavigationStack { DetailView(section: .library) }
                }
                Tab(role: .search) {                        // system search tab (iOS 18+)
                    NavigationStack { SearchResults(query: query) }
                        .searchable(text: $query)
                }
            }
        }
    }
}
```

## Decision aid: which root?

- **`NavigationSplitView`** — list/detail content that benefits from two panes on iPad (mail, notes,
  settings, a catalog). It's the default adaptive choice.
- **`NavigationStack`** — a linear push/pop flow with no master/detail split (checkout, onboarding).
  Put it *inside* a split-view column when you need both.
- **`TabView`** — peer top-level destinations. On iOS 26 use the `Tab(...)` declaration; combine with
  a split view per tab for iPad.
- **Don't** force a split view on a single-section app just for iPad — you'll get an empty sidebar.

## Related skills

- `global-skills/apple/swift-ios26-native-ux-patterns/SKILL.md` — the tab-bar collapse, bottom
  accessory, and `backgroundExtensionEffect()` that pair with the detail column.
- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — glass that the split-view /
  tab chrome renders for free; don't double it.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify navigation APIs each WWDC.

## Sources

- [NavigationSplitView](https://developer.apple.com/documentation/swiftui/navigationsplitview) · [NavigationSplitViewVisibility](https://developer.apple.com/documentation/swiftui/navigationsplitviewvisibility) · [navigationSplitViewStyle(_:)](https://developer.apple.com/documentation/swiftui/view/navigationsplitviewstyle(_:)) · [TabRole.search](https://developer.apple.com/documentation/swiftui/tabrole/search)
- WWDC25 [323 Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/) · WWDC22 [The SwiftUI cookbook for navigation](https://developer.apple.com/videos/play/wwdc2022/10054/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`NavigationSplitView` iOS 16+,
`TabRole.search` iOS 18+ confirmed live) + WWDC25 #323. Note: `TabRole`/the search tab debuted iOS 18;
the **floating/collapsible** tab bar is the iOS 26 visual layer on the same `Tab(...)` declaration.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (the tab bar redesign shifted in
the iOS 26 cycle; re-confirm the `Tab(...)` surface each major).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
