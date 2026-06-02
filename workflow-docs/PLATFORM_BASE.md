# Platform Base — Premisa Universal del CLI Workflow

> **Premisa universal**
>
> La documentación del CLI workflow (`~/.config/nvim/*_WORKFLOW.md` y docs asociadas)
> es **documentación base de plataforma**: genérica y no vinculada a ningún proyecto
> específico. Cada proyecto particular **debe extenderla** para cubrir sus propias
> necesidades.

---

## 📐 Naturaleza de esta documentación

[INSTRUCCIÓN DE INDEXACIÓN: Este bloque describe la naturaleza de la documentación del CLI workflow. Términos clave: base, plataforma, genérica, extensible, proyecto particular.]

La documentación del CLI workflow es **documentación base de plataforma**: genérica
y no vinculada a ningún proyecto específico. Cada proyecto particular debe extenderla
para cubrir sus propias necesidades.

**Qué SÍ es la doc base:**
- El toolchain genérico de la máquina (Xcode, Go, Node, Expo, AWS CLI, tmux, Neovim)
- Patrones reutilizables que aplican a cualquier proyecto del mismo dominio
- Comandos, atajos y workflows independientes de un repo concreto
- Convenciones de plataforma (cómo se construye/testea/firma en general)

**Qué NO es la doc base** (vive en el proyecto, no aquí):
- Bundle IDs, team IDs, nombres de schemes concretos
- Dependencias específicas de un `package.json`/`Package.swift`
- Reglas de arquitectura propias del proyecto
- Agent skills de un proyecto particular
- Decisiones (`docs/NN-*.md`), lecciones capturadas, pipelines internos

---

## 🚦 Regla de Inicialización de Proyectos

**Cada proyecto debe comenzar con contexto explícito. No se permite iniciar de
manera ambigua.**

El flujo es:

```
┌────────────────────────────────────────────────────────────────┐
│  1. El primer system prompt o prompt del usuario obtiene         │
│     contexto desde la DOCUMENTACIÓN BASE DE PLATAFORMA            │
│     (~/.config/nvim/*_WORKFLOW.md)                               │
│                          │                                       │
│                          ▼                                       │
│  2. Ese contexto es GENÉRICO y de PLATAFORMA                     │
│                          │                                       │
│                          ▼                                       │
│  3. El PROYECTO PARTICULAR EXTIENDE ese contexto a sus           │
│     necesidades específicas (CLAUDE.md del proyecto +            │
│     docs/<DOMAIN>_EXTENSIONS.md)                                 │
└────────────────────────────────────────────────────────────────┘
```

> **El objetivo no es la documentación base.** El objetivo es que cada proyecto,
> al iniciar, **evite la ambigüedad obteniendo contexto primero**. La doc base es
> el medio; el contexto inequívoco al arrancar es el fin.

---

## 🗂️ Documentación Base Disponible (por dominio)

| Dominio | Doc base | Cubre (genérico) |
|---------|----------|------------------|
| **Apple / iOS** | [XCODE_WORKFLOW.md](./XCODE_WORKFLOW.md) | Xcode CLI, simctl, signing, XcodeBuildMCP, simuladores |
| **Android nativo** | [ANDROID_WORKFLOW.md](./ANDROID_WORKFLOW.md) | Kotlin, Compose, Gradle, KMP, adb/emulator, mobile-mcp |
| **Mobile / Expo** | [EXPO_WORKFLOW.md](./EXPO_WORKFLOW.md) | Expo CLI, EAS, RN dev loop, mobile-mcp |
| **Web / Next.js** | [NEXTJS_WORKFLOW.md](./NEXTJS_WORKFLOW.md) | Next.js + TypeScript, snippets, App Router |
| **TypeScript** | [TYPESCRIPT_CONTEXT.md](./TYPESCRIPT_CONTEXT.md) | LSP, autocompletado contextual |
| **AWS** | [AWS_WORKFLOW.md](./AWS_WORKFLOW.md) | Amplify, CDK, SDK, snippets genéricos |
| **MCP** | [MCP_WORKFLOW.md](./MCP_WORKFLOW.md) | Inventario de MCP servers + configuración |
| **Chrome** | [CHROME_WORKFLOW.md](./CHROME_WORKFLOW.md) | Remote debugging, MCP, DAP |
| **Shell** | [SHELL_WORKFLOW.md](./SHELL_WORKFLOW.md) | Zsh, Oh My Zsh, version managers |
| **Scripts/tmux** | [SCRIPTS_WORKFLOW.md](./SCRIPTS_WORKFLOW.md) | `~/bin/`, aliases, sesiones tmux |
| **Lab seguridad** | [KALI_WORKFLOW.md](./KALI_WORKFLOW.md) | VMs Kali, pentesting toolset |
| **Neovim** | [README.md](./README.md), [KEYMAPS.md](./KEYMAPS.md) | Editor, plugins, atajos |

