---
name: dossier-driven-skill-update
description: >-
  Repair a technical skill that a freshness audit flagged STALE or SUPERSEDED. Re-runs a
  focused research dossier against current official docs for just the drifted APIs, rewrites
  the affected sections of the SKILL.md body (preserving structure and voice), refreshes the
  freshness frontmatter and citations, and resets status to current. The end-to-end
  detect→research→rewrite loop that closes the maintenance cycle. Use after
  skill-pattern-freshness-audit reports non-current skills.
version: "1.0.0"
disable-model-invocation: true
freshness:
  verified_against:
    - source: "AWS — cdk drift command reference (worked example of a pattern that drifted)"
      url: "https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html"
      version: "CDK CLI 2.1110.0"
    - source: "Android — AndroidX Hilt release notes (worked example: hiltViewModel artifact rename)"
      url: "https://developer.android.com/jetpack/androidx/releases/hilt"
      version: "Hilt 1.3.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "FRESHNESS_SPEC.md v2, or Claude Code skill-format change"
    or_date: "2026-12-03"
  decay_risk: low
  status: current
---

# Dossier-Driven Skill Update

`skill-pattern-freshness-audit` *detects* drift; this skill *repairs* it. It takes a single
skill flagged `stale` or `superseded`, runs a tight research pass against current docs for
only the APIs that drifted, rewrites the affected sections, refreshes the provenance, and
resets the status. It is the body-rewriting counterpart that the audit deliberately does not
do (so detection can stay cheap and frequent).

`disable-model-invocation: true` — this skill **writes to a published skill file**, so it
only runs on explicit `/dossier-driven-skill-update <skill-path>`, never automatically.

## When to invoke

- After `/skill-pattern-freshness-audit <domain>` reports one or more `stale` / `superseded`
  skills and you've decided which warrant a real refresh (not every drift needs a rewrite — a
  trivial rename can be a one-line note).
- When a platform release breaks a pattern you depend on and you want the skill corrected, not
  just flagged.
- `/dossier-driven-skill-update global-skills/android/compose-feature-scaffold`

**Announce on invoke:** "Using `dossier-driven-skill-update` to re-research and rewrite the drifted sections of `<skill>`."

## Worked example (why this exists)

The audit found `compose-feature-scaffold` was `stale`: it wrote
`androidx.hilt:hilt-navigation-compose` for a non-navigation `hiltViewModel()`, but Hilt 1.3.0
([release notes](https://developer.android.com/jetpack/androidx/releases/hilt)) moved that API
to `androidx.hilt:hilt-lifecycle-viewmodel-compose`. This skill's job: confirm the rename
against the release notes, rewrite the dependency block and the example, add a migration note,
update the `freshness.verified_against` Hilt version, and flip `status` back to `current` — all
without disturbing the rest of the well-verified scaffold.

## Methodology

### Step 1 — Scope the damage

Read the audit report (or run a single-skill audit). Identify the **specific APIs/strings**
that drifted and which sections of the body reference them. Do **not** rewrite the whole skill —
a freshness drift is usually localized (one dependency, one renamed call, one superseded tool).
Whole-body rewrites risk regressing the parts that were correct.

### Step 2 — Focused dossier (research only the drift)

For each drifted API, verify the replacement against **primary vendor docs + release notes**:

- The old API's current doc page (is there a deprecation banner? a "renamed to…" note?).
- The release notes for the window between the skill's `verified_on` and today.
- The replacement API's doc page (confirm the new canonical form, its version gate, and any
  migration caveats — e.g., "DataStore is not encrypted by default" when moving off
  `EncryptedSharedPreferences`).

Capture each finding with a citation URL. This mini-dossier is narrower than a full
`skill-extraction-pattern` step-3 research; it targets only what changed.

### Step 3 — Rewrite the affected sections (preserve the rest)

Edit only the sections that reference drifted APIs:

- Replace the stale API/artifact/command with the current canonical one.
- Update the concrete example(s) so they compile against the new API.
- Add a short **migration note** if readers may have the old form in their code (especially for
  `superseded`: "If you still use `EncryptedSharedPreferences`, migrate to DataStore + Tink —
  Jetpack Security 1.1.0 is its terminal release").
- Keep the skill's structure, voice, and the correct sections untouched.

Match the surrounding code's idiom and the doc's existing density — the goal is a seamless
patch, not a visibly bolted-on correction.

### Step 4 — Refresh provenance

Update the `freshness` frontmatter:

- Add/replace `verified_against` entries with the new citations (and bump their `version`).
- Set `verified_on` to today.
- Recompute `recheck_after` per the domain cadence in `FRESHNESS_SPEC.md`.
- Reset `status` to `current`.
- Re-assess `decay_risk` (a surface that just broke may deserve bumping `medium` → `high`).

Update the body's "Last verified" footer to match.

### Step 5 — Commit with a traceable message

Use a convention that records what drifted and against which version, so the history is an
audit trail:

```
chore(skills): refresh compose-feature-scaffold against Hilt 1.3.0

hiltViewModel() artifact moved hilt-navigation-compose →
hilt-lifecycle-viewmodel-compose (non-nav use). Updated dep block,
example, migration note. freshness.status stale → current.
```

### Step 6 — Re-audit to confirm

Run `/skill-pattern-freshness-audit <domain>` on just the repaired skill to confirm it now
reports `current`. If it still flags drift, the rewrite missed a reference — return to step 1.

## Rules

1. **Localized patch, not full rewrite.** Touch only the drifted sections; preserve everything
   the audit didn't flag.
2. **Every change carries a citation.** The new canonical API must be backed by a vendor doc /
   release note URL, recorded in `verified_against`.
3. **`superseded` needs a migration note.** Readers may have the old pattern in production;
   tell them how to move, and flag traps (e.g., DataStore not being encrypted by default).
4. **Refresh provenance, always.** A body fix without a frontmatter `verified_on` bump leaves
   the skill lying about when it was checked.
5. **Re-audit before done.** The loop only closes when the audit agrees the skill is `current`.
6. **Explicit invocation only** (`disable-model-invocation: true`) — this writes to published
   files.

## Relationship to the other meta-skills

```
skill-extraction-pattern        →  creates a skill (from private → public)
            │
            ▼
skill-pattern-freshness-audit   →  detects when it has drifted (cheap, frequent, read-only)
            │  (flags stale/superseded)
            ▼
dossier-driven-skill-update     →  repairs it (research + rewrite + re-verify)  ← you are here
            │
            └──────────────────────►  back to audit: confirm current
```

Together they form a closed lifecycle: **create → monitor → repair → re-monitor.** Extraction
is the birth; audit is the checkup; this skill is the treatment.

## Related skills

- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — the detector that feeds this.
- `global-skills/meta/skill-extraction-pattern/SKILL.md` — full research pipeline; this skill
  is its narrowed, repair-focused re-application (steps 3–4 only).
- `global-skills/FRESHNESS_SPEC.md` — the frontmatter contract this skill refreshes.

---

**Last verified:** 2026-06-03 against the AWS `cdk drift` docs and Android Hilt 1.3.0 release
notes (both used as worked drift-repair examples).
**Re-check after:** `FRESHNESS_SPEC.md` v2 or a Claude Code skill-format change, or by 2026-12-03.
**Decay risk:** low.
