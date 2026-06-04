# Xcode CLI Workflow - Apple Platforms Development

> **📐 Documentación base de plataforma — genérica, no vinculada a proyecto.**
> Cubre el toolchain Apple/iOS de esta máquina y patrones reutilizables de Xcode CLI.
> Cada proyecto Apple particular **extiende** esta base (ver
> [PLATFORM_BASE.md](./PLATFORM_BASE.md)). Nada aquí debe contener bundle IDs,
> team IDs, nombres de schemes ni arquitectura de un repo concreto.

**Stack:** Xcode 26.4.1 + Swift 6.3.1 + iOS 26.4 SDK + XcodeBuildMCP + mcp__xcode
**Hardware:** MacBook Pro M4 Pro (48GB unified memory, 14 cores)
**Filosofía:** CLI-first, simulador-first, MCP-driven (sin abrir Xcode.app salvo storyboards/asset catalogs)

Este es el **par iOS** de [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md). Un proyecto que
busca paridad iOS/Android nativa usa ambas bases + KMP para el código compartido.

---

## 📋 Resumen Ejecutivo

Workflow profesional para desarrollo Apple **sin abrir Xcode.app** en el 80% de los casos:

- ✅ **Build & test** desde terminal (xcodebuild + xcbeautify)
- ✅ **Simuladores** controlados por simctl + XcodeBuildMCP (snapshot, tap, swipe, screenshot)
- ✅ **Code signing** con identities Distribution + Development
- ✅ **MCP integration** para que Claude opere Xcode directamente (`mcp__xcode__BuildProject`, `mcp__XcodeBuildMCP__*`)
- ✅ **Documentación oficial** en línea via `mcp__apple-docs__*` (busca WWDC, símbolos, frameworks)

**Ganancia de productividad:** Build + test + screenshot loop **5-10x más rápido** que clicks en Xcode.app

---

## 🎯 Sistema Base

### Toolchain Instalado

| Componente | Versión | Ubicación |
|-----------|---------|-----------|
| Xcode | **26.4.1** (Build 17E202) | `/Applications/Xcode.app` (5.0 GB) |
| Swift | **6.3.1** (swiftlang-6.3.1.1.2) | Embebido en Xcode |
| Swift Driver | 1.148.6 | Target: arm64-apple-macosx26.0 |
| Command Line Tools | Activos | `/Applications/Xcode.app/Contents/Developer` |
| sourcekit-lsp | System | `/usr/bin/sourcekit-lsp` |
| swiftformat | 0.61.1 | `/opt/homebrew/bin/swiftformat` |
| swiftlint | 0.63.2 | `/opt/homebrew/bin/swiftlint` |
| xcbeautify | 3.2.1 | `/opt/homebrew/bin/xcbeautify` |

### SDKs Disponibles

```
iOS 26.4              -sdk iphoneos26.4
iOS Simulator 26.4    -sdk iphonesimulator26.4
macOS 26.4            -sdk macosx26.4
tvOS 26.4             -sdk appletvos26.4
watchOS               (vía Xcode)
visionOS              (vía Xcode)
DriverKit 25.4        -sdk driverkit25.4
```

### Storage Footprint

```
Xcode.app:                                5.0 GB
$HOME/Library/Developer/Xcode/DerivedData: ~600 MB  (limpiar mensualmente)
$HOME/.swiftpm/ (symlinks):               < 1 KB
```

---

## 📱 Inventario de Simuladores

### Runtimes Instalados

```
iOS 18.5    (22F77)     ← Compatibilidad con apps legacy
iOS 26.0    (23A343)    ← Producción actual
iOS 26.1    (23B86)
iOS 26.2    (23C54)
iOS 26.4    (23E244)    ← SDK match
iOS 26.4.1  (23E254a)
watchOS 11.5 (22T572)
watchOS 26.0 (23R353)
watchOS 26.2 (23S303)
```

### Devices Activos

| Categoría | Modelos disponibles |
|-----------|---------------------|
| **iPhone iOS 18.5** | iPhone 16, 16 Plus, 16 Pro, 16 Pro Max, 16e |
| **iPhone iOS 26.x** | iPhone 17, 17 Pro, 17 Pro Max, iPhone Air, 16e |
| **iPad iOS 18.5** | Pro 11"/13" (M4), Air 11"/13" (M3), mini (A17 Pro), iPad (A16) |
| **iPad iOS 26.x** | Pro 11"/13" (M4 + M5), Air 11"/13" (M3), mini, iPad |

