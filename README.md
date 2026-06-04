# AI-Native Engineering Workspace

A public library of **41 reusable Claude Code Agent Skills**, the **platform-base workflow docs**
they extend, and a **freshness system** that re-verifies each pattern against the official docs it
cites.

Each skill is a generic extraction from production engineering, rewritten from scratch (not
copy-pasted) and verified against vendor documentation: Apple Developer, AWS, Android, Anthropic.
Every skill records its own provenance: which sources it was checked against, when, at which
version, and when to re-check it.

Scope: working patterns and conventions across Apple (Swift 6 / iOS 26), AWS (Lambda Go / CDK),
Android (Kotlin / Compose / KMP), and Claude Code / MCP. Not a tutorial collection — these are
the patterns an AI-native engineer applies, published for reuse.

---

## What's inside

| Area | Count | What it covers |
|------|------:|----------------|
| **`global-skills/apple/`** | 13 | SwiftUI iOS 26: Liquid Glass, adaptive layouts, native UX, chat / feed / marketplace / social-graph UI; Clean Architecture + error patterns + feature scaffolding; on-device AI (Foundation Models); an append-only Apple anti-pattern registry |
| **`global-skills/apple-auth/`** | 6 | Post-quantum CryptoKit, auth security audit + checklist, performance benchmarks, Swift Testing patterns — provider-agnostic (Cognito / Firebase / Auth0 as examples) |
| **`global-skills/aws-go/`** | 8 | AWS SDK Go v2 version policy, canonical error handling, Lambda cold-start segregation, refactor blast-radius audit, MongoDB TTL correctness, CDK drift detection, adversarial pre-build audit, provider-API property verification |
| **`global-skills/android/`** | 5 | Compose Clean Architecture + feature scaffolding, JUnit5/Turbine/MockK testing, security checklist (DataStore + Tink + Keystore), Kotlin Multiplatform extraction |
| **`global-skills/claude-code-workflow/`** | 6 | Project bootstrap, lesson-capture pipeline, build-sync pipeline, MCP orchestration + the Android-specific twins |
| **`global-skills/meta/`** | 3 | Skills *about* skills: extraction methodology, freshness audit, dossier-driven update |
| **`workflow-docs/`** | 13 | The generic platform base (Apple, Android, Expo, AWS, Next.js, MCP, Chrome, TypeScript, Shell, Kali, + the platform premise). Skills extend these. |
| **`reference-projects/`** | 2 | Sanitized architecture docs for a native iOS app and a native Android app — Clean Architecture, tech-research decisions, iOS 26 UX patterns |

**Total: 41 skills + 13 base docs**, roughly 14,000 lines of documented engineering.

---

## The idea: platform base + extensions

A project should never start from an ambiguous context. The flow this repo encodes:

```
workflow-docs/  (generic platform base)         ← context loads here first
        │  a project's CLAUDE.md @imports the relevant base docs
        ▼
project CLAUDE.md + docs/<DOMAIN>_EXTENSIONS.md  ← project-specific detail lives here
```

The base docs are generic and reusable across any project of the same kind. A specific project
**extends** them — it never edits the base to inject its own bundle IDs, account IDs, or
architecture. The `project-init` skill automates this bootstrap; `PLATFORM_BASE.md` states the
full premise.

---

## The freshness system (why this stays trustworthy)

Technical patterns for fast-moving platforms rot in months. During this repo's own research,
five patterns were already stale or mythical at the moment of verification:

| Pattern | Naive assumption | Verified reality |
|---|---|---|
| CDK drift | "use `aws cloudformation detect-stack-drift` by hand" | `cdk drift` command (CDK CLI 2.1017.0, May 2025) |
| Android secrets | `EncryptedSharedPreferences` | deprecated → DataStore + Tink + Keystore |
| Hilt + Compose | `hilt-navigation-compose` | renamed to `hilt-lifecycle-viewmodel-compose` (Hilt 1.3.0) |
| Coroutine tests | `runBlockingTest` | deprecated → `runTest` (kotlinx-coroutines-test 1.6+) |
| Constant-time compare | `CryptoKit.timingSafeEqual` | **never existed** — use `HMAC.isValidAuthenticationCode` |

