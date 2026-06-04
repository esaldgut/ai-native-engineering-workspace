# Android Workflow - Native Android Platform (Kotlin + Compose + KMP)

> **📐 Documentación base de plataforma — genérica, no vinculada a proyecto.**
> Cubre el toolchain Android nativo de esta máquina y patrones reutilizables.
> Cada proyecto Android particular **extiende** esta base (ver
> [PLATFORM_BASE.md](./PLATFORM_BASE.md)). Nada aquí debe contener bundle IDs,
> applicationIds, dependencias o arquitectura de un repo concreto.

**Stack:** Kotlin + Jetpack Compose + Gradle (Kotlin DSL) + Kotlin Multiplatform (KMP)
**Hardware:** MacBook Pro M4 Pro (48GB, arm64)
**Filosofía:** CLI-first, emulator + adb + Gradle wrapper, mobile-mcp para UI automation, Neovim para edición (Android Studio solo para layouts/profiler)

---

## 📋 Resumen Ejecutivo

Workflow base para desarrollo Android **nativo** (Kotlin/Compose), complementario al
mobile cross-platform (Expo/RN — ver [EXPO_WORKFLOW.md](./EXPO_WORKFLOW.md)). Apunta a:

- ✅ **Build & test** desde terminal con Gradle wrapper (`./gradlew`)
- ✅ **Emuladores** controlados por `adb` + `emulator` + mobile-mcp
- ✅ **Kotlin Multiplatform (KMP)** para compartir lógica con iOS (modelos, networking, use cases)
- ✅ **Compose** como UI declarativa (análogo Android de SwiftUI)
- ✅ **mobile-mcp** para automatización de UI (mismo server que iOS — ver [MCP_WORKFLOW.md](./MCP_WORKFLOW.md))

Este es el **par Android** de [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md). Un proyecto que
busca paridad iOS/Android nativa usa ambas bases + KMP para el código compartido.

---

## 🎯 Sistema Base

### Toolchain Instalado (verificado)

| Componente | Versión | Ubicación |
|-----------|---------|-----------|
| **JDK** | 17.0.12 LTS (Oracle, via SDKMAN) | `~/.sdkman/candidates/java/current` |
| **Android Studio** | 2025.3 (AI-253.32098.37) | `/Applications/Android Studio.app` |
| **adb** | 1.0.41 (37.0.0) | `$ANDROID_HOME/platform-tools/platform-tools/adb` |
| **emulator** | (bundled) | `$ANDROID_HOME/emulator/emulator/emulator` |
| **Kotlin** | (vía Gradle wrapper / Android Studio) | No global en SDKMAN |
| **Gradle** | (vía wrapper `./gradlew` por proyecto) | No global instalado |
| **cmdline-tools** | latest | `$ANDROID_HOME/cmdline-tools/latest` |

> **JDK:** SDKMAN tiene 11.0.29-tem y 17.0.12-oracle. **Android moderno requiere JDK 17+**
> (AGP 8.x). `current` apunta a 17 — correcto. Para cambiar: `sdk use java 17.0.12-oracle`.
>
> **Kotlin/Gradle no son globales** — eso es lo correcto. Cada proyecto trae su Gradle
> wrapper (`./gradlew`) que fija la versión de Gradle, y el Kotlin Gradle Plugin fija
> la de Kotlin. Nunca instales Gradle global para proyectos Android.

### Android SDK

| Componente | Versión instalada |
|-----------|-------------------|
| **build-tools** | 36.0.0, 36.1.0, 37.0.0 |
| **platforms** | android-36.1 (API 36.1) |
| **platform-tools** | 37.0.0 |
| **emulator** | instalado |
| **cmdline-tools** | latest |

### ⚠️ El SSD Externo DEBE ser APFS — Nota Crítica (corrupción con ExFAT)

> **Lección aprendida (2026-05-03):** your external SSD venía formateado como **ExFAT**, y
> ExFAT **corrompe el Android SDK** — no soporta permisos POSIX ni la estructura de
> symlinks/anidamiento que el SDK requiere. Síntomas: `source.properties` perdidos,
> componentes mal anidados (`platform-tools/platform-tools/`, `build-tools/36.0.0/android-16/`),
> `sdkmanager` ausente, y builds que fallan con *"Failed to find target android-XX"* o
> *"build-tool has corrupt source.properties"*. Además macOS ensucia ExFAT con AppleDouble (`._*`).
>
> **Regla:** el SSD que aloje el Android SDK **debe ser APFS** (o Mac OS Extended). El
> código fuente plano sí tolera ExFAT, pero el SDK no. Si el SSD es ExFAT → reformatear a
> APFS (Disk Utility, destructivo: respaldar primero) antes de instalar el SDK.

