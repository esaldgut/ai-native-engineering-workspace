---
name: swift-ecommerce-marketplace-ui-patterns
description: >-
  Compose e-commerce / marketplace UI in SwiftUI from primitives (no Apple "commerce kit") — product
  grids with LazyVGrid + GridItem(.adaptive), a pinch-zoom gallery via MagnifyGesture + scaleEffect,
  an image carousel with TabView(.page), a multi-step booking wizard anchored on .sheet(item:) +
  presentationDetents, filter chips, and a payment redirect that opens the provider's checkout in
  SFSafariViewController (never WKWebView, never a custom URL scheme). iOS 26 adds Liquid Glass on
  CTAs/filters. Use when building product cards, a gallery, a booking/checkout flow, or wiring a
  hosted payment redirect the secure, Apple-recommended way.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — GridItem (.adaptive/.flexible/.fixed) + LazyVGrid"
      url: "https://developer.apple.com/documentation/swiftui/griditem"
      version: "iOS 14.0"
    - source: "Apple Developer — MagnifyGesture"
      url: "https://developer.apple.com/documentation/swiftui/magnifygesture"
      version: "iOS 17.0"
    - source: "Apple Developer — SFSafariViewController (hosted checkout redirect)"
      url: "https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller"
      version: "iOS 9.0+ (verified current 2026-06-03)"
    - source: "Apple Developer — presentationDetents(_:)"
      url: "https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)"
      version: "iOS 16.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote, or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# E-commerce / marketplace UI patterns (SwiftUI, iOS 26)

