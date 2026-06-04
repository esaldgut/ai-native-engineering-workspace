---
name: skill-pattern-freshness-audit
description: >-
  Detect (don't repair) drift between a published technical skill and the current state of the
  official SDK/framework/spec it documents. Reads the skill's freshness frontmatter, re-verifies
  each cited API against live vendor docs, and sets each skill's status to current / needs-recheck
  / stale / superseded with the citation that proves it. Use quarterly per domain, after a major
  platform release (new iOS / Compose BOM / AWS SDK major / Claude Code minor), or before relying
  on a skill past its recheck date. Repair of a flagged skill is handled by dossier-driven-skill-update.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — Agent Skills best practices"
      url: "https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices"
      version: "2026-06"
    - source: "AWS CDK — cdk drift command reference (canonical drift-detection example)"
      url: "https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html"
      version: "CDK CLI 2.1017.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code skill-frontmatter schema change, or FRESHNESS_SPEC.md v2"
    or_date: "2026-12-03"
  decay_risk: low
  status: current
---

# Skill Pattern Freshness Audit

Technical patterns for fast-moving platforms rot in months. This skill turns *rot* into a
*queryable signal*: it walks a directory of skills, reads each one's `freshness` block (see
`global-skills/FRESHNESS_SPEC.md`), re-verifies the cited APIs against current official
documentation, and reports drift with citations — so a stale recommendation is caught here,
not in someone's production code.

## When to invoke

**Auto-invoke when any of these triggers fire:**

1. A major platform release lands for a domain you publish skills in: a new **iOS major**
   (post-WWDC), a new **Compose BOM major** (post-Google I/O), an **AWS SDK Go v2 major** or
   **CDK CLI major**, a **Claude Code minor** release.
2. A skill's `freshness.recheck_after.or_date` has passed (run `/skill-pattern-freshness-audit <domain>`).
3. Before quoting or relying on a skill whose `freshness.status` is `needs-recheck`, `stale`,
   or `superseded`.
4. Quarterly housekeeping per domain, even with no release — the cheapest insurance against
   silent deprecations.

**Announce on invoke:** "Using `skill-pattern-freshness-audit` to re-verify `<domain>` skills against current docs."

## Why this is necessary

`global-skills/FRESHNESS_SPEC.md` documents five real drift cases found in a single verification
pass — `cdk drift` superseding a manual idiom, `EncryptedSharedPreferences` deprecated,
`hiltViewModel()` artifact renamed, `runBlockingTest` → `runTest`, and the mythical
`CryptoKit.timingSafeEqual`. Each is the kind of silent rot this audit exists to catch *as a
status flag* before a reader copies it into production. (See the spec's table for the cited
sources; this skill doesn't restate it.)

## Inputs

```
/skill-pattern-freshness-audit <domain> [--fix] [--strict]
```

- `<domain>` — one of `apple`, `apple-auth`, `aws-go`, `android`, `claude-code-workflow`,
  `meta`, or `all`. Maps to `global-skills/<domain>/`. (At this repo's current stage only `meta/`
  is populated; the other domains are populated by the skill-extraction phase. Auditing an empty
  domain returns an empty report, which is correct.)
- `--fix` — after reporting, update each skill's `freshness.status` field in place, and append a
  dated audit line to the body footer. It never rewrites the body's *instructions* — that's the
  job of `dossier-driven-skill-update`. So this skill writes status + a footer line only; it is
  not a body editor.
- `--strict` — treat any unresolvable citation URL as a failure (default: warn).

## Methodology

### Step 0 — Lint the freshness block (fast, no network)

For each `SKILL.md` in the domain, validate the block against `FRESHNESS_SPEC.md` rules 1–6:
≥2 `verified_against` entries, ≥1 primary vendor source, resolving URLs, `or_date >
verified_on`, valid enums, and ≥1 concrete public API string in the body (anti-sterilization).
A skill that fails the lint is reported `INVALID` and skipped from drift detection (you can't
audit a pattern with no provenance).

### Step 1 — Window check (no network)

Compare today's date and any known platform releases against each skill's `recheck_after`:

- Past `or_date` → mark `needs-recheck`.
- `trigger` keyword matches a release you know happened (e.g., trigger says "WWDC26" and it's
  July 2026) → mark `needs-recheck`.
- Otherwise → eligible to stay `current`, pending drift detection.

### Step 2 — Drift detection (network; this is the core)

For each skill flagged `needs-recheck` (or all, if a release just dropped), re-verify the
**concrete API strings** the body names. For each one:

1. Fetch the cited `verified_against[].url` (WebFetch). If it 404s or redirects to a "this
   has moved / deprecated" page → strong drift signal.
2. Search the vendor's current docs for the literal API string (WebSearch:
   `site:developer.apple.com "<API>"`, `site:developer.android.com "<artifact>"`,
   `site:docs.aws.amazon.com "<command>"`, `site:code.claude.com "<feature>"`).
