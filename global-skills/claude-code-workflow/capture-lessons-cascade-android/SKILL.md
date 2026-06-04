---
name: capture-lessons-cascade-android
description: >-
  After merging a PR in an Android Kotlin/Compose/KMP repo, extract the hard-won lesson and cascade it to
  three surfaces — the project memory file, the relevant agent SKILL.md, and a docs/lessons/ file — routed
  by Conventional-Commits PR-title scope (feat(compose), fix(r8), refactor(hilt), fix(kmp)). Specializes in
  Android-only failure categories: R8/ProGuard keep-rules (release-only crashes), Compose recomposition
  (key(), lambda allocation), Hilt scoping (@Singleton vs @ViewModelScoped), KSP/AGP mismatches, coroutine
  dispatcher misuse. Proposes diffs for human approval; never auto-commits. Run post-merge or via /capture-lessons.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Claude Code — Extend Claude with skills (SKILL.md format, directory layout)"
      url: "https://code.claude.com/docs/en/skills"
      version: "2026-06"
    - source: "Android — Enable app optimization with R8 / shrink code (keep rules for reflection)"
      url: "https://developer.android.com/studio/build/shrink-code"
      version: "AGP 8.x"
    - source: "Android — About keep rules (R8 cannot see reflection; rules in proguard-rules.pro)"
      url: "https://developer.android.com/topic/performance/app-optimization/keep-rules-overview"
      version: "AGP 8.x"
    - source: "Conventional Commits (PR-title scope routing)"
      url: "https://www.conventionalcommits.org/"
      version: "1.0.0"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code minor release"
    or_date: "2026-09-03"
  decay_risk: medium
  status: current
---

# capture-lessons-cascade-android — post-merge lesson propagation

After every merge, the knowledge earned (a release-only R8 crash, a Compose recomposition fix, a Hilt
scoping bug) should land where it will be reused — not be re-learned next quarter. This skill reads a
merged PR, classifies the lesson by its Conventional-Commits title scope, and **proposes** updates to
three surfaces for human approval. It is the Android twin of the iOS `capture-lessons` skill, adding the
Android-specific failure categories (R8, Compose, Hilt, KSP, coroutines).

## When to invoke

- Right after merging a PR to the integration branch (`main` / `develop`) in a Kotlin/Compose/KMP repo.
- Trigger phrases: "capture lessons", "post-merge review", or `/capture-lessons <PR>`. Omit the number to
  scan for the latest unprocessed merge.

**Announce on invoke:** "Using `capture-lessons-cascade-android` to extract the merged PR's lesson and
propose diffs to memory / the relevant SKILL.md / docs/lessons — Android quirks included. No
auto-commits."

## The three surfaces (route by scope, don't dump everything everywhere)

| Surface | Path | Update when |
|---|---|---|
| **Memory** | `~/.claude/projects/<project>/memory/MEMORY.md` | project-wide, recurring lessons (small, hot context) |
| **Agent skill** | `<project>/.claude/skills/<skill>/SKILL.md` | lesson tied to one skill's domain → its "Pitfalls" |
| **Project docs** | `<project>/docs/lessons/LXX-<slug>.md` | detailed, code-citing one-off (PR link, diff, root cause, prevention) |

## Workflow

