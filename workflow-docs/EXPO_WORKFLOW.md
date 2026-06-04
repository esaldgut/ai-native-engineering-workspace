# Expo / React Native Workflow - Mobile Cross-Platform

> **📐 Documentación base de plataforma — genérica, no vinculada a proyecto.**
> Cubre el toolchain Expo/EAS de esta máquina y patrones reutilizables de mobile
> cross-platform. Cada app Expo particular **extiende** esta base (ver
> [PLATFORM_BASE.md](./PLATFORM_BASE.md)). Nada aquí debe contener bundle IDs,
> build numbers, perfiles EAS ni la lista de dependencias de un repo concreto.

**Stack:** Expo SDK 52 + React Native 0.76.9 + AWS Amplify v6 + EAS CLI 18.0.1
**Hardware:** MacBook Pro M4 Pro 48GB
**Filosofía:** Mobile dev sin Xcode IDE — Expo CLI + EAS + mobile-mcp + Neovim

Para el lado **nativo iOS** (prebuild + signing) ver [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md);
para **Android nativo** (Kotlin/Compose) ver [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md).

---

## 📋 Resumen Ejecutivo

Workflow profesional para mobile cross-platform (iOS + Android) usando **Expo Router 4**
sobre RN 0.76.9 con **New Architecture habilitada**.

- ✅ **EAS Build** con perfiles (development, preview, production)
- ✅ **AWS Amplify v6** integrado (`@aws-amplify/react-native`, `@aws-amplify/ui-react-storage`)
- ✅ **Deep linking** configurado (custom scheme + universal links)
- ✅ **mobile-mcp** para automatización de testing manual (ver [MCP_WORKFLOW.md](./MCP_WORKFLOW.md))
- ✅ **Hot reload / Fast Refresh** en simulador iOS

**Distinción clave:** A diferencia de apps Swift puras (ver [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md)),
una app Expo **NO se compila desde Xcode IDE** — se compila desde Expo/EAS y opcionalmente
abre el `ios/` prebuild.

---

## 🎯 Sistema Base

### Toolchain Instalado

| Componente | Versión | Path |
|-----------|---------|------|
| **Node.js** | 22.17.0 | via fnm (`$HOME/.local/state/fnm_multishells/...`) |
| **fnm** | 1.39.0 | Auto-switch con `.node-version` / `.nvmrc` |
| **EAS CLI** | 18.0.1 | `$HOME/.local/state/fnm_multishells/.../bin/eas` |
| **Expo CLI** | 55.0.27 (npx) | Ejecutado vía `npx expo` (sin instalación global) |
| **Watchman** | (sistema) | Para hot reload eficiente |
| **Xcode** | 26.4.1 | Para iOS prebuild + signing |
| **Android Studio** | (opcional) | Para Android prebuild |

### Cuenta Expo

```bash
# Verificar sesión activa
npx expo whoami
npx eas whoami
```

> El username de Expo, user ID y state path (`$HOME/.expo/state.json`) son
> **específicos de la cuenta** → no van en esta base.

---

## 📂 Estructura de un Proyecto Expo

> Ejemplos con una app genérica `example-mobile` (slug `example-mobile`,
> bundle `com.example.mobile`, scheme `app`). Sustituye por los valores reales
> de tu proyecto.

### Configuración (app.json)

