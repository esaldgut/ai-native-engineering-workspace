---
name: swift-infinite-scroll-video-feed-ios26
description: >-
  Build a vertical, full-screen video feed (TikTok/Reels-style) in SwiftUI with the memory discipline
  AVFoundation requires — a bounded AVPlayer LRU pool of ~3 (current ± 1), not 50 players; vertical
  paging via .scrollTargetBehavior(.paging) + .containerRelativeFrame(.vertical); visibility-driven
  play/pause with onScrollVisibilityChange; "load more" fired by onScrollTargetVisibilityChange (NOT
  .onAppear, which fires prematurely in a lazy stack); cursor pagination at the data layer; a skeleton
  shimmer via phaseAnimator; and optimistic like/follow updates. Documents tearing down AVPlayerLayer
  for off-screen cells and pinning the pool to @MainActor. Use when implementing an autoplaying scroll
  feed and you must keep memory and battery bounded.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — scrollTargetBehavior(.paging) / PagingScrollTargetBehavior"
      url: "https://developer.apple.com/documentation/swiftui/scrolltargetbehavior/paging"
      version: "iOS 17.0"
    - source: "Apple Developer — onScrollVisibilityChange(threshold:_:) (+ onScrollTargetVisibilityChange)"
      url: "https://developer.apple.com/documentation/swiftui/view/onscrollvisibilitychange(threshold:_:)"
      version: "iOS 18.0"
    - source: "Apple Developer — AVPlayerItem.preferredForwardBufferDuration / AVQueuePlayer"
      url: "https://developer.apple.com/documentation/avfoundation/avplayeritem/preferredforwardbufferduration"
      version: "AVFoundation (verified current 2026-06-03)"
    - source: "Apple Developer — phaseAnimator(_:content:animation:) (skeleton shimmer)"
      url: "https://developer.apple.com/documentation/swiftui/view/phaseanimator(_:content:animation:)"
      version: "iOS 17.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (SwiftUI scroll + AVFoundation changes), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# Infinite-scroll video feed (SwiftUI + AVFoundation, iOS 26)