1. `gh pr view <N> --json title,body,files,commits,reviews` to pull the merged PR.
2. **Route by PR-title scope** (Conventional Commits — keep the regex specific, pair `feat:` with a scope
   so it doesn't match everything):
   - `feat\(compose\):` / `fix\(compose\):` → propose update to the `compose-*` skills
   - `fix\(r8\)` / `fix\(proguard\)` → propose a `docs/lessons/` file + a memory note (release-only risk)
   - `refactor\(hilt\):` / `fix\(hilt\):` → propose update to the Hilt-touching skills
   - `fix\(kmp\):` → propose update to the `kmp-shared-extraction` skill
3. Generate a **unified diff** per proposed surface.
4. Show the diffs; **require explicit human approval** before writing any file.
5. After approval, write the files. **Never auto-commit** memory or skills.

## Android-specific lesson categories (the high-value ones)

- **R8 / ProGuard keep-rules — release-only crashes.** R8 cannot see reflection (`Class.getDeclaredMethod`,
  `getAnnotation`), so it may strip or rename reflectively-accessed members; the fix is a keep rule in
  `proguard-rules.pro`. This bites **kotlinx.serialization** models, **Retrofit/Ktor** model classes, and
  any **Compose preview / reflection** path. (Hilt/Dagger don't use runtime reflection, so they rarely
  need keep rules — be precise about which lib actually broke.) These lessons are uniquely valuable
  because they pass in `debug` and only crash in the `release` variant — so the cascade should ask: *"was
  the release variant tested?"* as a pre-merge gate whenever a reflection-using library changed.
- **Compose recomposition.** Lambda allocation in composition, missing `key()` in `items()`, wrong
  `remember`/`LaunchedEffect` keys. These rarely surface as test failures — they show up as jank or
  recomposition-count regressions, so capture *"did Layout Inspector show the recomposition counts?"* as
  evidence, not just "tests pass."
- **Hilt scoping.** `@Singleton` vs `@ViewModelScoped` vs unscoped — wrong scope causes either
  over-retention (memory leak) or premature reconstruction (lost state).
- **KSP / AGP / Gradle mismatches.** `Unsupported class file major version`, KSP-vs-KAPT incompatibilities,
  AGP↔Kotlin-compiler↔Compose-BOM version skew.
- **Coroutines.** `runBlocking` in production, missing `Dispatchers.IO` for filesystem/network, unscoped
  coroutines leaking past lifecycle.

## The rules (load-bearing)

1. **Memory is a budget, not a dumping ground.** It ships with every invocation. Keep it to project-wide
   recurring lessons (~tens of bullet links, not hundreds). Single-skill lessons go in that `SKILL.md`;
   deep one-offs go in `docs/lessons/`. Prune memory items once a lesson is codified as a skill rule.
2. **PR-title regex must be scoped.** Bare `feat:` matches everything and routes to the wrong skill. Use
   `feat\(<area>\):` matching the project's CONTRIBUTING.md scope vocabulary.
3. **Never auto-commit memory/skill changes.** The trust boundary is: Claude proposes a diff → human
   reviews → human commits. Auto-writing to memory or skills breaks it.
4. **R8 lessons trigger a release-build gate prompt.** Any PR touching a reflection-using library should be
   asked "release variant tested?" before merge — this skill surfaces that as part of the cascade.
5. **Enforce Conventional Commits in CI** if you rely on title routing — a malformed title silently
   misroutes the lesson.

## Example SKILL.md routing block

```markdown
## Routing (Conventional Commits scope -> surface)
- feat\(compose\):   -> .claude/skills/compose-*/SKILL.md  (Pitfalls)
- fix\(r8\):         -> docs/lessons/  + MEMORY.md note  (release-only)
- refactor\(hilt\):  -> .claude/skills/*hilt*/SKILL.md
- fix\(kmp\):        -> .claude/skills/kmp-shared-extraction/SKILL.md
- everything else    -> propose docs/lessons/ only; ask before touching memory
```

## Decision aids

- **Memory location:** `~/.claude/projects/<project>/memory/MEMORY.md` (per-user, per-project) vs
  `<repo>/.claude/MEMORY.md` (in-repo, shared). Default to **per-user** for personal lessons; use in-repo
  only when the whole team should inherit them.
- **Auto-detect last unprocessed merge?** Track processed PRs in a `.lessons-state.json` marker in the
  repo so the scan knows where it left off. Worth adding once you run this regularly.
- **Human-gating cadence:** require approval on every cascade until ~5 clean runs build trust, then
  consider a lighter touch — but memory/skill **writes** still go through review.

## Related skills

- `global-skills/claude-code-workflow/android-build-sync/SKILL.md` — the gate that surfaces many of these
  lessons (R8 only fires in `release`, which this build gate's `assembleDebug` does **not** cover — a
  lesson worth capturing).
- `global-skills/android/compose-clean-architecture-module-scaffold/SKILL.md` ·
  `global-skills/android/kmp-shared-extraction/SKILL.md` — common cascade targets for `feat(compose)` /
  `fix(kmp)` lessons.

## Sources

- [Claude Code — Extend Claude with skills](https://code.claude.com/docs/en/skills) ·
  [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [R8 / shrink your code](https://developer.android.com/studio/build/shrink-code) ·
  [About keep rules](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview) ·
  [Conventional Commits](https://www.conventionalcommits.org/)

---

**Last verified:** 2026-06-03 against code.claude.com (SKILL.md format) + developer.android.com (R8 keep
rules; reflection cannot be seen by R8) + Conventional Commits 1.0.0.
**Re-check after:** next Claude Code minor release, or by 2026-09-03. **Decay risk:** medium (Claude Code
skill format + Android tooling both evolve).
**Found a drift?** Run `/skill-pattern-freshness-audit claude-code-workflow`.
