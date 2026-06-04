# claude-code-workflow — Claude Code + MCP automation skills

6 skills for working AI-natively with Claude Code: bootstrapping project context, capturing
lessons, build-consistency pipelines, and orchestrating MCP servers. This is the newest domain, so
each skill explicitly marks which patterns are documented-canonical vs. opinionated convention.
Verified against `code.claude.com`, `platform.claude.com`, and `modelcontextprotocol.io`.

## Project context & orchestration

| Skill | What it does |
|-------|--------------|
| [`project-init`](project-init/SKILL.md) | Initialize/audit a project against a generic platform base so work never starts ambiguously — generates the `## Platform Base Context` block via `@import` |
| [`mcp-orchestration-pattern`](mcp-orchestration-pattern/SKILL.md) | Assemble multiple MCP servers so each owns one primary domain, with a task→MCP routing table and a Phase-0 sweep before architecture |

## Pipelines

| Skill | What it does |
|-------|--------------|
| [`cli-build-sync-pipeline`](cli-build-sync-pipeline/SKILL.md) | Platform-agnostic pre-commit consistency pass — regenerate project file, format, lint, build, + IDE-Navigator second-opinion check |
| [`android-build-sync`](android-build-sync/SKILL.md) | The Android-specific variant — ktlint + detekt + `lintDebug` + `assembleDebug`, environment-agnostic (no hardcoded paths) |

## Lesson capture

| Skill | What it does |
|-------|--------------|
| [`claude-mcp-lesson-capture-pipeline`](claude-mcp-lesson-capture-pipeline/SKILL.md) | After a merge, cascade the lesson to three surfaces — auto-memory → agent skills → human docs — via a `Stop` hook (correct exit-0 + `decision:block` form) |
| [`capture-lessons-cascade-android`](capture-lessons-cascade-android/SKILL.md) | The Android-specific variant, with quirks for R8, Compose recomposition, and Hilt scoping |

> The lesson-capture and build-sync skills come in a generic + Android-specific pair. The generic
> ones (`claude-mcp-lesson-capture-pipeline`, `cli-build-sync-pipeline`) describe the platform-neutral
> shape; the Android twins (`capture-lessons-cascade-android`, `android-build-sync`) specialize it.

---

**Honesty about canon:** patterns like the lesson-capture cascade and the MCP routing table are
*emerging practice / opinionated convention* built on top of Claude Code's canonical primitives
(`Stop`/`PostToolUse` hooks, `@import`, `.mcp.json`, `disable-model-invocation`). Each skill flags
which is which.

**Freshness:** Claude Code evolves fast — re-check after a **Claude Code minor release** or by
**2026-09-03**. Several here are `decay_risk: high`. Run `/skill-pattern-freshness-audit claude-code-workflow`.