```json
{
  "expo": {
    "name": "example-mobile",
    "slug": "example-mobile",
    "version": "1.0.0",
    "scheme": "app",
    "orientation": "portrait",
    "userInterfaceStyle": "automatic",
    "newArchEnabled": true,
    "ios": {
      "buildNumber": "1",
      "supportsTablet": true,
      "bundleIdentifier": "com.example.mobile",
      "associatedDomains": ["applinks:example.com"]
    },
    "android": {
      "package": "com.example.mobile",
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

### Layout del Proyecto

```
example-mobile/
├── app/                  ← Expo Router (file-based routing)
│   ├── _layout.tsx       ← Root layout
│   └── (auth)/           ← Auth group
├── components/           ← Componentes compartidos
├── constants/            ← Theme, colors, config
├── hooks/                ← Custom hooks
├── assets/               ← Images, fonts
├── amplify/              ← AWS Amplify backend (Gen2) — si aplica
├── android/              ← Prebuild nativo Android (generado)
├── ios/                  ← Prebuild nativo iOS (generado)
├── credentials.json      ← EAS credentials reference (si local creds)
├── eas.json              ← EAS Build profiles
├── app.json              ← Expo config
├── babel.config.js       ← Babel + Reanimated plugin
├── metro.config.js       ← Metro bundler config
└── global.css            ← Tailwind directives (si NativeWind)
```

> `android/`, `ios/` y `amplify_outputs.json` son **generados** — usualmente
> `.gitignore`-ados en flujos managed (se regeneran con `expo prebuild` / `ampx`).

---

## 📦 Stack de Dependencias (ilustrativo)

> La lista concreta de dependencias (y sus versiones pinneadas) es **específica del
> proyecto** y vive en su `package.json` / `docs/EXPO_EXTENSIONS.md`. Lo de abajo son
> las **categorías** que un app mobile completo suele cubrir, con ejemplos
> representativos del ecosistema.

### Core Expo + RN

```json
{
  "expo": "~52.0.0",
  "react-native": "0.76.9",
  "expo-router": "~4.0.0",
  "expo-dev-client": "~5.0.0",
  "expo-updates": "~0.27.0"
}
```

### AWS Amplify v6 (si el backend es Amplify)

```json
{
  "aws-amplify": "^6.12.0",
  "@aws-amplify/react-native": "^1.1.0",
  "@aws-amplify/rtn-web-browser": "^1.1.0",
  "@aws-amplify/ui-react-storage": "^3.7.0"
}
```

### Categorías típicas (ejemplos representativos)

| Categoría | Paquetes ejemplo |
|---|---|
| **Navegación** | `@react-navigation/native`, `react-native-screens`, `react-native-gesture-handler`, `react-native-safe-area-context` |
| **Media (cámara/video/PDF)** | `expo-camera`, `expo-image`, `expo-image-picker`, `expo-video`, `react-native-pdf` |
| **Mapas** | `@maplibre/maplibre-react-native` (open source, evita billing externo) |
| **Forms & pickers** | `@react-native-picker/picker`, `@react-native-community/datetimepicker`, `expo-checkbox` |
| **UI components** | `react-native-paper`, `react-native-svg`, `@expo/vector-icons`, `expo-symbols`, `expo-linear-gradient` |
| **Animaciones** | `react-native-reanimated`, `react-native-gesture-handler` |
| **DB local** | `@react-native-async-storage/async-storage`, WatermelonDB |
| **Storage & files** | `expo-file-system`, `expo-document-picker`, `expo-media-library`, `expo-sharing` |
| **Sensores & permisos** | `expo-location`, `expo-device`, `expo-notifications`, `expo-crypto`, `expo-linking` |
| **Web support** | `react-native-web`, `react-native-webview` |

> **Regla:** declarar versiones con `npx expo install <pkg>` (no `npm install`) para que
> Expo resuelva la versión compatible con tu SDK. Validar con `npx expo-doctor`.

---

## 🚀 Scripts NPM

```json
{
  "start": "expo start",
  "ios": "expo run:ios",
  "android": "expo run:android",
  "web": "expo start --web",
  "test": "jest --watchAll --coverage=false",
  "test:coverage": "jest --watchAll",
  "lint": "expo lint",
  "reset-project": "node ./scripts/reset-project.js"
}
```

---

## 🏗️ EAS Build Profiles

`eas.json` define los perfiles de build. El **patrón canónico** son 3 perfiles
(development / preview / production); proyectos grandes añaden variantes:

| Profile | Plataforma | Configuración clave | Uso |
|---------|-----------|---------------------|-----|
| **development** | iOS | `simulator: true` | Builds para iOS Simulator (no firma) |
| **preview** | Android | `buildType: "apk"` | APK para distribución directa / internal testing |
| **preview (variant)** | (general) | `developmentClient: true` o `distribution: "internal"` | Dev client / internal testers |
| **production** | iOS/Android | `credentialsSource: "local"` o remote | Production build (App Store / Play Store) |
| **production (pinned)** | iOS | `node: "20.x"`, `image: "macos-...-xcode-..."` | Build pinned a versiones específicas para reproducibilidad |

**`eas.json` genérico:**
```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true }
    },
    "preview": {
      "distribution": "internal",
      "android": { "buildType": "apk" }
    },
    "production": {
      "autoIncrement": true
    }
  }
}
```

### Comandos EAS

```bash
# Login (1 vez)
eas login

# Configurar proyecto (1 vez)
eas build:configure

# Build iOS para simulador (rápido, sin signing)
eas build --profile development --platform ios

# Build iOS production
eas build --profile production --platform ios

# Build Android APK
eas build --profile preview --platform android

# Build ambas plataformas
eas build --profile production --platform all

