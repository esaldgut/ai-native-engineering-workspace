---
name: mcp-orchestration-pattern
description: >-
  Assemble multiple specialized MCP servers so each owns one primary domain, with a task→MCP
  routing table in CLAUDE.md, a Phase 0 sweep before architecture, and Tool Search left on by
  default (deferral) instead of alwaysLoad-ing everything. Avoids the tool-overflow problem where
  Claude has 150 tools and picks badly. Uses Claude Code's MCP scope precedence (local > project >
  user), .mcp.json, alwaysLoad, and the MCP spec's server "instructions". Use when wiring more than
  one MCP server, when Claude isn't using an installed server, or when designing a cross-domain
  task (build + docs + design + backend) that spans several servers.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Anthropic — Connect Claude Code to tools via MCP (scope precedence, .mcp.json, alwaysLoad, Tool Search, 2KB instructions)"
      url: "https://code.claude.com/docs/en/mcp"
      version: "Claude Code 2026-06 (alwaysLoad v2.1.121+)"
    - source: "Model Context Protocol — Lifecycle (server returns optional `instructions` in initialize result)"
      url: "https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle"
      version: "MCP spec 2025-06-18"
    - source: "Anthropic — Advanced tool use / Tool Search evals (deferral improves accuracy, not just context)"
      url: "https://www.anthropic.com/engineering/advanced-tool-use"
      version: "2026"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code minor release (MCP scope / Tool Search / alwaysLoad changes) or new MCP spec revision"
    or_date: "2026-09-03"
  decay_risk: high
  status: current
---

# mcp-orchestration-pattern

How to run several specialized MCP servers together so each is the **single primary source for one
domain**, and the agent skills point to "which MCP for which task." This is a concrete instantiation
of Anthropic's **domain-isolation** guidance — and it is now backed by data, not just intuition:
with Tool Search enabled, Anthropic's MCP evals show accuracy *improving* (Opus 4: 49% → 74%) as
large tool catalogs stop causing decision paralysis ([advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use)).

The four moves below — one primary domain per server, a routing table, a Phase 0 sweep, and Tool
Search by default — are the user's crystallization of that guidance. The *crystallization* is
opinionated; every underlying mechanism it relies on is canonical and verified.

## When to invoke

- You're wiring more than one MCP server into a repo.
- Claude "isn't using" an installed MCP server (often just Tool Search deferral — see pitfalls).
- You're about to design a task that spans domains (build + API docs + design + backend) and need to
  know which server answers which question.

**Announce on invoke:** "Using `mcp-orchestration-pattern` to map each MCP server to its primary domain and run the Phase 0 routing sweep."

## Move 1 — One primary domain per server (routing table in CLAUDE.md)

Keep a task→MCP table as **facts Claude should always hold** in `CLAUDE.md` (this is what CLAUDE.md
is for). Generic example:

```markdown
## MCP server routing (Phase 0 sweep — read before designing)

| Task                                  | Primary MCP                          | Fallback / second opinion        |
|---------------------------------------|--------------------------------------|----------------------------------|
| Build/test an iOS scheme on a sim     | XcodeBuildMCP                        | mcp__xcode__BuildProject         |
| List warnings + analyzer issues       | mcp__xcode__XcodeListNavigatorIssues | xcodebuild + grep `warning:`     |
| Look up a platform API symbol         | a docs-search MCP                    | WebFetch on vendor docs          |
| UI-tap a real device / emulator       | a device-automation MCP              | sim-only build tool              |
| Run a GraphQL query against a backend | a graphql MCP                        | curl via Bash                    |
| Read design context for a screen      | a design MCP (get_design_context)    | a screenshot tool                |
```

> The routing table is **opinionated structure**, not Claude Code spec. Frame it as a project
> convention that operationalizes domain isolation.

## Move 2 — Phase 0 sweep before architecture

Before proposing a design, sweep the table: for each sub-question the task raises, name the **primary
MCP** that answers it and confirm that server is connected (`/mcp`). This front-loads "do I even have
a tool for this?" so the architecture isn't built on a capability that isn't installed.

## Move 3 — Tool Search by default; `alwaysLoad` sparingly

Verified against the [MCP docs](https://code.claude.com/docs/en/mcp):

- **Tool Search is on by default.** MCP tool *definitions* are deferred; only tool **names** and
  **server instructions** load at session start. Claude calls the `ToolSearch` meta-tool to pull a
  server's full schemas when a task needs them. (Without Tool Search — e.g. on Vertex AI or a
  non-first-party `ANTHROPIC_BASE_URL` — Claude uses `WaitForMcpServers` instead.)