### Comandos simctl Esenciales

```bash
# Ver simuladores booted
xcrun simctl list devices booted

# Boot un device específico
xcrun simctl boot "iPhone 17 Pro"

# Open Simulator app
open -a Simulator

# Screenshot del simulador booted
xcrun simctl io booted screenshot $HOME/Desktop/sim-shot-$(date +%H%M%S).png

# Grabar video
xcrun simctl io booted recordVideo $HOME/Desktop/sim-rec.mov

# Install + launch app
xcrun simctl install booted /path/to/MyApp.app
xcrun simctl launch booted com.example.app

# Geolocation
xcrun simctl location booted set 19.4326,-99.1332  # CDMX

# Push notification
xcrun simctl push booted com.example.app payload.json

# Erase (reset al estado inicial)
xcrun simctl erase "iPhone 17 Pro"

# Shutdown all
xcrun simctl shutdown all
```

---

## 🔐 Code Signing & Distribution

### Identities (verificar las que existen)

```bash
# Listar identities de code signing instaladas
security find-identity -p codesigning -v
```

Patrón típico de un setup profesional:

```
1) Apple Distribution: <APPLE_DEV_NAME> (<TEAM_ID>)
   Uso: App Store / TestFlight / Ad-Hoc

2) Apple Development: <email> (<DEV_TEAM_ID>)
   Uso: Desarrollo local + simulador
```

> Los valores reales (nombre del developer, Team ID, hashes de certificados) son
> **específicos de la cuenta** → van en `docs/APPLE_EXTENSIONS.md` del proyecto +
> Keychain, nunca en esta base ni commiteados.

### Provisioning Profiles

```
$HOME/Library/MobileDevice/Provisioning Profiles/
```

### Workflow de distribución

```bash
# Verificar identities antes de archivar
security find-identity -p codesigning -v

# Refrescar profiles desde Apple ID
xcrun altool --list-providers -u "you@example.com" -p "@keychain:AC_PASSWORD"

# Validar archive
xcrun altool --validate-app -f MyApp.ipa -t ios -u "you@example.com" -p "@keychain:AC_PASSWORD"

# Subir a TestFlight/App Store
xcrun altool --upload-app -f MyApp.ipa -t ios -u "you@example.com" -p "@keychain:AC_PASSWORD"
```

---

## 🛠️ xcodebuild — Comandos Core

> Ejemplos con un proyecto genérico `MyApp.xcodeproj` (scheme `MyApp`,
> bundle `com.example.app`). Sustituye por los nombres reales de tu proyecto.

### List Schemes & Configurations

```bash
# Schemes en proyecto
xcodebuild -list -project MyApp.xcodeproj

# Schemes en workspace
xcodebuild -list -workspace MyApp.xcworkspace

# Build settings de un scheme
xcodebuild -showBuildSettings -project MyApp.xcodeproj -scheme MyApp \
  | grep -E "PRODUCT_NAME|BUNDLE_ID|VERSION"
```

### Build (con xcbeautify para output limpio)

```bash
# Build para simulador (rápido, sin code signing)
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro,OS=26.0' \
  | xcbeautify

# Build para device (requiere code signing)
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'generic/platform=iOS' \
  -allowProvisioningUpdates \
  | xcbeautify

# Clean build
xcodebuild clean build \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  | xcbeautify
```

### Test

```bash
# Run all tests
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro,OS=26.0' \
  | xcbeautify

# Run tests con coverage
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -enableCodeCoverage YES \
  -resultBundlePath ./TestResults.xcresult \
  | xcbeautify

# Run un test específico
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:MyAppTests/UserModelTests/testCreateUser \
  | xcbeautify

# Ver coverage report
xcrun xccov view --report ./TestResults.xcresult
xcrun xccov view --report --json ./TestResults.xcresult > coverage.json
```

### Archive & Export

```bash
# Crear archive (.xcarchive)
xcodebuild archive \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -archivePath ./build/MyApp.xcarchive \
  -destination 'generic/platform=iOS' \
  -allowProvisioningUpdates \
  | xcbeautify

# Export .ipa desde archive (requiere ExportOptions.plist)
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportPath ./build/ \
  -exportOptionsPlist ExportOptions.plist \
  -allowProvisioningUpdates \
  | xcbeautify
```

