---
name: skill-extraction-pattern
description: >-
  Extract public, reusable Agent Skills from a corpus of private, NDA-locked project skills.
  Seven-step pipeline — inventory, classify, research-verify, rewrite-from-scratch,
  re-classify, anchor-with-a-concrete-example, self-applicability check — that strips the
  secret architecture and business context while keeping the reusable pattern. Use when you
  want to publish (open-source, portfolio, share) skills you built inside a confidential
  codebase without leaking the client's stack.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — Agent Skills best practices (one skill / one job; description is the trigger)"
      url: "https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices"
      version: "2026-06"
    - source: "Anthropic — Equipping agents for the real world with Agent Skills"
      url: "https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills"
      version: "2026-06"
    - source: "Gamma, Helm, Johnson, Vlissides — Design Patterns (1994), Ch. 1 (pattern extraction methodology)"
      url: "https://en.wikipedia.org/wiki/Design_Patterns"
      version: "GoF 1994"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code skill format change, or a revision of this methodology"
    or_date: "2026-12-03"
  decay_risk: low
  status: current
---

# Skill Extraction Pattern

You built valuable Agent Skills inside a private codebase. The *patterns* in them are generic
and worth sharing; the *context* (real ARNs, account IDs, customer names, internal
architecture, a specific GraphQL schema) is confidential and must never leave. This skill is
the disciplined pipeline for separating the two — turning NDA-locked project skills into
public, reusable ones **without ever pasting source text into the output**.

It is **self-applicable**: this skill was itself produced by applying this pipeline to a
corpus of private skills. The methodology that extracts skills must be able to extract a
future version of itself (step 7).

## When to invoke

- You want to open-source / publish / put in a portfolio the skills you wrote inside a
  confidential client or employer codebase.
- You're starting a public "engineering workspace" repo and need to seed it from private work.
- A reviewer / hiring manager / client asks to see *how* you work, and your best evidence is
  locked behind an NDA.
- `/skill-extraction-pattern <source-dir> <target-dir>` to run the inventory step over a
  skills directory.

**Announce on invoke:** "Using `skill-extraction-pattern` to separate the reusable pattern from the confidential context in `<source>`."

## Precedent (this is not new)

Pattern extraction from concrete systems is a 30-year-old discipline:

- **Gang of Four (1994)** extracted 23 design patterns from concrete C++/Smalltalk frameworks
  (ET++, MacApp, HotDraw) by stripping framework-specific detail and naming the recurring
  structure. Their template — Intent, Motivation, Applicability, Structure, Consequences — is
  exactly what a good generic SKILL.md needs.
- **Anthropic's own skill guidance** reinforces the target shape: it emphasizes single-purpose
  skills and warns against over-generic descriptions that never fire. Those two principles drive
  the two hardest decisions in extraction (when to **split** a mega-skill, and how to keep the
  **trigger specific** while removing the **project specificity**).

## The pipeline (7 steps)

This is the methodology, refined past the naive linear "inventory → research → rewrite" by an
explicit methodology audit (see the four critiques below).

### 1. Inventory — read frontmatter only

List every source skill. **Read only the `name` + `description`**, never the body yet. Reading
bodies early biases you toward copying; the goal is to first understand the *shape* of the
corpus. Output a flat list.

### 2. Classify — with an effort + risk matrix

Label each skill. The matrix carries effort and leak-risk so a later writing phase can be
budgeted:

```
| origin_name             | classification        | generic_name                    | effort_min | risk_of_leak | research_depth |
|-------------------------|-----------------------|---------------------------------|------------|--------------|----------------|
| lazy-init-segregated…   | partially-extractable | lambda-cold-start-segregation   | 45         | low          | medium         |
| mongo-ttl-canonical     | extractable           | mongo-ttl-bson-tag-canonical    | 30         | low          | low            |
| reviewing-<person>      | not-extractable       | (personal — stays private)      | 0          | high         | -              |
```

The three classes:

- **`extractable`** — the pattern is fully generic; the project is mere illustration. Change a
  couple of example names and it's universal.
- **`partially-extractable`** — generic core, but examples are welded to the project's stack.
  Requires a real rewrite of the examples, not a find/replace.
- **`not-extractable`** — the *value itself* is confidential data: a person's PR history, a map
  of real account IDs, a list of internal helpers. There's no generic pattern under it. Leave
  it private. (Be honest here — the temptation is to over-classify as extractable.)

### 3. Research-verify — against official sources, never the source skill

