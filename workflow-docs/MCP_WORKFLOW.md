# MCP Workflow - Model Context Protocol Servers

**Stack:** Claude Code CLI + Claude Desktop + 7 MCP servers en Claude Code + 1 en Desktop
**Filosofía:** MCPs como "manos" de Claude. Cada uno extiende capacidades específicas (Xcode, Apple docs, mobile, AppSync, GraphQL, Figma, Chrome).
**Caso de uso central:** Claude opera herramientas externas sin que el usuario tenga que copiar/pegar entre ventanas.

---

## 📋 Resumen Ejecutivo

**MCP (Model Context Protocol)** es el estándar abierto que permite a Claude conectarse a sistemas externos a través de servidores que exponen tools, resources y prompts.

**Arquitectura en este sistema:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code (CLI)                         │
│  ~/.claude.json  →  7 MCP servers configurados              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ apple-docs    │ XcodeBuildMCP  │ xcode (mcpbridge)   │   │
│  │ mobile-mcp    │ appsync        │ graphql             │   │
│  │ figma         │                │                     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  Claude Desktop (GUI)                        │
│  ~/Library/.../Claude/claude_desktop_config.json            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ chrome-devtools (puerto 9222)                        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                Per-Project Configs                           │
│  ~/.claude/chrome-mcp.json (override CLI con --mcp-config)  │
│  .mcp.json en project root (auto-detect por Claude Code)    │
└─────────────────────────────────────────────────────────────┘
```

**Total de tools expuestos:** ~120+ (sumando todos los servers)

---

## 🗺️ Inventario Completo

### Claude Code CLI (`~/.claude.json`)

| # | Server | Comando | Status auth | Uso principal |
|---|--------|---------|-------------|---------------|
| 1 | **xcode** | `xcrun mcpbridge` | Auto (system) | Xcode IDE operations (cuando está abierto) |
| 2 | **XcodeBuildMCP** | `npx -y xcodebuildmcp@latest mcp` | No requiere | Build/test/sim genéricos |
| 3 | **apple-docs** | `npx -y @kimsungwhee/apple-docs-mcp@latest` | No requiere | Search Apple docs + WWDC |
| 4 | **mobile-mcp** | `npx -y @mobilenext/mobile-mcp@latest` | No requiere | Control simuladores iOS/Android |
| 5 | **appsync** | `uvx awslabs.aws-appsync-mcp-server@latest` | AWS SSO | AppSync API operations |
| 6 | **graphql** | `npx -y mcp-graphql` | Por endpoint | GraphQL introspection + queries |
| 7 | **figma** | `http` (remote) | OAuth | Diseño Figma → código |

### Claude Desktop (`claude_desktop_config.json`)

| # | Server | Comando | Uso |
|---|--------|---------|-----|
| 8 | **chrome-devtools** | `npx -y @modelcontextprotocol/server-chrome-devtools` | Inspeccionar Chrome remoto (puerto 9222) |

### Per-Project (override / scope)

| Path | Propósito |
|------|-----------|
| `~/.claude/chrome-mcp.json` | Activable con `claude --mcp-config ~/.claude/chrome-mcp.json` (también en Claude Code) |
| `.mcp.json` (project root) | Auto-detect por Claude Code al abrir proyecto |
| `~/.next-devtools-mcp/` | Next.js DevTools MCP — tracking activo desde 2026-04-13 (3624 líneas en mcp.log) |

---

## 🍎 Apple Docs MCP

**Package:** `@kimsungwhee/apple-docs-mcp@latest`

### Tools Expuestos

| Tool | Función |
|------|---------|
| `search_apple_docs` | Búsqueda full-text en docs Apple |
| `get_apple_doc_content` | Contenido de un símbolo/framework |
| `search_framework_symbols` | Lista de símbolos de un framework |
| `get_related_apis` | APIs relacionadas a una dada |
| `find_similar_apis` | APIs similares semánticamente |
| `get_platform_compatibility` | iOS/macOS/visionOS/etc. support |
| `list_technologies` | Frameworks disponibles |
| `get_technology_overviews` | Overview de un framework |
| `get_documentation_updates` | Cambios recientes en docs |
| `list_wwdc_years` | Años con sesiones WWDC |
| `list_wwdc_videos` | Videos de un año/topic |
| `get_wwdc_video` | Detalle de un video específico |
| `search_wwdc_content` | Búsqueda en transcripts |
| `find_related_wwdc_videos` | Videos para una API dada |
| `get_wwdc_code_examples` | Snippets de código de un video |
| `browse_wwdc_topics` | Topics WWDC navegables |
| `get_sample_code` | Sample projects oficiales |
| `resolve_references_batch` | Resolver múltiples refs en una llamada |

### Casos de Uso

**Caso 1: Investigar API nueva**
```
Usuario: "¿Cómo uso Live Activities en iOS 26?"
→ search_apple_docs "Live Activities"
→ get_apple_doc_content "ActivityKit/Activity"
→ get_platform_compatibility "Activity"
→ find_related_wwdc_videos "ActivityKit"
→ get_wwdc_code_examples del video más reciente
```

**Caso 2: Migración de API deprecada**
```
Usuario: "UIApplicationDelegate dice deprecated, ¿qué uso?"
→ get_apple_doc_content "UIApplicationDelegate"
→ find_similar_apis "UIApplicationDelegate"
→ get_documentation_updates --since "2025-09" --framework "UIKit"
```

**Caso 3: Explorar framework nuevo**
```
Usuario: "Quiero aprender Foundation Models"
→ list_technologies | grep -i model
→ get_technology_overviews "FoundationModels"
→ search_framework_symbols "FoundationModels"
→ get_sample_code "FoundationModels"
```

**Tiempo ahorrado:** ~6x vs navegar developer.apple.com manualmente.

---

## 🔨 XcodeBuildMCP

**Package:** `xcodebuildmcp@latest`
**Workflows habilitados:** Solo simulador por defecto (configurable para device, macOS, debugging, UI automation).

### Tools por Categoría

#### Session Defaults (configurar 1 vez)
| Tool | Función |
|------|---------|
| `session_show_defaults` | Ver project/scheme/sim activos |
| `session_set_defaults` | Setear defaults para evitar parámetros repetitivos |
| `session_clear_defaults` | Reset |
| `session_use_defaults_profile` | Switch entre perfiles guardados |

#### Discovery
| Tool | Función |
|------|---------|
| `discover_projs` | Encontrar .xcodeproj/.xcworkspace en directorio |
| `list_schemes` | Schemes del proyecto |
| `show_build_settings` | Settings activos |
| `list_sims` | Simuladores disponibles |

#### Simuladores
| Tool | Función |
|------|---------|
| `boot_sim` | Bootear simulador |
| `open_sim` | Abrir Simulator.app |
| `clean` | Clean derived data |

#### Build & Test
| Tool | Función |
|------|---------|
| `build_sim` | Build sin lanzar |
| `build_run_sim` | Build + install + launch |
| `test_sim` | xcodebuild test |
| `install_app_sim` | Solo install (sin build) |
| `launch_app_sim` | Solo launch (app ya instalada) |
| `stop_app_sim` | Terminar app |
| `launch_app_logs_sim` | Stream de logs en vivo |
| `start_sim_log_cap` / `stop_sim_log_cap` | Captura buffered |
| `get_sim_app_path` | Path del .app generado |
| `get_app_bundle_id` | Bundle ID desde .app |
| `get_coverage_report` | Coverage tras test |
| `get_file_coverage` | Coverage de un archivo específico |

#### UI Automation
| Tool | Función |
|------|---------|
| `screenshot` | Captura PNG |
| `snapshot_ui` | Jerarquía de vistas con coordenadas (JSON) |
| `record_sim_video` | Grabación |

### Workflow Estándar

```
1. session_show_defaults     ← OBLIGATORIO antes del primer build
2. (si vacío) session_set_defaults con project + scheme + simulator
3. boot_sim                  ← si simulador no booted
4. build_run_sim             ← build + install + launch
5. snapshot_ui               ← inspeccionar UI
6. screenshot                ← evidencia
```

### Caso de Uso Real

**Escenario:** Verificar que el botón de login navega al dashboard.

```
1. session_set_defaults
   { project: "esag.xcodeproj", scheme: "esag", simulator: "iPhone 17 Pro" }