**Tanto el SDK como los proyectos Android viven en el SSD externo your external SSD (APFS), NO
en el disco interno** — el disco interno está al ~96% (rebasado); el SSD lo extiende. Es la
convención de esta máquina para Android, **válida solo con el SSD en APFS**.

**SDK:**
```bash
ANDROID_HOME=$EXTERNAL_SSD/Users/you/Library/Android/sdk
```
Configurado en `~/.zshrc:126-129`:
```bash
# Android Studio
export ANDROID_HOME="$EXTERNAL_SSD/Users/you/Library/Android/sdk"
export PATH="$ANDROID_HOME/platform-tools/platform-tools:$PATH"
export PATH="$ANDROID_HOME/emulator/emulator:$PATH"
```

**Proyectos Android** — raíz canónica:
```
$EXTERNAL_SSD/Users/you/dev/src/android/
```

> ⚠️ **No usar `~/dev/src/android/`** (disco interno) para proyectos Android. Esa carpeta
> existe pero debe quedar vacía — los proyectos Android reales viven en el SSD. Razón:
> los proyectos Android son pesados (build artifacts, `.gradle/`, emulador system images);
> mantenerlos junto al SDK en el SSD libera el disco interno y evita rutas partidas.

**Implicaciones de tener todo en SSD externo:**
- ❌ Si your external SSD no está montado → `adb`, `emulator`, builds Gradle Y el código del proyecto son inaccesibles
- ✅ Verificar antes de trabajar: `ls $EXTERNAL_SSD >/dev/null 2>&1 && echo "SSD OK" || echo "⚠️ Mount your external SSD"`
- ⚠️ `sdkmanager` y `avdmanager` NO están en PATH (viven en `$ANDROID_HOME/cmdline-tools/latest/bin/`) — agregar si los usas seguido (ver Aliases)
- ⚠️ `ANDROID_SDK_ROOT` está vacío — Google lo deprecó en favor de `ANDROID_HOME`, está bien, pero algunas tools viejas lo buscan
- ⚠️ macOS crea archivos `._*` (AppleDouble) en el SSD (filesystem no-APFS) — inofensivos, pero añadir `._*` y `.DS_Store` al `.gitignore` del proyecto

### Verificación de Entorno (correr al inicio de sesión Android)

```bash
# Pre-flight check
ls $EXTERNAL_SSD >/dev/null 2>&1 || { echo "⚠️ Mount your external SSD primero"; }
java -version           # debe ser 17+
echo $ANDROID_HOME      # debe resolver a un path existente
adb version             # confirma SDK accesible
adb devices             # emuladores/devices conectados
ls $EXTERNAL_SSD/Users/you/dev/src/android/   # proyectos Android
```

---

## 📱 Emuladores & Devices (adb + emulator)

### Gestión de AVDs

```bash
# Listar AVDs existentes
emulator -list-avds

# Crear AVD (requiere sdkmanager/avdmanager en PATH — ver Aliases)
sdkmanager "system-images;android-36;google_apis;arm64-v8a"
avdmanager create avd -n Pixel_8_API36 -k "system-images;android-36;google_apis;arm64-v8a" -d pixel_8

# Lanzar emulador (headless o con ventana)
emulator -avd Pixel_8_API36 &
emulator -avd Pixel_8_API36 -no-window -no-audio &      # headless (CI / rápido)

# Cold boot (sin restaurar snapshot)
emulator -avd Pixel_8_API36 -no-snapshot-load &
```

### Comandos adb Esenciales

