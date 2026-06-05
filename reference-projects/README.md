# reference-projects

Sanitized architecture documentation from two real native apps — a native **iOS** app and a native
**Android** app. These are *worked examples* of the patterns in [`../global-skills/`](../global-skills/):
how Clean Architecture, tech-stack decisions, and platform UX actually played out in production,
with every project-specific identifier removed.

These are **documentation, not runnable code** — no source, no business logic, no secrets. They
exist to show the patterns in context.

## [`ios-native-app/`](ios-native-app/)

A native iOS app (Swift 6 / iOS 26, MVVM + Clean Architecture).

| Doc | Topic |
|-----|-------|
| [`CLAUDE.md`](ios-native-app/CLAUDE.md) | The project-governance file — how Claude Code is steered: architecture conventions, concurrency rules, the agent-skill set, and the lesson-capture discipline |
| [`docs/02-tech-research.md`](ios-native-app/docs/02-tech-research.md) | Stack comparison — Apollo vs REST, Nuke vs SDWebImage, post-quantum TLS, Foundation Models |
| [`docs/03-architecture-decision.md`](ios-native-app/docs/03-architecture-decision.md) | Clean Architecture + MVVM rationale; why not Amplify |
| [`docs/04-module-structure.md`](ios-native-app/docs/04-module-structure.md) | The full module tree with per-layer actor-isolation rules (Core / Domain / Data / Generated / Features / Presentation) |
| [`docs/05-agent-skills.md`](ios-native-app/docs/05-agent-skills.md) | The agent-skill catalog + dependency graph + 20 compiler-verified Swift 6 lessons + WWDC citations |
| [`docs/06-ios26-ux-design-patterns.md`](ios-native-app/docs/06-ios26-ux-design-patterns.md) | iOS 26 UX — Liquid Glass, tab collapse, zoom transitions |
| [`docs/07-competitor-ux-patterns.md`](ios-native-app/docs/07-competitor-ux-patterns.md) | Reverse-engineered UX patterns from leading social/marketplace apps |
| [`docs/08-mcp-servers.md`](ios-native-app/docs/08-mcp-servers.md) | The MCP server set used during development |
| [`docs/09-auth-module.md`](ios-native-app/docs/09-auth-module.md) | A production auth module — Cognito + federated OAuth/PKCE, `state` CSRF protection, constant-time comparison, 102 tests |
| `.swiftformat`, `.swiftlint.yml` | The Swift 6 strict-concurrency lint/format configs |

## [`android-native-app/`](android-native-app/)

A native Android app (Kotlin / Compose / KMP), the platform peer of the iOS app.

| Doc | Topic |
|-----|-------|
| [`docs/02-architecture.md`](android-native-app/docs/02-architecture.md) | Clean Architecture + MVVM in Compose |
| [`docs/03-gradle-performance.md`](android-native-app/docs/03-gradle-performance.md) | Gradle build performance tuning for Apple Silicon |

---

The reusable, generic versions of everything here live in [`../global-skills/`](../global-skills/).
These reference docs show one concrete instantiation; the skills are the portable patterns.
