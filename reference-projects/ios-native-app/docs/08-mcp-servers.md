# 08 — MCP Servers Configuration

7 MCP servers connected to Claude Code for the MyApp iOS project.

## Installed Servers

### 1. Apple Xcode MCP (native, Xcode 26.3+)

```bash
claude mcp add --transport stdio xcode -s user -- xcrun mcpbridge
```

**Prerequisite**: Xcode must be running with the project open.
Enable: Xcode > Settings > Intelligence > Model Context Protocol > Toggle "Xcode Tools" ON.

**Tools (20):**
- File System: `XcodeRead`, `XcodeWrite`, `XcodeUpdate`, `XcodeGlob`, `XcodeGrep`, `XcodeLS`, `XcodeMakeDir`, `XcodeRM`, `XcodeMV`
- Build & Test: `BuildProject`, `GetBuildLog`, `RunAllTests`, `RunSomeTests`, `GetTestList`
- Diagnostics: `XcodeListNavigatorIssues`, `XcodeRefreshCodeIssuesInFile`
- Intelligence: `ExecuteSnippet` (Swift REPL), `RenderPreview` (SwiftUI screenshots), `DocumentationSearch` (Apple docs + WWDC via MLX)
- Workspace: `XcodeListWindows`

**Architecture**: Claude Code → MCP → `mcpbridge` → XPC → Xcode

Source: [Apple — Giving external agentic coding tools access to Xcode](https://developer.apple.com/documentation/xcode/giving-agentic-coding-tools-access-to-xcode)

### 2. XcodeBuildMCP (getsentry, 4.9k stars)

```bash
claude mcp add XcodeBuildMCP -s user -e XCODEBUILDMCP_SENTRY_DISABLED=true -- npx -y xcodebuildmcp@latest mcp
```

59 structured tools. Works **headless** (no Xcode GUI required).
Covers: build operations, device management, testing, logs/debugging, project analysis.

Source: https://github.com/getsentry/XcodeBuildMCP

### 3. Apple Docs MCP (1.1k stars)

```bash
claude mcp add apple-docs -s user -- npx -y @kimsungwhee/apple-docs-mcp@latest
```

14 tools: `search_apple_docs`, `get_apple_doc_content`, `list_technologies`,
`search_framework_symbols`, `get_related_apis`, `get_platform_compatibility`,
`search_wwdc_videos`, `get_wwdc_video_details`, `get_sample_code`

Source: https://github.com/kimsungwhee/apple-docs-mcp

### 4. Mobile MCP (4.2k stars)

```bash
claude mcp add mobile-mcp -s user -- npx -y @mobilenext/mobile-mcp@latest
```

Full mobile automation: device management, app install/launch/terminate,
screenshots, UI element listing, tap/swipe/type, open URLs.

Source: https://github.com/mobile-next/mobile-mcp

### 5. AWS AppSync MCP (awslabs official)

```bash
claude mcp add appsync -s user -- uvx awslabs.aws-appsync-mcp-server@latest
```

Requires `uv` (`brew install uv`).
Tools: create/inspect APIs, data sources, resolvers, schemas, channel namespaces.
Add `--allow-write` for write operations.

Source: https://awslabs.github.io/mcp/servers/aws-appsync-mcp-server

### 6. GraphQL MCP (375 stars)

```bash
claude mcp add graphql -s user -- npx -y mcp-graphql
```

Tools: `introspect-schema`, `query-graphql`.
Configure endpoint via env vars: `ENDPOINT`, `HEADERS`, `SCHEMA`.

Source: https://github.com/blurrah/mcp-graphql

## Rejected: SwiftLens

SwiftLens (SourceKit-LSP bridge) was evaluated and rejected:
- 44% accuracy for expressions within function bodies
- Incompatible with current Python version
- `Grep` + `Glob` + `Read` cover 95% of the use cases at 100% reliability

## Key Xcode MCP Workflow

**MANDATORY after every code change:**
```
1. mcp__xcode__XcodeListWindows → get tabIdentifier
2. mcp__xcode__XcodeListNavigatorIssues(tabIdentifier, severity: "warning") → verify 0 issues
```

**For UI development:**
```
mcp__xcode__RenderPreview(tabIdentifier, sourceFilePath) → preview screenshot
```

**For API verification:**
```
mcp__xcode__DocumentationSearch(query, frameworks) → verify against Apple docs
```

**For testing logic:**
```
mcp__xcode__ExecuteSnippet(tabIdentifier, codeSnippet, sourceFilePath, purpose) → run in project context
```

## MCP × Agent Skills Matrix

| MCP Server | Skills that use it |
|---|---|
| **xcode** (`XcodeListNavigatorIssues`) | **ALL 23 skills** — MANDATORY verification step |
| **xcode** (`RenderPreview`) | `feature-scaffold`, `liquid-glass`, `adaptive-layout`, `ios26-ux-patterns`, `social-feed-patterns` |
| **xcode** (`DocumentationSearch`) | `swift-module`, `auth-scaffold`, `security-review`, `swift-error-patterns`, `liquid-glass`, `adaptive-layout`, `ios26-ux-patterns`, `apple-security-patterns`, `foundation-models` |
| **xcode** (`ExecuteSnippet`) | `auth-scaffold`, `auth-test-suite`, `auth-security-audit`, `swift-error-patterns` |
| **xcode** (`RunAllTests/RunSomeTests`) | `test-scaffold`, `auth-test-suite`, `auth-performance-test` |
| **XcodeBuildMCP** | `project-sync`, `swift-module`, `graphql-codegen`, `appsync-operation`, `feature-scaffold`, `auth-scaffold`, `test-scaffold`, `websocket-scaffold`, `social-feed-patterns`, `chat-patterns`, `marketplace-patterns`, `social-graph-patterns` |
| **apple-docs** | `swift-module`, `feature-scaffold`, `auth-scaffold`, `security-review`, `liquid-glass`, `adaptive-layout`, `ios26-ux-patterns`, `foundation-models`, `apple-security-patterns`, `social-graph-patterns` |
| **mobile-mcp** | `feature-scaffold`, `test-scaffold`, `liquid-glass`, `adaptive-layout`, `ios26-ux-patterns`, `social-feed-patterns`, `chat-patterns`, `marketplace-patterns`, `social-graph-patterns` |
| **appsync** | `graphql-codegen`, `appsync-operation`, `websocket-scaffold`, `chat-patterns`, `marketplace-patterns` |
| **graphql** | `graphql-codegen`, `appsync-operation`, `websocket-scaffold`, `marketplace-patterns`, `social-graph-patterns` |
| **figma** | `ui-design-workflow` |

### 7. Figma MCP (official)

```bash
claude mcp add figma -s user -- npx -y figma-developer-mcp --stdio
```

**Auth:** OAuth via `figma.com/oauth/mcp`. Account: `<your-figma-account>`.
**Plan:** Starter (6 tool calls/month). For heavy extraction, use manual PDF export.

**Key Tools:**
- `get_design_context` — Extract design as code + screenshot + metadata
- `get_screenshot` — Capture node screenshot
- `generate_figma_design` — Write designs back to Figma (exempt from rate limit)
- `whoami` — Check auth status and plan (exempt from rate limit)

**Skills:** `ui-design-workflow`

## Verification

```bash
claude mcp list
```

All 7 should show `✓ Connected`.