2. build_run_sim
   → app corriendo

3. snapshot_ui
   → JSON: { "Continue button": [200, 540], "Email field": [200, 380] }

4. screenshot → /tmp/before.png

5. (tap (200, 540) via simctl o XcodeBuildMCP UI tools)

6. snapshot_ui
   → JSON: { "Dashboard title": [187, 120], "Logout": [350, 50] }

7. screenshot → /tmp/after.png

→ Confirmado: navegación funciona
```

---

## 🛠️ Xcode (mcpbridge)

**Comando:** `xcrun mcpbridge` (system-provided con Xcode 26+)
**Cuándo usar:** Cuando Xcode IDE ESTÁ abierto con un proyecto.

### Tools Expuestos

#### Build & Run
| Tool | Función |
|------|---------|
| `BuildProject` | Build del proyecto activo |
| `GetBuildLog` | Log del último build |
| `RunAllTests` | Ejecuta todos los tests |
| `RunSomeTests` | Ejecuta tests específicos |
| `GetTestList` | Lista de tests disponibles |

#### Code Issues
| Tool | Función |
|------|---------|
| `XcodeListNavigatorIssues` | Errores/warnings del Navigator |
| `XcodeRefreshCodeIssuesInFile` | Re-trigger diagnósticos |

#### Files (dentro del proyecto)
| Tool | Función |
|------|---------|
| `XcodeRead` | Leer archivo |
| `XcodeWrite` | Crear/sobrescribir |
| `XcodeUpdate` | Edit string-based |
| `XcodeMV` | Renombrar/mover |
| `XcodeRM` | Eliminar |
| `XcodeMakeDir` | Crear directorio |
| `XcodeGlob` | Glob pattern matching |
| `XcodeGrep` | Búsqueda regex |
| `XcodeLS` | Listar contenido |

#### SwiftUI & Snippets
| Tool | Función |
|------|---------|
| `RenderPreview` | Renderizar SwiftUI Preview |
| `ExecuteSnippet` | Run código Swift ad-hoc |

#### Workspace
| Tool | Función |
|------|---------|
| `XcodeListWindows` | Ventanas Xcode abiertas |
| `DocumentationSearch` | Search docs desde Xcode |

### Caso de Uso Real

**Escenario:** Iterar sobre un SwiftUI preview sin abrir Xcode preview canvas.

```
1. XcodeRead "Views/UserCard.swift"
2. XcodeUpdate (cambiar padding de 8 a 16)
3. RenderPreview "UserCard_Previews"
   → PNG renderizado del preview