**ExportOptions.plist mínimo (App Store):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>          <string>app-store-connect</string>
  <key>destination</key>     <string>upload</string>
  <key>teamID</key>          <string>&lt;TEAM_ID&gt;</string>
  <key>uploadSymbols</key>   <true/>
  <key>signingStyle</key>    <string>automatic</string>
  <key>manageAppVersionAndBuildNumber</key>  <true/>
  <key>stripSwiftSymbols</key>               <true/>
</dict>
</plist>
```

> `manageAppVersionAndBuildNumber: true` deja que App Store Connect asigne build
> numbers automáticamente — útil para CI sin colisiones.

---

## 🤖 Integración MCP (Claude opera Xcode)

> **Filosofía:** Claude Code tiene 2 stacks de tools para Apple. Conoce las diferencias.
> Ver inventario completo en [MCP_WORKFLOW.md](./MCP_WORKFLOW.md).

### XcodeBuildMCP (genérico, multi-proyecto)

**Cuándo usar:** Proyectos sin abrir en Xcode IDE (Swift packages, Expo prebuild, multi-target).

**Setup de sesión** (obligatorio antes del primer build):
```
mcp__XcodeBuildMCP__session_show_defaults
mcp__XcodeBuildMCP__session_set_defaults  ← project + scheme + simulator
```

**Tools clave:**
| Tool | Uso |
|------|-----|
| `discover_projs` | Encontrar .xcodeproj/.xcworkspace |
| `list_schemes` | Schemes del proyecto |
| `list_sims` | Simuladores disponibles |
| `boot_sim` | Bootear simulador |
| `build_sim` | Build sin lanzar |
| `build_run_sim` | Build + install + launch |
| `test_sim` | xcodebuild test wrapper |
| `screenshot` | Captura del simulador |
| `snapshot_ui` | Jerarquía de vistas con coords |
| `get_coverage_report` | Coverage tras test |
| `get_app_bundle_id` | Bundle ID desde el .app |

**Workflow típico:**
```
1. session_show_defaults     → verificar config
2. boot_sim "iPhone 17 Pro"  → si no está booted
3. build_run_sim             → build + install + launch
4. snapshot_ui               → ver elementos con coords
5. screenshot                → evidencia visual
```

### mcp__xcode (cuando Xcode IDE está abierto)

**Cuándo usar:** Proyectos abiertos en Xcode.app (xcodeproj + signing manual + previews).

**Tools clave:**
| Tool | Uso |
|------|-----|
| `BuildProject` | Build del proyecto activo en Xcode |
| `RunAllTests` / `RunSomeTests` | Test runner integrado |
| `GetTestList` | Lista de tests |
| `GetBuildLog` | Log del último build |
| `XcodeListNavigatorIssues` | Errores/warnings en Navigator |
| `XcodeRefreshCodeIssuesInFile` | Re-trigger diagnósticos |
| `RenderPreview` | Render de SwiftUI Preview |
| `ExecuteSnippet` | Run código Swift ad-hoc |
| `XcodeRead` / `XcodeWrite` / `XcodeUpdate` | File ops dentro del project tree |
| `XcodeGlob` / `XcodeGrep` | Búsqueda dentro del proyecto |
| `XcodeListWindows` | Ventanas Xcode abiertas |
| `DocumentationSearch` | Search en docs Apple desde Xcode |

**Workflow típico:**
```
1. XcodeListWindows           → confirmar proyecto activo
2. BuildProject               → build
3. XcodeListNavigatorIssues   → revisar warnings
4. RunSomeTests               → tests específicos
5. RenderPreview              → ver SwiftUI live
```

> **Gotcha útil:** El listado de warnings del CLI puede divergir de los warnings que
> muestra Xcode en su Navigator. Cuando importe (p.ej. pre-release), verificar con
> `mcp__xcode__XcodeListNavigatorIssues` además del output de `xcodebuild`.

### apple-docs MCP

**Tools clave:**
- `search_apple_docs` — búsqueda full-text
- `get_apple_doc_content` — contenido de un símbolo/framework
- `search_framework_symbols` — todos los símbolos de un framework
- `get_related_apis` / `find_similar_apis` — descubrir APIs relacionadas
- `list_wwdc_videos` / `get_wwdc_video` / `search_wwdc_content` / `get_wwdc_code_examples`
- `find_related_wwdc_videos` — para una API, qué videos la cubren
- `get_documentation_updates` — qué cambió en docs recientemente
- `get_platform_compatibility` — iOS/macOS/visionOS support por API
- `get_sample_code` — proyectos sample oficiales
- `resolve_references_batch` — resolver múltiples refs en una llamada

**Caso de uso real:**
```
> "¿Cómo uso async/await con URLSession en iOS 18+?"
  → search_apple_docs "URLSession async"
  → get_apple_doc_content del símbolo
  → find_related_wwdc_videos
  → get_wwdc_code_examples