# Ver builds en cola
eas build:list --status in-queue

# Ver detalle / cancelar
eas build:view <build-id>
eas build:cancel <build-id>

# Submit a App Store / Play Store
eas submit --profile production --platform ios
eas submit --profile production --platform android

# Update OTA (sin pasar por stores)
eas update --branch production --message "Hot fix login bug"

# Ver branches / updates OTA
eas branch:list
eas update:list --branch production
```

### Credenciales

```bash
# Ver credenciales actuales
eas credentials

# Configurar por plataforma (push notifications Apple/FCM, signing)
eas credentials --platform ios
eas credentials --platform android
```

**`credentials.json` típico (para `credentialsSource: "local"`):**
```json
{
  "ios": {
    "provisioningProfilePath": "./credentials/profile.mobileprovision",
    "distributionCertificate": {
      "path": "./credentials/dist-cert.p12",
      "password": "ENV:CERT_PASSWORD"
    }
  },
  "android": {
    "keystore": {
      "keystorePath": "./credentials/release.keystore",
      "keystorePassword": "ENV:KEYSTORE_PASSWORD",
      "keyAlias": "release",
      "keyPassword": "ENV:KEY_PASSWORD"
    }
  }
}
```

> Los archivos de credenciales (`.p12`, `.mobileprovision`, `.keystore`) y passwords son
> **secretos** → nunca commitear; referenciar vía `ENV:` y mantener fuera del repo.

---

## 💻 Dev Loop Diario

### 1. Iniciar Dev Server

```bash
# Opción A: Expo dev server (web QR + multi-plataforma)
npx expo start

# Opción B: Build + run directo iOS
npm run ios            # = npx expo run:ios

# Opción C: Build + run directo Android
npm run android

# Opción D: Web (preview rápido sin sim)
npm run web

# Opción E: Con dev client (requiere build previo)
npx expo start --dev-client
```

### 2. Hot Reload Workflow

Una vez que el dev server está corriendo:

| Atajo | Acción |
|-------|--------|
| `r` (en terminal expo) | Reload bundle |
| `i` (en terminal expo) | Open iOS sim |
| `a` (en terminal expo) | Open Android emulator |
| `j` (en terminal expo) | Open debugger |
| `m` (en terminal expo) | Toggle dev menu |
| `Cmd+D` (en sim) | Abrir dev menu |
| `Cmd+R` (en sim) | Reload |

### 3. Edit en Neovim

```bash
# En otra terminal/tmux pane
nvim "app/(auth)/login.tsx"
```

**LSP en Neovim para RN:**
- typescript-tools.nvim auto-detecta `tsconfig.json`
- Auto-imports incluyen `react-native`, `expo-router`, `aws-amplify`
- Ver [TYPESCRIPT_CONTEXT.md](./TYPESCRIPT_CONTEXT.md) para el setup LSP

### 4. Testing

```bash
# Watch mode
npm test

# Run una vez / un archivo
npx jest
npx jest path/to/test.spec.ts

# Coverage report
npm run test:coverage
```

---

## 📱 Workflow con mobile-mcp

`mobile-mcp` (ver [MCP_WORKFLOW.md](./MCP_WORKFLOW.md)) permite a Claude controlar el
simulador directamente. Es plataforma-agnóstico (mismo server para iOS y Android).

### Setup

```bash
# 1. Boot simulator
xcrun simctl boot "iPhone 17 Pro"

# 2. Build + install
npm run ios

# 3. Claude Code ya tiene mobile-mcp activo
```

### Comandos en Claude

```
> "Listame los devices disponibles"
→ mobile-mcp.mobile_list_available_devices

> "Lanza la app"
→ mobile-mcp.mobile_launch_app "com.example.mobile"

> "Hazme un screenshot"
→ mobile-mcp.mobile_take_screenshot

> "¿Qué elementos hay en pantalla?"
→ mobile-mcp.mobile_list_elements_on_screen   → JSON con coords

> "Tap en el botón de Login"
→ mobile-mcp.mobile_click_on_screen_at_coordinates X, Y

> "Type 'user@example.com'"
→ mobile-mcp.mobile_type_keys "user@example.com"

> "Swipe up"
→ mobile-mcp.mobile_swipe_on_screen "up"

> "Hubo crashes?"
→ mobile-mcp.mobile_list_crashes
   mobile-mcp.mobile_get_crash <id>