For each `extractable` / `partially-extractable` skill, verify the pattern against **primary
vendor docs + standards + community canon** — *not* by re-reading the private skill. This
research dossier becomes the scaffolding for the rewrite. Anthropic's docs tell you *what*
Claude Code can do; add a **community-canon check** (engineering blogs, reputable newsletters)
for *which combinations work in production* — that's where emerging-canonical patterns live
before they hit official docs.

This step is also where freshness is established: the citations you gather here become the
skill's `freshness.verified_against` block (see `FRESHNESS_SPEC.md`).

### 4. Rewrite from scratch — never copy-paste

Write the generic SKILL.md fresh, using the dossier as scaffolding. **Never paste source text
into the output** — that's how real ARNs, hostnames, and customer names leak. Search the source
skill for the principle, re-derive it, and write the body in your own words against the public
citations.

### 5. Re-classify — extraction reveals mistakes

Real extraction discovers, mid-rewrite, that step 2 was wrong: a "fully extractable" skill
turns out to lean on a non-extractable cousin, or a "single" skill is secretly three jobs that
should be **split** ("one skill, one job"). Loop back and fix the classification. The pipeline
is iterative, not linear.

### 6. Anchor with a concrete example

Over-genericizing is the dominant failure mode — a skill that says "use the appropriate
modifier" never fires and can't be freshness-audited. Require **at least one fully concrete
example** in the body naming a real public API/library/tool (`.glassEffect()`, `cdk drift`,
`runTest`, `hilt-lifecycle-viewmodel-compose`). The concrete example anchors the trigger and
gives auditors a literal string to check; the prose around it generalizes.

### 7. Self-applicability check

Confirm the result could be produced *by this pipeline applied to a future version of itself*.
If `skill-extraction-pattern` couldn't extract a future skill-extraction skill, the methodology
is incomplete.

## Extraction anti-patterns (the failure modes)

1. **Over-genericizing kills the skill.** Keep *trigger specificity* (specific situation,
   specific signals) while removing *project specificity*. "Use when refactoring code" is so
   vague Claude never fires it. "Use when writing Next.js Server Actions that authenticate with
   an OIDC provider" still fires and names nothing confidential.
2. **Mega-skill leak.** A source skill bundling three concerns (Go conventions + AWS SDK init +
   cold-start analysis) should be **split** during extraction, not preserved whole.
3. **Copy-paste leaks secrets.** Project skills contain real ARNs, account IDs, internal
   hostnames, customer names. Rewrite-from-scratch is the *only* safe path. If you ever paste,
   you've already lost.
4. **Losing the war story.** A project skill's strength is often one concrete failure that
   motivated it. The generic version loses the specific incident but must keep a *generic*
   example carrying the same insight — this is GoF's "Motivation" section. ("When a Lambda has
   N clients with disjoint use paths, segregate `sync.Once` by path" preserves the lesson
   without the incident.)
5. **Self-recursion blind spot.** The meta-skill must apply to itself (step 7).

## Decision aid: extractable vs not

Ask, in order:

1. **Is the value the data, or the pattern?** If removing every proper noun leaves nothing
   useful → `not-extractable`. (A map of real Route53 zones is data; a rule about how to
   choose a Route53 routing policy is a pattern.)
2. **Does the trigger survive de-branding?** If the description only makes sense with the
   client's product names → rewrite the trigger as a *class of situation*, or it's not
   extractable.
3. **Is it one job or several?** Several → split before extracting.
4. **Can the examples be re-derived from public docs?** No (they depend on a private schema /
   internal endpoint) → `partially-extractable` at best; rewrite examples generically.

## Optional runnable inventory

The skill may ship a small `scripts/inventory.sh` that reads frontmatter from every
`SKILL.md` under a directory and emits the matrix YAML skeleton — automating step 1 so the
human starts from a populated table rather than a blank one.

## Related skills

- `global-skills/FRESHNESS_SPEC.md` — step 3's citations populate this block.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — keeps extracted skills honest
  over time (the maintenance counterpart to this creation pipeline).
- `global-skills/meta/dossier-driven-skill-update/SKILL.md` — re-runs steps 3–4 when an audit
  flags a skill `stale`/`superseded`.

---

**Last verified:** 2026-06-03 against Anthropic Agent Skills best-practices + the GoF (1994)
pattern-extraction methodology.
**Re-check after:** any Claude Code skill-format change, or by 2026-12-03. **Decay risk:** low.
**Improve the methodology?** This skill is self-applicable — extract a better version and
open a PR.
