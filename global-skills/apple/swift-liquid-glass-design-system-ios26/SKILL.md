---
name: swift-liquid-glass-design-system-ios26
description: >-
  Apply iOS 26 Liquid Glass to custom SwiftUI views the canonical way — glassEffect(_:in:),
  GlassEffectContainer, the .interactive() variant, and id-based morphing with glassEffectID.
  Enforces the load-bearing rules (no glass on glass; .clear needs a contrast strategy; iOS 26+
  only, so guard with a fallback) and the accessibility musts (Reduce Transparency / Increase
  Contrast / Reduce Motion). Use when building or restyling SwiftUI surfaces — toolbars, custom
  controls, floating accessories, cards — that target iOS 26+ and should adopt the system glass
  material rather than hand-rolled blurs.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — View.glassEffect(_:in:)"
      url: "https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)"
      version: "iOS 26.0"
    - source: "Apple Developer — Glass (struct) + Glass.interactive(_:) + glassEffectID(_:in:)"
      url: "https://developer.apple.com/documentation/swiftui/glass"
      version: "iOS 26.0"
    - source: "Apple Developer — Applying Liquid Glass to custom views (authoritative article)"
      url: "https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views"
      version: "iOS 26.0"
    - source: "WWDC25 Session 323 — Build a SwiftUI app with the new design"
      url: "https://developer.apple.com/videos/play/wwdc2025/323/"
      version: "WWDC25"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual SwiftUI surface change), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Liquid Glass design system (iOS 26 SwiftUI)

Liquid Glass is the adaptive material Apple introduced at WWDC25 for controls and navigational
elements across iOS / iPadOS / macOS 26 and the rest of the 26 family. In SwiftUI you reach it
through two canonical entry points — the `glassEffect(_:in:)` view modifier and the
`GlassEffectContainer` view — plus a small set of variants and an id-based morph mechanism. This
skill applies it the way Apple's
[Applying Liquid Glass to custom views](https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views)
article and the
[Landmarks sample](https://developer.apple.com/documentation/swiftui/landmarks-building-an-app-with-liquid-glass)
prescribe — not by hand-rolling `.ultraThinMaterial` blurs.

## When to invoke

- You're building or restyling a SwiftUI surface that targets **iOS 26+** and should adopt the
  system glass material: a custom toolbar, a floating action control, a bottom accessory
  ("mini-player") slot, a card, a capsule of buttons.
- You see hand-rolled `.ultraThinMaterial` / `.regularMaterial` blurs imitating glass — replace
  with the real `glassEffect`.
- You're animating one glass shape into another (a control that expands, a toolbar that
  reconfigures) and need the morph to read correctly.

**Announce on invoke:** "Using `swift-liquid-glass-design-system-ios26` to apply the system glass material per Apple's custom-views guidance."

Do **not** reach for this when the built-in components already provide glass for free — standard
`TabView`, `.toolbar`, sheets, and `NavigationStack` chrome render Liquid Glass automatically on
iOS 26. Adopt the new component APIs first; use `glassEffect` for *custom* views the system
doesn't style for you.

## The canonical APIs (verified iOS 26.0+)

| API | Signature (verified) | Use |
|---|---|---|
| `glassEffect(_:in:)` | `nonisolated func glassEffect(_ glass: Glass = .regular, in shape: some Shape = DefaultGlassEffectShape()) -> some View` | Apply glass to a custom view, clipped to a shape |
| `GlassEffectContainer` | `GlassEffectContainer(spacing:) { … }` | Group glass shapes so they can sample/morph together |
| `Glass` | `struct Glass` — variants `.regular` (default), `.clear`, `.identity` | The material configuration |
| `Glass.interactive(_:)` | `func interactive(_ isEnabled: Bool = true) -> Glass` | Make custom glass respond to tap/press |
| `glassEffectID(_:in:)` | `nonisolated func glassEffectID(_ id: (some Hashable & Sendable)?, in namespace: Namespace.ID) -> some View` | Tag glass shapes so SwiftUI morphs them across transitions |
| `GlassButtonStyle` / `GlassProminentButtonStyle` | `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)` | Glass on standard `Button`s without manual `glassEffect` |
| `GlassEffectTransition` | type | Customize the morph transition between tagged shapes |
| `DefaultGlassEffectShape` | type | The default clip shape `glassEffect` uses when none given |

> Note: `glassEffectID` takes `(some Hashable & Sendable)?` — the id is **optional** and must be
> `Sendable`. Passing a non-`Sendable` id, or forgetting the optionality, won't match Apple's
> signature.

## The rules (load-bearing — break them and the material breaks)

### 1. No glass on glass

Glass **cannot sample other glass.** Two `.glassEffect()` views that overlap or need to interact
must live inside a **single** `GlassEffectContainer`. Nesting glass without a shared container
produces broken, doubled visuals (a well-known iOS 26 pitfall). One container per cluster of
glass that belongs together; don't wrap your whole view tree in one giant container either.

