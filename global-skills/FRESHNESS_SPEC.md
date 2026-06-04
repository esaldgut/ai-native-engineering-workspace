# Freshness Frontmatter Specification

> **What this is.** Every skill in this repo carries machine-readable provenance: *what
> official source it was verified against, when, at which version, and when it should be
> re-checked.* This turns a static skill library into a **maintainable** one — drift can
> be detected mechanically instead of discovered when a reader copies a stale pattern into
> production.

## Why it exists

Technical patterns for fast-moving platforms (Apple, AWS, Android, Claude Code) age in
months, not years. During this repo's own research phase, five patterns were found already
stale or mythical at the moment of verification:

| Pattern | Naive assumption | Verified reality | Decay window |
|---|---|---|---|
| CDK drift detection | "use `aws cli describe`" | `cdk drift` command shipped (CDK 2.1110.0, Mar 2026) | ~3 months |
| Android secrets | `EncryptedSharedPreferences` | deprecated → DataStore + Tink + Keystore | ~1–2 years |
| Hilt + Compose | `hilt-navigation-compose` artifact | moved to `hilt-lifecycle-viewmodel-compose` (Hilt 1.3.0) | ~9 months |
| Coroutine tests | `runBlockingTest` | deprecated → `runTest` (kotlinx 1.6+) | already obsolete |
| Constant-time compare | `CryptoKit.timingSafeEqual` | **never existed** — common myth | permanent |

A skill without provenance silently rots into one of those traps. The freshness block makes
the rot **visible and queryable**: the `skill-pattern-freshness-audit` skill reads these
blocks, compares against current docs, and flags `STALE` / `BREAKING_CHANGE` / `STILL_VALID`.

## The block

Append a `freshness:` key to the standard Agent Skill frontmatter. The skill's existing
`name`, `description`, and optional `version` / `disable-model-invocation` keys are unchanged
— `freshness` is additive and ignored by Claude Code's skill loader (it only reads `name` /
`description`).

```yaml
---
name: swift-liquid-glass-design-system-ios26
description: >-
  Apply iOS 26 Liquid Glass to SwiftUI views — .glassEffect(), GlassEffectContainer,
  glass button styles, and the "no glass on glass" rule. Use when building or restyling
  SwiftUI surfaces that target iOS 26+.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer Documentation — glassEffect(_:in:)"
      url: "https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)"
      version: "iOS 26.0"
    - source: "WWDC25 Session 323 — Build a SwiftUI app with the new design"
      url: "https://developer.apple.com/videos/play/wwdc2025/323/"
      version: "WWDC25"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "WWDC26 keynote (annual API surface change)"
    or_date: "2026-12-01"
  decay_risk: medium        # low | medium | high
  status: current           # current | needs-recheck | stale | superseded
---
```

## Field reference

### `freshness.verified_against` (required, list)

One entry per **primary source** the pattern was checked against. Prefer official docs over
blog posts; if a community source is the only canonical reference, mark it and still cite the
nearest official page.

| Sub-field | Required | Notes |
|---|---|---|
| `source` | yes | Human-readable name of the doc / session / RFC / spec |
| `url` | yes | Citable URL. Must resolve. Prefer `developer.apple.com`, `docs.aws.amazon.com`, `developer.android.com`, `docs.claude.com`, `modelcontextprotocol.io`, IETF RFC, OWASP |
| `version` | yes | The platform/SDK/spec version the claim was true for (e.g., `iOS 26.0`, `AWS SDK Go v2 1.30`, `Compose BOM 2026.05.01`, `Claude Code v2.1.121`, `RFC 8252`) |

Minimum **2** entries per skill (the anti-sterilization rule: at least one must be a primary
vendor doc, not only a blog).

### `freshness.verified_on` (required, date)

ISO `YYYY-MM-DD` of the verification. This is the anchor the audit skill diffs against.

### `freshness.recheck_after` (required, object)

When the pattern should be re-verified. Two complementary triggers — whichever fires first:

| Sub-field | Required | Notes |
|---|---|---|
| `trigger` | yes | An event that invalidates the pattern (e.g., `"WWDC26 keynote"`, `"next Compose BOM major"`, `"AWS SDK Go v2 major bump"`, `"Claude Code minor release"`). Human-readable; the audit skill matches keywords |
| `or_date` | yes | ISO date hard backstop. If neither event is observed by this date, re-check anyway |

Cadence guidance by domain (default `or_date` horizon):

| Domain | Event trigger | Hard backstop |
|---|---|---|
| `apple/*` | post-WWDC (June) | 6 months |
| `apple-auth/*` | post-WWDC + any CryptoKit/Security release | 6 months |
| `aws-go/*` | AWS SDK Go v2 major + CDK CLI major | 3 months |
| `android/*` | post-Google I/O (May) + Compose BOM major | 6 months |
| `claude-code-workflow/*` | Claude Code minor release | 1 month |
| `meta/*` | when the methodology itself is revised | 6 months |

### `freshness.decay_risk` (required, enum)

How fast this specific pattern tends to rot, independent of cadence:

- `low` — stable primitive unlikely to change (e.g., a Clean Architecture layering convention, an RFC-grounded auth rule). Re-check is cheap insurance.
- `medium` — vendor API surface that evolves but rarely breaks (most SwiftUI/Compose view APIs).
- `high` — actively churning surface where breaking renames/deprecations are routine (Hilt artifacts, Jetpack Security, Claude Code features, CDK CLI verbs).

### `freshness.status` (required, enum)

Current verification state. Set by the author at write time; **updated by the audit skill**:

- `current` — verified, within its `recheck_after` window, no known drift.
- `needs-recheck` — past `recheck_after.or_date` OR its `trigger` event fired; not yet re-verified.
- `stale` — audit found the pattern partially wrong (an API was renamed/deprecated; a better tool exists). Body needs an update note but core may still hold.
- `superseded` — audit found the pattern fundamentally replaced (e.g., a skill recommending `EncryptedSharedPreferences`). Body must carry a migration pointer.

## Body convention: the "Last verified" footer

In addition to the frontmatter, every SKILL.md ends with a short human-visible footer so a
reader scanning the rendered file (not the YAML) sees provenance:

```markdown
---

**Last verified:** 2026-06-03 against iOS 26.0 (Apple Developer docs + WWDC25 #323).
**Re-check after:** WWDC26, or by 2026-12-01. **Decay risk:** medium.
**Found a drift?** Open an issue or run `/skill-pattern-freshness-audit apple`.
```

## Anti-sterilization rule (enforced)

Borrowed from the `skill-extraction-pattern` methodology audit (critique 3): every skill body
must contain **at least one fully concrete example** naming a real public API/library/tool
(e.g., `.glassEffect()`, `cdk drift`, `hilt-lifecycle-viewmodel-compose`, `runTest`). The
concrete example anchors the trigger and gives the audit skill a literal string to grep for
when checking if the named API still exists. A skill with only abstract prose ("apply the
appropriate modifier") cannot be freshness-audited and is rejected.

## Validation

A skill's freshness block is valid iff:

1. `verified_against` has ≥2 entries, each with resolving `url` + non-empty `version`.
2. At least one `verified_against` entry is a primary vendor/standards source (not a blog/Medium/dev.to).
3. `verified_on` ≤ today.
4. `recheck_after.or_date` > `verified_on`.
5. `decay_risk` ∈ {low, medium, high}; `status` ∈ {current, needs-recheck, stale, superseded}.
6. The body contains ≥1 concrete public API/tool string (anti-sterilization).

The `skill-pattern-freshness-audit` skill checks 1–6 as a lint pass before doing drift detection.

---

**Spec version:** 1.0.0
**Established:** 2026-06-03
**Applies to:** every `SKILL.md` under `global-skills/`
**Audited by:** `global-skills/meta/skill-pattern-freshness-audit/SKILL.md`