- **`alwaysLoad: true`** in a server's `.mcp.json` entry forces *all* its tools into context upfront,
  bypassing deferral. Requires **v2.1.121+**. It also **blocks startup** until that server connects
  (capped at the 5s connect timeout). Per-tool opt-in: `"anthropic/alwaysLoad": true` in the tool's
  `_meta`.

```json
{ "mcpServers": { "core-tools": { "type": "http", "url": "https://mcp.example.com/mcp", "alwaysLoad": true } } }
```

Default to **leaving Tool Search on**. Reach for `alwaysLoad` only for the 1–2 tools genuinely needed
*every* turn (e.g. a build server's `session_show_defaults`). `alwaysLoad`-ing five servers can load
~150 schemas upfront and eat a large fraction of the window before the first prompt.

## Move 4 — Make servers discoverable via `instructions`

A server's **`instructions`** field is how Claude knows *when to search for its tools*. This is
canonical at two levels:

- **MCP spec (2025-06-18):** the `initialize` result carries an optional `"instructions"` string —
  "Optional instructions for the client."
- **Claude Code:** with Tool Search on, server instructions become the routing signal (like a skill
  description). Claude Code **truncates tool descriptions and server instructions at 2KB each** — put
  the "when to search this server" hint at the very start.

## Scope precedence (verified, exact order)

When the same server is defined in more than one place, Claude Code connects **once**, using the
highest-precedence source — **the whole entry; fields are not merged**:

1. **Local** — `~/.claude.json`, current project only (the default `--scope`).
2. **Project** — `.mcp.json` at repo root, committed/shared.
3. **User** — `~/.claude.json`, all projects.
4. **Plugin-provided** servers.
5. **claude.ai connectors.**

(Scopes match by name; plugins/connectors match by endpoint.)

## Pitfalls (verified)

- **"My MCP is broken" is usually Tool Search deferral.** `/mcp` shows tools that *exist*, not tools
  Claude currently sees — Claude sees ~0 MCP tool schemas until it calls `ToolSearch`. Not a bug.
- **`alwaysLoad: true` is the silent context killer.** Setting it broadly loads everything upfront and
  delays startup. Use deferral; override only where every turn needs the tool.
- **Two servers claiming one domain → non-determinism.** If two servers both expose, say,
  `search_docs`, name collisions resolve by scope precedence — but Claude picks by the *description*
  it sees, not the precedence. Install one, or namespace the server names.
- **Project `.mcp.json` needs approval on first use.** Freshly-cloned project servers show as
  `⏸ Pending approval` in `claude mcp list`; CI can't approve them. Reset with
  `claude mcp reset-project-choices` in an interactive session.
- **Instructions over 2KB truncate.** A block documenting 30 tools gets cut — lead with the routing
  hint.
- **Output caps.** MCP tool output warns at 10K tokens and defaults to a 25K cap
  (`MAX_MCP_OUTPUT_TOKENS`); a server can raise a single tool via `_meta.anthropic/maxResultSizeChars`
  up to a 500K-char ceiling.

## Related skills

- `global-skills/claude-code-workflow/cli-build-sync-pipeline/SKILL.md` — consumes the routing table:
  its step 5 picks the build vs. Issue-Navigator MCP based on this domain map.
- `global-skills/claude-code-workflow/project-init/SKILL.md` — seeds CLAUDE.md, the home of the
  routing table, when a project initializes.
- `global-skills/meta/skill-pattern-freshness-audit/SKILL.md` — re-checks scope precedence,
  `alwaysLoad`, and Tool Search behavior after each Claude Code minor release.

## Sources

- [Connect Claude Code to tools via MCP (scope precedence, .mcp.json, alwaysLoad, Tool Search, 2KB instructions)](https://code.claude.com/docs/en/mcp)
- [MCP spec — Lifecycle (server `instructions` in initialize result)](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle)
- [Anthropic — Advanced tool use (Tool Search accuracy evals)](https://www.anthropic.com/engineering/advanced-tool-use)

---

**Last verified:** 2026-06-03. Confirmed live: scope precedence **local > project > user > plugin >
claude.ai connector** with whole-entry (non-merged) resolution; Tool Search **on by default**
(`ToolSearch` meta-tool; `WaitForMcpServers` fallback); `alwaysLoad` v2.1.121+ blocks startup at the
5s connect cap; server `instructions` truncate at **2KB**; the MCP spec's `initialize` result carries
an optional `instructions` string. **Flagged as opinionated:** the routing table, Phase 0 sweep, and
"one primary domain per server" are conventions operationalizing Anthropic's domain-isolation
guidance — not named Claude Code features.
**Re-check after:** any Claude Code minor release or new MCP spec revision, or by 2026-09-03.
**Decay risk:** high. **Found a drift?** Run `/skill-pattern-freshness-audit claude-code-workflow`.
