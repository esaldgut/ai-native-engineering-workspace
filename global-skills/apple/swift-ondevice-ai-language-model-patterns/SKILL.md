---
name: swift-ondevice-ai-language-model-patterns
description: >-
  Use Apple's on-device Foundation Models framework (iOS 26) from Swift — gate on
  SystemLanguageModel.default.availability BEFORE any call, run a LanguageModelSession with
  instructions, get structured output via guided generation (@Generable types + @Guide constraints,
  with a PartiallyGenerated mirror for streaming), stream snapshots with streamResponse(...), and
  extend the model with the Tool protocol for function calling. Covers the hardware gate (Apple
  Intelligence-capable device + enabled in Settings), the 4,096-token context window, and graceful
  degradation when unavailable. Use when adding on-device summarization, tagging, smart search, or
  generated content — and you must fail soft on unsupported devices.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — Foundation Models / LanguageModelSession (class)"
      url: "https://developer.apple.com/documentation/foundationmodels/languagemodelsession"
      version: "iOS 26.0"
    - source: "Apple Developer — Generable (protocol) + @Generable / @Guide macros"
      url: "https://developer.apple.com/documentation/foundationmodels/generable"
      version: "iOS 26.0"
    - source: "Apple Developer — Tool (protocol) for function calling"
      url: "https://developer.apple.com/documentation/foundationmodels/tool"
      version: "iOS 26.0"
    - source: "WWDC25 Session 286 — Meet the Foundation Models framework"
      url: "https://developer.apple.com/videos/play/wwdc2025/286/"
      version: "WWDC25"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (Apple Intelligence model/API changes), or any iOS 27 beta"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# On-device AI with Foundation Models (iOS 26)

