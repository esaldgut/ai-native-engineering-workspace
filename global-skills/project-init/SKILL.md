---
name: project-init
description: Inicializa o audita un proyecto contra la documentación base de plataforma del CLI workflow (~/.config/nvim/*_WORKFLOW.md). Aplica la premisa universal "ningún proyecto inicia de manera ambigua": el proyecto obtiene contexto desde la base genérica de plataforma y luego la extiende. Auto-invocar al abrir un proyecto sin bloque "Platform Base Context" en su CLAUDE.md, cuando el usuario pida "inicializar/bootstrap contexto", o al crear un proyecto nuevo. Genera/verifica el bloque de contexto base en CLAUDE.md y sugiere docs/<DOMAIN>_EXTENSIONS.md.
---

# project-init — Inicialización de Contexto de Plataforma

Implementa la **regla de inicialización de proyectos** definida en
`~/.config/nvim/PLATFORM_BASE.md`:

> Cada proyecto debe comenzar con contexto explícito. No se permite iniciar de
> manera ambigua. El primer prompt obtiene contexto desde la documentación base
> de plataforma (genérica); el proyecto particular la extiende a sus necesidades.

El objetivo **no** es la doc base. El objetivo es que el proyecto, al iniciar,
**evite la ambigüedad obteniendo contexto primero**.

---

## Cuándo invocar este skill

**Auto-invocar siempre que ocurra alguno de estos triggers:**

1. Se abre un proyecto cuyo `CLAUDE.md` **no tiene** un bloque `## Platform Base Context`.
2. El proyecto **no tiene** `CLAUDE.md` (bootstrap de proyecto nuevo).
3. El usuario pide explícitamente "inicializar contexto", "bootstrap", "carga el contexto base", o "alinear este proyecto con la plataforma".
4. Se va a empezar trabajo sustancial en un repo que el modelo no reconoce.

**Anunciar al invocarse:** "Usando `project-init` para cargar el contexto base de plataforma y verificar la inicialización de este proyecto."

---

## La Documentación Base (fuente de verdad)

Vive en `~/.config/nvim/`. Es **genérica, de plataforma, no vinculada a proyecto**.

| Dominio | Doc base | Señales en el repo (cómo se detecta) |
|---------|----------|--------------------------------------|
| Apple/iOS | `XCODE_WORKFLOW.md` | `*.xcodeproj`, `*.xcworkspace`, `Package.swift`, `project.yml` |
| Android nativo | `ANDROID_WORKFLOW.md` | `build.gradle(.kts)`, `settings.gradle(.kts)`, `gradlew`, `AndroidManifest.xml`, `*.kt` |
| Mobile/Expo | `EXPO_WORKFLOW.md` | `app.json` con `"expo"`, `eas.json`, `expo` en deps |
| Web/Next.js | `NEXTJS_WORKFLOW.md` | `next.config.*`, `next` en deps |
| TypeScript | `TYPESCRIPT_CONTEXT.md` | `tsconfig.json` |
| AWS | `AWS_WORKFLOW.md` | `amplify/`, `cdk.json`, `samconfig.*`, SDK aws en deps |
| MCP | `MCP_WORKFLOW.md` | `.mcp.json`, MCP servers en `~/.claude.json` |
| Chrome | `CHROME_WORKFLOW.md` | front-end con remote debugging / e2e |
| Shell | `SHELL_WORKFLOW.md` | siempre aplicable (zsh) |
| Scripts/tmux | `SCRIPTS_WORKFLOW.md` | `~/bin/` scripts, sesiones tmux |
| Lab seguridad | `KALI_WORKFLOW.md` | trabajo de pentesting / VMs |

**Premisa:** la base es el medio. No se edita la base para meter algo específico de
un proyecto. Si se descubre conocimiento **genérico reutilizable**, SÍ se aporta a la
base; lo específico va al proyecto.

---

## Workflow

### Paso 1 — Detectar dominios aplicables

Si el usuario pasó dominios como argumento (`/project-init apple,aws,mcp`), úsalos.
Si no, detéctalos escaneando el repo:

```bash
ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$ROOT"

# Apple
ls *.xcodeproj *.xcworkspace Package.swift project.yml 2>/dev/null && echo "→ apple"
# Android nativo
ls build.gradle build.gradle.kts settings.gradle settings.gradle.kts gradlew 2>/dev/null && echo "→ android"
find . -maxdepth 3 -name AndroidManifest.xml 2>/dev/null | head -1 | grep -q . && echo "→ android (manifest)"
# Expo
[ -f app.json ] && grep -q '"expo"' app.json 2>/dev/null && echo "→ expo"
[ -f eas.json ] && echo "→ expo (eas)"
# Desambiguación: si es Expo, el android/ es prebuild de RN, NO Android nativo.
# Trata como "expo", no como "android", aunque haya build.gradle dentro de android/.
# Next.js
ls next.config.* 2>/dev/null && echo "→ nextjs"
# TypeScript
[ -f tsconfig.json ] && echo "→ typescript"
# AWS
ls cdk.json samconfig.* 2>/dev/null && echo "→ aws"
[ -d amplify ] && echo "→ aws (amplify)"
# MCP
[ -f .mcp.json ] && echo "→ mcp"
```

### Paso 2 — Listar los docs base correspondientes

Por cada dominio detectado, mapea al doc base de la tabla de arriba. Verifica que
exista:

```bash
for doc in XCODE_WORKFLOW EXPO_WORKFLOW NEXTJS_WORKFLOW AWS_WORKFLOW MCP_WORKFLOW; do
  [ -f ~/.config/nvim/$doc.md ] && echo "✓ $doc.md" || echo "✗ $doc.md (falta)"
done
```

**Lee** los docs base aplicables para tener el contexto genérico cargado antes de
operar en el proyecto. Este es el corazón de la regla: contexto primero.

### Paso 3 — Verificar / generar el bloque en CLAUDE.md

Comprueba si el `CLAUDE.md` del proyecto ya tiene el bloque:

```bash
grep -q "## Platform Base Context" "$ROOT/CLAUDE.md" 2>/dev/null \
  && echo "✓ bloque presente" || echo "✗ falta bloque Platform Base Context"
```

Si falta, propón insertarlo cerca del inicio del `CLAUDE.md` (gated por el usuario):

```markdown
## Platform Base Context

Este proyecto extiende la documentación base de plataforma del CLI workflow.
Antes de operar, el contexto base aplicable es:

- **<Dominio>:** `~/.config/nvim/<DOMAIN>_WORKFLOW.md`
  <... un bullet por dominio detectado ...>

Premisa: `~/.config/nvim/PLATFORM_BASE.md` — la base es genérica; lo específico de
este proyecto vive aquí y en `docs/<DOMAIN>_EXTENSIONS.md`.
```

Si el proyecto **no tiene** `CLAUDE.md`, ofrece crearlo con este bloque + un esqueleto
mínimo (Project Overview, Build & Test, Module Structure).

### Paso 4 — Sugerir extension docs

Por cada dominio donde el proyecto tenga especificidad real (bundle IDs, deps,
arquitectura propia, agent skills), sugiere crear:

```
docs/<DOMAIN>_EXTENSIONS.md
```

Cada uno arranca con:

```markdown
# <DOMAIN> Extensions — <proyecto>

> Extiende `~/.config/nvim/<DOMAIN>_WORKFLOW.md` con lo específico de este proyecto.
```

No los crees automáticamente; lístalos como recomendación salvo que el usuario pida
generarlos.

### Paso 5 — Reportar estado

Imprime un checklist de inicialización:

```
Inicialización de <proyecto>:
  Dominios base:          apple, aws, mcp
  Docs base cargados:     XCODE_WORKFLOW.md, AWS_WORKFLOW.md, MCP_WORKFLOW.md
  CLAUDE.md bloque base:  [✓/✗ — propuesto]
  Extension docs:         docs/XCODE_EXTENSIONS.md (sugerido)
  → Contexto inequívoco:  [SÍ / pendiente de aprobar bloque]
```

---

## Reglas

1. **No editar la doc base para meter algo específico de un proyecto.** Lo específico
   va al proyecto (`CLAUDE.md` + `docs/<DOMAIN>_EXTENSIONS.md`).
2. **Conocimiento genérico reutilizable SÍ se aporta a la base** — propónlo aparte,
   gated por el usuario.
3. **Gating humano** para escribir en `CLAUDE.md` del proyecto: proponer el bloque,
   esperar aprobación, luego aplicar.
4. **Idempotente:** si el bloque ya existe y es correcto, no duplicar — solo verificar
   que los dominios listados siguen coincidiendo con el repo.
5. **El contexto se carga primero, se opera después.** Leer los docs base aplicables
   ES el paso que elimina la ambigüedad; no saltárselo.

---

## Ejemplo de uso

```
Usuario: /project-init
  → Detecta: apple (MyApp.xcodeproj + project.yml), aws (schema graphql ref), mcp (.claude.json)
  → Carga: XCODE_WORKFLOW.md, AWS_WORKFLOW.md, MCP_WORKFLOW.md
  → CLAUDE.md ya tiene Module Structure pero NO bloque "Platform Base Context"
  → Propone insertar el bloque referenciando los 3 docs base
  → Sugiere docs/XCODE_EXTENSIONS.md para mover la especificidad iOS
  → Reporta: contexto base cargado, bloque pendiente de aprobar
```

---

**Relacionado:**
- `~/.config/nvim/PLATFORM_BASE.md` — premisa universal
- `~/.config/nvim/README.md` — índice de docs base
- Project memory: regla de inicialización de proyectos