```

---

## 🚀 Workflows Profesionales

### 1. Nueva Feature SwiftUI (5 min)

```bash
# 1. Crear branch
git checkout -b feature/user-profile

# 2. Editar en Neovim (LSP via sourcekit-lsp)
nvim MyApp/Views/UserProfileView.swift

# 3. Build + run sin tocar Xcode
xcodebuild build \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  | xcbeautify

# 4. Si build pasa, install + launch
xcrun simctl install booted \
  $HOME/Library/Developer/Xcode/DerivedData/MyApp-*/Build/Products/Debug-iphonesimulator/MyApp.app
xcrun simctl launch booted com.example.app

# 5. Screenshot para PR
xcrun simctl io booted screenshot $HOME/Desktop/feature-shot.png

# 6. Commit (p.ej. \gg en Neovim → LazyGit)
```

### 2. TDD Loop (Swift Testing / XCTest)

```bash
# 1. Escribir test
nvim MyAppTests/UserModelTests.swift

# 2. Run solo ese test (rápido)
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:MyAppTests/UserModelTests/testCreateUser \
  | xcbeautify

# 3. Implementar
nvim MyApp/Models/User.swift

# 4. Re-run + coverage
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -enableCodeCoverage YES \
  -resultBundlePath /tmp/coverage.xcresult \
  | xcbeautify

xcrun xccov view --report /tmp/coverage.xcresult | grep User
```

### 3. UI Automation con MCP

**Escenario:** Verificar que un botón navegue correctamente.

```
1. mcp__XcodeBuildMCP__build_run_sim       → app corriendo
2. mcp__XcodeBuildMCP__snapshot_ui         → JSON con coords de todos los elementos
3. Identificar coord del botón "Continue"  → (200, 540)
4. mcp__XcodeBuildMCP__screenshot          → estado inicial
5. (tap via XcodeBuildMCP UI tools)
6. mcp__XcodeBuildMCP__screenshot          → estado post-tap
7. (stream de logs)                        → confirmar log "Navigated to Profile"
```

### 4. Distribuir a TestFlight (15 min)

```bash
# 1. Bump version + build number
agvtool bump -all
agvtool new-marketing-version 1.2.0

# 2. Archive
xcodebuild archive \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -archivePath ./build/MyApp.xcarchive \
  -destination 'generic/platform=iOS' \
  -allowProvisioningUpdates \
  | xcbeautify

# 3. Export .ipa (ExportOptions.plist gestiona signing automático + upload)
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportPath ./build/ \
  -exportOptionsPlist ExportOptions.plist \
  -allowProvisioningUpdates \
  | xcbeautify

# 4. Upload a App Store Connect (si no usaste destination=upload)
xcrun altool --upload-app \
  -f ./build/MyApp.ipa \
  -t ios \
  -u "you@example.com" \
  -p "@keychain:ASC_API_KEY"

# 5. Verificar en App Store Connect (web)
open https://appstoreconnect.apple.com
```

### 5. Debugging con Logs en Vivo

```bash
# Stream logs del simulador (filtrado por bundle)
xcrun simctl spawn booted log stream \
  --predicate 'processImagePath endswith "MyApp"' --style compact

# Logs estructurados (Console.app style)
xcrun simctl spawn booted log show \
  --predicate 'subsystem == "com.example.app"' --last 5m

# Logs de un device físico
xcrun devicectl device console --device <UDID>
```

### 6. Comparar comportamiento entre versiones de iOS

```bash
# Boot ambos simuladores
xcrun simctl boot "iPhone 16 Pro"      # iOS 18.5
xcrun simctl boot "iPhone 17 Pro"      # iOS 26.0

