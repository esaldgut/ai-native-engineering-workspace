# 🌐 Chrome DevTools + MCP Integration Workflow

**Fecha:** 2025-11-18
**Stack:** Chrome 142 + Claude Desktop + Neovim DAP + MCP
**Propósito:** Debugging profesional de aplicaciones web con integración completa

---

## 📋 Índice

1. [Setup Completo](#1-setup-completo)
2. [Workflows de Debugging](#2-workflows-de-debugging)
3. [Integración con Claude Desktop + MCP](#3-integración-con-claude-desktop--mcp)
4. [chrome-cli: Automatización](#4-chrome-cli-automatización)
5. [Keymaps y Atajos](#5-keymaps-y-atajos)
6. [Troubleshooting](#6-troubleshooting)
7. [Tips Profesionales](#7-tips-profesionales)

---

## 1. Setup Completo

### 1.1 Herramientas Instaladas

| Herramienta | Versión | Ubicación | Propósito |
|-------------|---------|-----------|-----------|
| Google Chrome | 142.0.7444.176 | /Applications/Google Chrome.app | Browser principal |
| Claude Desktop | 0.14.10 | /Applications/Claude.app | AI Assistant con MCP |
| chrome-cli | 1.11.0 | /opt/homebrew/bin/chrome-cli | Control CLI de Chrome |
| chromedriver | 142.0.7444.175 | /opt/homebrew/bin/chromedriver | Selenium WebDriver |

### 1.2 Scripts Creados

**chrome-debug.sh** (`~/bin/chrome-debug.sh`)
```bash
#!/bin/bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile \
  --no-first-run \
  --no-default-browser-check \
  "$@"
```

**Uso:**
```bash
chrome-debug                          # Lanza Chrome en modo debug
chrome-debug http://localhost:3000    # Abre URL específica
```

### 1.3 Aliases Disponibles

| Alias | Comando Completo | Uso |
|-------|------------------|-----|
| `chrome-debug` | `~/bin/chrome-debug.sh` | Lanzar Chrome con debugging |
| `chrome-open` | `chrome-cli open` | Abrir URL en Chrome |
| `chrome-tabs` | `chrome-cli list tabs` | Listar tabs abiertos |
| `chrome-exec` | `chrome-cli execute` | Ejecutar JavaScript |
| `chrome-close` | `chrome-cli close` | Cerrar tab activo |
| `chrome-reload` | `chrome-cli reload` | Recargar tab activo |

### 1.4 Configuración MCP

**Archivo:** `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-chrome-devtools"
      ],
      "env": {
        "CHROME_REMOTE_DEBUGGING_PORT": "9222"
      }
    }
  }
}
```

**Verificar configuración:**
```bash
cat ~/Library/Application\ Support/Claude/claude_desktop_config.json | jq .
```

---

## 2. Workflows de Debugging

### 2.1 Debugging Next.js desde Neovim

**Prerequisitos:**
- Proyecto Next.js con `npm run dev` corriendo
- Neovim abierto con DAP configurado
- js-debug-adapter instalado (ver Fase 2)

**Paso a paso:**

```bash
# Terminal 1: Iniciar Next.js dev server
cd ~/dev/src/react/nextjs/mi-proyecto
npm run dev

# Terminal 2 (o tmux pane): Lanzar Chrome en modo debug
chrome-debug

# Neovim: Abrir archivo TypeScript/JavaScript
nvim app/page.tsx

# En Neovim:
# 1. Establecer breakpoint
<leader>db    # Tu keymap existente

# 2. Iniciar debugging
<leader>dc    # Start/continue debug

# 3. Chrome se conecta automáticamente
# 4. Navegar en Chrome activará los breakpoints
# 5. DAP UI muestra variables, call stack, console logs
```

**Configuración DAP existente** (en `~/.config/nvim/lua/config/dap.lua`):
```lua
-- Next.js debugging (puerto 9229)
{
  type = 'pwa-node',
  request = 'attach',
  name = 'Next.js: Attach',
  skipFiles = { '<node_internals>/**', 'node_modules/**' },
  port = 9229,
  cwd = '${workspaceFolder}',
  sourceMaps = true,
}
```

### 2.2 Debugging React/TypeScript (General)

**Para aplicaciones React sin Next.js:**

```bash
# 1. Lanzar Chrome en modo debug
chrome-debug http://localhost:3000

# 2. En Neovim, abrir archivo
nvim src/App.tsx

# 3. Establecer breakpoints
<leader>db

# 4. En Chrome DevTools (F12)
# - Sources → Filesystem → Add folder to workspace
# - Seleccionar tu proyecto
# - Los breakpoints de Neovim se sincronizan

# 5. Alternativamente: Usar DAP attach
:lua require('dap').run({
  type = 'pwa-chrome',
  request = 'attach',
  name = 'Attach to Chrome',
  port = 9222,
  webRoot = vim.fn.getcwd(),
})
```

### 2.3 Debugging Tests con Jest

**Ya configurado en tu DAP:**

```vim
" En archivo de test (.test.ts, .spec.ts)
<leader>tr    " Run nearest test (Neotest)
<leader>db    " Establecer breakpoint antes del test
<leader>dc    " Debug test

" DAP adjunta a Jest runner
" Breakpoints funcionan dentro de tests
" Variables de test visibles en DAP UI
```

### 2.4 Workflow Completo: Feature → Debug → Fix

```bash
# 1. Crear feature branch
git checkout -b feature/nueva-funcionalidad

# 2. Desarrollo en Neovim
nvim app/features/nueva-funcionalidad.tsx

# 3. Testing con Neotest
<leader>tr    # Run test

# 4. Si falla, debug
<leader>db    # Breakpoint en función problemática
<leader>dc    # Debug

# 5. Inspeccionar variables en DAP UI
<leader>du    # Toggle DAP UI

# 6. Fix y verificar
<leader>tr    # Re-run test

# 7. Commit cuando pasa
<leader>gg   # LazyGit
c            # Commit
```

---

## 3. Integración con Claude Desktop + MCP

### 3.1 ¿Qué es MCP?

**Model Context Protocol (MCP)** permite a Claude Desktop interactuar directamente con Chrome DevTools:

- Inspeccionar elementos DOM
- Ejecutar JavaScript en página
- Analizar performance
- Capturar screenshots
- Obtener console logs
- Network analysis

### 3.2 Activar MCP en Claude Desktop

**Paso 1: Iniciar Chrome con debugging**
```bash
chrome-debug http://localhost:3000
```

**Paso 2: Verificar puerto 9222**
```bash
lsof -i :9222
# Debe mostrar Chrome escuchando
```

**Paso 3: Abrir Claude Desktop**
```bash
open -a Claude
```

**Paso 4: Verificar MCP connection**
- Settings → Developer → MCP Servers
- Debe aparecer: `chrome-devtools` (connected)

### 3.3 Comandos MCP Disponibles

**En Claude Desktop, puedes decir:**

```
"Inspecciona la página en http://localhost:3000"
→ Claude usa MCP para conectarse y mostrar estructura DOM

"Ejecuta console.log('test') en la página actual"
→ Ejecuta JavaScript y muestra resultado

"Toma un screenshot de la aplicación"
→ Captura imagen de la página

"Analiza el performance de esta página"
→ Genera reporte de métricas (LCP, FID, CLS)

"Muestra los console logs"
→ Extrae y formatea todos los logs

"¿Qué requests de red está haciendo?"
→ Lista todas las peticiones HTTP/HTTPS
```

### 3.4 Workflow: Debugging con AI

**Escenario:** Bug visual en componente React

```bash
# 1. Reproducir bug en Chrome
chrome-debug http://localhost:3000/problema

# 2. Abrir Claude Desktop

# 3. En Claude:
"Inspecciona el elemento con clase 'problematic-component'"

# Claude responde con:
<div class="problematic-component" style="display: none;">
  <!-- Contenido... -->
</div>

# 4. Identificar problema (display: none no intencionado)

# 5. En Neovim, buscar y corregir
:Telescope live_grep display: none

# 6. Verificar fix en Claude:
"Verifica que el elemento ahora sea visible"
```

### 3.5 MCP + Neovim: Workflow Híbrido

```
┌─────────────────────────────────────────────────┐
│             Desarrollo Híbrido                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Neovim (Coding)                             │
│     ├── Escribir código TypeScript/React       │
│     ├── LSP autocompletion                     │
│     └── Snippets (Next.js, AWS)                │
│                                                 │
│  2. Chrome (Testing)                            │
│     ├── Renderizar aplicación                  │
│     ├── Breakpoints visuales                   │
│     └── Remote debugging (port 9222)           │
│                                                 │
│  3. Claude Desktop + MCP (AI Analysis)         │
│     ├── Inspeccionar DOM                       │
│     ├── Analizar performance                   │
│     ├── Sugerir fixes                          │
│     └── Generar screenshots                    │
│                                                 │
│  4. Neovim DAP (Deep Debugging)                │
│     ├── Breakpoints en código                  │
│     ├── Step debugging                         │
│     ├── Variables inspection                   │
│     └── Call stack analysis                    │
└─────────────────────────────────────────────────┘
```

### 3.6 Integración con Claude Code (CLI)

**Claude Code** es la herramienta CLI para coding (lo que estás usando ahora mismo), diferente de Claude Desktop (GUI).

#### Configuración MCP para Claude Code

**Opción 1: Archivo MCP Centralizado (Recomendado)**

```bash
# Archivo ya creado en ~/.claude/chrome-mcp.json
cat ~/.claude/chrome-mcp.json
```

**Uso con flag `--mcp-config`:**
```bash
# Lanzar Claude Code con Chrome MCP
claude --mcp-config ~/.claude/chrome-mcp.json

# Con debugging habilitado
claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug
```

**Opción 2: Por Proyecto (.mcp.json)**

```bash
# En tu proyecto Next.js
cd ~/dev/src/react/nextjs/my-app
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

# Claude Code detectará automáticamente .mcp.json al iniciar
claude
```

#### Verificar MCP en Claude Code

```bash
# Iniciar con debug para ver logs del MCP server
claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug

# Dentro de Claude Code, preguntar:
> Can you list the Chrome DevTools MCP tools available?

# Debería mostrar herramientas como:
# - chrome_devtools_navigate
# - chrome_devtools_execute
# - chrome_devtools_screenshot
# - chrome_devtools_console_logs
```

#### Comandos Útiles para Claude Code + Chrome

**En una sesión de Claude Code con MCP habilitado:**

```bash
# 1. Iniciar Chrome en modo debug
chrome-debug http://localhost:3000

# 2. En otra terminal, iniciar Claude Code
claude --mcp-config ~/.claude/chrome-mcp.json

# 3. Dentro de Claude Code, puedes pedir:
> Navigate to http://localhost:3000 and take a screenshot

> Execute console.log('Hello from Claude Code') in the active tab

> Get all console logs from the current page

> Analyze the page performance and show me metrics
```

#### Alias Útil para Claude Code + Chrome

Agregar a `~/.zshrc`:
```bash
alias claude-chrome="claude --mcp-config ~/.claude/chrome-mcp.json --mcp-debug"
```

Luego:
```bash
# Uso rápido
claude-chrome
> Inspect the DOM of http://localhost:3000
```

#### Diferencias: Claude Desktop vs Claude Code

| Característica | Claude Desktop (GUI) | Claude Code (CLI) |
|----------------|----------------------|-------------------|
| **Config MCP** | `~/Library/Application Support/Claude/claude_desktop_config.json` | `~/.claude/chrome-mcp.json` o `.mcp.json` |
| **Auto-detect** | ✅ Auto-load al abrir app | ⚠️ Requiere `--mcp-config` flag o `.mcp.json` en proyecto |
| **Scope** | Global | Por sesión / por proyecto |
| **Debugging** | Limited visibility | `--mcp-debug` flag |
| **Uso típico** | Quick inspections, screenshots | Automated testing, CI/CD integration |

---

## 4. chrome-cli: Automatización

### 4.1 Comandos Básicos

```bash
# Abrir URL
chrome-cli open https://github.com

# Abrir en nueva ventana
chrome-cli open -n https://example.com

# Listar tabs
chrome-cli list tabs
# Output:
# [0] Google - https://google.com
# [1] GitHub - https://github.com

# Listar ventanas
chrome-cli list windows

# Activar tab por índice
chrome-cli activate 0

# Cerrar tab
chrome-cli close
chrome-cli close 1  # Cerrar tab específico

# Recargar página
chrome-cli reload

# Ejecutar JavaScript
chrome-cli execute "document.title"
chrome-cli execute "console.log('Hello from CLI')"
```

### 4.2 Scripts de Automatización

**test-deployment.sh**
```bash
#!/bin/bash
# Script para testing de deployment

# 1. Abrir staging
chrome-cli open https://staging.miapp.com

# 2. Esperar carga
sleep 3

# 3. Verificar título
TITLE=$(chrome-cli execute "document.title")
echo "Page title: $TITLE"

# 4. Verificar si hay errores
ERRORS=$(chrome-cli execute "console.log(window.errors || 'No errors')")
echo "Console errors: $ERRORS"

# 5. Capturar estado
chrome-cli execute "localStorage.getItem('user')"
```

**multi-env-testing.sh**
```bash
#!/bin/bash
# Abrir múltiples ambientes en tabs

chrome-cli open http://localhost:3000          # Local
chrome-cli open https://staging.miapp.com      # Staging
chrome-cli open https://production.miapp.com   # Production

# Comparar títulos
for i in 0 1 2; do
  chrome-cli activate $i
  TITLE=$(chrome-cli execute "document.title")
  echo "Tab $i: $TITLE"
done
```

### 4.3 Integración con Testing E2E

**Ejemplo con Selenium + chromedriver:**

```javascript
// tests/e2e/login.test.js
const { Builder, By, until } = require('selenium-webdriver');
const chrome = require('selenium-webdriver/chrome');

describe('Login Flow', () => {
  let driver;

  beforeAll(async () => {
    const options = new chrome.Options();
    options.addArguments('--remote-debugging-port=9222');

    driver = await new Builder()
      .forBrowser('chrome')
      .setChromeOptions(options)
      .build();
  });

  test('should login successfully', async () => {
    await driver.get('http://localhost:3000/login');

    await driver.findElement(By.id('email')).sendKeys('test@example.com');
    await driver.findElement(By.id('password')).sendKeys('password123');
    await driver.findElement(By.css('button[type="submit"]')).click();

    await driver.wait(until.urlIs('http://localhost:3000/dashboard'), 5000);

    const title = await driver.getTitle();
    expect(title).toContain('Dashboard');
  });

  afterAll(async () => {
    await driver.quit();
  });
});
```

---

## 5. Keymaps y Atajos

### 5.1 Neovim DAP (Ya Configurados)

| Keymap | Acción | Descripción |
|--------|--------|-------------|
| `<leader>db` | Toggle breakpoint | Establecer/quitar breakpoint |
| `<leader>dc` | Start/Continue | Iniciar debugging o continuar |
| `<leader>dt` | Terminate | Detener debugging |
| `<leader>dr` | Restart | Reiniciar debugger |
| `<leader>du` | Toggle UI | Mostrar/ocultar DAP UI |
| `<leader>dh` | Hover | Ver valor de variable |
| `<F5>` | Continue | Continuar ejecución |
| `<F10>` | Step Over | Siguiente línea (sin entrar en función) |
| `<F11>` | Step Into | Entrar en función |
| `<F12>` | Step Out | Salir de función |

### 5.2 Terminal Aliases (Recién Agregados)

| Alias | Función |
|-------|---------|
| `chrome-debug` | Lanzar Chrome en modo debugging |
| `chrome-open <url>` | Abrir URL en Chrome |
| `chrome-tabs` | Listar tabs abiertos |
| `chrome-exec "<js>"` | Ejecutar JavaScript |
| `chrome-close` | Cerrar tab activo |
| `chrome-reload` | Recargar página actual |

### 5.3 Combinaciones Útiles

**Debugging Flow Completo:**
```vim
" 1. Abrir archivo con bug
:e app/components/Problematic.tsx

" 2. Establecer breakpoint en línea sospechosa
<leader>db

" 3. Iniciar debugging
<leader>dc

" 4. En Chrome, reproducir bug
" → Execution se detiene en breakpoint

" 5. Inspeccionar variables
<leader>dh    " Hover sobre variable
<leader>du    " Ver DAP UI con todas las variables

" 6. Step through código
<F10>         " Next line
<F11>         " Step into function

" 7. Fix el código mientras debuggeas
:w            " Guardar

" 8. Restart debugging con nuevo código
<leader>dr
```

---

## 6. Troubleshooting

### 6.1 Chrome no conecta en modo debug

**Síntoma:** Chrome no escucha en puerto 9222

**Diagnóstico:**
```bash
lsof -i :9222
# Si no muestra nada, Chrome no está en modo debug
```

**Solución:**
```bash
# Opción 1: Usar script
chrome-debug

# Opción 2: Lanzar manualmente
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug-profile
```

### 6.2 MCP Server no conecta en Claude Desktop

**Síntoma:** Claude Desktop muestra "MCP server error"

**Diagnóstico:**
```bash
# Verificar config existe
cat ~/Library/Application\ Support/Claude/claude_desktop_config.json

# Verificar npx funciona
which npx
npx --version

# Test manual del MCP server
npx -y @modelcontextprotocol/server-chrome-devtools
```

**Solución:**
```bash
# 1. Verificar Node.js instalado
node --version
npm --version

# 2. Verificar sintaxis JSON
cat ~/Library/Application\ Support/Claude/claude_desktop_config.json | jq .

# 3. Reiniciar Claude Desktop
pkill Claude
open -a Claude

# 4. Ver logs de Claude
cat ~/Library/Logs/Claude/*.log
```

### 6.3 Neovim DAP no encuentra js-debug-adapter

**Síntoma:** Error al iniciar debugging: "Adapter not found"

**Diagnóstico:**
```bash
# Verificar instalación
ls -la ~/.local/share/nvim/mason/packages/js-debug-adapter
```

**Solución:**
```vim
" En Neovim
:Mason
:MasonInstall js-debug-adapter

" Verificar health
:checkhealth dap
```

### 6.4 Puerto 9222 ya en uso

**Síntoma:** Chrome no puede abrir puerto 9222

**Diagnóstico:**
```bash
lsof -i :9222
# Muestra proceso usando el puerto
```

**Solución:**
```bash
# Opción 1: Matar proceso
kill -9 <PID>

# Opción 2: Usar puerto diferente
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9223  # Puerto alternativo
```

### 6.5 chromedriver Gatekeeper Error

**Síntoma:** chromedriver bloqueado por macOS security

**Solución:**
```bash
# Remover quarantine attribute
xattr -d com.apple.quarantine /opt/homebrew/bin/chromedriver

# O permitir en System Preferences
# Security & Privacy → General → "Allow chromedriver"
```

### 6.6 Breakpoints no funcionan en TypeScript

**Síntoma:** Breakpoints aparecen como grises (no activos)

**Diagnóstico:**
- Source maps no están disponibles
- Build tool no genera .map files

**Solución:**
```javascript
// next.config.js
module.exports = {
  productionBrowserSourceMaps: true,  // Habilitar source maps
};

// tsconfig.json
{
  "compilerOptions": {
    "sourceMap": true  // Generar source maps
  }
}
```

---

## 7. Tips Profesionales

### 7.1 Performance Optimization

**Minimizar overhead de debugging:**

```bash
# Usar perfil temporal (ya en chrome-debug.sh)
--user-data-dir=/tmp/chrome-debug-profile

# Deshabilitar extensiones
--disable-extensions

# Deshabilitar GPU (si hay problemas gráficos)
--disable-gpu
```

### 7.2 Debugging Múltiples Puertos

**Escenario:** Next.js (3000) + API (4000) + Database UI (5432)

```bash
# Terminal 1: Next.js
npm run dev

# Terminal 2: API
cd api && go run main.go

# Terminal 3: Chrome con debugging
chrome-debug http://localhost:3000

# Neovim DAP puede adjuntar a ambos:
# - Next.js en puerto 9229
# - Chrome en puerto 9222
```

### 7.3 Snapshots de Estado

**Capturar estado completo de aplicación:**

```javascript
// En Chrome DevTools Console o via chrome-cli
const appState = {
  url: window.location.href,
  localStorage: { ...localStorage },
  sessionStorage: { ...sessionStorage },
  cookies: document.cookie,
  redux: window.__REDUX_DEVTOOLS_EXTENSION__?.store?.getState(),
  console: console.history || []
};

console.log(JSON.stringify(appState, null, 2));
```

**Automatizar con chrome-cli:**
```bash
chrome-cli execute "JSON.stringify({url: location.href, storage: {...localStorage}})" > app-state.json
```

### 7.4 Conditional Breakpoints

**En Neovim DAP:**

```vim
" Establecer breakpoint condicional
<leader>db

" En el prompt que aparece, escribir condición:
user.id === 123

" Breakpoint solo se activa si user.id es 123
```

### 7.5 Logpoints (Breakpoints sin parar)

**En lugar de console.log(), usar logpoints:**

```vim
" Establecer logpoint
:lua require('dap').set_breakpoint(nil, nil, "User ID: {userId}")

" Imprime mensaje sin detener ejecución
" Aparece en DAP UI console
```

### 7.6 Watch Expressions

**Monitorear expresiones durante debugging:**

```vim
" En DAP UI, agregar watch
:lua require('dap.ui.widgets').hover()

" O en DAP UI sidebar:
" Click en "+" en sección Watches
" Agregar: user.email, state.isLoading, etc.
```

### 7.7 Remote Debugging (AWS Amplify)

**Debugging en ambiente staging:**

```bash
# 1. Habilitar source maps en producción
# (Ya configurado en next.config.js con productionBrowserSourceMaps: true)

# 2. Abrir app staging en Chrome
chrome-debug https://staging.miapp.amplifyapp.com

# 3. En Chrome DevTools:
# Sources → Filesystem → Add folder
# Mapear código local a staging

# 4. Breakpoints funcionan en código local
# Ejecutan contra staging
```

### 7.8 Testing Matrix con chrome-cli

**Probar múltiples rutas automáticamente:**

```bash
#!/bin/bash
# test-routes.sh

ROUTES=(
  "/"
  "/dashboard"
  "/profile"
  "/settings"
)

for route in "${ROUTES[@]}"; do
  chrome-cli open "http://localhost:3000$route"
  sleep 2

  # Verificar sin errores
  ERRORS=$(chrome-cli execute "window.errors?.length || 0")
  echo "Route $route: $ERRORS errors"

  chrome-cli close
done
```

---

## 📊 Resumen de Puertos

| Puerto | Servicio | Uso |
|--------|----------|-----|
| 9222 | Chrome Remote Debugging | MCP + debugging externo |
| 9229 | Node.js Inspector (Next.js) | Neovim DAP → Next.js |
| 3000 | Next.js Dev Server | Aplicación frontend |
| 4000 | API Backend (ejemplo) | Backend services |

**Verificar todos los puertos:**
```bash
lsof -i :9222,9229,3000,4000
```

---

## 🎯 Workflows Completos (Ejemplos Reales)

### Workflow 1: Fix Bug en Producción

```bash
# 1. Reproducir en local
npm run dev
chrome-debug http://localhost:3000/bug-page

# 2. Debugging con Neovim
nvim app/bug-component.tsx
<leader>db    # Breakpoint
<leader>dc    # Debug

# 3. Identificar causa raíz
# (Usar DAP UI para inspeccionar)

# 4. Probar fix
:w
<leader>dr    # Restart debugging

# 5. Verificar con MCP
# Claude: "Verifica que el bug esté corregido"

# 6. Test automatizado
npm test

# 7. Commit y deploy
<leader>gg    # LazyGit
# feat(bug): fix [descripción]
```

### Workflow 2: Analizar Performance

```bash
# 1. Lanzar Chrome con profiling
chrome-debug --enable-precise-memory-info http://localhost:3000

# 2. En Claude Desktop (MCP):
"Analiza el performance de la página actual"

# Claude muestra:
# - LCP: 2.1s (Good)
# - FID: 45ms (Needs Improvement)
# - CLS: 0.05 (Good)

# 3. Identificar bottlenecks
chrome-cli execute "performance.getEntries().filter(e => e.duration > 100)"

# 4. Optimizar código en Neovim

# 5. Re-test
chrome-reload
# Claude: "Analiza performance nuevamente"
```

### Workflow 3: Debug E2E Test Failure

```bash
# 1. Test falla en CI
npm run test:e2e  # Falla

# 2. Reproducir localmente con debugging
chrome-debug
npm run test:e2e:debug

# 3. Breakpoint en test file
nvim tests/e2e/checkout.test.js
<leader>db    # En línea que falla

# 4. Ver estado de aplicación
<leader>du    # DAP UI muestra variables

# 5. Ejecutar JavaScript ad-hoc
chrome-cli execute "document.querySelector('.checkout-button')"

# 6. Fix selector/timing issue
:w

# 7. Re-run test
npm run test:e2e  # Ahora pasa ✓
```

---

## 🔗 Referencias

### Documentación

- **Chrome DevTools Protocol:** https://chromedevtools.github.io/devtools-protocol/
- **Model Context Protocol:** https://modelcontextprotocol.io
- **chrome-cli GitHub:** https://github.com/prasmussen/chrome-cli
- **Neovim DAP:** https://github.com/mfussenegger/nvim-dap

### Archivos de Configuración

- MCP Config: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Neovim DAP: `~/.config/nvim/lua/config/dap.lua`
- Chrome Script: `~/bin/chrome-debug.sh`
- Aliases: `~/.zshrc` (líneas 53-59)

### Otros Workflows

- **Neovim General:** `~/.config/nvim/README.md`
- **Next.js Workflow:** `~/.config/nvim/NEXTJS_WORKFLOW.md`
- **AWS Workflow:** `~/.config/nvim/AWS_WORKFLOW.md`
- **Keymaps Completos:** `~/.config/nvim/KEYMAPS.md`

---

**Creado:** 2025-11-18
**Última Actualización:** 2025-11-18
**Autor:** Erick Aldama
**Stack:** Next.js 15 + AWS Amplify v6 + Chrome 142 + Claude Desktop + Neovim

---

**Happy Debugging!** 🐛🔍