4. XcodeListNavigatorIssues
   → 0 errors
5. RunSomeTests "UserCardTests"
   → ✓ all pass
```

---

## 📱 Mobile MCP

**Package:** `@mobilenext/mobile-mcp@latest`
**Plataformas:** iOS Simulator + Android Emulator + (opcionalmente) device físico vía USB

### Tools Expuestos

#### Discovery
| Tool | Función |
|------|---------|
| `mobile_list_available_devices` | Devices/sims disponibles |
| `mobile_list_apps` | Apps instaladas |
| `mobile_list_crashes` | Lista de crashes |
| `mobile_get_crash` | Detalle de un crash |
| `mobile_get_screen_size` | Resolución de pantalla |
| `mobile_get_orientation` | Portrait/Landscape |
| `mobile_set_orientation` | Cambiar orientación |

#### App Lifecycle
| Tool | Función |
|------|---------|
| `mobile_install_app` | Install .ipa/.apk |
| `mobile_launch_app` | Launch por bundle ID |
| `mobile_terminate_app` | Kill app |
| `mobile_uninstall_app` | Uninstall |
| `mobile_open_url` | Deep link / URL scheme |

#### UI Interaction
| Tool | Función |
|------|---------|
| `mobile_list_elements_on_screen` | Elementos visibles con coords |
| `mobile_click_on_screen_at_coordinates` | Tap |
| `mobile_double_tap_on_screen` | Double tap |
| `mobile_long_press_on_screen_at_coordinates` | Long press |
| `mobile_swipe_on_screen` | Swipe direccional |
| `mobile_type_keys` | Type text |
| `mobile_press_button` | Hardware buttons (home, volume, power) |

#### Capture
| Tool | Función |
|------|---------|
| `mobile_take_screenshot` | PNG |
| `mobile_save_screenshot` | Save a path específico |
| `mobile_start_screen_recording` / `mobile_stop_screen_recording` | Grabación |

### Caso de Uso Real

**Escenario:** Test E2E de flujo de login en example-mobile sin escribir Detox/Maestro.

```
1. mobile_list_available_devices → "iPhone 17 Pro (iOS 26.0)"
2. mobile_launch_app "com.example.mobile"
3. mobile_take_screenshot → /tmp/01-splash.png
4. mobile_list_elements_on_screen
   → { "Email": [200, 380], "Password": [200, 440], "Sign In": [200, 540] }