There's no Apple "feed" primitive — a vertical video feed is composition plus **memory discipline**.
Apple's own developer-forum guidance is blunt: don't keep dozens of `AVPlayer`s alive. The canonical
shape is a **bounded LRU player pool** (current ± 1, ~3 players) driving cells that page with
[`.scrollTargetBehavior(.paging)`](https://developer.apple.com/documentation/swiftui/scrolltargetbehavior/paging)
and [`.containerRelativeFrame(.vertical)`](https://developer.apple.com/documentation/swiftui/view/containerrelativeframe(_:alignment:)),
autoplay gated by [`onScrollVisibilityChange`](https://developer.apple.com/documentation/swiftui/view/onscrollvisibilitychange(threshold:_:)),
and pagination fired by `onScrollTargetVisibilityChange`. This skill is **opinionated** on the pool
size — 3 is a sane default, but it's a tunable, not an Apple constant.

## When to invoke

- You're building an **autoplaying, full-screen, vertically-paged feed** and must keep memory and
  battery bounded.
- You need **cursor-based "load more"** that fires reliably (not the broken `.onAppear`-on-last-cell
  trick).
- You're adding **skeleton/shimmer** placeholders or **optimistic** like/follow updates to a feed.

**Announce on invoke:** "Using `swift-infinite-scroll-video-feed-ios26` for the AVPlayer LRU pool, paging, visibility autoplay, and cursor pagination."

Do **not** reach for this for a short, fixed list of videos — a handful of `VideoPlayer`s without a
pool is fine. The pool earns its keep at feed scale.

## The canonical APIs (verified)

| API | Signature / form (verified) | iOS gate | Use |
|---|---|---|---|
| `.scrollTargetBehavior(.paging)` | `static var paging: PagingScrollTargetBehavior` | 17.0+ | Snap one cell per swipe |
| `.containerRelativeFrame(_:alignment:)` | `func containerRelativeFrame(_ axes: Axis.Set, alignment: Alignment = .center) -> some View` | 17.0+ | Make each cell full-screen |
| `.onScrollVisibilityChange(threshold:_:)` | `func onScrollVisibilityChange(threshold: Double = 0.5, _ action: @escaping (Bool) -> Void) -> some View` | 18.0+ | Play/pause by visibility |
| `.onScrollTargetVisibilityChange(idType:threshold:_:)` | fires with the set of visible target ids | 18.0+ | "Load more" trigger |
| `AVPlayer` / `AVQueuePlayer` / `AVPlayerLooper` | `class` | 4.1+ | The player(s); looper for repeat |
| `AVPlayerItem.preferredForwardBufferDuration` | `var preferredForwardBufferDuration: TimeInterval` | 10.0+ | Pre-roll only ~1 chunk |
| `AVPlayer.preferredPeakBitRate` | `var preferredPeakBitRate: Double` | 8.0+ | Cap quality on cellular |
| `.phaseAnimator(_:content:animation:)` | `func phaseAnimator<Phase>(_ phases: some Sequence, content:, animation:) -> some View` | 17.0+ | Shimmer placeholder |
| `AVAssetDownloadURLSession` | `class` | 9.0+ | True offline HLS (heavyweight) |

## The rules (load-bearing)

### 1. Bounded LRU player pool, pinned to `@MainActor`

Keep at most ~3 players (visible ± 1). On eviction, **tear down**, don't just pause. The pool mutates
shared state under concurrent play/pause calls coming from visibility events, so isolate it on
`@MainActor` (or make it an `actor`) — otherwise you violate Swift 6 `Sendable`.

```swift
@MainActor
final class PlayerPool {
    private var players: [URL: AVPlayer] = [:]
    private var order: [URL] = []
    private let capacity = 3
    func player(for url: URL) -> AVPlayer {
        if let p = players[url] { return p }
        let item = AVPlayerItem(url: url)
        item.preferredForwardBufferDuration = 1.0          // pre-roll ~1 HLS chunk
        let p = AVPlayer(playerItem: item)
        players[url] = p; order.append(url)
        if order.count > capacity, let evict = order.first {
            players[evict]?.replaceCurrentItem(with: nil)  // real teardown
            players[evict] = nil; order.removeFirst()
        }
        return p
    }
}
```

### 2. Tear down `AVPlayerLayer` for off-screen cells, not just `pause()`

Pausing keeps decode/render resources alive. For cells scrolled out of the pool window, remove the
player layer entirely (set the item to `nil`). This is the single biggest memory win in a feed.

### 3. Each cell is full-screen via `.containerRelativeFrame(.vertical)`; paging snaps it

```swift
ScrollView {
    LazyVStack(spacing: 0) {
        ForEach(items) { item in
            FeedCell(item: item).containerRelativeFrame([.horizontal, .vertical])
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.paging)
```

### 4. Autoplay with `onScrollVisibilityChange` — tune the threshold up for video

Default threshold is 0.5; for a video feed, 0.6–0.75 avoids flicker as cells cross. Gate
`player.play()` / `player.pause()` on the boolean it hands you.

### 5. "Load more" fires from `onScrollTargetVisibilityChange`, NOT `.onAppear`

`.onAppear` on the last cell of a `LazyVStack` fires **prematurely** because lazy realization runs
ahead of the viewport. Use `onScrollTargetVisibilityChange` (iOS 18+) and request the next cursor
page when a near-the-end id becomes visible. Cursor pagination itself is a **data-layer** concern (a
`nextToken`/cursor in the response), not a SwiftUI API.

### 6. Skeleton shimmer with `phaseAnimator`; optimistic updates flip state immediately

A `LinearGradient` masked and driven by `.phaseAnimator` gives a dependency-free shimmer. For
like/follow, mutate the `@Observable` item optimistically, fire the request, and revert on failure —
don't block the UI on the network.

## Canonical example

```swift
struct VideoFeed: View {
    @State private var store: FeedStore                     // @MainActor @Observable, owns the pool
    var body: some View {
        ScrollView {
            LazyVStack(spacing: 0) {
                ForEach(store.items) { item in
                    FeedCell(item: item, player: store.pool.player(for: item.videoURL))
                        .containerRelativeFrame([.horizontal, .vertical])
                        .onScrollVisibilityChange(threshold: 0.7) { visible in
                            store.setPlaying(item.id, visible)        // play/pause
                        }
                }
            }
            .scrollTargetLayout()
        }
        .scrollTargetBehavior(.paging)
        .ignoresSafeArea()
        .onScrollTargetVisibilityChange(idType: FeedItem.ID.self) { visibleIDs in
            if store.isNearEnd(visibleIDs) { Task { await store.loadNextPage() } }  // cursor pagination
        }
    }
}
```

## Decision aid: when NOT to / trade-offs

- **Pool size is a tunable.** 3 is conservative; a high-end device can hold 4–5, a low-RAM device may
  want 2. Don't hard-code it as if it were an Apple constant.
- **`CachingPlayerItem` is third-party, not Apple.** If you need in-feed caching, know that
  AVFoundation has no "download-while-streaming" primitive — the supported offline path is the
  heavyweight `AVAssetDownloadURLSession`, or a local reverse proxy. Mention it; don't pretend it's free.
- **Don't autoplay with sound by default** — respect the silent switch and start muted, matching feed
  conventions and HIG.

## Related skills

- `global-skills/apple/swift-ios26-native-ux-patterns/SKILL.md` — the zoom transition and
  visibility-autoplay modifier shared with this feed.
- `global-skills/apple/swift-clean-architecture-module-scaffold/SKILL.md` — the `@MainActor` pool
  isolation and `Sendable` cursor-paginated repository.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify scroll + AVFoundation APIs.

## Sources

- [scrollTargetBehavior(.paging)](https://developer.apple.com/documentation/swiftui/scrolltargetbehavior/paging) · [containerRelativeFrame(_:alignment:)](https://developer.apple.com/documentation/swiftui/view/containerrelativeframe(_:alignment:)) · [onScrollVisibilityChange(threshold:_:)](https://developer.apple.com/documentation/swiftui/view/onscrollvisibilitychange(threshold:_:)) · [AVPlayerItem.preferredForwardBufferDuration](https://developer.apple.com/documentation/avfoundation/avplayeritem/preferredforwardbufferduration) · [AVQueuePlayer](https://developer.apple.com/documentation/avfoundation/avqueueplayer) · [phaseAnimator(_:content:animation:)](https://developer.apple.com/documentation/swiftui/view/phaseanimator(_:content:animation:))
- WWDC23 [Beyond scroll views](https://developer.apple.com/videos/play/wwdc2023/10159/) · WWDC24 [What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2024/10144/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (`.paging` iOS 17, `onScrollVisibilityChange`
/ `onScrollTargetVisibilityChange` iOS 18, `preferredForwardBufferDuration` confirmed live). The
AVPlayer-pool sizing and "tear down, don't pause" guidance trace to Apple developer-forum advice on
feed memory.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (scroll-visibility APIs are iOS 18-new).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