# Build & install en ambos (UDIDs reales de tu `simctl list`)
APP=$HOME/Library/Developer/Xcode/DerivedData/MyApp-*/Build/Products/Debug-iphonesimulator/MyApp.app

xcrun simctl install <UDID_iOS18> $APP
xcrun simctl install <UDID_iOS26> $APP

xcrun simctl launch <UDID_iOS18> com.example.app
xcrun simctl launch <UDID_iOS26> com.example.app

# Screenshots de ambos para diff visual
xcrun simctl io <UDID_iOS18> screenshot $HOME/Desktop/ios18.png
xcrun simctl io <UDID_iOS26> screenshot $HOME/Desktop/ios26.png
```

### 7. WWDC research desde terminal

```
"Necesito implementar Live Activities para iOS 26"

→ mcp__apple-docs__search_wwdc_content "Live Activities"
→ mcp__apple-docs__list_wwdc_videos --topic "ActivityKit"
→ mcp__apple-docs__get_wwdc_video --id "<wwdc-video-id>"
→ mcp__apple-docs__get_wwdc_code_examples --videoId "<wwdc-video-id>"
→ mcp__apple-docs__get_apple_doc_content --path "ActivityKit/Activity"
→ mcp__apple-docs__get_platform_compatibility --api "Activity"
```

### 8. Limpiar Caché (mensual)

```bash
# DerivedData (build artifacts)
rm -rf $HOME/Library/Developer/Xcode/DerivedData/*

# Module cache
rm -rf $HOME/Library/Developer/Xcode/iOS\ DeviceSupport/

# SwiftPM cache
rm -rf $HOME/Library/Caches/org.swift.swiftpm

# Simuladores no usados
xcrun simctl delete unavailable
```

**Espacio recuperable típico:** 5-15 GB

---

## 🏗️ XcodeGen (project.yml como source of truth) — Patrón Recomendado

Para evitar merge conflicts en `.xcodeproj` y tener un proyecto reproducible, un patrón
maduro es generar el `.xcodeproj` desde un `project.yml` declarativo con
[XcodeGen](https://github.com/yonaskolb/XcodeGen). **El `.xcodeproj` no se edita a mano.**

**Flujo:** Editas `project.yml` → `xcodegen generate` → `.xcodeproj` se regenera → buildeas.

**`project.yml` genérico (placeholders donde aplique):**

```yaml
options:
  xcodeVersion: "26.2"
  deploymentTarget:
    iOS: "26.4"
  bundleIdPrefix: com.example

configFiles:
  Debug:   MyApp/Config/Dev.xcconfig
  Release: MyApp/Config/Prod.xcconfig

settings:
  base:
    DEVELOPMENT_TEAM: <TEAM_ID>
    SWIFT_VERSION: "5.0"
    SWIFT_APPROACHABLE_CONCURRENCY: YES
    SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor      # ← Swift 6 strict concurrency
    TARGETED_DEVICE_FAMILY: "1,2"                 # iPhone + iPad
    LOCALIZATION_PREFERS_STRING_CATALOGS: YES

targets:
  MyApp:
    preBuildScripts:
      - script: swiftformat "$SRCROOT/MyApp"      # ← format pre-compile
        name: SwiftFormat
        basedOnDependencyAnalysis: false
    postCompileScripts:
      - script: swiftlint                          # ← lint post-compile
```

> Los valores reales (`DEVELOPMENT_TEAM`, `bundleIdPrefix`, nombre del target,
> módulos) son **específicos del proyecto** → viven en su repo, no en esta base.

---

## 🧹 Code Quality

### swiftformat (auto-format)

```bash
# Format un archivo
swiftformat MyApp/Models/User.swift

# Format todo el proyecto (config en .swiftformat)
swiftformat .

# Dry-run (mostrar cambios sin aplicar)
swiftformat --dryrun .
```

**`.swiftformat` recomendado** (proyecto root):
```
--swiftversion 6.0
--indent 2
--smarttabs enabled
--maxwidth 120
--wraparguments before-first
--wrapparameters before-first
--wrapcollections before-first
--self remove
--importgrouping testable-last      # @testable al final (Apple convention)
```

### swiftlint (linter)

```bash
# Lint
swiftlint

# Auto-fix issues que se pueden arreglar
swiftlint --fix

# Solo errores (sin warnings)
swiftlint --strict
```

**`.swiftlint.yml` con opt-in rules para Swift 6 strict concurrency:**
```yaml
disabled_rules:
  - trailing_whitespace
  - line_length
  - trailing_comma          # lo maneja SwiftFormat
opt_in_rules:
  # Concurrency
  - async_without_await
  - unhandled_throwing_task
  # SwiftUI
  - private_swiftui_state
  # Quality
  - force_unwrapping
  - direct_return
  - redundant_type_annotation
  - prefer_self_in_static_references
included:
  - MyApp
excluded:
  - Carthage
  - Pods
  - .build
```

> **Patrón recomendado:** SwiftFormat como build phase **pre-build** (formato consistente
> antes de compilar) + SwiftLint **post-compile** (warnings nativos en Xcode).

### xcbeautify (output legible)

```bash
# Pipe cualquier output de xcodebuild
xcodebuild ... | xcbeautify

# Renderizar como reporter para CI
xcodebuild test ... | xcbeautify --renderer github-actions
```

---

## 🧩 Convenciones Swift 6.3 (genéricas, reutilizables)

Patrones de concurrencia que aplican a cualquier proyecto con
`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` (Swift 6.3 strict concurrency). Estos son
"compiler-verified gotchas" del SDK/lenguaje — no son específicos de ningún proyecto:

| Patrón | Regla |
|---|---|
| **`nonisolated` explícito** | Con `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, **TODO tipo** en capas no-UI (modelos, repos, infra) DEBE marcarse `nonisolated` explícitamente (SE-0449) — structs, enums, protocolos y extensiones incluidos. Sin esto el compilador aplica `@MainActor` y los singletons `.shared` rompen con *"call to main actor-isolated initializer"*. |
| **Extensions no heredan** | `extension Foo` necesita su propio `nonisolated extension Foo` — la isolation no se hereda del tipo base. Genera cascade warnings invisibles en CLI. |
| **SE-0478 roto en 26.4** | El typealias `DefaultIsolation` (workaround `private typealias DefaultIsolation = nonisolated`) **NO compila** en Xcode 26.4 → usar `nonisolated` per-type. |
| **ViewModels** | `@MainActor @Observable final class` consumidos vía `@State` (no `@StateObject`/`@ObservedObject`). |
| **Core services** | `nonisolated final class: Sendable`; usar `actor` solo si hay estado mutable real (p.ej. un monitor con `NWPathMonitor`). |
| **Domain models** | `nonisolated struct: Sendable`. |
| **Repository protocols** | `nonisolated protocol: Sendable` con métodos `async`. |

Otras convenciones de código genéricas:

- **Indent:** 2 espacios (gestionado por SwiftFormat)
- **Trailing commas:** Siempre en colecciones multiline (SwiftFormat)
- **Import ordering:** Imports normales primero, `@testable` al final (`--importgrouping testable-last`)
- **Doc comments:** `///` con `- Parameter:`, `- Returns:`, `- Throws:` (DocC)
- **`@unknown default`** obligatorio en switches sobre Apple SDK / third-party enums. No requerido en enums internos del proyecto (frozen by definition)
- **Keychain:** TODA data sensible vía un `KeychainManager`. NUNCA `UserDefaults` para secrets
- **Storage by sensitivity:** `@AppStorage` solo para no-sensible (keys centralizadas en un `nonisolated enum`, nunca literals inline); `KeychainManager` para todo lo sensible
- **Unit tests:** Swift Testing (`import Testing`, `@Test`); **UI tests:** XCTest (`XCUIApplication`)

> Gotchas adicionales de Swift Testing observados en la práctica: los error enums necesitan
> `Equatable` para `#expect(throws:)`; las test functions con `await` deben ser
> `async throws`; `INFOPLIST_KEY_*` solo funciona con keys Apple-predefinidas (custom keys
> necesitan `Info.plist` con `$(VARIABLE)`).

---

## 🐛 Troubleshooting

### "No suitable application records were found"

```bash
# Refrescar provisioning profiles
xcodebuild -allowProvisioningUpdates -project ...

# O manual
open $HOME/Library/MobileDevice/Provisioning\ Profiles/
# Borrar profiles caducados, luego refrescar desde Xcode > Settings > Accounts > Download Manual Profiles
```

### Build lento (>1 min para incremental)

```bash
# 1. Limpiar DerivedData
rm -rf $HOME/Library/Developer/Xcode/DerivedData/*

# 2. Verificar tamaño de módulos
du -sh $HOME/Library/Developer/Xcode/DerivedData/*/ModuleCache.noindex/