5. mobile_click_on_screen_at_coordinates 200, 380
6. mobile_type_keys "test@example.com"
7. mobile_click_on_screen_at_coordinates 200, 440
8. mobile_type_keys "password123"
9. mobile_click_on_screen_at_coordinates 200, 540
10. mobile_take_screenshot → /tmp/02-dashboard.png
11. mobile_list_elements_on_screen
    → confirmar elementos del dashboard presentes
```

**Tiempo:** ~30s vs ~10 min escribiendo test E2E.

---

## ☁️ AppSync MCP

**Package:** `awslabs.aws-appsync-mcp-server@latest` (uvx)
**Auth:** AWS SSO (usa credenciales de tu perfil activo)

### Tools Expuestos

| Tool | Función |
|------|---------|
| `create_api` | Create AppSync Event API |
| `create_graphql_api` | Create GraphQL API |
| `create_api_cache` | Habilitar caché |
| `create_api_key` | Generar API key |
| `create_channel_namespace` | Channels para Event API |
| `create_datasource` | Conectar DynamoDB/Lambda/HTTP |
| `create_domain_name` | Custom domain |
| `create_function` | Pipeline function |
| `create_resolver` | Resolver para campo GraphQL |
| `create_schema` | Schema GraphQL |

### Caso de Uso Real

**Escenario:** Crear nuevo API GraphQL para feature de chat en the-platform.

```
1. (asumiendo SSO activo)
   aws sso login --profile example-dev

2. create_graphql_api
   { name: "example-chat-api", authType: "AMAZON_COGNITO_USER_POOLS", userPoolId: "..." }

3. create_schema
   { apiId: "...", definition: "<contenido de schema.graphql>" }

4. create_datasource
   { apiId: "...", name: "MessagesTable", type: "AMAZON_DYNAMODB", tableName: "example-messages" }

5. create_resolver
   { apiId: "...", typeName: "Query", fieldName: "getMessages", dataSourceName: "MessagesTable" }
```

> **Atención:** AppSync MCP crea recursos en AWS. Verifica el perfil activo (`aws sts get-caller-identity --profile example-dev`) antes de operar.

---

## 🔍 GraphQL MCP

**Package:** `mcp-graphql`
**Tools:** Solo 2 (introspección + query)

| Tool | Función |
|------|---------|
| `introspect-schema` | Obtener schema completo del endpoint |
| `query-graphql` | Ejecutar query/mutation |

### Caso de Uso Real

**Escenario:** Explorar API GraphQL desconocido.

```
1. introspect-schema { endpoint: "https://api.example.com/graphql" }
   → schema completo

