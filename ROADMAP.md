# Roadmap

What this repo is, and where it's going. Current state: **42 skills + 13 platform-base docs**, all
verified against live vendor docs, 0 confidential identifiers.

## Now (shipped)

- ✅ 13 generic platform-base workflow docs (`workflow-docs/`)
- ✅ 42 Agent Skills across 7 domains (`global-skills/`)
- ✅ The freshness framework — `FRESHNESS_SPEC.md` + the 3 meta-skills (extract / audit / repair)
- ✅ Per-domain navigation READMEs + a coverage matrix
- ✅ Sanitized iOS + Android reference-project architecture docs

## Next

- **Keep skills current.** Run `/skill-pattern-freshness-audit <domain>` on the documented cadence
  (Apple/Android post-conference, AWS quarterly, Claude Code monthly) and repair drift with
  `dossier-driven-skill-update`. This is the repo's standing maintenance loop, not a one-off.
- **More base docs as extensions accrue.** Additional `workflow-docs/*.md` as new platform domains
  come into regular use.
- **Deepen the iOS reference project.** More sanitized architecture docs (module structure, auth
  module) from the iOS reference app.

## Considered / later

- **Runtime AI agents in Go.** A separate track exploring agent runtimes (e.g. Google ADK Go,
  langgraphgo, langsmithgo) — distinct from the *declarative* Claude Code skills here. Those are
  design-time guidance; runtime agents execute. If pursued, they'd live in their own repo and be
  referenced from here.
- **Extractable open-source tools.** The freshness framework (`FRESHNESS_SPEC.md` + audit/update
  skills) and a `claude-code-skills-template` could be packaged as standalone, installable tools.

## Non-goals

- **Not a tutorial site.** These are working patterns with verified provenance, not a learning
  course.
- **Not a leak surface.** Skills that can't be generalized without exposing a private codebase stay
  private — see the coverage matrix in [`global-skills/README.md`](global-skills/README.md). 9 source
  skills were deliberately not published for exactly this reason.

---

**Verification dates** throughout the repo reflect when each pattern was last checked. The roadmap's
most important recurring item is simply: keep them true.