# 3. Habilitar build timing summary
xcodebuild ... -showBuildTimingSummary | xcbeautify
```

### Simulador no bootea

```bash
# Reset Core Simulator service
sudo killall -9 com.apple.CoreSimulator.CoreSimulatorService
xcrun simctl shutdown all
xcrun simctl erase all

# Reabrir
open -a Simulator
```

### "Could not find module ... for target 'arm64-apple-ios-simulator'"

Arquitectura mismatch (proyecto buildeado para device, no simulator).

```bash
# Force clean + rebuild para sim
xcodebuild clean build \
  -destination 'generic/platform=iOS Simulator' \
  -arch arm64 \
  | xcbeautify
```

### sourcekit-lsp en Neovim no autocompleta

```bash
# Verificar que sourcekit-lsp está disponible
which sourcekit-lsp     # → /usr/bin/sourcekit-lsp

# Verificar que el proyecto tiene compile_commands.json o .build
swift package generate-xcodeproj    # solo SPM
# o
xcodebuild -project ... -scheme ... -showBuildSettings  # genera index store
```

> **Gap típico:** Si Neovim no tiene `sourcekit-lsp` configurado en `lua/config/lsp.lua`,
> agregar:
> ```lua
> vim.lsp.config.sourcekit = {
>   cmd = { 'sourcekit-lsp' },
>   filetypes = { 'swift', 'objective-c', 'objective-cpp' },
>   root_markers = { 'Package.swift', '*.xcodeproj', '*.xcworkspace' },
> }
> vim.lsp.enable('sourcekit')
> ```

---

## ⌨️ Aliases y Scripts Recomendados

Agregar a `~/.zshrc`:

```bash
# Xcode CLI shortcuts
alias xcb='xcodebuild | xcbeautify'
alias xclean='rm -rf $HOME/Library/Developer/Xcode/DerivedData/* && echo "DerivedData cleared"'
alias xcdevices='xcrun simctl list devices booted'
alias xcboot='xcrun simctl boot'
alias xcshot='xcrun simctl io booted screenshot $HOME/Desktop/sim-$(date +%Y%m%d-%H%M%S).png && echo "Saved to Desktop"'
alias xcrec='xcrun simctl io booted recordVideo $HOME/Desktop/sim-rec-$(date +%H%M%S).mov'
alias xcerase='xcrun simctl erase'
alias xclist='xcrun simctl list devices available'
alias xclogs='xcrun simctl spawn booted log stream --style compact'