2. query-graphql
   {
     endpoint: "https://api.example.com/graphql",
     query: "query { user(id: 1) { name email posts { title } } }"
   }
```

**Combinable con AppSync MCP** para validar APIs recién creadas.

---

## 🎨 Figma MCP

**Tipo:** HTTP remote (no local npm package)
**Auth:** OAuth — requiere `mcp__figma__authenticate` + `complete_authentication`

### Tools de Auth

| Tool | Función |
|------|---------|
| `mcp__figma__authenticate` | Iniciar OAuth flow |
| `mcp__figma__complete_authentication` | Finalizar con código |

> **Estado actual:** En `~/.claude/mcp-needs-auth-cache.json` aparece `figma` con timestamp — lo que indica que aún no completaste la autenticación.

### Casos de Uso Típicos (post-auth)

- Extraer specs de un frame Figma → componente SwiftUI/React
- Listar variables de design system → mapear a tokens en Tailwind/Stylesheet
- Sync de assets (imágenes, iconos) → `assets/` del proyecto

---

## 🌐 Chrome DevTools MCP

**Package:** `@modelcontextprotocol/server-chrome-devtools`
**Activación:**
- **Claude Desktop:** auto-conecta al iniciar (config en `claude_desktop_config.json`)
- **Claude Code:** explícito con `claude --mcp-config ~/.claude/chrome-mcp.json` o vía `.mcp.json` en proyecto

### Pre-requisito: Chrome en modo debug

```bash
chrome-debug                              # alias → ~/bin/chrome-debug.sh
# o
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile
```

### Capacidades

- Inspeccionar DOM
- Ejecutar JavaScript en página activa
- Capturar screenshots
- Analizar performance (LCP, FID, CLS)
- Extraer console logs
- Network analysis (requests/responses)

### Caso de Uso Real

**Escenario:** Debug visual de feature en example-web.

```bash
# Terminal
chrome-debug http://localhost:3000/dashboard
claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug

# En Claude Code:
> "Inspecciona el sidebar y dime qué componentes están renderizados"
→ chrome usa MCP para navegar DOM, devuelve estructura

> "Ejecuta document.querySelectorAll('[data-testid]').length"
→ devuelve count

> "Analiza performance de esta página"
→ devuelve métricas Core Web Vitals
```

Ver detalles completos en `CHROME_WORKFLOW.md`.

---

## ⚡ Next DevTools MCP

**Path:** `~/.next-devtools-mcp/`
**Estado:** Activo desde 2026-04-13 — `mcp.log` tiene 3624 líneas

> **Nota importante:** Este MCP **no aparece** en `~/.claude.json` ni en `claude_desktop_config.json`. Está corriendo standalone (probablemente lanzado por Next.js DevTools cuando levantas un proyecto Next 16+).

### Verificar Estado

```bash
# Ver si está corriendo
ps aux | grep next-devtools-mcp

# Ver últimas operaciones
tail -50 ~/.next-devtools-mcp/mcp.log

# Telemetry IDs (no compartir)
cat ~/.next-devtools-mcp/telemetry-id
```

### Casos de Uso (cuando se acopla a un Next 16 project)

- Inspeccionar componentes RSC vs Client
- Analizar boundaries de Server Actions
- Debug de cache (revalidate, tags)
- Trace de Turbopack builds

---

## ⚙️ Configuración de MCPs

### Estructura de `~/.claude.json` (Claude Code)

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",                  // o "uvx", "xcrun", binario directo
      "args": ["-y", "package@latest"],
      "env": {
        "VAR_NAME": "value"
      }
    }
  }
}
```

**Ejemplo: Agregar nuevo MCP**