3. Classify the result:

| Finding | Status to set |
|---|---|
| API still documented at cited version or later, no deprecation banner | `current` |
| API documented but with a **deprecation/renamed** notice, or a newer canonical tool exists | `stale` |
| API absent / replaced / never existed | `superseded` |

Cross-check against the platform's release notes for the window between `verified_on` and
today (Apple "what's new", Android "AndroidX releases", AWS SDK Go v2 `CHANGELOG`, Claude Code
release notes). A rename in release notes that the skill body doesn't mention = `stale`.

### Step 3 — Report

Emit a table, one row per skill (the skill names below are illustrative of an `android` domain
once populated):

```
DOMAIN: android   (audited 2026-09-15)

skill                              status       evidence
─────────────────────────────────────────────────────────────────────────────
compose-clean-architecture-module  current      collectAsStateWithLifecycle still canonical
compose-feature-scaffold            stale        hilt-navigation-compose → hilt-lifecycle-viewmodel-compose (Hilt 1.3.0)
android-security-checklist          superseded   EncryptedSharedPreferences deprecated (Jetpack Security 1.1.0 terminal)
android-testing-patterns            stale        runBlockingTest → runTest (kotlinx 1.6+); Turbine API unchanged
kmp-shared-extraction               current      CMP iOS Stable since 1.8.0; expect/actual unchanged
─────────────────────────────────────────────────────────────────────────────
2 current · 2 stale · 1 superseded · 0 invalid

Next action: run /dossier-driven-skill-update android for the 3 non-current skills.
```

For each non-`current` skill, include: the exact API that drifted, the replacement, and the
citation URL proving it. That citation is what makes the report actionable instead of alarmist.

### Step 4 — Optionally update status (`--fix`)

With `--fix`, write the new `status` value back into each skill's `freshness` frontmatter and
append a dated line to the body footer (e.g., `Audit 2026-09-15: hiltViewModel artifact
renamed — see issue`). Never touch the body's instructions; that is a separate, gated step
(`dossier-driven-skill-update`), because rewriting guidance needs a fresh research dossier.

## Rules

1. **Read frontmatter + body, fetch live docs — never trust the skill's own claim.** The whole
   point is to catch the skill being wrong.
2. **Every status change carries a citation.** A `stale`/`superseded` flag without a URL is a
   bug; downgrade to `needs-recheck` and ask a human.
3. **Primary sources outrank blogs.** A Medium post saying "X is deprecated" is a lead, not a
   verdict; confirm on the vendor's docs or release notes before flagging `superseded`.
4. **Don't rewrite bodies here.** This skill *detects*; `dossier-driven-skill-update` *repairs*.
   Keeping them separate means detection can run cheaply and often.
5. **Asynchronous-friendly.** Drift detection over a whole repo is slow (many fetches); run it
   per domain, and schedule it (see below) rather than blocking interactive work.

## Automation (recommended)

Schedule the audit per domain on the cadence in `FRESHNESS_SPEC.md` (Apple/Android twice a
year post-conference, AWS quarterly, Claude Code monthly). In Claude Code this can be a
`schedule` routine or a cron-driven headless run:

```
# Quarterly AWS audit, monthly Claude Code audit
/skill-pattern-freshness-audit aws-go --fix
/skill-pattern-freshness-audit claude-code-workflow --fix
```

The `--fix` keeps `status` honest automatically; a human reviews the report and decides which
`stale`/`superseded` skills warrant a full body refresh.

## Related skills

- `global-skills/FRESHNESS_SPEC.md` — the frontmatter contract this skill reads and lints.
- `global-skills/meta/dossier-driven-skill-update/SKILL.md` — repairs the skills this audit
  flags (re-researches + rewrites the body, then resets `status` to `current`).
- `global-skills/meta/skill-extraction-pattern/SKILL.md` — the methodology that produced the
  audited skills in the first place; freshness is its maintenance counterpart.

---

**Last verified:** 2026-06-03 against Anthropic Agent Skills best practices + the AWS `cdk drift`
docs (the canonical example of a documented pattern superseded by a new first-class command).
**Re-check after:** any Claude Code skill-frontmatter schema change, or by 2026-12-03. **Decay risk:** low.
**Found a drift in this skill itself?** Open an issue or re-run the audit on `meta`.