# Build helpers
alias xcbuild-sim='xcodebuild build -destination "platform=iOS Simulator,name=iPhone 17 Pro" | xcbeautify'
alias xctest-sim='xcodebuild test -destination "platform=iOS Simulator,name=iPhone 17 Pro" | xcbeautify'

# Code signing
alias xccerts='security find-identity -p codesigning -v'
alias xcprofiles='ls $HOME/Library/MobileDevice/Provisioning\ Profiles/'

# Quality
alias xcfmt='swiftformat .'
alias xclint='swiftlint --strict'
alias xcfix='swiftlint --fix'

# Storage cleanup
alias xcclean-all='xclean && rm -rf $HOME/Library/Developer/Xcode/iOS\ DeviceSupport/* && xcrun simctl delete unavailable'
```

### Script: `~/bin/xc-new-feature.sh`

```bash
#!/bin/bash
# Bootstrap nueva feature: branch + boot sim + open Simulator
set -e

FEATURE=$1
[ -z "$FEATURE" ] && { echo "Usage: xc-new-feature <feature-name>"; exit 1; }

git checkout -b "feature/$FEATURE"
xcrun simctl boot "iPhone 17 Pro" 2>/dev/null || true
open -a Simulator
echo "✓ Branch created: feature/$FEATURE"
echo "✓ Simulator booted"
echo "→ Open files in Neovim and start coding"
```

---

## 📊 Métricas de Productividad

| Tarea | Antes (Xcode IDE) | Con CLI + MCP | Mejora |
|-------|-------------------|---------------|--------|
| Build + run en simulador | ~45s (UI lag + spinners) | ~12s (xcodebuild) | **3.7x** |
| Run un test específico | ~30s (click + run + tabs) | ~5s (-only-testing) | **6x** |
| Screenshot del simulador | ~15s (Cmd+S → save dialog) | <1s (simctl) | **15x** |
| Stream de logs filtrado | ~20s (Console.app + filtro) | ~3s (log stream) | **6.5x** |
| Cambiar simulador | ~10s (menú + boot) | ~3s (boot_sim) | **3.3x** |
| Buscar API en docs | ~25s (developer.apple.com) | ~4s (apple-docs MCP) | **6x** |
| Setup de proyecto | ~3 min (clicks signing) | ~30s (xcodebuild flags) | **6x** |

**Promedio:** **~6.6x más rápido** vs Xcode IDE puro

---

## 🧩 Cómo un Proyecto Apple Extiende esta Base

Siguiendo la [regla de inicialización](./PLATFORM_BASE.md), un proyecto iOS nuevo:

1. **Corre `/project-init apple`** → detecta el dominio Apple y genera el bloque
   `## Platform Base Context` en su `CLAUDE.md` referenciando este doc.