```json
{
  "mcpServers": {
    "filesystem-extra": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "$HOME/dev",
        "$HOME/Documents"
      ]
    }
  }
}
```

### Override Per-Sesión

```bash
# Solo para esta sesión
claude --mcp-config ~/.claude/chrome-mcp.json

# Múltiples configs (mergea)
claude --mcp-config ~/.claude/chrome-mcp.json --mcp-config ~/.claude/extra.json

# Con debug logs
claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug
```

### Auto-detect en Proyecto

Crea `.mcp.json` en el root del proyecto:

```bash
cd ~/dev/src/react/nextjs/example-web
cat > .mcp.json << 'EOF'
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-chrome-devtools"],
      "env": {"CHROME_REMOTE_DEBUGGING_PORT": "9222"}
    }
  }
}
EOF

# Claude Code lo detecta automáticamente al abrir aquí
claude
```

---

## 🔥 Workflows Multi-MCP (combinaciones poderosas)

### Workflow 1: Feature iOS de A-Z

```
1. apple-docs       → research API
2. xcode/XcodeBuildMCP → crear archivo, build
3. mobile-mcp       → test en simulador
4. graphql          → validar API call
```

**Ejemplo concreto:**
```
> "Quiero agregar pull-to-refresh con UIRefreshControl"

→ apple-docs.search_apple_docs "UIRefreshControl"
→ apple-docs.get_apple_doc_content "UIRefreshControl"
→ apple-docs.find_related_wwdc_videos "UIRefreshControl"
→ XcodeBuildMCP.build_run_sim
→ mobile-mcp.mobile_swipe_on_screen (simular pull-to-refresh)
→ mobile-mcp.mobile_take_screenshot (verificar)
```

### Workflow 2: Diseño Figma → SwiftUI Component

```
1. figma             → extraer specs del frame
2. apple-docs        → buscar APIs SwiftUI necesarias
3. xcode/XcodeWrite  → crear View
4. xcode/RenderPreview → validar visualmente
5. mobile-mcp        → probar en simulador
```

### Workflow 3: Bug en Production (Next.js + AppSync)

```
1. chrome-devtools   → reproducir bug en navegador
2. graphql           → verificar query/response del API
3. appsync           → revisar configuración del resolver
4. (Neovim + DAP)    → debugging local
```

### Workflow 4: Migración API Deprecada

```
1. apple-docs.get_documentation_updates --since "2025-01"
2. apple-docs.find_similar_apis para cada API deprecada
3. xcode/XcodeGrep → encontrar usos en codebase
4. xcode/XcodeUpdate → reemplazar
5. xcode/RunAllTests → validar
```

---

## 🔐 Seguridad y Permisos

### MCPs que tocan recursos AWS

- **appsync** → puede crear APIs, schemas, resolvers en tu cuenta AWS
- Riesgo: usar perfil incorrecto = recursos en cuenta equivocada

**Mitigación:**
```bash
# Antes de usar appsync MCP, confirmar perfil activo
aws sts get-caller-identity --profile example-dev

# Si está mal, switch
aws sso login --profile example-dev
export AWS_PROFILE=example-dev
```

### MCPs que envían datos a terceros

- **figma** → datos de tu Figma a Anthropic + Figma API
- **apple-docs** → queries van a un servidor third-party (no Apple oficial)
- **graphql** → queries pueden incluir tokens si no se separan

**Buena práctica:** No exponer secrets vía args/env en `.mcp.json` que se commitee.

### MCPs que ejecutan código local

- **mobile-mcp**, **XcodeBuildMCP**, **xcode** → corren xcrun, npx, etc. en tu sistema
- **chrome-devtools** → ejecuta JS en Chrome remoto

**Mitigación:** Solo configurar MCPs de fuentes confiables (`@modelcontextprotocol/*`, `awslabs/*`, packages oficiales).

### Variables Sensibles

