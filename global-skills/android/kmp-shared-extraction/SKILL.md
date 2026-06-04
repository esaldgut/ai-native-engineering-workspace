---
name: kmp-shared-extraction
description: >-
  CROSS-PLATFORM skill (requires BOTH a native Android app AND a native iOS app). Extract shared business
  logic — domain models, UseCases, validation, and optionally Ktor networking / kotlinx.serialization /
  SQLDelight persistence — into a Kotlin Multiplatform commonMain module, abstract platform APIs with
  expect/actual, and build an XCFramework (XCFramework() Kotlin DSL, baseName="Shared", iosArm64() +
  iosSimulatorArm64(), ./gradlew assembleSharedXCFramework) for iOS consumption. Includes the
  what-to-share-vs-keep-native matrix. Use when deduplicating logic across an existing native Android+iOS pair.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Kotlin — Multiplatform overview (Stable since Nov 2023; share business logic, native UI)"
      url: "https://kotlinlang.org/docs/multiplatform.html"
      version: "Kotlin 2.3.x"
    - source: "Kotlin — Build final native binaries (XCFramework, iosArm64/iosSimulatorArm64, assemble<Name>XCFramework)"
      url: "https://kotlinlang.org/docs/multiplatform/multiplatform-build-native-binaries.html"
      version: "Kotlin 2.3.x"
    - source: "Kotlin — Expected and actual declarations (expect/actual across commonMain/androidMain/iosMain)"
      url: "https://kotlinlang.org/docs/multiplatform/multiplatform-expect-actual.html"
      version: "Kotlin 2.3.x"
    - source: "Android (Google) — Kotlin Multiplatform (officially supported for shared business logic)"
      url: "https://developer.android.com/kotlin/multiplatform"
      version: "Google I/O 2024+"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Google I/O (May) + Compose BOM major"
    or_date: "2026-12-01"
  decay_risk: medium
  status: current
---

# KMP shared extraction (Android ⇄ iOS shared logic)

> **Prerequisite — read first.** This is a **cross-platform** skill. It applies only when you have
> **both** a native Android app **and** a native iOS app (SwiftUI/UIKit) and want to stop maintaining the
> same business logic twice. It is **not** a "rewrite the app in KMP" skill. The output is a shared
> Kotlin module consumed as a normal Gradle dependency on Android and as an **XCFramework** on iOS, while
> **each platform keeps its own native UI.**

[Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html) is **Stable since November 2023**
and was endorsed by Google at I/O 2024 for sharing business logic between Android and iOS. The canonical
pattern for a native-pair app is **KMP for logic + native UI per platform** — *not* Compose Multiplatform
for UI (which is Stable on iOS as of May 2025 but is a different, larger decision).

## When to invoke

- You have a native Android app and a native iOS app and the same domain model / validation / API client
  is implemented twice and drifting.
- You're standing up a `shared` KMP module and need the `commonMain` / `androidMain` / `iosMain` source
  sets, the `expect`/`actual` seams, and the XCFramework Gradle wiring.

**Announce on invoke:** "Using `kmp-shared-extraction` to move shared business logic into a KMP
`commonMain` module and emit an XCFramework for the iOS app — native UI stays on each platform."

## What to share vs keep native (the matrix)

| Layer | `commonMain`? | How |
|---|---|---|
| Domain entities (`data class`) | **YES** | pure Kotlin |
| UseCases / business logic | **YES** | pure Kotlin |
| Validation rules | **YES** | pure Kotlin |
| Networking (HTTP) | **YES** | **Ktor** — engines via `expect/actual` (OkHttp on Android, Darwin on iOS) |
| Serialization | **YES** | **kotlinx.serialization** |
| Persistence (SQL) | **YES** | **SQLDelight** (Android `AndroidSqliteDriver`, iOS `NativeSqliteDriver`) |
| Date/time | **YES** | **kotlinx-datetime** |
| Dependency injection | **YES** | **Koin** (Hilt is JVM-only — see below) |
| **UI** (Compose / SwiftUI) | **NO** (native-UI projects) | each platform keeps its native UI |
| Platform APIs (Keychain, biometrics, push, camera) | **NO** directly | abstract behind `expect/actual` |
| Localized UI strings | **NO** | share **keys** only; strings live in Android resources / `Localizable.strings` |

## The rules (load-bearing)

1. **`expect`/`actual` is the platform seam.** Declare `expect fun getPlatform(): Platform` (no body) in
   `commonMain`; provide `actual fun getPlatform(): Platform = AndroidPlatform()` in `androidMain` and
   `= IOSPlatform()` in `iosMain`. The compiler enforces an `actual` in every target for every `expect`.
2. **Hilt does NOT work in `commonMain`.** Hilt is JVM/Android-only. Use **Koin** (or manual factories)
   for the shared module's DI. The Android app can still use Hilt for Android-only deps and bridge to the
   Koin-provided shared singletons.
3. **Provide both current iOS targets: `iosArm64()` (devices) AND `iosSimulatorArm64()` (Apple-silicon
   simulator).** A module with only `iosArm64()` + `iosX64()` (Intel simulator) fails to link on M-series
   Macs. `iosX64()` is rarely needed in 2026.
