---
name: swift-ui-design-iteration-mcp-loop
description: >-
  A methodology for iterating SwiftUI UI with a visual, MCP-driven feedback loop — write a #Preview,
  render it headlessly, screenshot the running view, evaluate the bitmap against Apple's Human
  Interface Guidelines, then refine the code and repeat, keeping code + screenshot + evaluation on one
  commit so the loop doesn't drift. Encodes the HIG rules to check each pass (contrast, Dynamic Type,
  Reduce Transparency/Motion, touch targets, "glass on controls, not content"), a SwiftUI performance
  checklist (state granularity, no GeometryReader in lazy stacks, .drawingGroup for static layers),
  and which MCP tools do which step. Use when designing or polishing a view and you want grounded
  visual evaluation instead of guessing from code.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — Preview(_:body:) macro (#Preview)"
      url: "https://developer.apple.com/documentation/swiftui/preview(_:body:)"
      version: "Xcode 15+ / iOS 13.0+ availability stamp"
    - source: "Apple Human Interface Guidelines (evaluation rubric)"
      url: "https://developer.apple.com/design/human-interface-guidelines/"
      version: "2026 (verified current 2026-06-03)"
    - source: "WWDC23 Session 10160 — Demystify SwiftUI performance"
      url: "https://developer.apple.com/videos/play/wwdc2023/10160/"
      version: "WWDC23"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (HIG + SwiftUI performance guidance updates), or any Xcode 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# UI design iteration with an MCP visual loop (SwiftUI)

This is a **workflow skill**, not an API. It runs a tight loop:
[`#Preview`](https://developer.apple.com/documentation/swiftui/preview(_:body:)) → render →
screenshot → evaluate the bitmap against the
[Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/) →
refine → repeat. The point is **grounded** visual feedback — judging a rendered pixel against HIG
instead of guessing from source. It leans on the project's MCP servers (Xcode + XcodeBuild + Apple
docs) to do each step without leaving the agent.

## When to invoke

- You're **designing or polishing a view** and want to see it, evaluate it against HIG, and iterate —
  not ship the first compile.
- You're checking a view across **Dynamic Type, dark mode, Reduce Transparency** before calling it done.
- You suspect a **performance** problem (jank, over-invalidation) and want a checklist pass.

**Announce on invoke:** "Using `swift-ui-design-iteration-mcp-loop` to render, screenshot, and evaluate this view against HIG, then iterate."

Do **not** use this as a substitute for running real interaction tests — the loop evaluates
*appearance and layout*. Behavior (gestures, navigation, data flow) still needs the app run and tests.

## The loop (each pass)

1. **Write/extend a `#Preview`** with the states that matter (light, dark, Dynamic Type XXL, an empty
   and a populated state). The preview `body` is `@MainActor`; provide model data inline.
2. **Render headlessly** — `mcp__xcode__RenderPreview` returns the preview bitmap; or build+run on a
   simulator and capture with `mcp__XcodeBuildMCP__screenshot`.
3. **Evaluate the bitmap** against the HIG checklist below (multimodal: actually look at the pixels).
   Query specifics with `mcp__xcode__DocumentationSearch` / `mcp__apple-docs__search_apple_docs`.
4. **Refine the code**, then re-render. Keep code + screenshot + evaluation on the **same commit** so
   the feedback doesn't drift across changes.

## The HIG rules to check each pass

| Check | Rule | How |
|---|---|---|
| Contrast | Text on any surface meets **WCAG 4.5:1** | Inspect the screenshot; darken/strengthen if marginal |
| Dynamic Type | Layout survives `.dynamicTypeSize(.accessibility3)` | Add a Dynamic Type XXL `#Preview` |
| Reduce Transparency | Glass/material falls back to opaque | Preview with the trait; provide a solid fallback |
| Reduce Motion | Zoom/morph transitions simplify | Suppress `glassEffectID` morphs and large motion |
| Touch targets | Interactive elements ≥ **44×44 pt** | Measure controls in the screenshot |
| Glass scope | Glass on **controls/navigation**, not body content | No frosted panels behind paragraphs |
| Color semantics | Use semantic colors (`.primary`, `.tint`) not hardcoded hex | Verify in light AND dark |
| Safe areas | Content respects notch/Dynamic Island/home indicator | Screenshot on a notched device |

## The SwiftUI performance checklist

- **State granularity:** the smallest view possible owns each `@State`; prefer `@Observable` with
  per-property tracking so a change invalidates only what depends on it.
- **No `GeometryReader` inside `LazyVStack`/`LazyHStack`** — it forces eager layout and defeats
  laziness. Use `containerRelativeFrame`/`onScrollGeometryChange` instead.
- **`.drawingGroup()` (Metal) for static, expensive layers** (complex gradients/shadows) — but not on
  frequently-animated content, where it can hurt.
- **Hoist constants out of `body`;** `body` runs often. Don't allocate formatters/arrays per render.
- **`equatable()` / `EquatableView`** to short-circuit re-renders of pure subviews when inputs are
  unchanged.
- Measure with **Instruments** and the WWDC23 "Demystify SwiftUI performance" guidance — don't guess.

## The MCP tooling map

| Step | Tool (this environment) |
|---|---|
| Render a `#Preview` headlessly | `mcp__xcode__RenderPreview` |
| Build + run on a simulator | `mcp__XcodeBuildMCP__build_run_sim` |
| Screenshot the running view | `mcp__XcodeBuildMCP__screenshot` |
| Inspect the view hierarchy / coordinates | `mcp__XcodeBuildMCP__snapshot_ui` |
| Look up an HIG rule or API | `mcp__xcode__DocumentationSearch`, `mcp__apple-docs__search_apple_docs` |

Tool names are environment-specific; if these MCP servers aren't connected, the loop degrades to:
build in Xcode → screenshot manually → evaluate → edit. The *methodology* is the durable part.

## Canonical example: the previewable unit you iterate on

```swift
struct StatCard: View {
    let title: String
    let value: String
    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            Text(title).font(.subheadline).foregroundStyle(.secondary)   // semantic color
            Text(value).font(.title2).bold()
        }
        .padding()
        .background(.regularMaterial, in: .rect(cornerRadius: 16))       // material, not faux glass
        .frame(minWidth: 44, minHeight: 44)                             // touch-target floor
    }
}

#Preview("Light")            { StatCard(title: "Followers", value: "12.4k").preferredColorScheme(.light) }
#Preview("Dark")             { StatCard(title: "Followers", value: "12.4k").preferredColorScheme(.dark) }
#Preview("Dynamic Type XXL") { StatCard(title: "Followers", value: "12.4k").dynamicTypeSize(.accessibility3) }
#Preview("Reduce Transparency") {
    StatCard(title: "Followers", value: "12.4k").environment(\.accessibilityReduceTransparency, true)
}
```

## Decision aid: when NOT to / trade-offs

- **Don't iterate forever.** Two or three passes against the checklist is usually enough; perfectionism
  on a screen nobody's blocked on is waste.
- **`#Preview` async caveat:** unstructured `Task {}` in a preview can behave oddly — drive async work
  through `.task` with preview-only model data, not detached tasks.
- **The loop can echo.** If you change code between rendering and evaluating, you're judging stale
  pixels — keep each pass on one commit hash.
- **HIG changes yearly.** Re-query the guidelines each major OS cycle; a rule that held last year may
  have moved.

## Related skills

- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — the glass rules this loop
  checks ("glass on controls, not content"; Reduce Transparency fallback).
- `global-skills/apple/swift-ios26-native-ux-patterns/SKILL.md` — the motion/transitions this loop
  evaluates against Reduce Motion.
- `global-skills/apple/apple-anti-patterns/SKILL.md` — the registry this loop feeds when it catches a
  repeatable mistake.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify HIG + performance guidance.

## Sources

- [Preview(_:body:) macro](https://developer.apple.com/documentation/swiftui/preview(_:body:)) · [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/) · [DocC](https://developer.apple.com/documentation/docc)
- WWDC23 [Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/) · WWDC25 [Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`#Preview` macro confirmed:
`Preview(_ name: String? = nil, @ViewBuilder body: @escaping @MainActor () -> any View)`) + HIG +
WWDC23 #10160. MCP tool names reflect this environment's Xcode/XcodeBuild/apple-docs servers.
**Re-check after:** WWDC26 / Xcode 27, or by 2026-12-01. **Decay risk:** medium (HIG and the MCP tool
surface both evolve; the loop methodology is stable).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