```

### Caso de Uso: Test Manual Automatizado

**Escenario:** Validar flujo de login + redirect a dashboard.

```
1. mobile_launch_app "com.example.mobile"
2. mobile_take_screenshot → /tmp/01-splash.png
3. mobile_list_elements_on_screen
   → identificar coords de Email, Password, Sign In
4. mobile_click_on_screen_at_coordinates <email>
5. mobile_type_keys "test@example.com"
6. mobile_click_on_screen_at_coordinates <password>
7. mobile_type_keys "Test123!"
8. mobile_click_on_screen_at_coordinates <signin>
9. mobile_take_screenshot → /tmp/02-loading.png
10. (esperar response del backend)
11. mobile_take_screenshot → /tmp/03-dashboard.png
12. mobile_list_elements_on_screen → confirmar dashboard
```

**Tiempo:** ~30s vs ~10 min escribiendo un test Detox/Maestro.

---

## 🛠️ Workflows Profesionales

### 1. Nueva Feature con Expo Router (5 min)

```bash
# 1. Branch
git checkout -b feature/profile-edit

# 2. Crear screen (file-based routing crea la ruta automáticamente)
nvim app/profile/edit.tsx

# 3. Hot reload (ya activo si dev server corriendo)
#    Navegar a la screen via deep link:
xcrun simctl openurl booted "app:///profile/edit"

# 4. Iterar — cambios se reflejan en <1s vía Fast Refresh

# 5. Commit
git add app/profile/edit.tsx
git commit -m "feat: add profile edit screen"
```

### 2. Hot Fix OTA (sin pasar por stores) — 10 min

**Casos válidos para OTA:**
- ✅ Bug en JS code (lógica, UI text, validaciones)
- ✅ Cambio de copy
- ✅ Tweak de estilos
- ❌ NO funciona para cambios en código nativo
- ❌ NO funciona para nuevas dependencias nativas
- ❌ NO funciona para permisos nuevos

```bash
# 1. Fix en JS puro
nvim "app/(auth)/login.tsx"

# 2. Verificar local (bundle production-like)
npx expo start --no-dev --minify

# 3. Push update OTA
eas update --branch production --message "Fix: validation regex on email"

# 4. Verificar
eas update:list --branch production --limit 5
```

### 3. Build & Submit a App Store (30-60 min)

```bash
# 1. Bump version en app.json (version + buildNumber)

# 2. Build production (esperar ~15-30 min en EAS cloud)
eas build --profile production --platform ios

# 3. Submit
eas submit --profile production --platform ios

# 4. Confirmar en App Store Connect
open https://appstoreconnect.apple.com
```

### 4. Build & Submit Android Play Store

```bash
# 1. Build production (genera AAB para Play Store)
eas build --profile production --platform android

# 2. Submit
eas submit --profile production --platform android
```

### 5. Debug con Reanimated / Gesture Handler

```bash
# 1. Habilitar Reanimated logger en código:
#    import { configureReanimatedLogger } from 'react-native-reanimated';
#    configureReanimatedLogger({ level: 'warn' });

# 2. Stream logs filtrados (sustituye "MyApp" por el process name real)
xcrun simctl spawn booted log stream \
  --predicate 'processImagePath contains "MyApp"' --style compact

# 3. Si crash:
#    mobile-mcp.mobile_get_crash <id>
```

### 6. Profiling de Performance

```bash
# Build production-like
npx expo start --no-dev --minify

# Hermes Profiler en sim: Cmd+D → Open Performance Monitor

# Profiling deep:
npx react-native profile-hermes
```

### 7. Cambio de Dependencia Nativa (rebuild requerido)

```bash
# 1. Instalar dep
npx expo install expo-camera@latest

# 2. Si requiere rebuild nativo:
npx expo prebuild --clean

# 3. Rebuild & install
npm run ios

# 4. Si production: nuevo EAS build (NO OTA)
eas build --profile production --platform ios
```

---

## 🌐 Deep Linking

**Configurado en `app.json`:**
```json
{
  "scheme": "app",
  "ios": {
    "associatedDomains": ["applinks:example.com"]
  }
}
```

### Probar Deep Links

```bash
# Custom scheme
xcrun simctl openurl booted "app:///profile/123"
xcrun simctl openurl booted "app:///auth/login"