4. **`baseName` must be a valid Swift module identifier and drives the Gradle task.** `baseName =
   "Shared"` ⇒ Swift `import Shared` ⇒ the assemble task is `./gradlew assembleSharedXCFramework`.
   (Declaring an XCFramework registers `assemble<Name>XCFramework` plus `…Debug`/`…Release` variants.)
   Naming the framework after the gradle module ("shared-domain") is an invalid Swift name.
5. **The new memory model is the default (Kotlin 1.7.20+).** Pre-2023 guidance about `freeze()` /
   `InvalidMutabilityException` for coroutines on iOS is stale — ignore it.
6. **Kotlin↔Swift bridging has sharp edges.** A Kotlin `companion object` surfaces as `Shared.companion`
   in Swift, `sealed` classes need `as?` casts, and Kotlin `Result<T>` does not bridge cleanly — keep the
   exported API surface in plain types (data classes, enums, `suspend` funcs map to Swift `async`).

## Canonical example

```kotlin
// commonMain/.../domain/MyItem.kt — shared, pure Kotlin
data class MyItem(val id: String, val title: String)

// commonMain/.../usecase/FetchItemUseCase.kt
class FetchItemUseCase(private val api: MyApi) {
    suspend operator fun invoke(id: String): MyItem = api.fetchItem(id)
}

// commonMain/.../platform/Logger.kt — the expect seam
expect class Logger() {
    fun log(message: String)
}

// androidMain/.../platform/Logger.android.kt
actual class Logger {
    actual fun log(message: String) = android.util.Log.d("Shared", message)
}

// iosMain/.../platform/Logger.ios.kt
actual class Logger {
    actual fun log(message: String) = platform.Foundation.NSLog("Shared: %@", message)
}
```

```kotlin
// shared/build.gradle.kts — XCFramework wiring
import org.jetbrains.kotlin.gradle.plugin.mpp.apple.XCFramework

plugins {
    kotlin("multiplatform")
    id("com.android.library")
}

kotlin {
    androidTarget()
    val xcf = XCFramework("Shared")
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            xcf.add(this)
        }
    }
    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.kotlinx.serialization.json)
            implementation(libs.kotlinx.datetime)
        }
        androidMain.dependencies { implementation(libs.ktor.client.okhttp) }
        iosMain.dependencies { implementation(libs.ktor.client.darwin) }
    }
}
// Build: ./gradlew assembleSharedXCFramework  ->  build/XCFrameworks/release/Shared.xcframework
```

## Decision aids

- **DI: Koin vs manual factories?** Koin is the canonical KMP DI and reads cleanly across platforms;
  manual factories avoid a dependency. Default to Koin once the shared graph has more than a couple of
  objects.
- **iOS consumption: direct Xcode drop vs SPM?** Dropping `Shared.xcframework` into the Xcode project is
  simplest to bootstrap; **Swift Package Manager** (via a community plugin like
  `multiplatform-swiftpackage`) is the modern distribution path for a versioned shared module. Show the
  direct drop first; graduate to SPM when versioning matters.
- **Compose Multiplatform for shared UI?** Out of scope here — this skill's line is firm: shared **logic**,
  native UI. CMP is a separate, larger decision; revisit only if both teams want to converge UI.

## Related skills

- `global-skills/android/compose-clean-architecture-module-scaffold/SKILL.md` — the Android-side Clean
  Architecture layers; the Domain layer is exactly what migrates into `commonMain` here.
- `global-skills/apple/swift-module/SKILL.md` (iOS twin scaffold) — the iOS side consuming the
  `import Shared` XCFramework.
- `global-skills/android/android-security-checklist/SKILL.md` — Keychain/Keystore are `expect/actual`
  platform APIs (kept native), referenced in the matrix above.

## Sources

- [Kotlin Multiplatform (stable)](https://kotlinlang.org/docs/multiplatform.html) ·
  [Build final native binaries (XCFramework)](https://kotlinlang.org/docs/multiplatform/multiplatform-build-native-binaries.html) ·
  [Expected and actual declarations](https://kotlinlang.org/docs/multiplatform/multiplatform-expect-actual.html)
- [Google — Kotlin Multiplatform](https://developer.android.com/kotlin/multiplatform) ·
  [Ktor client](https://ktor.io/docs/client.html) · [SQLDelight](https://cashapp.github.io/sqldelight/)

---

**Last verified:** 2026-06-03 against kotlinlang.org (KMP Stable, XCFramework DSL, expect/actual) +
developer.android.com (Google KMP support). XCFramework registers `assemble<Name>XCFramework`;
`iosArm64()` + `iosSimulatorArm64()` confirmed as the current target pair.
**Re-check after:** Google I/O 2026 + next Compose BOM major (and any Kotlin major), or by 2026-12-01.
**Decay risk:** medium (KMP surface is stable; tooling/distribution evolves).
**Found a drift?** Run `/skill-pattern-freshness-audit android`.
