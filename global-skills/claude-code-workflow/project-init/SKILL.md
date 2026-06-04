---
name: project-init
description: >-
  Initialize or audit a project against a generic platform-base documentation set so that work
  never starts from an ambiguous context. Applies the rule "no project starts ambiguously": the
  project loads context from the generic platform base first, then extends it. Auto-invoke when a
  project's CLAUDE.md has no "Platform Base Context" block, when the user asks to initialize or
  bootstrap context, or when starting work in an unfamiliar repo. Generates/verifies the base
  context block in CLAUDE.md (via @import) and suggests per-domain extension docs.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — How Claude remembers your project (CLAUDE.md scopes + @import)"
      url: "https://code.claude.com/docs/en/memory"
      version: "2026-06"
    - source: "Anthropic — Agent Skills best practices"
      url: "https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices"
      version: "2026-06"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code CLAUDE.md / memory schema change (e.g., @import behavior)"
    or_date: "2026-12-03"
  decay_risk: medium
  status: current
---

# project-init — Platform Base Context Initialization

Implements one rule: **a project must begin with explicit context — never ambiguously.** At the
start of work, the project loads context from a generic, project-independent **platform base**
(a set of `*_WORKFLOW.md` docs), then the specific project **extends** that base for its own
needs. The goal is not the base docs themselves — it's that the project, when it starts, *avoids
ambiguity by obtaining context first.*