```bash
# Devices conectados
adb devices

# Si hay varios, targetear uno
adb -s emulator-5554 <comando>

# Install / uninstall
adb install app/build/outputs/apk/debug/app-debug.apk
adb install -r app-debug.apk          # reinstall conservando data
adb uninstall <applicationId>

# Launch / stop (genérico — el applicationId lo define el proyecto)
adb shell am start -n <applicationId>/.MainActivity
adb shell am force-stop <applicationId>

# Logcat filtrado
adb logcat --pid=$(adb shell pidof -s <applicationId>)
adb logcat *:E                        # solo errores
adb logcat -c                         # limpiar buffer

# Screenshot
adb exec-out screencap -p > ~/Desktop/android-shot-$(date +%H%M%S).png

# Screen record
adb shell screenrecord /sdcard/rec.mp4    # Ctrl+C para parar
adb pull /sdcard/rec.mp4 ~/Desktop/

# Deep link (scheme lo define el proyecto)
adb shell am start -a android.intent.action.VIEW -d "myscheme://path"

# Push notification test, permisos, etc.
adb shell pm grant <applicationId> android.permission.CAMERA
adb shell dumpsys package <applicationId> | grep -A5 "granted=true"

# Geolocation (emulator)
adb emu geo fix -99.1332 19.4326       # lon lat (CDMX)

# Wipe data
adb shell pm clear <applicationId>
```

---

## 🔨 Gradle (build, test, lint) vía wrapper

> **Siempre `./gradlew`, nunca `gradle` global.** El wrapper fija la versión correcta.

```bash
# Tareas disponibles
./gradlew tasks

# Build debug
./gradlew assembleDebug

# Build release
./gradlew assembleRelease

# Install en device/emulator booted
./gradlew installDebug

# Build + install + launch (un solo paso)
./gradlew installDebug && adb shell am start -n <applicationId>/.MainActivity

# Unit tests (JVM, rápidos)
./gradlew test
./gradlew testDebugUnitTest

# Instrumented tests (en emulator/device)
./gradlew connectedAndroidTest
./gradlew connectedDebugAndroidTest

# Un test específico
./gradlew testDebugUnitTest --tests "com.example.UserRepositoryTest"

# Lint
./gradlew lint
./gradlew lintDebug

# Build + verificar todo (CI-style)
./gradlew build

# Limpiar
./gradlew clean

# Dependencias (árbol)
./gradlew app:dependencies --configuration debugRuntimeClasspath

# Build con report de tiempos
./gradlew assembleDebug --profile

# Offline (si SSD lento o sin red)
./gradlew assembleDebug --offline
```

### Gradle Performance en M4 Pro (48GB)

`gradle.properties` recomendado (el proyecto lo define, esto es la base genérica):

```properties
org.gradle.jvmargs=-Xmx6g -XX:+UseParallelGC
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
kotlin.incremental=true
android.useAndroidX=true
```

> En M4 Pro 48GB puedes dar 6-8GB a la JVM de Gradle sin problema. `configuration-cache`
> acelera builds incrementales notablemente.

---

## 🧬 Kotlin Multiplatform (KMP) — compartir con iOS

KMP permite compartir lógica de negocio (modelos, networking, use cases, validación)
entre Android e iOS nativos, manteniendo UI nativa por plataforma (Compose / SwiftUI).

### Estructura típica de un módulo `shared`

```
shared/
├── src/
│   ├── commonMain/kotlin/      # Lógica compartida (expect declarations)
│   ├── androidMain/kotlin/     # actual para Android (JVM)
│   └── iosMain/kotlin/         # actual para iOS (Native)
└── build.gradle.kts            # targets: androidTarget(), iosArm64(), iosSimulatorArm64()
```

### Build del framework iOS desde KMP

```bash
# Generar el framework iOS (.framework / XCFramework)
./gradlew :shared:assembleXCFramework

# Sync con el lado iOS (si se consume vía SPM o CocoaPods)
./gradlew :shared:podPublishXCFramework      # si usa CocoaPods plugin
```

### Qué compartir vs qué NO (regla genérica)

| Compartir en `commonMain` | Mantener nativo |
|---|---|
| Modelos de dominio (data classes) | UI (Compose / SwiftUI) |
| Networking (Ktor client) | Navegación |
| Serialización (kotlinx.serialization) | Permisos / sensores específicos |
| Use cases / lógica de negocio | Auth flows con SDK nativo |
| Validación, mappers | Push notifications |

> **Sinergia con esta máquina:** Si un proyecto Android nace como par de uno iOS
> (p.ej. paridad de un app nativo iOS), KMP es el puente. El código compartido vive
> en `commonMain`; Android lo consume vía Gradle, iOS vía XCFramework. Ver
> [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md) para el lado iOS.

---

## 🎨 Jetpack Compose (UI declarativa)

Análogo Android de SwiftUI. Patrones base genéricos:

```kotlin
// Composable + state hoisting
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("Count: $count") }
}

// State holder (ViewModel) — análogo a @Observable de iOS
class CounterViewModel : ViewModel() {
    var count by mutableStateOf(0)
        private set
    fun increment() { count++ }
}
```

### Compose Preview (análogo a SwiftUI Preview / RenderPreview)

```kotlin
@Preview(showBackground = true)
@Composable
fun CounterPreview() { Counter(count = 0, onIncrement = {}) }
```

> Compose Previews se renderizan en Android Studio. No hay equivalente CLI directo
> (a diferencia de `mcp__xcode__RenderPreview` en iOS). Para feedback visual headless,
> usar emulator + mobile-mcp screenshot.

---

## 🔐 Code Signing & Distribution

### Debug signing (automático)

Android usa un debug keystore auto-generado en `~/.android/debug.keystore` — sin
configuración para desarrollo local.

### Release signing (el proyecto define el keystore)

```bash
# Generar release keystore (1 vez, guardar SEGURO — no commitear)
keytool -genkey -v -keystore release.keystore \
  -alias release -keyalg RSA -keysize 2048 -validity 10000

# Verificar firma de un APK/AAB
jarsigner -verify -verbose -certs app-release.aab
apksigner verify --print-certs app-release.apk    # apksigner en build-tools/
```

> El keystore, passwords y `applicationId` son **específicos del proyecto** → van en
> `docs/<DOMAIN>_EXTENSIONS.md` del proyecto + variables de entorno, nunca en esta base
> ni commiteados.

### Build de release para Play Store

```bash
# AAB (Android App Bundle — formato Play Store)
./gradlew bundleRelease
# → app/build/outputs/bundle/release/app-release.aab

# APK (distribución directa / testing)
./gradlew assembleRelease
# → app/build/outputs/apk/release/app-release.apk
```

---

## 🤖 Integración MCP (mobile-mcp para Android)

El mismo `mobile-mcp` que controla simuladores iOS controla **emuladores Android**
(ver inventario en [MCP_WORKFLOW.md](./MCP_WORKFLOW.md)).

### Workflow de UI automation

```
1. mobile_list_available_devices    → detecta emulator-5554
2. mobile_launch_app <applicationId>
3. mobile_take_screenshot           → estado inicial
4. mobile_list_elements_on_screen   → JSON con coords + labels
5. mobile_click_on_screen_at_coordinates X, Y
6. mobile_type_keys "texto"
7. mobile_swipe_on_screen "up"
8. mobile_take_screenshot           → verificar
9. mobile_list_crashes / mobile_get_crash  → si crash
```

**Caso de uso:** test E2E manual de un flow sin escribir Espresso/UI Automator.
Tiempo ~30s vs ~10 min escribiendo el test instrumentado.

> mobile-mcp es plataforma-agnóstico: el mismo skill sirve para iOS y Android. La
> diferencia la pone el device target (`emulator-XXXX` vs `iPhone XX`).

---

## 🚀 Dev Loop Diario (genérico)

```bash
# 1. Pre-flight
ls $EXTERNAL_SSD >/dev/null 2>&1 || echo "⚠️ Montar SSD"
adb devices

# 2. Boot emulator (si no hay device)
emulator -avd Pixel_8_API36 &
adb wait-for-device

# 3. Build + install + launch
./gradlew installDebug
adb shell am start -n <applicationId>/.MainActivity

# 4. Editar en Neovim (Kotlin LSP vía kotlin-language-server si configurado)
nvim app/src/main/kotlin/.../SomeScreen.kt

# 5. Hot reload: Compose tiene Live Edit en Android Studio;
#    desde CLI, rebuild incremental:
./gradlew installDebug && adb shell am start -n <applicationId>/.MainActivity

# 6. Logs
adb logcat --pid=$(adb shell pidof -s <applicationId>)
```

---

## 📊 Métricas de Productividad (base genérica, M4 Pro 48GB)

| Tarea | Android Studio UI | CLI + mobile-mcp | Mejora |
|-------|-------------------|------------------|--------|
| Build + install debug | ~40s (Studio + sync) | ~15s (`./gradlew installDebug`) | **2.7x** |
| Screenshot emulator | ~10s (Studio capture) | <1s (`adb exec-out screencap`) | **10x** |
| Logcat filtrado | ~15s (Logcat panel + filtro) | ~3s (`adb logcat --pid`) | **5x** |
| Run un unit test | ~25s (Studio gutter run) | ~6s (`--tests`) | **4x** |
| UI test flow (3 pasos) | ~10 min (escribir Espresso) | ~30s (mobile-mcp) | **20x** |
| Cambiar emulator | ~12s (AVD manager) | ~4s (`emulator -avd`) | **3x** |

