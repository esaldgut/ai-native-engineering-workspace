---
name: claude-mcp-lesson-capture-pipeline
description: >-
  Capture lessons from a merged PR and propagate them in a cascade — auto-memory, then agent
  skills, then human docs — so hard-won knowledge (compiler quirks, security findings, framework
  bugs) is never re-learned. Built on Claude Code's auto memory, a Stop hook (exit 0 +
  decision:block to continue), and a side-effectful skill marked disable-model-invocation so it
  runs only on /capture-lessons. Use after a merge to a long-lived branch, when a PR title matches
  chore(lessons): capture PR#N, or when asked to record lessons from a diff. This is the generic
  cross-platform pattern; platform-specific variants (e.g. an Android cascade) exist separately.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — How Claude remembers your project (auto memory + @import + MEMORY.md path)"
      url: "https://code.claude.com/docs/en/memory"
      version: "Claude Code 2026-06 (auto memory v2.1.59+)"
    - source: "Anthropic — Automate workflows with hooks (Stop hook exit codes + decision/continue)"
      url: "https://code.claude.com/docs/en/hooks"
      version: "Claude Code 2026-06"
    - source: "Anthropic — Extend Claude with skills (disable-model-invocation semantics)"
      url: "https://code.claude.com/docs/en/skills"
      version: "Claude Code 2026-06"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code minor release (hooks / skills / memory schema change)"
    or_date: "2026-09-03"
  decay_risk: high
  status: current
---

# claude-mcp-lesson-capture-pipeline

A post-merge automation that turns one merged PR into durable, reusable knowledge. It composes
three things Claude Code formalizes — **auto memory**, a **`Stop` hook**, and a skill with
**`disable-model-invocation`** — into a single cascade: lessons flow first to project auto memory,
then to the relevant agent skills, then to human-facing docs.

The lesson-capture-after-merge composition is **emerging community practice**, not a named
Anthropic feature. The three primitives it stands on are fully canonical and verified below; the
*orchestration* (and the auto-unlock gate in Step 6) is opinionated. Both are flagged where they
appear.

## When to invoke

- Right after a merge to the long-lived branch (e.g. `main`/`develop`).
- A `Stop` hook detected the latest commit subject matches `chore(lessons): capture PR#<N>`.
- The user runs `/capture-lessons 42` (or asks to "record the lessons from PR 42").

