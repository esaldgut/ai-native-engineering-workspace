---
name: android-build-sync
description: >-
  One-command Android build-consistency gate, environment-agnostic — format (ktlint via
  org.jlleitschuh.gradle.ktlint), static analysis (detekt via io.gitlab.arturbosch.detekt, with the
  Compose ruleset), platform lint (./gradlew lintDebug), then compile/package (./gradlew assembleDebug),
  with pre-flight checks for JDK (17/21) and the Gradle wrapper. Enforces ./gradlew over system gradle,
  a detekt baseline for retrofitting, and parallel+caching gradle.properties. Use to set up or run a
  pre-commit/pre-push/CI build gate for a Kotlin/Compose project. No hardcoded machine paths.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Android — Enable app optimization with R8 / shrink code (lint + build pipeline context)"
      url: "https://developer.android.com/studio/build/shrink-code"
      version: "AGP 8.x"
    - source: "Android — Android Lint (lint / lintDebug tasks, variant-aware)"
      url: "https://developer.android.com/studio/write/lint"
      version: "AGP 8.x"
    - source: "Android — AGP releases & JDK compatibility (AGP 8.x requires JDK 17)"
      url: "https://developer.android.com/build/releases/gradle-plugin"
      version: "AGP 8.x (JDK 17 min)"
    - source: "detekt — static analysis + git pre-commit hook"
      url: "https://detekt.dev/"
      version: "detekt 1.23.x"
    - source: "ktlint Gradle plugin (JLLeitschuh)"
      url: "https://github.com/JLLeitschuh/ktlint-gradle"
      version: "ktlint-gradle 12.x"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Claude Code minor release"
    or_date: "2026-09-03"
  decay_risk: medium
  status: current
---

# android-build-sync — one-command Kotlin/Compose build gate

A single consistency gate that runs the consensus 2025-26 pipeline — **ktlint → detekt → lintDebug →
assembleDebug** — so "it compiles, it's formatted, it passes static analysis, it links" is one command,
not four remembered ones. **Environment-agnostic by design:** it uses the project-local `./gradlew`
wrapper and relative paths only — no absolute machine paths, no external-drive paths, nothing
machine-specific. It works identically on any developer's machine and in CI.

## When to invoke

- Setting up or running a build-consistency gate for a Kotlin/Compose project (pre-commit, pre-push, or
  CI).
- Before opening a PR, to catch formatting/lint/compile breakage locally.
- You see CI running `gradle` (system-installed) instead of `./gradlew`, or formatting failures that pass
  locally but fail in CI — fix per the rules below.

**Announce on invoke:** "Using `android-build-sync` to run the ktlint → detekt → lintDebug →
assembleDebug gate via the project's `./gradlew` wrapper."

## The pipeline

| Stage | Task | Catches |
|---|---|---|
| 1. Format | `./gradlew ktlintFormat` (or `ktlintCheck` in CI) | style / JetBrains Kotlin formatting |
| 2. Static analysis | `./gradlew detekt` | complexity, smells, Compose recomposition pitfalls |
| 3. Platform lint | `./gradlew lintDebug` | manifest issues, deprecated APIs, missing translations, Compose lints |
| 4. Compile + package | `./gradlew assembleDebug` | "does the whole thing actually compile and link?" |

## The rules (load-bearing)

1. **`./gradlew`, never system `gradle`.** The wrapper pins the exact Gradle version — the
   reproducible-build invariant. Running system `gradle` produces non-reproducible builds; CI must always
   use `./gradlew`.
2. **JDK 17 minimum for AGP 8.x (21 supported on AGP 8.7+).** Building with JDK 11 fails with
   `Unsupported class file major version 61`. The pre-flight checks the active JDK before doing work.
3. **`lintDebug`, not `lint`, as the gate.** Plain `lint` runs every variant (release + debug + all
   flavors) and is slow on multi-flavor apps; `lintDebug` is the pragmatic single-variant gate.
4. **Use a detekt baseline to retrofit onto an existing codebase.** `./gradlew detektBaseline` writes
   `detekt-baseline.xml`; subsequent runs fail only on **new** violations, so you adopt detekt without
   fixing every legacy finding at once.
5. **Pin the ktlint version.** ktlint releases disagree on formatting (import ordering changed in 0.50);
   an unpinned version causes "passes locally, fails in CI" loops. Pin it in the plugin config.
6. **Enable parallel builds and caching.** Put `org.gradle.parallel=true` and `org.gradle.caching=true`
   in `gradle.properties` — these dominate CI time for medium apps.
