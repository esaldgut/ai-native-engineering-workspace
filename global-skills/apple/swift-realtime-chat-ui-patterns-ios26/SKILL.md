---
name: swift-realtime-chat-ui-patterns-ios26
description: >-
  Build a real-time chat UI in SwiftUI from primitives (there is no Apple "ChatKit") — date-grouped
  message bubbles with LazyVStack(pinnedViews:[.sectionHeaders]), scroll-to-bottom-on-new-message via
  ScrollPosition(.bottom), swipe-to-reply with .swipeActions on List rows, a typing indicator scoped
  to its own @Observable state so it doesn't redraw the thread, and a URLSessionWebSocketTask stream
  bridged to @Observable conversation state with optimistic local-first updates. Documents which
  pieces are iOS 15/17/18 and that presence/typing require a server push (no client API). Use when
  implementing a messaging thread, status indicators, typing, or swipe-to-reply the native way.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — ScrollPosition (scroll-to-bottom) + scrollPosition(_:anchor:)"
      url: "https://developer.apple.com/documentation/swiftui/scrollposition"
      version: "iOS 18.0"
    - source: "Apple Developer — swipeActions(edge:allowsFullSwipe:content:)"
      url: "https://developer.apple.com/documentation/swiftui/view/swipeactions(edge:allowsfullswipe:content:)"
      version: "iOS 15.0"
    - source: "Apple Developer — URLSessionWebSocketTask"
      url: "https://developer.apple.com/documentation/foundation/urlsessionwebsockettask"
      version: "iOS 13.0"
    - source: "Apple Developer — LazyVStack / PinnedScrollableViews"
      url: "https://developer.apple.com/documentation/swiftui/lazyvstack"
      version: "iOS 14.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (SwiftUI scroll/list changes), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Real-time chat UI patterns (SwiftUI, iOS 26)