2. **Declara lo específico** en `docs/APPLE_EXTENSIONS.md`:
   - Bundle ID, Team ID, nombre del developer/distribución, schemes concretos
   - Provisioning / signing config (paths, env vars — nunca secrets inline)
   - Dependencias concretas (Apollo, Nuke, etc.) y por qué
   - Módulos del proyecto y arquitectura específica (MVVM/Clean, capas, isolation por capa)
   - `project.yml` real de XcodeGen (si se usa)
   - Simuladores objetivo y deployment target
   - Si usa KMP: qué comparte con su par Android (ver [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md))
   - Agent skills del proyecto (scaffolding de features, codegen, auth, etc.)

3. **Mantiene esta base limpia** — conocimiento Xcode/Swift genérico reutilizable se aporta
   aquí; lo del repo concreto vive en el proyecto.

> **Plantilla de bloque para el `CLAUDE.md` del proyecto Apple:**
> ```markdown
> ## Platform Base Context
>
> Este proyecto extiende la documentación base de plataforma del CLI workflow:
> - **Apple/iOS:** `~/.config/nvim/XCODE_WORKFLOW.md`
> - **MCP:**       `~/.config/nvim/MCP_WORKFLOW.md` (XcodeBuildMCP, apple-docs, mobile-mcp)
> - (si KMP) **Android:** `~/.config/nvim/ANDROID_WORKFLOW.md` (par nativo Android)
>
> Premisa: `~/.config/nvim/PLATFORM_BASE.md`.
> Específico de este proyecto: `docs/APPLE_EXTENSIONS.md`.
> ```

---

## 📚 Referencias

- **Xcode Build System:** https://developer.apple.com/documentation/xcode/build-system
- **xcodebuild man page:** `man xcodebuild`
- **simctl docs:** `xcrun simctl help`
- **xcbeautify:** https://github.com/cpisciotta/xcbeautify
- **swiftformat:** https://github.com/nicklockwood/SwiftFormat
- **swiftlint:** https://github.com/realm/SwiftLint
- **XcodeGen:** https://github.com/yonaskolb/XcodeGen
- **XcodeBuildMCP:** https://github.com/getsentry/XcodeBuildMCP

### Documentación Relacionada

- [PLATFORM_BASE.md](./PLATFORM_BASE.md) — Premisa universal + regla de inicialización
- [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md) — Par nativo Android (para proyectos con KMP)
- [EXPO_WORKFLOW.md](./EXPO_WORKFLOW.md) — Mobile cross-platform (Expo SDK + prebuild iOS)
- [MCP_WORKFLOW.md](./MCP_WORKFLOW.md) — Inventario completo de MCP servers
- [README.md](./README.md) — Setup general de Neovim

---

**Tipo:** Documentación base de plataforma (genérica, no vinculada a proyecto)
**Stack:** Xcode 26.4.1 + Swift 6.3.1 + iOS 26.4 SDK
**Hardware:** MacBook Pro M4 Pro 48GB
**Filosofía:** CLI-first, MCP-driven, sin Xcode.app abierto en 80% del tiempo
**Extiende con:** `/project-init apple` → `docs/APPLE_EXTENSIONS.md` por proyecto