### 2. Morphing requires a shared namespace inside one container

`glassEffectID(_:in:)` only morphs shapes whose ids share the **same** `Namespace.ID` **and**
live in the **same** `GlassEffectContainer`. Source and destination across different containers
or namespaces will not animate into each other — they'll cross-fade or pop.

### 3. `.clear` is opt-in legibility risk

`Glass.regular` is legible by default. `Glass.clear` is more transparent and, per Apple HIG,
**needs an explicit contrast strategy** (a dimming layer, a shadow, or content-aware tinting) on
busy backgrounds. Default to `.regular`; reach for `.clear` only when you control what's behind it.

### 4. iOS 26+ only — there is no backport

Every API here is **iOS 26.0+** (and the 26-family equivalents). There is no shim for iOS
17/18/25. Guard and provide a non-glass fallback:

```swift
if #available(iOS 26, *) {
    content.glassEffect(.regular, in: .capsule)
} else {
    content.background(.regularMaterial, in: .capsule)   // graceful pre-26 fallback
}
```

### 5. Accessibility is not optional

Liquid Glass must respect the system settings. Honor:

- **Reduce Transparency** — fall back to an opaque background.
- **Increase Contrast** — strengthen separation; don't rely on the glass blur alone.
- **Reduce Motion** — suppress or simplify `glassEffectID` morphs.

Maintain **WCAG 4.5:1** contrast for text over glass. Read these via `@Environment`
(`accessibilityReduceTransparency`, `accessibilityReduceMotion`, `colorSchemeContrast`).

## Canonical example

A toolbar of two buttons that share a container so they morph and don't double-sample. Verified
against the signatures above:

```swift
struct GlassToolbar: View {
    @Namespace private var glassNS
    @Environment(\.accessibilityReduceTransparency) private var reduceTransparency

    var body: some View {
        GlassEffectContainer(spacing: 12) {
            HStack(spacing: 12) {
                Button("Save") {}
                    .glassEffect(reduceTransparency ? .identity : .regular.interactive(),
                                 in: .capsule)
                    .glassEffectID("save", in: glassNS)

                Button("Cancel") {}
                    .glassEffect(reduceTransparency ? .identity : .regular,
                                 in: .capsule)
                    .glassEffectID("cancel", in: glassNS)
            }
            .padding()
        }
    }
}
```

For standard buttons that don't need custom shapes, prefer the style instead of manual glass:

```swift
Button("Continue") {}.buttonStyle(.glassProminent)   // GlassProminentButtonStyle
```

## Decision aid: when NOT to use glass

- **On scrolling content edges** the system already adapts glass; use
  `.scrollEdgeEffectStyle(_:for:)` rather than stacking your own glass at the edge.
- **For plain backgrounds** that aren't controls/navigation, glass is the wrong material — use a
  solid color or standard material. Apple's HIG scopes glass to *controls and navigational
  elements*, not arbitrary surfaces.
- **Never as a "frosted" decoration** behind body text — that's the legibility trap rule 3 warns
  about.

## Migration from hand-rolled blurs

If the codebase fakes glass with `.ultraThinMaterial` + custom shadows:

1. Replace the material background with `.glassEffect(.regular, in: <shape>)` inside a
   `GlassEffectContainer`.
2. Delete the manual shadow/border imitating depth — glass provides it.
3. Keep the pre-26 path as the `#available` fallback (rule 4).
4. Re-test under Reduce Transparency and in both color schemes.

## Related skills

- `global-skills/apple/apple-anti-patterns/SKILL.md` — registers the "no glass on glass" and
  "wrap an Apple-canonical API in a hand-rolled struct" anti-patterns this skill avoids.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verifies these glass APIs
  after each WWDC, since the material's API surface evolved through the iOS 26 beta cycle.

## Sources

- [View.glassEffect(_:in:)](https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)) · [GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer) · [Glass](https://developer.apple.com/documentation/swiftui/glass) · [Glass.interactive(_:)](https://developer.apple.com/documentation/swiftui/glass/interactive(_:)) · [glassEffectID(_:in:)](https://developer.apple.com/documentation/swiftui/view/glasseffectid(_:in:))
- [Applying Liquid Glass to custom views](https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views) · [Landmarks: Building an app with Liquid Glass](https://developer.apple.com/documentation/swiftui/landmarks-building-an-app-with-liquid-glass)
- WWDC25 [219 Meet Liquid Glass](https://developer.apple.com/videos/play/wwdc2025/219/) (design language) · [323 Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/) (implementation)

---

**Last verified:** 2026-06-03 against Apple Developer docs (glassEffect / Glass / glassEffectID
signatures confirmed live, iOS 26.0+) + WWDC25 #323. `Glass.tint(_:)` was checked and does **not**
exist as a documented API — use the standard `tint(_:)` modifier instead.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (the glass API surface shifted
during the iOS 26 beta cycle; re-confirm signatures each major).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