# Universal link (requiere apple-app-site-association servido en el dominio)
xcrun simctl openurl booted "https://example.com/profile/123"
```

### Verificar AASA

```bash
# El archivo debe estar en https://<dominio>/.well-known/apple-app-site-association
curl https://example.com/.well-known/apple-app-site-association | jq
```

---

## 🔐 Permisos Nativos

`app.json` declara los usage descriptions necesarios. Ejemplos comunes:

**iOS:**
- `NSLocationWhenInUseUsageDescription`, `NSLocationAlwaysAndWhenInUseUsageDescription`
- `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription`
- `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`
- `LSApplicationQueriesSchemes` (esquemas de apps externas a las que se enlaza)

**Android:**
- `RECORD_AUDIO`, `READ/WRITE_EXTERNAL_STORAGE`
- `ACCESS_MEDIA_LOCATION`, `ACCESS_COARSE/FINE_LOCATION`

### Cambiar Permisos = Rebuild

```bash
# Después de cambiar app.json
npx expo prebuild --clean
eas build --profile production --platform ios
```

---

## ☁️ Integración AWS Amplify (si aplica)

Si el backend es **Amplify Gen2**, el código vive en `amplify/` del proyecto.

### Verificar Backend

```bash
# Ver schema
cat amplify/data/resource.ts

# Sandbox dev (requiere AWS SSO al perfil correspondiente)
aws sso login --profile <dev-profile>
npx ampx sandbox
# Despliega backend efímero en AWS y genera amplify_outputs.json
```

### Uso en Código

```typescript
// app/_layout.tsx
import { Amplify } from 'aws-amplify';
import outputs from '../amplify_outputs.json';

Amplify.configure(outputs);

// En cualquier screen:
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '@/amplify/data/resource';

const client = generateClient<Schema>();
const { data } = await client.models.User.list();
```

Ver detalles en [AWS_WORKFLOW.md](./AWS_WORKFLOW.md). El profile AWS, region y nombres de
recursos son **específicos del proyecto**.

---

## 🐛 Troubleshooting

### "Unable to resolve module" tras agregar dep

```bash
# Restart Metro con cache clear
npx expo start --clear

# Si persiste: full rebuild
rm -rf node_modules ios/Pods
yarn install   # o npm install
cd ios && pod install && cd ..
npm run ios
```

### Build iOS falla con "Code signing error"

```bash
# Refrescar credentials
eas credentials --platform ios

# Si production con local creds, verificar que existen:
ls -la credentials/
```

### "Native module RNGestureHandlerModule is null"

```bash
# Reinstalar y rebuild nativo
npx expo install react-native-gesture-handler --fix
npx expo prebuild --clean
npm run ios
```

### Hot Reload se rompe

```bash
watchman watch-del-all
npx expo start --clear
```

### Reanimated no funciona

Verificar que `babel.config.js` incluye el plugin **al final**:

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      'react-native-reanimated/plugin',  // SIEMPRE último
    ],
  };
};
```

### EAS Build queda en cola mucho tiempo

```bash
eas build:list --status in-queue
# Tier free tiene cola más larga; producción usa paid tier
```

### "JavaScript code did not load"

```bash
# Bundle no creado o Metro no corriendo
killall node
npx expo start --clear
```

### Amplify "No credentials"

```bash
aws sts get-caller-identity --profile <dev-profile>   # verificar SSO activo
aws sso login --profile <dev-profile>                 # re-login
export AWS_PROFILE=<dev-profile>
npx ampx sandbox
```

### Video player / crash tras update de SDK

```bash
npx expo install expo-video@latest
npx expo-doctor
```

### App muy lenta en debug

- Debug builds son ~5x más lentos que release
- Para profiling real, usar `expo start --no-dev --minify`
- O EAS Build con `developmentClient: false`

---

## 📊 Métricas de Productividad

| Tarea | Sin este setup | Con este setup | Mejora |
|-------|----------------|----------------|--------|
| Iniciar dev (cold) | ~3 min (Xcode build) | ~45s (`npm run ios` + cache) | **4x** |
| Hot reload | ~5s | <1s (Fast Refresh) | **5x** |
| Test manual flow | ~5 min (manual taps) | ~30s (mobile-mcp) | **10x** |
| OTA hot fix | ~24h (App Review) | ~5 min (eas update) | **288x** |
| Build production iOS | ~30 min local | ~20 min EAS cloud (en bg) | **1.5x + paralelo** |
| Cambio de copy | ~24h (rebuild + review) | ~5 min (OTA) | **288x** |

---

## ⌨️ Aliases Recomendados

Agregar a `~/.zshrc` (genéricos — sustituye paths/bundle por los de tu proyecto):