There is **no Apple-blessed "ChatKit"** — a chat thread is composition over SwiftUI primitives plus
Swift Concurrency for the live stream. The load-bearing pieces are
[`LazyVStack`](https://developer.apple.com/documentation/swiftui/lazyvstack) with pinned section
headers for date grouping,
[`ScrollPosition`](https://developer.apple.com/documentation/swiftui/scrollposition) for
scroll-to-bottom, [`swipeActions`](https://developer.apple.com/documentation/swiftui/view/swipeactions(edge:allowsfullswipe:content:))
for swipe-to-reply, and a [`URLSessionWebSocketTask`](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
stream surfaced through `@Observable` state. This skill is **framework-agnostic** about the
transport — bring your own socket; iOS 26 only adds Liquid Glass to the bubbles for free.

## When to invoke

- You're implementing a **messaging thread**: bubbles, date separators, status (sent/delivered/read),
  typing indicator, swipe-to-reply, or scroll-to-bottom on a new message.
- You're bridging a **WebSocket message stream** into SwiftUI and want the invalidation to stay
  scoped (one new message shouldn't redraw the whole thread).

**Announce on invoke:** "Using `swift-realtime-chat-ui-patterns-ios26` to build the thread from SwiftUI primitives + a WebSocket-backed @Observable."

Do **not** invent a presence/typing API — Apple has none. Typing/online state is a **server push**
concern; this skill renders it but does not synthesize it.

## The canonical APIs (verified)

| API | Signature / form (verified) | iOS gate | Use |
|---|---|---|---|
| `LazyVStack(pinnedViews:)` | `init(alignment:spacing:pinnedViews:content:)`; `pinnedViews: [.sectionHeaders]` | 14.0+ | Date-section bubbles with sticky headers |
| `ScrollPosition` | `ScrollPosition(idType:)`; `.scrollTo(edge: .bottom)` | 18.0+ | Programmatic scroll-to-bottom |
| `.scrollPosition(_:anchor:)` | `func scrollPosition(_ position: Binding<ScrollPosition>, anchor: UnitPoint? = nil) -> some View` | 18.0+ | Keep newest message pinned |
| `.scrollTargetLayout()` | marks the layout whose items the position tracks | 17.0+ | Required for id-based scroll position |
| `.swipeActions(edge:allowsFullSwipe:content:)` | `func swipeActions<T>(edge: HorizontalEdge = .trailing, allowsFullSwipe: Bool = true, @ViewBuilder content:) -> some View` | 15.0+ | Swipe-to-reply / delete (List rows) |
| `URLSessionWebSocketTask` | `send(_:completionHandler:)`, `receive(completionHandler:)`; `ws:`/`wss:` | 13.0+ | The live message transport |
| `@Observable` | macro | 17.0+ | Scoped conversation / typing state |

## The rules (load-bearing)

### 1. Group by day with pinned section headers — `pinnedViews` only works in a *lazy* stack

`pinnedViews: [.sectionHeaders]` pins inside `LazyVStack`/`LazyHStack` only — a plain `VStack`
ignores it. Wrap the thread in `ScrollView { LazyVStack(pinnedViews: [.sectionHeaders]) { … } }` with
a `Section(header:)` per day.

### 2. Scroll-to-bottom: pick ONE mechanism

iOS 18's `ScrollPosition` (`.scrollTo(edge: .bottom)` or `.scrollPosition($pos, anchor: .bottom)`)
and the older `ScrollViewReader.scrollTo(id, anchor:)` **compete** inside the same `ScrollView` —
using both fights for control. Prefer `ScrollPosition` on iOS 18+; fall back to `ScrollViewReader`
only on iOS 17. Drive it from `.onChange(of: messages.last?.id)`.

### 3. Swipe-to-reply uses `.swipeActions`, which requires a `List` row

`.swipeActions` is a `List`-row affordance. If your thread is a `LazyVStack` (common, for custom
bubble layout), `.swipeActions` won't apply — implement reply-swipe with a `DragGesture` + offset on
the bubble instead. Don't expect `.swipeActions` to work on a non-`List` row.

### 4. Scope typing-indicator state so it doesn't redraw the thread

A typing indicator that lives on the same `@Observable` as `messages` will invalidate every bubble
on each keystroke event. Give it its **own** `@Observable TypingState`; Observation's per-property
tracking then redraws only the indicator view.

```swift
@Observable final class TypingState { var peersTyping: Set<UserID> = [] }   // isolated invalidation
```

### 5. Optimistic local-first, then reconcile — it's a pattern, not an API

Append the outgoing message to `@Observable` state immediately with a `.sending` status, send over
the socket, then flip to `.sent`/`.failed` on ack. Model status as an enum on the message; never
block the UI on the round-trip.

### 6. The socket stream is `Sendable` work — bridge it on the main actor

Read frames off the `URLSessionWebSocketTask` in a `Task`, decode to `Sendable` message values, and
hand them to the `@MainActor @Observable` store. Don't mutate UI state off-actor.

## Canonical example

```swift
struct Conversation: View {
    @State private var store: ChatStore                 // @MainActor @Observable
    @State private var position = ScrollPosition(idType: Message.ID.self)

    var body: some View {
        ScrollView {
            LazyVStack(pinnedViews: [.sectionHeaders]) {
                ForEach(store.days, id: \.date) { day in
                    Section {
                        ForEach(day.messages) { msg in
                            Bubble(message: msg)            // shows .sending/.sent/.read status
                                .id(msg.id)
                        }
                    } header: {
                        DateSeparator(date: day.date).background(.bar)
                    }
                }
            }
            .scrollTargetLayout()
        }
        .scrollPosition($position, anchor: .bottom)
        .onChange(of: store.messages.last?.id) { _, _ in
            withAnimation { position.scrollTo(edge: .bottom) }   // newest message into view
        }
        .safeAreaInset(edge: .bottom) {
            Composer { text in store.sendOptimistically(text) }   // local-first append
        }
        .task { await store.connect() }                         // URLSessionWebSocketTask stream
    }
}
```

## Decision aid: when NOT to / trade-offs

- **`List` vs `LazyVStack`:** `List` gives you `.swipeActions` and recycling for free but constrains
  bubble styling and insets. Custom bubbles → `LazyVStack` + manual swipe gesture. Pick based on how
  bespoke the bubble design is.
- **Don't ship a bundled socket client in the UI layer** — keep the transport behind a `Sendable`
  protocol so tests can inject a fake stream (an `AsyncStream` of messages).
- **Presence is server-truth.** Render "typing…" / "online" from pushed events; do not infer it
  client-side from request timing.

## Related skills

- `global-skills/apple/swift-clean-architecture-module-scaffold/SKILL.md` — the `Sendable` transport
  protocol and `@MainActor` store isolation this thread depends on.
- `global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md` — glass bubbles; one
  `GlassEffectContainer` per cluster, never glass-on-glass.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify scroll/list APIs each WWDC.

## Sources

- [LazyVStack](https://developer.apple.com/documentation/swiftui/lazyvstack) · [PinnedScrollableViews](https://developer.apple.com/documentation/swiftui/pinnedscrollableviews) · [ScrollPosition](https://developer.apple.com/documentation/swiftui/scrollposition) · [swipeActions(edge:allowsFullSwipe:content:)](https://developer.apple.com/documentation/swiftui/view/swipeactions(edge:allowsfullswipe:content:)) · [URLSessionWebSocketTask](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
- WWDC23 [Beyond scroll views](https://developer.apple.com/videos/play/wwdc2023/10159/) · WWDC23 [Discover Observation in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10149/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`ScrollPosition` iOS 18+, `swipeActions`
iOS 15+, `URLSessionWebSocketTask` iOS 13+ confirmed live). Confirmed there is **no** Apple chat-UI
framework — these are the canonical building blocks.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (scroll-position APIs modernized
in iOS 18; watch for further changes).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