---

## ⌨️ Aliases Recomendados

Agregar a `~/.zshrc` (después de los exports Android existentes en :126-129):

```bash
# Android SDK cmdline-tools en PATH (sdkmanager, avdmanager)
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

# Pre-flight
alias android-check='ls $EXTERNAL_SSD >/dev/null 2>&1 && echo "✓ SSD OK" || echo "⚠️ Mount your external SSD"; java -version; adb version | head -1'

# Emulator
alias avd-list='emulator -list-avds'
alias avd-boot='emulator -avd'                       # uso: avd-boot Pixel_8_API36
alias avd-headless='emulator -no-window -no-audio -avd'

# adb shortcuts
alias adevices='adb devices'
alias alogcat='adb logcat *:E'                       # solo errores
alias ashot='adb exec-out screencap -p > ~/Desktop/android-$(date +%Y%m%d-%H%M%S).png && echo "Saved to Desktop"'
alias arec='adb shell screenrecord /sdcard/rec.mp4'
alias awipe='adb shell pm clear'                     # uso: awipe <applicationId>

# Gradle wrapper (siempre desde root del proyecto)
alias gw='./gradlew'
alias gwd='./gradlew installDebug'
alias gwt='./gradlew testDebugUnitTest'
alias gwc='./gradlew clean'
alias gwb='./gradlew assembleDebug'
alias gwlint='./gradlew lintDebug'

# KMP
alias kmp-ios='./gradlew :shared:assembleXCFramework'
```

---

## 🧩 Cómo un Proyecto Android Extiende esta Base

Siguiendo la [regla de inicialización](./PLATFORM_BASE.md), un proyecto Android nuevo:

0. **Vive en la raíz canónica** `$EXTERNAL_SSD/Users/you/dev/src/android/<proyecto>`
   (SSD externo, junto al SDK — ver Nota Crítica arriba). NO en el disco interno.

1. **Corre `/project-init android`** → detecta el dominio Android y genera el bloque
   `## Platform Base Context` en su `CLAUDE.md` referenciando este doc.

2. **Declara lo específico** en `docs/ANDROID_EXTENSIONS.md`:
   - `applicationId` y namespace
   - Keystore / signing config (paths, env vars — nunca secrets inline)
   - Dependencias concretas (`libs.versions.toml`)
   - Módulos del proyecto (`:app`, `:shared`, `:feature-*`)
   - Arquitectura específica (MVVM/MVI, DI con Hilt/Koin, etc.)
   - AVD(s) objetivo y min/target SDK
   - Si usa KMP: qué comparte con su par iOS
   - Agent skills del proyecto (scaffolding de features Compose, etc.)

3. **Mantiene esta base limpia** — conocimiento Android genérico reutilizable se aporta
   aquí; lo del repo concreto vive en el proyecto.

> **Plantilla de bloque para el `CLAUDE.md` del proyecto Android:**
> ```markdown
> ## Platform Base Context
>
> Este proyecto extiende la documentación base de plataforma del CLI workflow:
> - **Android:** `~/.config/nvim/ANDROID_WORKFLOW.md`
> - **MCP:**     `~/.config/nvim/MCP_WORKFLOW.md` (mobile-mcp para emulator)
> - (si KMP) **Apple/iOS:** `~/.config/nvim/XCODE_WORKFLOW.md` (par nativo iOS)
>
> Premisa: `~/.config/nvim/PLATFORM_BASE.md`.
> Específico de este proyecto: `docs/ANDROID_EXTENSIONS.md`.
> ```

---

## 🐛 Troubleshooting

### "Failed to find target android-XX" / "build-tool has corrupt source.properties"

**Causa raíz:** el SDK vive en un volumen **ExFAT** (o se copió a uno). ExFAT no preserva
la estructura del SDK → `source.properties` perdidos, anidamiento roto, `sdkmanager` ausente.