So every skill carries a `freshness` block in its frontmatter: cited sources with versions, a
verification date, a re-check trigger, and a status. The **`skill-pattern-freshness-audit`** skill
reads those blocks, re-verifies each cited API against current docs, and sets each status to
`current`, `needs-recheck`, `stale`, or `superseded` with the citation that proves it. The
**`dossier-driven-skill-update`** skill repairs what the audit flags. The contract is in
[`global-skills/FRESHNESS_SPEC.md`](global-skills/FRESHNESS_SPEC.md).

The detect-then-repair loop makes drift a queryable signal rather than something a reader finds in
production.

---

## How a skill is structured

Every `SKILL.md` follows the same shape — a focused, single-job Agent Skill:

```yaml
---
name: swift-liquid-glass-design-system-ios26
description: >-
  Apply iOS 26 Liquid Glass to custom SwiftUI views the canonical way … Use when building
  or restyling SwiftUI surfaces that target iOS 26+ …
version: "1.0.0"
freshness:
  verified_against:
    - source: "Apple Developer — View.glassEffect(_:in:)"
      url: "https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)"
      version: "iOS 26.0"
  verified_on: "2026-06-03"
  recheck_after: { trigger: "WWDC26 keynote", or_date: "2026-12-01" }
  decay_risk: medium
  status: current
---
```

…followed by: when to invoke (with trigger signals), a table of **verified** APIs, the
load-bearing rules with code, a canonical example using generic names (`MyApp`,
`com.example.app`), a decision aid (when *not* to use it), cross-links, sources, and a
"Last verified" footer.

[`global-skills/apple/swift-liquid-glass-design-system-ios26/`](global-skills/apple/swift-liquid-glass-design-system-ios26/SKILL.md)
is the reference example for the format.

---

## Quality bar

Each skill meets every one of these:

- **Verified, not assumed.** Every API in a "canonical APIs" table was confirmed against the
  vendor's live docs. That verification caught more than a dozen errors during authoring: wrong
  version numbers, renamed artifacts, a nonexistent crypto symbol (`CryptoKit.timingSafeEqual`),
  and an API URL that returned 404.
- **At least two cited sources**, one of them a primary vendor or standards doc, each with a
  resolving URL and a version.
- **Anti-sterilization.** Every body names at least one concrete public API, which anchors the
  pattern and gives the freshness audit a literal symbol to re-check.
- **One skill, one job.** No mega-skills.
- **Honest about canon.** Where a domain is still emerging (Claude Code / MCP), skills mark which
  patterns are documented-canonical versus opinionated convention.
- **Zero confidential context.** No client or employer identifiers anywhere; these are generic
  patterns, not an extract of any private codebase.

---

## Using these skills

Drop a skill folder into a project's (or your user-level) `.claude/skills/` directory. Claude Code
auto-invokes it when its `description` matches the situation, or you can invoke it explicitly with
`/<skill-name>`. The `freshness` and `version` keys are inert to the loader (it reads only `name`
and `description`) — they exist for humans and for the audit tooling here.

To check whether a skill has drifted before you rely on it:

```
/skill-pattern-freshness-audit <domain>     # e.g. apple, aws-go, android
```

---

## Layout

```
ai-native-engineering-workspace/
├── README.md                  ← you are here
├── LICENSE                    ← MIT
├── workflow-docs/             ← 13 generic platform-base docs + PLATFORM_BASE.md
├── global-skills/
│   ├── FRESHNESS_SPEC.md       ← the provenance contract every skill follows
│   ├── meta/                   ← 3 skills about skills (extract / audit / update)
│   ├── apple/                  ← 13
│   ├── apple-auth/             ← 6
│   ├── aws-go/                 ← 8
│   ├── android/                ← 5
│   └── claude-code-workflow/   ← 6
└── reference-projects/         ← sanitized iOS + Android architecture docs
```

---

## Notes

- **Bilingual.** This README and the skills are in English; some platform-base workflow docs keep
  their original Spanish body. Each skill's `description` (the part Claude reads to decide whether
  to fire) is English.
- **License:** MIT — reuse freely.
- **Provenance dates** throughout reflect when each pattern was last verified; run the freshness
  audit to re-confirm against current docs.