This skill has side effects (it writes to memory, skills, and docs), so it is marked
`disable-model-invocation: true` — Claude will **not** auto-fire it; you trigger it with
`/capture-lessons`. Per the [skills docs](https://code.claude.com/docs/en/skills), that flag keeps
the skill out of Claude's auto-discovery context and reserves it for `/name` invocation, exactly
the intended pattern for `/commit`-/`/deploy`-class operations.

**Announce on invoke:** "Using `claude-mcp-lesson-capture-pipeline` to extract and cascade lessons from the merged PR."

## The canonical primitives (verified Claude Code 2026-06)

| Primitive | Verified fact | Source |
|---|---|---|
| Auto memory | Claude writes learnings to `~/.claude/projects/<project>/memory/MEMORY.md`; first **200 lines or 25KB** load each session; topic files alongside load **on demand**. Requires v2.1.59+. | [memory](https://code.claude.com/docs/en/memory) |
| `Stop` hook | Fires when Claude finishes a turn. Configured under `"hooks": { "Stop": [...] }` in `.claude/settings.json`. | [hooks](https://code.claude.com/docs/en/hooks) |
| `disable-model-invocation` | `true` hides the skill from auto-discovery and runs it only on `/name`. | [skills](https://code.claude.com/docs/en/skills) |
| `@import` | `@path` lines in `MEMORY.md`/`CLAUDE.md` pull other files in; max **4** hops. | [memory](https://code.claude.com/docs/en/memory) |

## The Stop-hook trigger — get the exit code right (load-bearing)

The single most common way to break this is the hook's exit semantics. Verified against the
[hooks reference](https://code.claude.com/docs/en/hooks):

- **Exit 0** = success. **stdout is parsed as JSON only on exit 0.**
- **Exit 2** = blocking error. Claude Code **ignores stdout/JSON** and feeds **stderr** back instead.
- To make the `Stop` hook keep Claude working, emit **exit 0 with `{"decision":"block","reason":"…"}`**.
  `decision:block` is what prevents the stop and continues the turn.
- Do **not** use `{"continue": true}` for this — `continue` defaults to `true` and does nothing to
  force continuation; its only active value is `continue:false`, which *stops* Claude entirely.
  (This corrects a common write-up that pairs `continue:true` with a reason.)

A `Stop` hook fires on **every** turn, not just after a merge — so the command must gate on the
commit subject and stay silent otherwise. `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "test \"$(git -C \"$CLAUDE_PROJECT_DIR\" log -1 --format=%s 2>/dev/null | grep -cE '^chore\\(lessons\\): capture PR#[0-9]+')\" = 1 && printf '{\"decision\":\"block\",\"reason\":\"Run /capture-lessons for the last commit, then stop.\"}' || true"
          }
        ]
      }
    ]
  }
}
```

`CLAUDE_PROJECT_DIR` is the documented project-root variable Claude Code exports to hook processes.
Because `disable-model-invocation` blocks Claude's *own* auto-discovery, the hook can't silently
auto-run the skill — it nudges Claude (via the `reason`) to invoke `/capture-lessons`, which is the
honest, supported path. If you'd rather decouple from Claude's lifecycle entirely, a git
`hooks/post-merge` script is a robust alternative trigger that runs outside Claude Code.

## Workflow — the cascade

The cascade propagates in one order: **cheapest-to-restate first, most-durable last** — memory →
skills → docs.

1. **Read the gate state** (the Step-6 file below).
2. **Gate (opinionated):** if `successful_runs < 5` and `--auto` was not passed, summarize the
   proposed writes and ask for confirmation before touching any file.
3. **Fetch** the PR with `gh pr view "$ARGUMENTS" --json title,body,files,comments,mergeCommit` and
   `gh pr diff "$ARGUMENTS"`.
4. **Distill** lessons: anti-patterns hit, canonical refinements, framework/compiler bugs, security
   findings.
5. **Cascade**, each edit gated by Step 2:
   - **Auto memory** — append a one-line entry to `MEMORY.md` (the index) and create/update a topic
     file beside it (e.g. `build-quirks.md`, `security-findings.md`). Anthropic's docs name topic
     files plainly (`debugging.md`, `api-conventions.md`).
     > **Opinionated, not canonical:** a strict prefix taxonomy (`feedback_*`, `project_*`,
     > `reference_*`, `user_*`) is a *convention some teams enforce*, not Claude Code spec — the
     > loader treats any `*.md` the same. Document it as a local rule if you adopt it.
   - **Agent skills** — patch the checklist/anti-pattern list of the relevant
     `.claude/skills/<skill>/SKILL.md` so the lesson fires *next time* that skill runs.
   - **Human docs** — append to `docs/lessons/` (or the changelog) for teammates who don't run Claude.
6. **Bump** `successful_runs += 1` in the state file.

## Step 6 — the auto-unlock gate (USER-INVENTED, not a Claude Code primitive)

There is **no** canonical Claude Code mechanism for "confirmation gates that decay." This is the
user's own pattern, implemented purely inside the skill as a state file:

```json
// ~/.claude/projects/<project>/memory/.capture-lessons-state.json
{ "successful_runs": 3 }
```

Rules that keep it safe:

- **Store it under the auto-memory dir**, which is **machine-local** by design. If you put it inside
  a committed project skill, it would be cloned with `successful_runs: 5` already set and skip the
  gate on every fresh checkout.
- `--auto` bypasses the gate immediately; otherwise the gate auto-unlocks after 5 successful runs.
- Treat unlock as advisory — keep the `decision:block` reason honest so a human stays in the loop on
  what gets written.

## Pitfalls (verified)

- **`Stop` hook runs every turn.** Without the commit-subject gate it fires on every reply. Gate it.
- **Wrong exit code swallows your JSON.** Exit 2 makes Claude Code read stderr, not your
  `decision`/`reason`. Use exit 0 for control JSON.
- **`disable-model-invocation` ≠ "scriptable from a hook."** It removes the skill from Claude's
  context entirely; you still invoke it with `/capture-lessons`. (Plugin skills historically didn't
  honor the flag at all — keep this as a *user* or *project* skill.)
- **Taxonomy drift.** Don't present a `feedback_/project_/reference_` prefix scheme as Anthropic
  spec. It isn't.

## Related skills

- `global-skills/claude-code-workflow/capture-lessons-cascade-android/SKILL.md` — the Android-specific
  variant of this cascade (Kotlin/Compose lessons, Gradle quirks). This skill is the generic version.
- `global-skills/claude-code-workflow/cli-build-sync-pipeline/SKILL.md` — the pre-merge consistency
  pass; lessons captured here often originate from build/lint failures that pipeline surfaces.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-verifies the hook/skill/memory
  claims after each Claude Code minor release (this domain churns monthly).

## Sources

- [How Claude remembers your project (auto memory, MEMORY.md path, @import)](https://code.claude.com/docs/en/memory)
- [Automate workflows with hooks (Stop hook, exit codes, decision/continue, CLAUDE_PROJECT_DIR)](https://code.claude.com/docs/en/hooks)
- [Extend Claude with skills (disable-model-invocation, $ARGUMENTS)](https://code.claude.com/docs/en/skills)

---

**Last verified:** 2026-06-03 against Claude Code docs. Confirmed live: `Stop` hook parses JSON
**only on exit 0**; `decision:block`+`reason` (not `continue:true`) is the continuation pattern;
`disable-model-invocation:true` hides the skill from auto-discovery and runs it on `/name`; auto
memory lives at `~/.claude/projects/<project>/memory/MEMORY.md` (200 lines / 25KB at startup).
**Flagged as user-invented:** the auto-unlock-after-5 gate and the `feedback_/project_/reference_`
memory prefix taxonomy — both opinionated, neither canonical.
**Re-check after:** any Claude Code minor release, or by 2026-09-03. **Decay risk:** high.
**Found a drift?** Run `/skill-pattern-freshness-audit claude-code-workflow`.