```bash
# Confirmar filesystem del volumen del SDK
diskutil info $EXTERNAL_SSD | grep "File System Personality"
#   → ExFAT  = PROBLEMA · APFS = OK

# Contar source.properties (deberían ser muchos; si <10, SDK corrupto)
find "$ANDROID_HOME" -name source.properties | wc -l
```

**Solución:** reformatear el SSD a **APFS** (Disk Utility — destructivo, respaldar primero) y
**reinstalar el SDK limpio** vía Android Studio (Settings → SDK Manager) o:
```bash
# cmdline-tools en PATH (post-reinstall)
sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0"
```
No intentar "reparar" un SDK ExFAT in-place — reinstalar de cero en APFS es lo correcto.

### `adb: command not found` / SDK no encontrado

```bash
# Casi siempre: SSD externo no montado
ls $EXTERNAL_SSD || echo "Mount your external SSD"

# O PATH no recargado
source ~/.zshrc
echo $ANDROID_HOME
```

### Gradle: "Unsupported class file major version" / JDK incorrecto

```bash
# AGP 8.x requiere JDK 17
sdk use java 17.0.12-oracle
java -version

# Verificar que Gradle usa el JDK correcto
./gradlew -version
```

### Emulator no bootea / "PANIC: Cannot find AVD"

```bash
emulator -list-avds                 # ver AVDs reales
# Si vacío, crear uno (requiere cmdline-tools en PATH)
sdkmanager "system-images;android-36;google_apis;arm64-v8a"
avdmanager create avd -n Pixel_8_API36 -k "system-images;android-36;google_apis;arm64-v8a"
```

### Build lento

```bash
# Habilitar configuration cache + parallel en gradle.properties (ver sección Gradle)
# Verificar que no estés en --offline sin querer
# En M4: dar más heap
./gradlew assembleDebug -Dorg.gradle.jvmargs=-Xmx8g
```

### `INSTALL_FAILED_UPDATE_INCOMPATIBLE` (firma distinta)

```bash
adb uninstall <applicationId>       # borra la versión con firma vieja
./gradlew installDebug
```

### Kotlin LSP en Neovim no autocompleta

```bash
# Requiere kotlin-language-server (no instalado por defecto)
brew install kotlin-language-server
# Configurar en ~/.config/nvim/lua/config/lsp.lua:
#   vim.lsp.config.kotlin_language_server = { ... }
# Nota: el LSP necesita el proyecto compilado al menos 1 vez para resolver deps Gradle
```

> **Gap actual:** Neovim no tiene `kotlin_language_server` configurado (igual que
> sourcekit-lsp para Swift). Para edición Kotlin con LSP completo, configurarlo o usar
> Android Studio para refactors pesados.

---

## 📚 Referencias

- **Android Developers:** https://developer.android.com
- **Jetpack Compose:** https://developer.android.com/jetpack/compose
- **Kotlin Multiplatform:** https://kotlinlang.org/docs/multiplatform.html
- **Gradle (Android):** https://developer.android.com/build
- **adb:** https://developer.android.com/tools/adb
- **AGP release notes:** https://developer.android.com/build/releases/gradle-plugin

### Documentación Relacionada

- [PLATFORM_BASE.md](./PLATFORM_BASE.md) — Premisa universal + regla de inicialización
- [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md) — Par nativo iOS (para proyectos con KMP)
- [EXPO_WORKFLOW.md](./EXPO_WORKFLOW.md) — Mobile cross-platform (RN, no nativo)
- [MCP_WORKFLOW.md](./MCP_WORKFLOW.md) — mobile-mcp (controla emulator Android)
- [SHELL_WORKFLOW.md](./SHELL_WORKFLOW.md) — exports Android en ~/.zshrc

---

**Creado:** 2026-05-03
**Tipo:** Documentación base de plataforma (genérica, no vinculada a proyecto)
**Stack:** Kotlin + Jetpack Compose + Gradle (Kotlin DSL) + KMP
**Toolchain verificado:** JDK 17.0.12, Android Studio 2025.3, SDK build-tools 36/37, API 36.1, adb 37.0.0
**Nota crítica:** SDK **y proyectos** Android viven en SSD externo your external SSD (**debe ser APFS**, no ExFAT — ExFAT corrompe el SDK) — montar antes de trabajar
**Raíz canónica de proyectos:** `$EXTERNAL_SSD/Users/you/dev/src/android/`
**Extiende con:** `/project-init android` → `docs/ANDROID_EXTENSIONS.md` por proyecto