```bash
# Project shortcuts (ajusta el path real del proyecto)
alias mob='cd $HOME/dev/src/<project>/apps/<app>'
alias mob-start='mob && npx expo start'
alias mob-ios='mob && npm run ios'
alias mob-android='mob && npm run android'
alias mob-web='mob && npm run web'
alias mob-prebuild='mob && npx expo prebuild --clean'

# EAS
alias eas-build-ios='eas build --profile production --platform ios'
alias eas-build-android='eas build --profile production --platform android'
alias eas-build-dev='eas build --profile development --platform ios'
alias eas-list='eas build:list --limit 10'
alias eas-update-prod='eas update --branch production'

# Expo
alias expo-clear='npx expo start --clear'
alias expo-doctor='npx expo-doctor'
alias expo-install='npx expo install'

# Deep links
alias mob-link='xcrun simctl openurl booted'
```

---

## 🧩 Cómo una App Expo Extiende esta Base

Siguiendo la [regla de inicialización](./PLATFORM_BASE.md), una app Expo nueva:

1. **Corre `/project-init expo`** → detecta el dominio Expo y genera el bloque
   `## Platform Base Context` en su `CLAUDE.md` referenciando este doc.

2. **Declara lo específico** en `docs/EXPO_EXTENSIONS.md`:
   - `name`, `slug`, `version`, `buildNumber`, bundle/package ID, scheme, associated domains
   - Cuenta Expo / EAS project ID
   - Perfiles EAS reales (`eas.json`) y su estrategia de credenciales
   - Lista concreta de dependencias (`package.json`) y por qué cada una
   - Backend (Amplify profile/region, o el que use) — ver [AWS_WORKFLOW.md](./AWS_WORKFLOW.md)
   - Permisos nativos declarados
   - Agent skills del proyecto (scaffolding de screens, etc.)

3. **Mantiene esta base limpia** — conocimiento Expo/EAS genérico reutilizable se aporta
   aquí; lo del repo concreto vive en el proyecto.

> **Plantilla de bloque para el `CLAUDE.md` del proyecto Expo:**
> ```markdown
> ## Platform Base Context
>
> Este proyecto extiende la documentación base de plataforma del CLI workflow:
> - **Mobile/Expo:** `~/.config/nvim/EXPO_WORKFLOW.md`
> - **iOS prebuild:** `~/.config/nvim/XCODE_WORKFLOW.md` (signing + simctl)
> - **MCP:**          `~/.config/nvim/MCP_WORKFLOW.md` (mobile-mcp)
> - (si backend AWS) **AWS:** `~/.config/nvim/AWS_WORKFLOW.md`
>
> Premisa: `~/.config/nvim/PLATFORM_BASE.md`.
> Específico de este proyecto: `docs/EXPO_EXTENSIONS.md`.
> ```

---

## 📚 Referencias

### Docs Oficiales

- **Expo SDK:** https://docs.expo.dev/
- **Expo Router:** https://docs.expo.dev/router/introduction/
- **EAS Build:** https://docs.expo.dev/build/introduction/
- **EAS Submit:** https://docs.expo.dev/submit/introduction/
- **EAS Update:** https://docs.expo.dev/eas-update/introduction/
- **AWS Amplify (React Native):** https://docs.amplify.aws/react-native/
- **React Native 0.76 (New Architecture):** https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture
- **New Architecture:** https://reactnative.dev/architecture/landing-page

### Documentación Relacionada

- [PLATFORM_BASE.md](./PLATFORM_BASE.md) — Premisa universal + regla de inicialización
- [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md) — iOS prebuild + signing + simctl
- [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md) — Android nativo (Kotlin/Compose)
- [MCP_WORKFLOW.md](./MCP_WORKFLOW.md) — Detalles de mobile-mcp
- [AWS_WORKFLOW.md](./AWS_WORKFLOW.md) — Amplify v6 + AppSync (backend)
- [NEXTJS_WORKFLOW.md](./NEXTJS_WORKFLOW.md) — Web sibling project
- [TYPESCRIPT_CONTEXT.md](./TYPESCRIPT_CONTEXT.md) — LSP setup para TS/RN

---

**Tipo:** Documentación base de plataforma (genérica, no vinculada a proyecto)
**Stack:** Expo SDK 52 + RN 0.76.9 + Amplify v6 + EAS 18.0.1
**Hardware:** MacBook Pro M4 Pro 48GB
**Filosofía:** Mobile dev sin Xcode IDE, OTA-first para fixes JS, EAS cloud para builds
**Extiende con:** `/project-init expo` → `docs/EXPO_EXTENSIONS.md` por proyecto
