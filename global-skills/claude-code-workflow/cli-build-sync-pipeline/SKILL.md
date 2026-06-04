---
name: cli-build-sync-pipeline
description: >-
  Run a platform-agnostic project-consistency pass before a commit/PR — regenerate the project
  file, format, lint, build, then take an IDE second opinion via the editor's MCP server to catch
  diagnostics the CLI build truncates or never emits. Works for Xcode, Gradle, Node, Go, etc. Uses
  Claude Code's PostToolUse hook (format-on-write) and the IDE-Navigator check (e.g. Xcode's
  XcodeListNavigatorIssues) as a complement to the build log. Use before declaring a change ready,
  on /sync, or when wiring format/lint/build automation. Generic cross-platform version; an
  Android-specific build-sync variant exists separately.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — Automate workflows with hooks (PostToolUse fires after a tool succeeds; stdin tool_input.file_path)"
      url: "https://code.claude.com/docs/en/hooks"
      version: "Claude Code 2026-06"
    - source: "XcodeBuildMCP server (build_sim / build_run_sim / test_sim / session defaults; structured build output)"
      url: "https://github.com/cameroncooke/XcodeBuildMCP"
      version: "XcodeBuildMCP 2026-06"
    - source: "Native Xcode MCP server (XcodeListNavigatorIssues / XcodeRefreshCodeIssuesInFile) — Issue Navigator state"
      url: "https://www.paperclipped.de/en/blog/xcode-agentic-coding-claude-codex/"
      version: "Xcode 26.3+"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code minor release, or a new Xcode/Android Studio MCP tool surface"
    or_date: "2026-09-03"
  decay_risk: high
  status: current
---

# cli-build-sync-pipeline

A repeatable consistency pass that brings a working tree to a known-good state before a commit or
PR. The pipeline itself — **regenerate → format → lint → build** — is universal and uncontroversial
(the `pre-commit` framework codifies the same chain). The **non-obvious, load-bearing step** is the
last one: read the **IDE's** diagnostics through its MCP server as a *second opinion*, because
`xcodebuild` / `./gradlew assemble` / `swift build` routinely miss issues the editor's Issue
Navigator shows.

## When to invoke

- Before declaring a change ready or opening a PR.
- On `/sync`.
- When wiring format-on-write or pre-commit automation for a repo.

**Announce on invoke:** "Using `cli-build-sync-pipeline` to regenerate, format, lint, build, and take an IDE second-opinion diagnostics pass."

## The five steps

| # | Step | iOS / Xcode | Android / Gradle | Node | Go |
|---|---|---|---|---|---|
| 1 | Regenerate project | `xcodegen generate` (or Tuist) | gradle wrapper sync | — | — |
| 2 | Format | `swiftformat .` | `./gradlew ktlintFormat` | `prettier --write` | `gofmt -w .` |
| 3 | Lint | `swiftlint --strict` | `./gradlew detekt` | `eslint` | `golangci-lint run` |
| 4 | Build | `xcodebuild … build` | `./gradlew assembleDebug` | `pnpm build` | `go build ./...` |
| 5 | **IDE second opinion** | `XcodeListNavigatorIssues` (native Xcode MCP) | Android Studio / JetBrains MCP inspections | — | — |

```bash
#!/usr/bin/env bash
# scripts/sync.sh — invoked by /sync (step 5 is a separate MCP tool call, see below)
set -euo pipefail
case "${1:-auto}" in
  ios)     xcodegen generate; swiftformat .; swiftlint --strict; xcodebuild -scheme "$SCHEME" -destination "$DEST" build -quiet ;;
  android) ./gradlew ktlintFormat detekt assembleDebug lintDebug ;;
  node)    pnpm install --frozen-lockfile && pnpm format && pnpm lint && pnpm build ;;
  go)      go mod tidy && gofmt -w . && golangci-lint run && go build ./... ;;
esac
```

## Step 5 — the IDE second opinion (the core insight)

**CLI build output ≠ Issue Navigator content.** `xcodebuild` runs without the editor's live
SourceKit-LSP indexing, so warnings like unused `let`/`var`, "Will never be executed," and static
analyzer findings ("Potential leak") are frequently emitted **only** in the IDE. After step 4 build
succeeds, query the editor's MCP server for issues it sees and the build log didn't surface.

For Apple platforms there are two complementary MCP servers — call them by their real tool names:

- **Native Xcode MCP (Xcode 26.3+).** `mcp__xcode__XcodeListNavigatorIssues` returns the Issue
  Navigator state (compiler diagnostics from SourceKit-LSP, the static analyzer, package-resolution
  warnings). `mcp__xcode__XcodeRefreshCodeIssuesInFile` re-runs diagnostics for one file.