In Claude Code terms, the mechanism is the CLAUDE.md **`@import`** syntax: a project's `CLAUDE.md`
imports the relevant base docs with `@path` lines, so the generic context loads at session start
([How Claude remembers your project](https://code.claude.com/docs/en/memory)). `@import` is the
concrete public API this skill is built on.

## When to invoke

**Auto-invoke when any of these is true:**

1. A project's `CLAUDE.md` has **no** `## Platform Base Context` block.
2. The project has **no** `CLAUDE.md` (new-project bootstrap).
3. The user explicitly asks to "initialize context", "bootstrap", "load the base context", or
   "align this project with the platform".
4. You're about to start substantial work in a repo the model doesn't recognize.

**Announce on invoke:** "Using `project-init` to load the platform base context and verify this project's initialization."

## The platform base (source of truth)

The base docs are generic, platform-level, and **not tied to any project**. In this repo they
live under `workflow-docs/`. The detection table maps a repo signal → the base doc that applies:

| Domain | Base doc | Repo signal (how it's detected) | In this repo? |
|--------|----------|---------------------------------|---------------|
| AWS | `AWS_WORKFLOW.md` | `amplify/`, `cdk.json`, `samconfig.*`, AWS SDK in deps | ✅ |
| Web/Next.js | `NEXTJS_WORKFLOW.md` | `next.config.*`, `next` in deps | ✅ |
| TypeScript | `TYPESCRIPT_CONTEXT.md` | `tsconfig.json` | ✅ |
| Chrome/web debug | `CHROME_WORKFLOW.md` | front-end with remote debugging / e2e | ✅ |
| Security lab | `KALI_WORKFLOW.md` | pentesting / VM work | ✅ |
| Platform premise | `PLATFORM_BASE.md` | always (defines the rule itself) | ✅ |
| Apple/iOS | `XCODE_WORKFLOW.md` | `*.xcodeproj`, `*.xcworkspace`, `Package.swift`, `project.yml` | ⏳ forthcoming |
| Android native | `ANDROID_WORKFLOW.md` | `build.gradle(.kts)`, `settings.gradle(.kts)`, `gradlew`, `AndroidManifest.xml`, `*.kt` | ⏳ forthcoming |
| Mobile/Expo | `EXPO_WORKFLOW.md` | `app.json` with `"expo"`, `eas.json`, `expo` in deps | ⏳ forthcoming |
| MCP | `MCP_WORKFLOW.md` | `.mcp.json`, MCP servers configured | ⏳ forthcoming |

> Docs marked ⏳ are part of this repo's roadmap. When a detected domain has no base doc yet,
> report it as a gap (Step 2) instead of importing a non-existent file.

**Premise:** the base is the means, not the end. Never edit the base to inject something
project-specific. If you discover **generic, reusable** knowledge, contribute it back to the
base; project-specific knowledge goes in the project.

## Workflow

### Step 1 — Detect applicable domains

If the user passed domains as an argument (`/project-init aws,nextjs,mcp`), use them. Otherwise
detect by scanning the repo:

```bash
ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$ROOT"

ls *.xcodeproj *.xcworkspace Package.swift project.yml 2>/dev/null && echo "→ apple"
ls build.gradle build.gradle.kts settings.gradle settings.gradle.kts gradlew 2>/dev/null && echo "→ android"
[ -f app.json ] && grep -q '"expo"' app.json 2>/dev/null && echo "→ expo"
# Disambiguation: an Expo project's android/ is an RN prebuild, NOT native Android.
# Treat it as expo even if a build.gradle exists under android/.
[ -f eas.json ] && echo "→ expo (eas)"
ls next.config.* 2>/dev/null && echo "→ nextjs"
[ -f tsconfig.json ] && echo "→ typescript"
ls cdk.json samconfig.* 2>/dev/null && echo "→ aws"
[ -d amplify ] && echo "→ aws (amplify)"
[ -f .mcp.json ] && echo "→ mcp"
```

### Step 2 — Resolve the base docs (and report gaps)

For each detected domain, map to its base doc and verify it exists. A detected domain whose base
doc is still forthcoming (⏳ above) is reported as a gap, not imported:

```bash
BASE_DIR="workflow-docs"     # adjust to where the base lives in the consuming repo
for doc in AWS_WORKFLOW NEXTJS_WORKFLOW TYPESCRIPT_CONTEXT; do
  [ -f "$BASE_DIR/$doc.md" ] && echo "✓ $doc.md" || echo "✗ $doc.md (gap — base doc not present)"
done
```

**Read** the applicable base docs so the generic context is loaded before you operate on the
project. This is the heart of the rule: context first.

### Step 3 — Verify / generate the CLAUDE.md block

Check whether the project's `CLAUDE.md` already has the block:

```bash
grep -q "## Platform Base Context" "$ROOT/CLAUDE.md" 2>/dev/null \
  && echo "✓ block present" || echo "✗ missing Platform Base Context block"
```

If missing, propose inserting it near the top of `CLAUDE.md` (gated on user approval). Use Claude
Code's `@import` so the base actually loads at session start:

```markdown
## Platform Base Context

This project extends the generic platform-base documentation. The applicable base context loads
via @import at session start:

@workflow-docs/PLATFORM_BASE.md
@workflow-docs/<DOMAIN>_WORKFLOW.md
<... one @import line per detected domain whose base doc exists ...>

Premise: the base is generic; project-specific detail lives here and in
`docs/<DOMAIN>_EXTENSIONS.md`.
```

If the project has **no** `CLAUDE.md`, offer to create it with this block plus a minimal skeleton
(Project Overview, Build & Test, Module Structure).

### Step 4 — Suggest extension docs

For each domain where the project has real specificity (bundle IDs, deps, its own architecture,
agent skills), suggest creating `docs/<DOMAIN>_EXTENSIONS.md`, each starting with:

```markdown
# <DOMAIN> Extensions — <project>

> Extends `<DOMAIN>_WORKFLOW.md` (platform base) with this project's specifics.
```

Don't create them automatically; list them as recommendations unless the user asks.

### Step 5 — Report state

```
Initialization of <project>:
  Detected domains:   aws, nextjs, mcp
  Base docs loaded:   AWS_WORKFLOW.md, NEXTJS_WORKFLOW.md
  Base doc gaps:      MCP_WORKFLOW.md (forthcoming)
  CLAUDE.md block:    [present / proposed]
  Extension docs:     docs/AWS_EXTENSIONS.md (suggested)
  → Unambiguous context: [yes / pending block approval]
```

## Rules

1. **Never edit the base to inject project-specific content.** Specifics go in the project
   (`CLAUDE.md` + `docs/<DOMAIN>_EXTENSIONS.md`).
2. **Generic, reusable knowledge IS contributed back to the base** — propose it separately, gated.
3. **Human gating** to write the project's `CLAUDE.md`: propose the block, wait for approval, then apply.
4. **Idempotent:** if the block already exists and is correct, don't duplicate — just verify the
   listed domains still match the repo.
5. **Context first, operate second.** Reading the applicable base docs IS the step that removes
   ambiguity; don't skip it.

## Example

```
User: /project-init
  → Detects: aws (cdk.json), nextjs (next.config.ts), mcp (.mcp.json)
  → Loads: AWS_WORKFLOW.md, NEXTJS_WORKFLOW.md; reports MCP_WORKFLOW.md as a forthcoming gap
  → CLAUDE.md has a Module Structure section but NO "Platform Base Context" block
  → Proposes inserting the block with @import lines for the two existing base docs
  → Suggests docs/AWS_EXTENSIONS.md to hold the project's IaC specifics
  → Reports: base context loaded, block pending approval
```

## Related

- `workflow-docs/PLATFORM_BASE.md` — the universal premise this skill enforces.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — keeps this skill's CLAUDE.md /
  `@import` claims current against Claude Code's memory docs.

---

**Last verified:** 2026-06-03 against Claude Code memory docs (CLAUDE.md scopes + `@import`) and
Agent Skills best practices.
**Re-check after:** any Claude Code CLAUDE.md / memory schema change, or by 2026-12-03. **Decay risk:** medium.