Apple's [Foundation Models](https://developer.apple.com/documentation/foundationmodels) framework
(WWDC25) exposes the on-device LLM behind Apple Intelligence to your app. You always **gate on
availability first**, then run a [`LanguageModelSession`](https://developer.apple.com/documentation/foundationmodels/languagemodelsession),
get type-safe output via guided generation
([`@Generable`](https://developer.apple.com/documentation/foundationmodels/generable) +
`@Guide`), stream partial results, and optionally extend the model with the
[`Tool`](https://developer.apple.com/documentation/foundationmodels/tool) protocol. This is a
**hardware-gated capability**, not just a UI pattern — the framework leads with availability for a
reason.

## When to invoke

- You're adding **on-device generation/understanding**: summarization, content tagging, smart search,
  suggested replies, structured extraction — and want it private and offline.
- You need **structured output** (a Swift type, not a raw string) from the model.
- You must **fail soft** on devices without Apple Intelligence.

**Announce on invoke:** "Using `swift-ondevice-ai-language-model-patterns` to gate availability, run a LanguageModelSession, and generate @Generable output."

Do **not** reach for this for tasks needing world knowledge, long context, or guaranteed
availability — the on-device model is small (4,096-token window) and absent on older hardware. For
those, a server model behind your own API is the right call.

## The canonical APIs (verified)

| API | Signature / form (verified) | Use |
|---|---|---|
| `SystemLanguageModel.default` | `static var default: SystemLanguageModel` | The on-device model handle |
| `.availability` | `var availability: SystemLanguageModel.Availability` → `.available` / `.unavailable(UnavailableReason)` | The mandatory pre-flight gate |
| `UnavailableReason` | `.appleIntelligenceNotEnabled`, `.deviceNotEligible`, `.modelNotReady` | Why it's off |
| `LanguageModelSession` | `final class`; `init(instructions:)`, `init(model:tools:instructions:)` | A stateful generation session |
| `respond(to:options:)` | returns `LanguageModelSession.Response<String>` | One-shot text response |
| `respond(generating:includeSchemaInPrompt:options:prompt:)` | returns `Response<Content>` (guided) | Structured output |
| `streamResponse(to:generating:includeSchemaInPrompt:options:)` | returns `LanguageModelSession.ResponseStream<Content>` | Streamed structured output |
| `@Generable` | macro → `Generable : ConvertibleFromGeneratedContent, ConvertibleToGeneratedContent` | Mark a type the model can produce |
| `@Guide(description:…)` | property macro; supports guides like `.count(_:)`, `.range(_:)` | Constrain / describe a field |
| `Tool` | `protocol Tool<Arguments, Output> : Sendable`; `call(arguments:)` | Function calling |
| `GenerationOptions` | `maximumResponseTokens`, sampling, temperature | Tune the request |

## The rules (load-bearing)

### 1. Gate on `SystemLanguageModel.default.availability` BEFORE constructing a session

Constructing/using a session on an unsupported device fails. Check availability and branch to a
fallback. The unavailable reasons are actionable (prompt the user to enable Apple Intelligence vs.
hide the feature on ineligible hardware):

```swift
switch SystemLanguageModel.default.availability {
case .available:
    // proceed
case .unavailable(.appleIntelligenceNotEnabled):
    // deep-link to Settings, or show "turn on Apple Intelligence"
case .unavailable(.deviceNotEligible), .unavailable(.modelNotReady):
    // hide the feature / degrade gracefully
}
```

### 2. Structured output = a `@Generable` type with `@Guide`d fields

Annotate the type with `@Generable`; describe properties with `@Guide`. The framework uses
constrained sampling so the model can't produce malformed output. Keep descriptions short — they
consume the context window.

```swift
@Generable
struct SearchSuggestions {
    @Guide(description: "Suggested search terms.", .count(4))
    var terms: [String]
}
```

### 3. Streaming yields a `ResponseStream` of partial snapshots — handle progressive reveal

`streamResponse(...)` returns a `LanguageModelSession.ResponseStream<Content>`; iterate it for
`Snapshot`s where the generated content is filled in progressively (a `PartiallyGenerated` mirror
with optional fields). Your UI must render half-filled state, then settle. Call `.collect()` to await
the final value instead.

### 4. The context window is 4,096 tokens — split large work into sessions

Instructions + all prompts + all outputs share one 4,096-token budget. Exceeding it throws
`exceededContextWindowSize(_:)`. For long inputs, chunk the work and run each chunk in a **new**
`LanguageModelSession`, then combine. Tool definitions also consume the window.

### 5. Tools must be `Sendable`; the model decides when to call them

A `Tool` carries a `name` + `description` (the model uses them to decide invocation) and a
`call(arguments:)` whose `Arguments` are themselves `@Generable`. Tools run concurrently, so the
protocol requires `Sendable`.

### 6. Wrap it in a `@MainActor @Observable` capability provider

Surface availability and streamed state to the View layer through an `@Observable` object on the main
actor. The same provider can gate other Apple Intelligence features (Writing Tools, Image Playground)
behind one availability check.

## Canonical example

```swift
import FoundationModels

@Generable
struct Recipe: Sendable {
    @Guide(description: "The recipe title.")            var title: String
    @Guide(description: "Ingredients with quantities.") var ingredients: [String]
    @Guide(description: "Ordered preparation steps.")   var steps: [String]
}

@MainActor @Observable
final class RecipeGenerator {
    enum State { case unsupported(SystemLanguageModel.Availability), idle, streaming(Recipe.PartiallyGenerated), done(Recipe) }
    private(set) var state: State = .idle

    func start() {
        if case .unavailable = SystemLanguageModel.default.availability {
            state = .unsupported(SystemLanguageModel.default.availability); return   // fail soft
        }
        state = .idle
    }

    func generate(prompt: String) async {
        guard case .available = SystemLanguageModel.default.availability else { return }
        let session = LanguageModelSession(
            instructions: "You are a chef. Produce a concise, safe recipe."
        )
        do {
            let stream = session.streamResponse(to: prompt, generating: Recipe.self)
            for try await snapshot in stream {            // progressive reveal
                state = .streaming(snapshot.content)
            }
            let final = try await stream.collect()
            state = .done(final.content)
        } catch {
            // includes exceededContextWindowSize — start a new session with a shorter prompt
        }
    }
}
```

## Decision aid: when NOT to / trade-offs

- **Not for world-knowledge or long context.** 4,096 tokens and a ~3B on-device model — use a server
  model when you need breadth or large inputs.
- **Availability is a moving target at runtime.** Battery, Game Mode, or a still-downloading model can
  flip it to `.modelNotReady`; re-check, don't cache "available" forever.
- **Don't ship a feature that hard-fails on A16 and older.** Those devices report `.deviceNotEligible`
  — design the fallback (hide, or route to a server) as a first-class path.

## Related skills

- `global-skills/apple/swift-clean-architecture-module-scaffold/SKILL.md` — the `@MainActor`
  `@Observable` capability provider and `Sendable` Tool isolation.
- `global-skills/apple/swift-feature-scaffold-mvvm-clean-arch/SKILL.md` — wiring the streamed state
  into a feature's `State` enum.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verify Foundation Models symbols
  after each WWDC (the model + API surface is new and evolving).

## Sources

- [Foundation Models](https://developer.apple.com/documentation/foundationmodels) · [LanguageModelSession](https://developer.apple.com/documentation/foundationmodels/languagemodelsession) · [SystemLanguageModel](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel) · [Generable](https://developer.apple.com/documentation/foundationmodels/generable) · [Tool](https://developer.apple.com/documentation/foundationmodels/tool) · [Generating Swift data structures with guided generation](https://developer.apple.com/documentation/foundationmodels/generating-swift-data-structures-with-guided-generation)
- WWDC25 [286 Meet the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2025/286/) · [259 Code-along: Bring on-device AI to your app](https://developer.apple.com/videos/play/wwdc2025/259/)

---

**Last verified:** 2026-06-03 against Apple Developer docs (live). Resolved draft open questions:
`@Generable`/`@Guide` confirmed real (`Generable` protocol page, sample uses `.count(4)`/`.range(1...10)`);
`streamResponse(to:generating:includeSchemaInPrompt:options:)` returns `ResponseStream<Content>`; `Tool`
is `protocol Tool<Arguments, Output> : Sendable`; the on-device **context window is 4,096 tokens** (Apple
docs), correcting the ~2000 estimate.
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium (new framework; model size and API
surface may shift).
**Found a drift?** Run `/skill-pattern-freshness-audit apple`.