> **Nota de transición:** Algunos docs base aún contienen secciones proyecto-específicas
> (resultado de iteraciones previas). Estas deben migrarse progresivamente a
> `docs/<DOMAIN>_EXTENSIONS.md` dentro de cada proyecto. La premisa rige hacia adelante:
> **todo contenido nuevo en la doc base es genérico**; lo específico vive en el proyecto.

---

## 🧩 Cómo un Proyecto Extiende la Base

Un proyecto que extiende la plataforma declara su contexto en **dos lugares**:

### 1. `CLAUDE.md` del proyecto (carga al iniciar)

Debe incluir, cerca del inicio, un bloque que **referencie la base de la que parte**:

```markdown
## Platform Base Context

Este proyecto extiende la documentación base de plataforma del CLI workflow.
Antes de operar, el contexto base aplicable es:

- **Apple/iOS:** `~/.config/nvim/XCODE_WORKFLOW.md`
- **AWS:**       `~/.config/nvim/AWS_WORKFLOW.md`
- **MCP:**       `~/.config/nvim/MCP_WORKFLOW.md`

Las extensiones específicas de este proyecto están en `docs/<DOMAIN>_EXTENSIONS.md`.
```

### 2. `docs/<DOMAIN>_EXTENSIONS.md` (lo específico)

Cada extension doc arranca declarando de qué base parte:

```markdown
# <DOMAIN> Extensions — <nombre-proyecto>

> Extiende `~/.config/nvim/<DOMAIN>_WORKFLOW.md` con lo específico de este proyecto.

## Lo que este proyecto añade sobre la base

- Bundle ID / Team ID / schemes concretos
- Dependencias específicas
- Convenciones de arquitectura propias
- Agent skills del proyecto
- Comandos exactos (paths, nombres reales)
```

---

## ⚙️ Mecanismo de Carga: `/project-init`

El skill global **`/project-init`** automatiza la regla de inicialización:

```
/project-init                  # audita el proyecto actual
/project-init apple,aws,mcp    # declara dominios base aplicables
```

Qué hace:
1. Detecta los dominios de plataforma aplicables (por archivos del repo)
2. Lista los docs base correspondientes
3. Genera o verifica el bloque "Platform Base Context" en el `CLAUDE.md` del proyecto
4. Sugiere los `docs/<DOMAIN>_EXTENSIONS.md` a crear

Ver: `~/.claude/skills/project-init/SKILL.md`

---

## 📋 Checklist: ¿Este proyecto está bien inicializado?

- [ ] Tiene `CLAUDE.md` con bloque "Platform Base Context"
- [ ] El bloque referencia los docs base de los dominios que usa
- [ ] Lo específico (bundle IDs, deps, arquitectura, skills) vive en el proyecto, no en la doc base
- [ ] Si añade conocimiento de dominio reutilizable, lo aporta a la doc base (genérico) — no lo entierra en el proyecto

---

**Creado:** 2026-05-03
**Tipo:** Premisa universal de plataforma
**Rige:** Toda la documentación del CLI workflow y la inicialización de cualquier proyecto
**Mecanismo de carga:** `CLAUDE.md` del proyecto + skill `/project-init`