```bash
# NUNCA commitear .mcp.json con secrets
echo ".mcp.json" >> .gitignore

# En su lugar, usar env vars
export GRAPHQL_ENDPOINT="..."
export GRAPHQL_TOKEN="..."

# Y referenciar en .mcp.json
{
  "mcpServers": {
    "graphql": {
      "command": "npx",
      "args": ["-y", "mcp-graphql"],
      "env": {
        "ENDPOINT": "${GRAPHQL_ENDPOINT}",
        "TOKEN": "${GRAPHQL_TOKEN}"
      }
    }
  }
}
```

---

## 🐛 Troubleshooting

### MCP no carga al iniciar Claude Code

```bash
# 1. Validar JSON
cat ~/.claude.json | jq . > /dev/null && echo "OK" || echo "JSON inválido"

# 2. Ver logs en sesión
claude --mcp-debug

# 3. Test manual del MCP server
npx -y @kimsungwhee/apple-docs-mcp@latest    # Ctrl+C para salir
```

### "MCP server timeout"

```bash
# Server tarda mucho en arrancar (npx descarga package)
# Solución: pre-instalar global
npm install -g xcodebuildmcp

# Luego cambiar config:
{
  "mcpServers": {
    "XcodeBuildMCP": {
      "command": "xcodebuildmcp",   // ya no usa npx
      "args": ["mcp"]
    }
  }
}
```

### "No tools available" en sesión

- Causa común: el MCP server crashed silenciosamente
- Verificar:
```bash
# Logs de Claude Code
ls ~/.claude/

# Re-iniciar sesión con --mcp-debug
claude --mcp-debug
```

### appsync MCP: "AccessDenied"

```bash
# SSO expirado
aws sso login --profile example-dev

# O perfil no exportado
export AWS_PROFILE=example-dev
claude
```

### figma MCP: "Authentication required"

```
> "Authenticate to Figma"
→ Claude llama mcp__figma__authenticate
→ devuelve URL → abrir en navegador → autorizar
→ Claude llama mcp__figma__complete_authentication con código
```

### chrome-devtools MCP no conecta

```bash
# Verificar Chrome corriendo con debugging
lsof -i :9222

# Si vacío:
chrome-debug
```

---

## 📊 Métricas de Productividad

| Tarea | Sin MCP | Con MCP | Mejora |
|-------|---------|---------|--------|
| Buscar API en Apple docs | ~25s (developer.apple.com) | ~4s (apple-docs MCP) | **6x** |
| Build + test iOS app | ~60s (Xcode UI clicks) | ~15s (XcodeBuildMCP) | **4x** |
| Test E2E mobile (3 pasos) | ~10 min (escribir Detox) | ~30s (mobile-mcp) | **20x** |
| Crear AppSync API completo | ~15 min (CDK + deploy) | ~3 min (appsync MCP) | **5x** |
| Inspeccionar GraphQL API | ~5 min (Postman + curl) | ~10s (graphql MCP) | **30x** |
| Debug visual web | ~3 min (DevTools manual) | ~30s (chrome-devtools MCP) | **6x** |
| Extraer specs Figma | ~2 min (manual + screenshots) | ~10s (figma MCP) | **12x** |
| Sync component design system | ~30 min (manual) | ~5 min (figma → code) | **6x** |

**Promedio:** ~11x más rápido en tareas de integración cross-tool

---

## 🎯 Casos de Uso Reales (Tu Stack)

### Caso 1: Desarrollar feature en example-mobile (Expo + AppSync)

```
1. apple-docs        → research de API iOS necesaria
2. mobile-mcp        → ver app actual corriendo
3. graphql           → introspect AppSync schema
4. appsync           → crear nuevo resolver si falta
5. (Neovim)          → escribir código RN
6. mobile-mcp        → screenshot + interacción de prueba
```

### Caso 2: Hot fix en producción example-web

