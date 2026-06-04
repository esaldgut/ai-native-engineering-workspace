# apple — SwiftUI iOS 26 skills

13 skills for native Apple development with Swift 6 and iOS 26. UI patterns, Clean Architecture,
on-device AI, and an anti-pattern registry. Every API was verified live against
`developer.apple.com` before it shipped.

## UI & design system

| Skill | What it does |
|-------|--------------|
| [`swift-liquid-glass-design-system-ios26`](swift-liquid-glass-design-system-ios26/SKILL.md) | iOS 26 Liquid Glass the canonical way — `glassEffect(_:in:)`, `GlassEffectContainer`, the no-glass-on-glass rule, accessibility musts. **Start here** — it's the exemplar. |
| [`swift-adaptive-layouts-ios26`](swift-adaptive-layouts-ios26/SKILL.md) | One app adapting across iPhone/iPad — `NavigationSplitView` as the adaptive root, size-class routing |
| [`swift-ios26-native-ux-patterns`](swift-ios26-native-ux-patterns/SKILL.md) | The iOS 26 UX cluster — collapsible floating tab bar, bottom accessory, zoom transitions |
| [`swift-realtime-chat-ui-patterns-ios26`](swift-realtime-chat-ui-patterns-ios26/SKILL.md) | Chat UI from primitives (no "ChatKit") — date-grouped bubbles, status, typing, swipe-to-reply |
| [`swift-infinite-scroll-video-feed-ios26`](swift-infinite-scroll-video-feed-ios26/SKILL.md) | Vertical full-screen video feed with the AVFoundation memory discipline (player pool) |
| [`swift-ecommerce-marketplace-ui-patterns`](swift-ecommerce-marketplace-ui-patterns/SKILL.md) | Marketplace UI from primitives — product grids, gallery, booking flow, payment redirect |
| [`swift-social-network-profile-patterns`](swift-social-network-profile-patterns/SKILL.md) | Profile UI — shrinking header, follow state machine, moments grid, notification center |

## Architecture & correctness

| Skill | What it does |
|-------|--------------|
| [`swift-clean-architecture-module-scaffold`](swift-clean-architecture-module-scaffold/SKILL.md) | Multi-module MVVM + Clean layers with correct actor isolation per layer (Swift 6 strict concurrency) |
| [`swift-feature-scaffold-mvvm-clean-arch`](swift-feature-scaffold-mvvm-clean-arch/SKILL.md) | One feature end-to-end — View + `@Observable` ViewModel + Repository + DTO + mapper + route |
| [`swift-localizederr-enum-patterns-ios26`](swift-localizederr-enum-patterns-ios26/SKILL.md) | Errors as a `Sendable` `LocalizedError` enum, exhaustive switch, external-error mapping |

## On-device AI & methodology

| Skill | What it does |
|-------|--------------|
| [`swift-ondevice-ai-language-model-patterns`](swift-ondevice-ai-language-model-patterns/SKILL.md) | Foundation Models (iOS 26) — `SystemLanguageModel` gating, `@Generable`, `@Guide`, streaming, tools |
| [`swift-ui-design-iteration-mcp-loop`](swift-ui-design-iteration-mcp-loop/SKILL.md) | Visual UI iteration with an MCP-driven feedback loop — `#Preview` → headless render → evaluate |
| [`apple-anti-patterns`](apple-anti-patterns/SKILL.md) | Append-only registry of Apple-framework anti-patterns, each with the canonical alternative + citation |

---

**Freshness:** these track Apple's annual cycle — re-check after **WWDC** (June) or by **2026-12-01**.
Run `/skill-pattern-freshness-audit apple` to re-verify against current docs.
