# android — Kotlin / Compose / KMP skills

5 skills for native Android with Kotlin and Jetpack Compose, plus one cross-platform skill for
sharing logic with iOS via KMP. Verified against `developer.android.com`, `kotlinlang.org`, and the
tool repos — with the recent breaking changes applied (Hilt 1.3.0 artifact rename,
`EncryptedSharedPreferences` deprecation, `collectAsStateWithLifecycle`, `runTest`).

## Architecture & scaffolding

| Skill | What it does |
|-------|--------------|
| [`compose-clean-architecture-module-scaffold`](compose-clean-architecture-module-scaffold/SKILL.md) | Scaffold one Kotlin file into the correct Clean layer (UI/Domain/Data) — twin of iOS `swift-clean-architecture-module-scaffold` |
| [`compose-feature-scaffold`](compose-feature-scaffold/SKILL.md) | A complete Compose feature at once — domain model + repository + `RepositoryImpl` + ViewModel + Screen + Hilt DI + nav entry |

## Testing & security

| Skill | What it does |
|-------|--------------|
| [`android-testing-patterns`](android-testing-patterns/SKILL.md) | The right test type per layer — JVM unit (domain/repo) with JUnit5 + Turbine + MockK; `runTest` (not `runBlockingTest`); Compose UI tests |
| [`android-security-checklist`](android-security-checklist/SKILL.md) | OWASP MASVS-aligned — secrets via DataStore + Google Tink + Keystore (**not** `EncryptedSharedPreferences`), Network Security Config pinning, Custom Tabs + PKCE for OAuth |

## Cross-platform

| Skill | What it does |
|-------|--------------|
| [`kmp-shared-extraction`](kmp-shared-extraction/SKILL.md) | **Requires BOTH a native Android and a native iOS app.** Extract shared business logic into a KMP `commonMain` module; `expect`/`actual`; XCFramework for iOS |

---

**Freshness:** re-check after **Google I/O** (May) or a **Compose BOM major**, or by **2026-12-01**.
Several here are `decay_risk: high` (Compose/Hilt churn fast). Run `/skill-pattern-freshness-audit android`.