A marketplace screen is composition over SwiftUI primitives — there's no Apple "commerce kit." The
load-bearing pieces are [`LazyVGrid`](https://developer.apple.com/documentation/swiftui/lazyvgrid)
with [`GridItem(.adaptive(minimum:))`](https://developer.apple.com/documentation/swiftui/griditem)
for product grids, [`MagnifyGesture`](https://developer.apple.com/documentation/swiftui/magnifygesture)
for gallery pinch-zoom, `.sheet(item:)` + [`presentationDetents`](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:))
for the booking wizard, and [`SFSafariViewController`](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller)
for the hosted-checkout redirect. iOS 26 layers Liquid Glass on CTAs and filter chips. This skill is
**provider-agnostic** about payments — it stops at the secure redirect boundary, not a specific SDK.

## When to invoke

- You're building **product cards / a grid**, a **gallery** with zoom, a **booking/checkout wizard**,
  filter chips, or a search/results screen.
- You need to send the user to a **hosted payment page** and bring them back.

**Announce on invoke:** "Using `swift-ecommerce-marketplace-ui-patterns` for the grid, gallery, booking sheet, and the SFSafariViewController payment redirect."

Do **not** use this for Apple's own in-app purchase of digital goods — that's StoreKit, not a web
redirect. This skill is for physical goods / bookings paid via an external processor.

## The canonical APIs (verified)

| API | Signature / form (verified) | Use |
|---|---|---|
| `LazyVGrid` + `GridItem` | `GridItem(.adaptive(minimum:maximum:))`, `.flexible(minimum:maximum:)`, `.fixed(_:)` | Responsive product grid |
| `MagnifyGesture` | `struct MagnifyGesture`; `.updating($state) { v, s, _ in s = v.magnification }` | Pinch-zoom in the gallery |
| `TabView` + `.tabViewStyle(.page)` | paging style with `PageTabViewStyle` | Image carousel |
| `.sheet(item:onDismiss:content:)` | `Item : Identifiable` | Anchor the booking wizard step |
| `.presentationDetents(_:)` | `Set<PresentationDetent>` (`.medium`, `.large`, `.fraction(_:)`, `.height(_:)`) | Partial-height booking sheet |
| `SFSafariViewController` | UIKit VC; present via `UIViewControllerRepresentable` | Hosted checkout redirect |
| `.onOpenURL(perform:)` / Universal Links | return path after payment | Reliable callback |
| `.glassEffect(.regular.interactive(), in: .capsule)` | iOS 26 | Glass CTA / filter chip |

## The rules (load-bearing)

### 1. Responsive grid = `GridItem(.adaptive(minimum:))`, not a fixed column count

`.adaptive(minimum: 160)` lets SwiftUI fit as many columns as the width allows — the same code looks
right on iPhone and iPad. Hard-coding `count: 2` wastes iPad width.

```swift
LazyVGrid(columns: [GridItem(.adaptive(minimum: 160), spacing: 12)], spacing: 12) {
    ForEach(products) { ProductCard(product: $0) }
}
```

### 2. Pinch-zoom needs `MagnifyGesture` + an anchor — the focal point isn't free

`MagnifyGesture` (iOS 17; the modern name — `MagnificationGesture` is the deprecated spelling)
reports `value.magnification` via `.updating`. Compose it with a `DragGesture` (use
`SimultaneousGesture`) and an `UnitPoint` anchor so zoom centers on the pinch, not the view center.

### 3. The booking wizard anchors on `.sheet(item:)`, stepped by an `Identifiable` enum

A `.sheet(isPresented: Bool)` can't carry *which* step. Drive the flow with `.sheet(item: $step)`
where `Step: Identifiable`, set `presentationDetents([.medium, .large])`, and on iOS 26 the sheet
adopts Liquid Glass automatically for the partial detent.

### 4. Payment redirect: `SFSafariViewController` — NEVER `WKWebView`, NEVER a bare custom scheme

Apple's guidance is explicit: to view arbitrary web content you don't control, present
`SFSafariViewController` (it isolates cookies/keychain from your app and supports AutoFill/Reader).
Use `WKWebView` only when you must inject/interact with the page; use `ASWebAuthenticationSession`
for OAuth-shaped flows. Return via **Universal Links + `onOpenURL`**, not a custom URL scheme alone
(deprecated security model).

```swift
struct CheckoutRedirect: UIViewControllerRepresentable {
    let url: URL
    func makeUIViewController(context: Context) -> SFSafariViewController { SFSafariViewController(url: url) }
    func updateUIViewController(_ vc: SFSafariViewController, context: Context) {}
}
```

### 5. Filter chips / CTAs: glass, but one container, never glass-on-glass

Wrap a row of glass chips in a single `GlassEffectContainer`. A chip's `.glassEffect()` sitting on a
card that's *also* glass double-samples — keep the card a solid/material surface and let only the
chips be glass.

## Canonical example

```swift
struct ProductCard: View {
    let product: Product
    @State private var bookingStep: BookingStep?

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            TabView {                                            // image carousel
                ForEach(product.imageURLs, id: \.self) { url in
                    AsyncImage(url: url) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
                }
            }
            .tabViewStyle(.page)
            .frame(height: 220)
            .clipShape(.rect(cornerRadius: 16))

            Text(product.title).font(.headline)
            Text(product.price.formatted(.currency(code: "USD"))).foregroundStyle(.secondary)

            Button("Reserve") { bookingStep = .dates }
                .buttonStyle(.glassProminent)                   // iOS 26 glass CTA
        }
        .sheet(item: $bookingStep) { step in
            BookingWizard(product: product, step: step)
                .presentationDetents([.medium, .large])
        }
    }
}

struct ZoomableImage: View {
    let url: URL
    @GestureState private var zoom = 1.0
    var body: some View {
        AsyncImage(url: url) { $0.resizable().scaledToFit() } placeholder: { ProgressView() }
            .scaleEffect(zoom)
            .gesture(MagnifyGesture().updating($zoom) { value, state, _ in state = value.magnification })
    }
}
```

## Decision aid: when NOT to / trade-offs

- **Don't commit the skill to one payment provider.** Stop at the `SFSafariViewController(url:)`
  boundary; the provider's checkout URL and the return Universal Link are app config, not Apple API.
- **`TabView(.page)` for galleries is fine up to a handful of images;** for 50+ media items prefer a
  paged `ScrollView` with `.scrollTargetBehavior(.paging)` and lazy loading (see the video-feed skill).
- **Glass is for controls, not product photos.** Keep imagery on solid surfaces; reserve glass for
  CTAs, filter chips, and floating toolbars.

## Related skills

- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — the glass CTA/chip rules
  (container, interactive, no glass-on-glass).
- `global-skills/apple/swift-feature-scaffold-mvvm-clean-arch/SKILL.md` — the `.sheet(item:)` wizard
  anchoring and ViewModel state.
- `global-skills/apple/swift-infinite-scroll-video-feed-ios26/SKILL.md` — paging for large media sets.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify gallery/sheet/redirect APIs.

## Sources

- [LazyVGrid](https://developer.apple.com/documentation/swiftui/lazyvgrid) · [GridItem](https://developer.apple.com/documentation/swiftui/griditem) · [MagnifyGesture](https://developer.apple.com/documentation/swiftui/magnifygesture) · [presentationDetents(_:)](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:)) · [SFSafariViewController](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller)
- WWDC23 [Design with SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10115/) · WWDC25 [Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`GridItem.adaptive`, `MagnifyGesture` iOS 17,
`SFSafariViewController` confirmed live). Correction vs. draft: the gallery gesture is **`MagnifyGesture`**
(the deprecated name was `MagnificationGesture`). `SFSafariViewController`'s own docs confirm the
"use this, not WKWebView; use ASWebAuthenticationSession for OAuth" guidance.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (glass styling on CTAs evolves; the
grid/gallery/redirect primitives are stable).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