```
1. chrome-devtools   → reproducir bug en chrome remoto
2. graphql           → confirmar response del API
3. (Neovim + DAP)    → fix
4. chrome-devtools   → validar fix
```

### Caso 3: Auditar uso de APIs deprecadas en ios-reference-app

```
1. apple-docs.get_documentation_updates --since "2024-09" --framework "UIKit"
   → lista de deprecaciones
2. xcode.XcodeGrep para cada API deprecada en proyecto
3. apple-docs.find_similar_apis → reemplazo recomendado
4. xcode.XcodeUpdate → migrar
5. xcode.RunAllTests → validar
```

### Caso 4: Onboarding de feature design en Figma

```
1. figma.get_frame "Login Flow"
2. apple-docs.search_apple_docs "TextField iOS 26 design"
3. xcode.XcodeWrite (crear View)
4. xcode.RenderPreview → comparar visualmente
5. (iterar hasta match)
```

---

## ⌨️ Aliases Recomendados

Agregar a `~/.zshrc`:

```bash
# MCP shortcuts
alias claude-chrome="claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug"
alias claude-debug="claude --mcp-debug"

# Verificar configs MCP
alias mcp-list="cat ~/.claude.json | jq '.mcpServers | keys'"
alias mcp-show="cat ~/.claude.json | jq '.mcpServers'"
alias mcp-validate="cat ~/.claude.json | jq . > /dev/null && echo '✓ JSON valid' || echo '✗ Invalid'"

# Logs
alias mcp-logs-next="tail -50 ~/.next-devtools-mcp/mcp.log"

# Auth helpers
alias aws-example-dev="export AWS_PROFILE=example-dev && aws sso login"
alias aws-example-prod="export AWS_PROFILE=example-prod && aws sso login"
```

---

## 🚀 Recomendaciones de MCPs Adicionales

Considerando tu stack, estos MCPs aportarían valor:

| MCP | Package | Por qué |
|-----|---------|---------|
| **filesystem** | `@modelcontextprotocol/server-filesystem` | Ya tienes acceso vía Read/Edit, pero útil para sandbox limitado |
| **github** | `@modelcontextprotocol/server-github` | Issues/PRs/releases sin salir de Claude |
| **postgres** | `@modelcontextprotocol/server-postgres` | Query directo a Aurora si lo usas |
| **slack** | `@modelcontextprotocol/server-slack` | Notificar deploys |
| **memory** | `@modelcontextprotocol/server-memory` | Conocimiento persistente cross-session |
| **fetch** | `@modelcontextprotocol/server-fetch` | HTTP requests sin tener que pedir Bash |
| **sentry** | `@sentry/mcp-server` | Triage de errores en producción |

**No agregar todos** — cada MCP carga al iniciar Claude. Mantener <10 para startup rápido.

---

## 📚 Referencias

- **MCP spec oficial:** https://modelcontextprotocol.io
- **MCP servers oficiales:** https://github.com/modelcontextprotocol/servers
- **Claude Code MCP docs:** https://docs.claude.com/en/docs/claude-code/mcp
- **Awesome MCP:** https://github.com/punkpeye/awesome-mcp-servers

### Documentación Relacionada

- [README.md](./README.md) — Setup general
- [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md) — Detalles de XcodeBuildMCP + xcode (mcpbridge)
- [CHROME_WORKFLOW.md](./CHROME_WORKFLOW.md) — chrome-devtools MCP en profundidad
- [EXPO_WORKFLOW.md](./EXPO_WORKFLOW.md) — mobile-mcp aplicado a example-mobile
- [AWS_WORKFLOW.md](./AWS_WORKFLOW.md) — appsync MCP en contexto del stack AWS

---

**Creado:** 2026-05-02
**MCPs activos:** 7 (Claude Code) + 1 (Claude Desktop) + 1 standalone (Next DevTools)
**Tools totales expuestos:** ~120+
**Filosofía:** Claude opera tools externos directamente. Nunca copy/paste entre ventanas.
