# 06 — iOS 26 / iPadOS 26 UX/UI Design Patterns for MyApp

Research compiled 2026-03-26 from Apple developer documentation, WWDC25 sessions,
and Human Interface Guidelines. Only verified patterns with sources.

---

## 1. iPadOS 26-Specific Patterns (vs iPhone)

### 1.1 Sidebar vs Tab Bar Navigation

**Source:** [Elevate the design of your iPad app — WWDC25-208](https://developer.apple.com/videos/play/wwdc2025/208/)

| Pattern | iPhone | iPad |
|---|---|---|
| Primary navigation | Tab bar (bottom, Liquid Glass) | Sidebar (Liquid Glass, inset) OR Tab bar |
| Sidebar | Not available | Floats with Liquid Glass, content flows behind it |
| Tab bar on iPad | Shows when sidebar is hidden | Morphs fluidly to/from sidebar |
| Search placement | Bottom toolbar (easier reach) | Top-trailing in Liquid Glass container |

Key rules:
- **Start with tab bar as default** for iPad; scale to sidebar when app has numerous sub-views or deeply nested content (Mail, Music pattern).
- Sidebar **flattens navigation hierarchy**, exposing content at top level.
- Sidebar can **morph fluidly into tab bar** in compact width (portrait, floating window).
- Adapt to **width changes**, not just orientation — floating windows also trigger compact layouts.

### 1.2 Split View / Multi-Column Layouts

**Source:** [TN3154: Adopting NavigationSplitView](https://developer.apple.com/documentation/technotes/tn3154-adopting-swiftui-navigation-split-view)

- `NavigationSplitView` supports 2-column and 3-column layouts.
- **Automatic adaptive behavior**: collapses to `NavigationStack` in compact size classes.
- Two-column: sidebar + detail. Three-column: sidebar + content + detail.
- Use `.balanced` or `.prominentDetail` style to control column priority.
- iPadOS 26 adds **interactive column resizing** via draggable separators.

### 1.3 Pointer / Trackpad / Keyboard Support

**Source:** [WWDC25-208](https://developer.apple.com/videos/play/wwdc2025/208/)

- **New pointer shape** in iPadOS 26: more precise, tracks 1:1 (no magnetization/rubber-banding).
- **Liquid Glass highlight effect** replaces the original hover morph effect — materializes directly on buttons.
- On button clusters, highlight **quickly catches up** as pointer moves across targets.
- Test app with new pointer behavior to identify unexpected results.

### 1.4 Stage Manager / Windowing

**Source:** [WWDC25-208](https://developer.apple.com/videos/play/wwdc2025/208/), [WWDC25-282](https://developer.apple.com/videos/play/wwdc2025/282/), [iPadOS 26 Newsroom](https://www.apple.com/newsroom/2025/06/ipados-26-introduces-powerful-new-features-that-push-ipad-even-further/)

- **`UIRequiresFullscreen` is deprecated** — remove from Info.plist. Apps must support flexible resizing.
- Window controls (close, minimize, resize, tile) on **leading edge of toolbar**.
- **Wrap toolbar around window controls** so they appear inline (system `NavigationBar` does this automatically).
- **Drag handle** in bottom-right corner for resizing into floating window.
- **Window persistence**: windows reopen at exact same size and position.
- **Additive windowing**: create a new window per document.
- **Expose**: shows all open windows for quick switching.
- New `windowingControlStyle(.unified)` API for UIKit scene delegate.
- Apps built with iOS 26 SDK automatically adapt to new screen sizes without resubmission.

### 1.5 iPad Menu Bar

**Source:** [WWDC25-208](https://developer.apple.com/videos/play/wwdc2025/208/)

- **New menu bar** revealed by swiping down or moving cursor to top edge.
- Uses same **Commands API** from macOS — same code creates menus on both platforms.
- Populate with: app menu (required), system defaults, custom menus.
- **Organize by frequency of use** (not alphabetically).
- Add **tabs as menu items** with keyboard shortcuts for quick switching.
- Include navigation toggle (sidebar show/hide).
- **Never hide menus contextually** — dim inactive items instead.
- Keep menu structure **static and predictable**.

### 1.6 iPad-Specific Liquid Glass Behavior

**Source:** [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

- Sidebar is **inset and built with Liquid Glass**, content flows behind.
- Use `backgroundExtensionEffect()` to extend content behind sidebar.
- Per-pane scroll edge effects in Split View (one per pane).
- Concentric shapes align with **window edge** on iPad (vs screen edge on iPhone).

---

## 2. iOS 26 Human Interface Guidelines Updates

### 2.1 Navigation Patterns

**Source:** [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/), [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

| Component | iPhone Behavior | iPad Behavior |
|---|---|---|
| `NavigationSplitView` | Collapses to stack | 2 or 3 columns with Liquid Glass sidebar |
| `NavigationStack` | Standard push/pop | Used within detail column |
| Toolbar | Floating Liquid Glass surface | Wraps around window controls |
| Back button | Standard | Auto-generated from column structure |

### 2.2 Tab Bar (Liquid Glass)

**Source:** [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/), [Apple HIG: Tab Bars](https://developer.apple.com/design/human-interface-guidelines/tab-bars)

New behaviors in iOS 26:
- Tab bar **floats above content** in Liquid Glass.
- **Collapses/minimizes on scroll down**, expands on scroll up.
- Control via `.tabBarMinimizeBehavior(.onScrollDown)` (options: `.automatic`, `.never`, `.onScrollDown`).
- **Dedicated Search tab** via `Tab(role: .search)` — transforms into search field when selected.
- **Bottom accessory** via `.tabViewBottomAccessory { }` — view above tab bar (like Apple Music "Now Playing").
- Read accessory placement via `@Environment(\.tabViewBottomAccessoryPlacement)` — returns `.expanded`, `.inline`, or nil.

Do NOT put in tab bar:
- Screen-specific actions (checkout buttons belong with content).
- Mixed elements from different UI parts.

### 2.3 Search Placement

**Source:** [SwiftUI Search Enhancements](https://nilcoalescing.com/blog/SwiftUISearchEnhancementsIniOSAndiPadOS26/), [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/)

| Platform | Default Placement | Style |
|---|---|---|
| iPhone | **Bottom toolbar** (easier reach) | Own Liquid Glass surface |
| iPad | **Top-trailing corner** | Liquid Glass container |
| iPad sidebar | Optional via `placement: .sidebar` | Anchored in sidebar column |

APIs:
- `.searchable(text:)` on `NavigationSplitView` — auto-places per platform.
- `.searchable(text:)` on `TabView` with `Tab(role: .search)` — dedicated search tab.
- `.searchToolbarBehavior(.minimize)` — collapses search to button when not primary feature.
- `DefaultToolbarItem` with `.search` kind for explicit placement control.

### 2.4 Sheet Presentation

**Source:** [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/), [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

- Partial-height sheets are **inset by default** with Liquid Glass background.
- At smaller heights, bottom edges **pull in**, nesting in display curved edges.
- Full-height transition: glass becomes opaque, anchors to screen edge.
- **Remove custom `presentationBackground`** — now automatic.
- **Action sheets spring from source action** (not bottom of screen).
- **Modal sheets**: Liquid Glass + dimming layer (interrupting flow).
- **Parallel task sheets**: Liquid Glass alone (no dimming, no modal feel).
- **Navigation zoom transition**: morph content from source view to sheet using `glassEffectID` + namespace.

### 2.5 Typography

**Source:** [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

- Text is **bolder and left-aligned** for improved readability.
- Strengthens clarity and structure in alerts, onboarding, key moments.
- No new font families — same SF system but adjusted weights/alignment defaults.

### 2.6 Color System

**Source:** [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

- System colors **adjusted subtly** across Light, Dark, and Increased Contrast.
- Improved **hue differentiation** while maintaining Apple's optimistic spirit.
- Colors work **in harmony with Liquid Glass**.
- Toolbar icons render **monochrome automatically** — use `tint()` only to convey meaning.

### 2.7 Shapes, Spacing & Concentricity

**Source:** [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/), [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/)

Three shape types:
1. **Fixed shapes** — constant corner radius.
2. **Capsules** — radius = half container height.
3. **Concentric shapes** — radius = parent radius minus padding (mathematically nested).

APIs:
- `ConcentricRectangle` — shape that matches container's corner style.
- `.concentric(corner: .containerConcentric)` — auto-matches container corners across displays/windows.
- Default button shape changed to **capsule** (harmonious with rounded design).
- `.buttonBorderShape()` to override.

Platform-specific control shapes:
- **iOS/iPadOS**: Large controls = capsule; Mini/Small/Medium = rounded rectangle.
- **macOS**: Mini/Small/Medium = rounded rectangle; Large/X-Large = capsule.

### 2.8 Scroll Edge Effects

**Source:** [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/), [WWDC25-356](https://developer.apple.com/videos/play/wwdc2025/356/)

- Replace hard dividers with **subtle blur at scroll edges**.
- Two styles via `.scrollEdgeEffectStyle()`:
  - **`.soft`** (default on iOS/iPadOS) — for interactive elements.
  - **`.hard`** — for pinned headers, text controls (macOS default).
- Use **one per view** (or one per pane in split view).
- Automatic when pinned controls overlap scroll views.

### 2.9 Accessibility

**Source:** [WWDC25-256](https://developer.apple.com/videos/play/wwdc2025/256/)

- **Accessibility default focus modifier** — suggest initial VoiceOver focus.
- **`AssistiveAccess` scene type** — show simplified UI for cognitive disabilities.
- All Liquid Glass elements maintain accessibility contrast requirements.
- Continue using `accessibilityElement(children:)`, `accessibilityLabel()`, etc.

---

## 3. Adaptive Layouts (Universal iPhone + iPad)

### 3.1 Size Class Strategy

**Source:** [NavigationSplitView docs](https://developer.apple.com/documentation/swiftui/navigationsplitview), [WWDC25-208](https://developer.apple.com/videos/play/wwdc2025/208/)

```swift
@Environment(\.horizontalSizeClass) var horizontalSizeClass
@Environment(\.verticalSizeClass) var verticalSizeClass
```

| horizontalSizeClass | verticalSizeClass | Context |
|---|---|---|
| `.compact` | `.regular` | iPhone portrait |
| `.compact` | `.compact` | iPhone landscape |
| `.regular` | `.regular` | iPad / large window |
| `.regular` | `.compact` | Rare edge case |

**Best approach for MyApp**: Use `NavigationSplitView` as root — it automatically collapses
to `NavigationStack` in compact width. No need for manual switching.

### 3.2 NavigationSplitView Adaptive Pattern

```swift
NavigationSplitView {
  // Sidebar (becomes tab bar on iPhone)
  SidebarView()
} detail: {
  // Detail (becomes full-screen stack on iPhone)
  DetailView()
}
```

- Two-column for most apps. Three-column for chat (sidebar + conversations + messages).
- `.balanced` style: sidebar and detail share space equally.
- `.prominentDetail` style: detail takes priority.

### 3.3 New iOS 26 Adaptive APIs

- `backgroundExtensionEffect()` — extends detail content behind sidebar on iPad; no-op on iPhone.
- `ConcentricRectangle` — adapts corner radius to window shape (different on iPad floating vs fullscreen).
- `TabView` with sidebar morphing — same TabView code automatically shows sidebar on iPad, tabs on iPhone.
- Menu bar Commands API — same code creates menus on macOS and iPad; ignored on iPhone.

---

## 4. New SwiftUI iOS 26 View Modifiers & Components

### 4.1 Liquid Glass

| API | Purpose |
|---|---|
| `.glassEffect()` | Apply Liquid Glass to custom views |
| `.glassEffect(in: .capsule)` | Glass with custom shape |
| `GlassEffectContainer { }` | Group multiple glass elements (required: glass cannot sample glass) |
| `.glassEffectID(_:in:)` | Enable fluid morphing transitions between glass elements |
| `.buttonStyle(.glass)` | Liquid Glass button |
| `.buttonStyle(.glassProminent)` | Prominent Liquid Glass button |
| `.tint()` on glass | Vibrant adaptive color tinting (use only for meaning) |
| `.interactive` | Enable scaling, bouncing, shimmering on interaction |

### 4.2 Tab & Navigation

| API | Purpose |
|---|---|
| `.tabBarMinimizeBehavior(.onScrollDown)` | Collapse tab bar on scroll |
| `.tabViewBottomAccessory { }` | View above tab bar (mini player pattern) |
| `@Environment(\.tabViewBottomAccessoryPlacement)` | Read accessory state: `.expanded`, `.inline` |
| `Tab(role: .search)` | Dedicated search tab |
| `.searchToolbarBehavior(.minimize)` | Collapse search to button |
| `.navigationSubtitle()` | Configure view subtitle for navigation |

### 4.3 Toolbar

| API | Purpose |
|---|---|
| `ToolbarSpacer(.fixed)` | Fixed spacing between toolbar groups |
| `ToolbarSpacer(.flexible)` | Expanding space between items |
| `.badge()` on toolbar items | Notification badges |
| `.sharedBackgroundVisibility(.hidden)` | Remove glass background from item |
| Placement: `.title`, `.subtitle`, `.largeTitle`, `.largeSubtitle` | New toolbar item placements |

### 4.4 Layout & Shapes

| API | Purpose |
|---|---|
| `ConcentricRectangle` | Shape matching container corner style |
| `.concentric(corner: .containerConcentric)` | Auto-concentric corner radius |
| `.backgroundExtensionEffect()` | Extend view behind sidebar/inspector with blur |
| `.scrollEdgeEffectStyle(.soft / .hard)` | Control scroll edge blur style |
| `safeAreaBar(edge:alignment:spacing:content:)` | Custom bar in safe area |

### 4.5 Controls

| API | Purpose |
|---|---|
| `Button` role initializers (`.cancel`, `.close`, `.confirm`, `.destructive`) | Role buttons with default labels |
| `.controlSize(.extraLarge)` | New extra-large control size |
| Slider `ticks()` closure | Manual tick mark placement |
| Slider `neutralValue()` | Start track fill from specific value |
| `TextEditor(text: $attributedString)` | Rich text editing with `AttributedString` |

### 4.6 Animation

| API | Purpose |
|---|---|
| `@Animatable` macro | Auto-synthesize `animatableData` |
| `@AnimatableIgnored` macro | Exclude properties from animation |
| Navigation zoom transitions | Morph from source to destination |
| Glass morphing via `glassEffectID` | Fluid element-to-element transitions |

### 4.7 Web Integration

| API | Purpose |
|---|---|
| `WebView(url:)` | Embed web content |
| `WebView(page:)` | Observable web page model |
| `WebPage` model | Programmatic navigation, JS calling |

### 4.8 Performance Improvements

- **Lists (macOS)**: 6x faster loading for 100k+ items; 16x faster updates.
- **Scrolling**: Improved scheduling of UI updates on iOS/macOS.
- **Nested lazy stacks**: `LazyVStack` in `ScrollView` now delays loading until needed.
- **High frame rate scrolling**: Reduced frame drops.
- **SwiftUI Performance Instrument** in Xcode — lanes for long view body updates.

### 4.9 Deprecations

| Deprecated | Replacement |
|---|---|
| `UIRequiresFullscreen` (Info.plist) | Remove it. Support flexible resizing. |
| Custom `presentationBackground` on sheets | Automatic Liquid Glass background |
| Hard dividers on toolbars/bars | Scroll edge effects (automatic) |
| Original pointer hover morph (iPad) | Liquid Glass highlight effect |

---

## 5. Media Playback UX Patterns

### 5.1 Video in Feed (Social App)

**Source:** [AVKit docs](https://developer.apple.com/documentation/avkit/videoplayer), tech research (`docs/02-tech-research.md`)

- Use `AVPlayer` + HLS (`.m3u8`) for adaptive bitrate streaming.
- SwiftUI: `VideoPlayer(player:)` from AVKit.
- For reels/stories: preload with `AVQueuePlayer`.
- Autoplay in scroll: manage `AVPlayer.play()/pause()` based on visibility using `onAppear`/`onDisappear` or `ScrollView` geometry reader.

### 5.2 Picture-in-Picture

**Source:** [Adopting PiP](https://developer.apple.com/documentation/avkit/adopting-picture-in-picture-in-a-standard-player)

- PiP supported via `AVPlayerViewController` (standard player) or `AVPictureInPictureController` (custom player).
- Works on iPhone and iPad.
- Automatic PiP on app background if configured.
- Enable via `AVAudioSession.Category.playback` + `Background Modes` capability.

### 5.3 Mini Player Pattern (Tab Bar Accessory)

**Source:** [WWDC25-323](https://developer.apple.com/videos/play/wwdc2025/323/)

iOS 26 provides a first-class pattern:
```swift
TabView {
  // tabs...
}
.tabViewBottomAccessory {
  MiniPlayerView()
}
```
- Accessory sits above tab bar.
- Collapses when tab bar minimizes on scroll.
- Read placement via `@Environment(\.tabViewBottomAccessoryPlacement)` to adapt layout.

### 5.4 Full-Screen Video Transitions

- Use navigation zoom transition with `glassEffectID` for fluid morph from mini player to full-screen.
- Or standard `.fullScreenCover()` / `.sheet()` with Liquid Glass background.

### 5.5 Audio Session

- `AVAudioSession.Category.playback` for media apps.
- Handle interruptions (phone calls) via `AVAudioSession.interruptionNotification`.
- Configure in `AppDelegate` or at first play.

### 5.6 WWDC25 Multiview Playback

**Source:** [WWDC25-302](https://developer.apple.com/videos/play/wwdc2025/302/)

- `AVPlaybackCoordinationMedium` — synchronize playback across multiple players.
- Useful for side-by-side video comparison or shared viewing.

---

## 6. Design Patterns for Social / Marketplace Apps

### 6.1 Feed Layouts

Apple does not provide a specific "feed" component, but recommended patterns:

- **`List` with custom rows** — most performant for long scrolling feeds.
- **`LazyVStack` in `ScrollView`** — more customizable, improved in iOS 26 (delayed loading).
- **Card pattern**: Use `RoundedRectangle` or `ConcentricRectangle` with shadow/glass.
- For video-heavy feeds (TikTok-style): `TabView` with `.tabViewStyle(.page)` for vertical paging, or custom `ScrollView` with `scrollTargetBehavior(.paging)`.

### 6.2 Card-Based Marketplace Layouts

- `LazyVGrid` with adaptive columns: `GridItem(.adaptive(minimum: 160))`.
- Cards with `ConcentricRectangle` for corner concentricity.
- Use `.glassEffect()` sparingly on cards — reserve glass for navigation/controls.
- HIG principle: Glass is for **functional layers** (navigation, controls), not content cards.

### 6.3 Chat Interfaces

- Three-column `NavigationSplitView` on iPad: conversations list + chat + details.
- Two-column on iPhone: conversations list + chat (collapsed stack).
- Inverted scroll for message lists.
- `.safeAreaBar(edge: .bottom)` for message input bar.
- Keyboard avoidance is automatic with `.safeAreaInset()`.

### 6.4 Profile Pages

- `ScrollView` with header image + `backgroundExtensionEffect()` for edge-to-edge hero.
- Concentric shapes for profile image container.
- Use `.glassEffect()` on floating action buttons (follow, message).
- Toolbar with `.sharedBackgroundVisibility(.hidden)` for transparent header toolbar.

### 6.5 Onboarding Flows

- HIG: Typography is **bolder and left-aligned** — leverage for onboarding headlines.
- Use `TabView` with `.tabViewStyle(.page)` for swipeable pages.
- Liquid Glass for primary CTA buttons: `.buttonStyle(.glassProminent)`.
- Sheet presentation for sign-in: partial-height inset sheet with automatic glass background.
- Navigation zoom transitions for fluid step-to-step animation.

---

## Key WWDC25 Sessions Reference

| Session | Topic |
|---|---|
| [323 — Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/) | SwiftUI Liquid Glass APIs, tab bar, toolbar, search, sheets |
| [356 — Get to know the new design system](https://developer.apple.com/videos/play/wwdc2025/356/) | Typography, color, shapes, concentricity, platform differences |
| [256 — What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2025/256/) | All new SwiftUI APIs, performance, animation macros |
| [208 — Elevate the design of your iPad app](https://developer.apple.com/videos/play/wwdc2025/208/) | iPad sidebar, windowing, menu bar, pointer, keyboard |
| [282 — Make your UIKit app more flexible](https://developer.apple.com/videos/play/wwdc2025/282/) | UIRequiresFullscreen migration, flexible resizing |
| [284 — Build a UIKit app with the new design](https://developer.apple.com/videos/play/wwdc2025/284/) | UIKit Liquid Glass adoption |
| [243 — What's new in UIKit](https://developer.apple.com/videos/play/wwdc2025/243/) | UIKit updates for iOS 26 |
| [359 — Design foundations from idea to interface](https://developer.apple.com/videos/play/wwdc2025/359/) | Design foundations |
| [Meet with Apple 208 — Showcase: Liquid Glass](https://developer.apple.com/videos/play/meet-with-apple/208/) | Real-world Liquid Glass integrations |

## Apple Documentation Links

- [GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)
- [glassEffect(\_:in:)](https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:))
- [Applying Liquid Glass to custom views](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views)
- [backgroundExtensionEffect()](https://developer.apple.com/documentation/SwiftUI/View/backgroundExtensionEffect())
- [ConcentricRectangle](https://developer.apple.com/documentation/swiftui/concentricrectangle)
- [safeAreaBar()](https://developer.apple.com/documentation/swiftui/view/safeareabar(edge:alignment:spacing:content:))
- [scrollEdgeEffectStyle()](https://developer.apple.com/documentation/SwiftUI/View/scrollEdgeEffectStyle(_:for:))
- [NavigationSplitView](https://developer.apple.com/documentation/swiftui/navigationsplitview)
- [TN3154: Adopting NavigationSplitView](https://developer.apple.com/documentation/technotes/tn3154-adopting-swiftui-navigation-split-view)
- [TN3192: Migrating from UIRequiresFullScreen](https://developer.apple.com/documentation/technotes/tn3192-Migrating-your-app-from-the-deprecated-UIRequiresFullScreen-key)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [SwiftUI Updates](https://developer.apple.com/documentation/updates/swiftui)