7. **Add the Compose detekt ruleset.** `io.nlopez.compose.rules:detekt` catches recomposition bugs
   (lambda allocation in composition, missing `key()` in `items()`) that plain detekt and lint miss.

## Canonical example

```bash
# scripts/build-sync.sh — environment-agnostic; run from the repo root.
# Uses the project-local ./gradlew only. No absolute or machine-specific paths.
set -euo pipefail

echo "==> Pre-flight: JDK (expect 17 or 21)"
java -version 2>&1 | grep -E 'version "(17|21)' || { echo "JDK 17/21 required"; exit 1; }

echo "==> Pre-flight: Gradle wrapper present + executable"
test -x ./gradlew || { echo "./gradlew missing — run 'gradle wrapper'"; exit 1; }

echo "==> 1/4 ktlint format";  ./gradlew ktlintFormat
echo "==> 2/4 detekt";          ./gradlew detekt
echo "==> 3/4 Android Lint";    ./gradlew lintDebug
echo "==> 4/4 assembleDebug";   ./gradlew assembleDebug
echo "==> All gates passed."
```

```kotlin
// build.gradle.kts (root) — plugin + detekt baseline wiring (versions are illustrative; pin yours)
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.1" apply false
    id("io.gitlab.arturbosch.detekt") version "1.23.7" apply false
}
subprojects {
    apply(plugin = "org.jlleitschuh.gradle.ktlint")
    apply(plugin = "io.gitlab.arturbosch.detekt")
    extensions.configure<io.gitlab.arturbosch.detekt.extensions.DetektExtension> {
        baseline = file("$rootDir/detekt-baseline.xml")   // relative to repo root — portable
        config.setFrom("$rootDir/detekt.yml")
    }
}
```

```properties
# gradle.properties — reproducible, fast, portable
org.gradle.parallel=true
org.gradle.caching=true
# Do NOT hardcode org.gradle.java.home to a machine path; let JAVA_HOME / toolchains resolve it.
```

> **Portability note:** never set `org.gradle.java.home` to an absolute path in a committed
> `gradle.properties` — it breaks every other machine and CI. Use Gradle JVM toolchains or the ambient
> `JAVA_HOME` instead. Likewise keep all script paths relative to the repo root.

## Decision aids

- **Hook split:** run the fast stages (`ktlintFormat` + `detekt`) in a `pre-commit` hook for tight
  feedback, and the slow stages (`lintDebug` + `assembleDebug`) in `pre-push` or CI. detekt documents the
  pre-commit pattern directly.
- **ktlint vs Spotless?** Plain `ktlint` is simplest; **Spotless** wraps ktlint and adds non-Kotlin
  formatting (XML, Markdown) at the cost of more config. Default to ktlint; adopt Spotless only if you
  need multi-language formatting.
- **Single aggregate task?** You can define a `./gradlew syncCheck` that `dependsOn` all four — cleaner to
  invoke but harder to bootstrap on an existing project. The shell script above is the portable starting
  point.

## Related skills

- `global-skills/android/android-testing-patterns/SKILL.md` — the test stage that runs after this gate
  (this skill stops at `assembleDebug`; tests are a separate `./gradlew test` step).
- `global-skills/claude-code-workflow/capture-lessons-cascade-android/SKILL.md` — captures R8 / Compose /
  Hilt-scoping lessons surfaced by builds gated here.

## Sources

- [Android Lint](https://developer.android.com/studio/write/lint) ·
  [Enable app optimization with R8 / shrink code](https://developer.android.com/studio/build/shrink-code) ·
  [AGP releases & JDK compatibility](https://developer.android.com/build/releases/gradle-plugin)
- [detekt](https://detekt.dev/) ·
  [detekt git pre-commit hook](https://detekt.dev/docs/gettingstarted/git-pre-commit-hook/) ·
  [ktlint Gradle plugin](https://github.com/JLLeitschuh/ktlint-gradle) ·
  [compose-rules (detekt)](https://github.com/mrmans0n/compose-rules)

---

**Last verified:** 2026-06-03 against developer.android.com (Android Lint variant tasks; AGP 8.x JDK 17
minimum), detekt 1.23.x, ktlint-gradle 12.x.
**Re-check after:** next Claude Code minor release, or by 2026-09-03. **Decay risk:** medium (Gradle
plugin versions move; the pipeline shape is stable).
**Found a drift?** Run `/skill-pattern-freshness-audit claude-code-workflow`.