- **XcodeBuildMCP.** Wraps `xcodebuild` into **structured JSON** build/test errors — agent-friendly
  where the raw log is megabytes. Verified tools include `build_sim`, `build_run_sim`, `test_sim`,
  `list_schemes`, `show_build_settings`, `discover_projs`, `clean`, and the **session defaults**
  pair `session_show_defaults` / `session_set_defaults` (set scheme + simulator once instead of
  re-passing them every call).

A robust pass calls **both**: XcodeBuildMCP for the build verdict, the native Xcode MCP for the
Navigator state. For Android the closest analog is the JetBrains / Android Studio MCP exposing
inspections; if that server isn't installed, document the gap rather than pretending the CLI
`lintDebug` covers it.

## Format-on-write via PostToolUse (verified)

To keep files formatted as Claude writes them, use a **`PostToolUse`** hook — **not** `PreToolUse`.
Verified against the [hooks reference](https://code.claude.com/docs/en/hooks): `PreToolUse` fires
*before* the tool and can block it; `PostToolUse` fires *after a tool call succeeds* — which is what
you want when formatting what was just written. Hooks receive their input as **stdin JSON**; the
edited path is `tool_input.file_path` (there is no `CLAUDE_TOOL_INPUT_FILE_PATH` env var — read it
from stdin):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command",
            "command": "f=$(jq -r '.tool_input.file_path // empty'); case \"$f\" in *.swift) swiftformat \"$f\" ;; *.kt) ktlint -F \"$f\" ;; *.ts|*.tsx) npx prettier --write \"$f\" ;; esac" }
        ]
      }
    ]
  }
}
```

The `matcher` is matched against **tool names** (`Write|Edit`), confirmed in the hooks docs.

## Pitfalls (verified)

- **Formatting generated files breaks codegen output.** Exclude `*.generated.swift`, `build/`,
  `.build/`, `DerivedData/`, Gradle plugin output. Enforce via `.swiftformat` / ktlint config so the
  hook and the script both respect it.
- **`PreToolUse` for format-on-write is wrong.** It fires before the write and can block the edit;
  `PostToolUse` formats what was actually written.
- **XcodeBuildMCP wants session defaults first.** Per its own instructions, call
  `session_show_defaults` before the first build/run/test; only call `discover_projs` when defaults
  are missing or wrong — calling it speculatively wastes a tool turn.
- **Project regeneration (step 1) can be lossy.** `xcodegen generate` rewrites `*.xcodeproj`; if the
  team hand-edits the project, make step 1 opt-in.

## Related skills

- `global-skills/claude-code-workflow/android-build-sync/SKILL.md` — the Android-specific build-sync
  (Gradle + Compose + Android Studio inspections). This skill is the generic cross-platform version.
- `global-skills/claude-code-workflow/claude-mcp-lesson-capture-pipeline/SKILL.md` — captures the
  lessons that build/lint failures here tend to produce.
- `global-skills/claude-code-workflow/mcp-orchestration-pattern/SKILL.md` — decides *which* MCP server
  owns the build/diagnostics domain so step 5 routes to the right tool.

## Sources

- [Automate workflows with hooks (PostToolUse timing, matcher on tool names, stdin tool_input)](https://code.claude.com/docs/en/hooks)
- [XcodeBuildMCP (structured build output, session defaults)](https://github.com/cameroncooke/XcodeBuildMCP)
- [Xcode 26.3 agentic coding — native MCP Issue Navigator tools](https://www.paperclipped.de/en/blog/xcode-agentic-coding-claude-codex/)

---

**Last verified:** 2026-06-03. Confirmed live: `PostToolUse` fires **after a tool succeeds** and the
edited path arrives via **stdin `tool_input.file_path`** (no `CLAUDE_TOOL_INPUT_FILE_PATH` env var);
`matcher` filters on tool names; XcodeBuildMCP tools (`build_sim`, `build_run_sim`, `test_sim`,
`session_show_defaults`) and native Xcode MCP tools (`XcodeListNavigatorIssues`,
`XcodeRefreshCodeIssuesInFile`) confirmed present in the live tool surface.
**Flagged:** the IDE-second-opinion step is emerging practice (well-evidenced for Xcode 26.3, not a
named Anthropic feature); the Android inspection MCP is a documented gap, not a guarantee.
**Re-check after:** any Claude Code minor release or new IDE-MCP tool surface, or by 2026-09-03.
**Decay risk:** high. **Found a drift?** Run `/skill-pattern-freshness-audit claude-code-workflow`.
