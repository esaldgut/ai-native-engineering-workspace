# global-skills

41 reusable Claude Code Agent Skills, organized by domain. Each is a focused, single-job skill
with verified provenance (see [`FRESHNESS_SPEC.md`](FRESHNESS_SPEC.md)). Drop any folder into a
`.claude/skills/` directory to use it; invoke with `/<skill-name>` or let Claude auto-fire it.

## Domains

| Domain | Skills | Focus |
|--------|-------:|-------|
| [`meta/`](meta/) | 3 | Skills *about* skills — extract, audit, repair |
| [`apple/`](apple/) | 13 | SwiftUI iOS 26 — UI, architecture, on-device AI |
| [`apple-auth/`](apple-auth/) | 6 | iOS auth security, crypto, testing |
| [`aws-go/`](aws-go/) | 8 | Lambda Go, AWS SDK v2, CDK, MongoDB |
| [`android/`](android/) | 5 | Kotlin / Compose / KMP |
| [`claude-code-workflow/`](claude-code-workflow/) | 6 | Claude Code + MCP workflow automation |

Each domain folder has its own README listing its skills.

## The freshness contract

Every skill carries a `freshness` block: cited sources with versions, a verification date, a
re-check trigger, and a `status`. [`FRESHNESS_SPEC.md`](FRESHNESS_SPEC.md) defines the contract;
the `meta/skill-pattern-freshness-audit` skill enforces it.

```
meta/skill-extraction-pattern   →  creates a skill (private → public, generically)
        ↓
meta/skill-pattern-freshness-audit  →  detects drift vs current docs (read-only, cheap)
        ↓  flags stale / superseded
meta/dossier-driven-skill-update    →  repairs it (re-research + localized rewrite)
        ↓
        └──────────────────────────►  back to audit: confirm current
```

---

## Coverage matrix — what was extracted, and what wasn't

These 41 skills were extracted from a larger corpus of **51 audited source skills** (private,
NDA-locked, across iOS / Android / global tooling). The extraction methodology
([`meta/skill-extraction-pattern`](meta/skill-extraction-pattern/SKILL.md)) classified each as
`extractable`, `partially-extractable`, or `not-extractable`. This table is the honest accounting.

### Published (41)

All `extractable` and `partially-extractable` source skills were rewritten from scratch, verified
against live docs, and published across the 6 domains above. Coverage by source area:

| Source area | Audited | Published | Rate |
|-------------|--------:|----------:|-----:|
| iOS (Swift / SwiftUI) | 26 | 24 | 92% |
| Android (Kotlin / Compose) | 7 | 7 | 100% |
| Global tooling (AWS / MCP / workflow) | 18 | 10 | 56% |
| **Total** | **51** | **41** | **80%** |

(Some source skills mapped to more than one published skill, and a few cross-platform twins —
e.g. iOS `swift-module` ↔ Android `compose-clean-architecture-module-scaffold` — are published in
both domains, so the per-domain counts above sum to the 41 shipped.)

### Deliberately NOT published (the honest part)

9 source skills were classified `not-extractable` — their entire value is confidential data, not a
reusable pattern. Publishing them would either leak a private codebase or be useless without it. By
category:

| Category | Why it can't be a generic skill |
|----------|---------------------------------|
| Two PR-review skills tuned to a specific person | Encode an individual's review history, tone calibration, and identity — not a pattern |
| A Route53 hosted-zone map | The value *is* the real AWS account IDs and zone records |
| A web deploy-flags matrix | The value *is* the project's real feature-flag names and defaults |
| An E2E smoke protocol | Built around one project's specific CloudWatch / ECS topology |
| Two "no hardcoded strings / reuse-before-create" disciplines | Index 150+ internal helper paths of one repo |
| Two Cognito-federation auth skills | Inseparable from one project's specific Cognito backend + Lambda triggers |

The *generalizable* security/auth knowledge from the Cognito skills survives — provider-agnostically
— in [`apple-auth/`](apple-auth/). What stayed private is the project-specific wiring, not the
principle.

> This distinction is the point of [`meta/skill-extraction-pattern`](meta/skill-extraction-pattern/SKILL.md):
> a skill is publishable only when removing every proper noun still leaves something useful. Where it
> doesn't, the skill stays private — and saying so plainly is more credible than pretending the whole
> corpus generalized.

---

**Skills:** 41 · **Source corpus audited:** 51 · **NDA-safe:** 0 confidential identifiers across the repo.
