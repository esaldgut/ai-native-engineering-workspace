# Neovim Keymaps Reference

> 📘 **Workflows Específicos:**
> - **Next.js 15 + TypeScript:** [NEXTJS_WORKFLOW.md](./NEXTJS_WORKFLOW.md)
> - **AWS Full Stack (Amplify, CDK, SDK):** [AWS_WORKFLOW.md](./AWS_WORKFLOW.md)

## Leader Key
- **Leader**: `\` (backslash)

---

## 🎯 LSP (Language Server Protocol)

### Navegación
- `gd` - Ir a definición
- `gD` - Ir a declaración
- `gi` - Ir a implementación
- `gt` - Ir a definición de tipo
- `gr` - Buscar referencias
- `K` - Mostrar documentación hover (doble `K` para entrar al floating window)

### Code Actions
- `\ca` - Code actions (fixes, refactors, imports)
- `\rn` - Renombrar símbolo (todo el proyecto)
- `\f` - Formatear buffer

### Diagnósticos
- `[d` - Diagnóstico anterior
- `]d` - Diagnóstico siguiente
- `\e` - Mostrar diagnóstico flotante

### Signature Help
- `<C-k>` (Insert mode) - Mostrar firma de función

### TypeScript Específico (archivos .ts/.tsx)
- `\to` - Organizar imports (auto-ordena y agrupa)
- `\ts` - Ordenar imports alfabéticamente
- `\tu` - Eliminar imports no usados
- `\ti` - Agregar imports faltantes
- `\tf` - Fix all (corrige todos los errores auto-arreglables)

---

## 📦 Autocompletado (nvim-cmp)

### Durante el completado
- `<C-x><C-o>` - Abrir menú de completado
- `<C-x><C-n>` - Siguiente item
- `<C-x><C-p>` - Item anterior
- `<Tab>` - Siguiente item / Expandir snippet
- `<S-Tab>` - Item anterior / Saltar snippet atrás
- `<CR>` - Confirmar selección
- `<C-e>` - Cancelar completado
- `<C-d>` - Scroll docs arriba
- `<C-f>` - Scroll docs abajo

### Snippets Next.js 15 (en archivos .tsx/.ts)
| Snippet | Descripción |
|---------|-------------|
| `nsc` | Server Component (App Router) |
| `ncc` | Client Component |
| `nsa` | Server Action |
| `nget` | GET Route Handler |
| `npost` | POST Route Handler |
| `nmeta` | Static Metadata |
| `ndmeta` | Dynamic Metadata |
| `us` | useState hook |
| `ue` | useEffect hook |
| `ucb` | useCallback hook |
| `um` | useMemo hook |
| `int` | TypeScript Interface |
| `typ` | TypeScript Type |
| `afn` | Async Function |

---

## 🔍 Telescope (Búsqueda Fuzzy)

### Archivos
- `<leader>ff` - Buscar archivos
- `<leader>fr` - Archivos recientes
- `<leader>fb` - Buffers abiertos

### Búsqueda
- `<leader>fg` - Live grep (buscar texto)
- `<leader>fh` - Help tags
- `<leader>fc` - Comandos disponibles
- `<leader>fk` - Keymaps

### Git
- `<leader>gs` - Git status
- `<leader>gc` - Git commits
- `<leader>gb` - Git branches

### LSP
- `<leader>ld` - Símbolos del documento
- `<leader>lw` - Símbolos del workspace

### Dentro de Telescope
- `<C-x>` - Cerrar
- `<C-n>` / `<C-p>` - Navegar resultados

---

## 📁 Oil.nvim (Explorador de Archivos)

### Abrir
- `-` - Abrir directorio padre
- `\-` - Abrir oil en ventana flotante
- `\n` - Abrir Oil estilo NERDTree (split vertical izquierdo)

### Navegación (dentro de Oil)
- `<CR>` - Seleccionar archivo/directorio
- `-` - Ir al directorio padre
- `g.` - Toggle archivos ocultos

### Splits
- `<C-x>s` - Abrir en split vertical
- `<C-x>h` - Abrir en split horizontal
- `<C-t>` - Abrir en nueva tab

### Acciones
- `<C-p>` - Preview
- `<C-c>` - Cerrar
- `<C-r>` - Refrescar
- `gx` - Abrir con aplicación externa
- `g?` - Mostrar ayuda

---

## 🪟 Ventanas y Buffers

### Navegación entre ventanas (Ctrl+x)
- `<C-x>h` - Ir a ventana izquierda
- `<C-x>j` - Ir a ventana abajo
- `<C-x>k` - Ir a ventana arriba
- `<C-x>l` - Ir a ventana derecha

### Splits
- `<C-x>|` - Split vertical
- `<C-x>-` - Split horizontal
- `<C-x>q` - Cerrar ventana

### Redimensionar ventanas
- `<C-x>H` - Disminuir ancho
- `<C-x>L` - Aumentar ancho
- `<C-x>J` - Disminuir altura
- `<C-x>K` - Aumentar altura

### Buffers
- `<leader>bn` - Siguiente buffer
- `<leader>bp` - Buffer anterior
- `<leader>bd` - Eliminar buffer

---

## 🔄 Git (GitSigns)

- `]c` - Siguiente hunk
- `[c` - Hunk anterior

### Git Fugitive
- `:Git` - Comandos git
- `:Git blame` - Ver blame

---

## ⌨️ Edición

### Mover líneas
- `J` (visual) - Mover líneas seleccionadas abajo
- `K` (visual) - Mover líneas seleccionadas arriba

### Navegación
- `j` / `k` - Respetar line wrapping
- `n` / `N` - Buscar y centrar cursor

### Clipboard del sistema
- `<leader>y` (visual) - Copiar a clipboard
- `<leader>Y` - Copiar línea a clipboard
- `<leader>p` - Pegar desde clipboard
- `<leader>P` - Pegar antes desde clipboard

### Utilidades
- `<leader>s` - Reemplazar palabra bajo cursor
- `<leader>+` - Incrementar número
- `<leader>-` - Decrementar número
- `<C-a>` - Seleccionar todo

### Text Wrapping
- `<leader>w` - Toggle wrap on/off
- `<leader>wf` - Format/hard wrap párrafo actual (usa textwidth)
- `<leader>wt` - Establecer textwidth para hard wrapping
- `<leader>ws` - Mostrar estado de wrapping actual

---

## 💾 Guardar y Salir

- `<C-s>` - Guardar archivo (normal/insert)
- `<C-x><C-c>` - Salir de todo

---

## 🖥️ Terminal

- `\t` - Terminal horizontal (split abajo)
- `\tv` - Terminal vertical (split derecha)
- `<Esc>` (terminal mode) - Salir a Normal mode
- `i` o `a` - Volver a Terminal mode

---

## 🔧 Quickfix

- `<C-x>n` - Siguiente item
- `<C-x>p` - Item anterior
- `<leader>co` - Abrir quickfix
- `<leader>cc` - Cerrar quickfix

---

## 🧩 Treesitter (Selección Incremental)

- `gnn` - Iniciar selección
- `grn` - Incrementar al nodo padre
- `grc` - Incrementar al scope superior
- `grm` - Decrementar selección

### Text Objects
- `af` - Función completa (outer)
- `if` - Cuerpo de función (inner)
- `ac` - Clase completa (outer)
- `ic` - Cuerpo de clase (inner)

---

## 🎨 Otros

- `<Esc>` - Limpiar highlights de búsqueda

---

## 🚀 Workflows Avanzados LSP

### **Fix Rápido de Errores**
```
1. ]d           # Saltar al siguiente error
2. \e           # Ver detalles del error
3. \ca          # Ver opciones de fix
4. Enter        # Aplicar fix
5. ]d           # Siguiente error
```

### **Refactoring Completo**
```
1. Visual mode  # Seleccionar código
2. \ca          # Extract function/variable
3. Enter        # Aplicar
4. \rn          # Renombrar (actualiza todo el proyecto)
```

### **Navegación de Código**
```
1. gd           # Ir a definición
2. K            # Ver documentación
3. gr           # Ver todas las referencias (Telescope)
4. Ctrl+o       # Volver (jumplist)
```

### **Imports y Organización (Go)**
```
1. \ca          # En archivo Go
2. Buscar "Organize imports"
3. Enter        # Auto-ordena y limpia imports
```

### **Fix Masivo con Quickfix**
```
1. \co          # Abrir quickfix con todos los errores
2. j/k          # Navegar errores
3. Enter        # Ir al error
4. \ca          # Fix
5. :cnext       # Siguiente error
```

### **Exploración de Tipos (Go)**
```
1. gt           # Ir a definición de tipo
2. gi           # Ver implementaciones de interface
3. gr           # Ver dónde se usa el tipo
```

### **Signature Help en Tiempo Real**
```
# En Insert mode, dentro de parámetros de función:
Ctrl+k          # Mostrar firma con tipos y documentación
```

---

## 🔍 Búsqueda Avanzada

### **Telescope con Args**
- `\fG` - Live grep con argumentos avanzados
  - Dentro: `<C-k>` agregar comillas, `<C-i>` filtrar por extensión

### **Spectre (Find & Replace)**
- `\sr` - Replace en todo el proyecto
- `\sw` - Replace palabra bajo cursor
- `\sf` - Replace en archivo actual

---

## 🧪 Testing (Neotest)

- `\tr` - Run nearest test
- `\tf` - Run file tests
- `\ts` - Toggle test summary
- `\to` - Show test output
- `\tS` - Stop test

---

## 🐛 Debugging (DAP)

- `\db` - Toggle breakpoint
- `\dc` - Continue/Start debug
- `\di` - Step into
- `\do` - Step over
- `\dO` - Step out
- `\dr` - Open REPL
- `\du` - Toggle DAP UI

---

## 📊 Code Context

- `\a` - Toggle Aerial (outline)
- Breadcrumbs automáticos en top bar

---

## 🔄 Git Avanzado

- `\gg` - LazyGit (UI completa)
- `\gf` - LazyGit archivo actual

---

## 🎨 Dashboard

Al abrir `nvim` sin archivo:
- `f` - Find file
- `p` - Recent projects
- `r` - Recent files
- `g` - LazyGit
- `c` - Config

---

## 📝 Snippets AWS

### **AWS Amplify v6**
- `ampauth` - Auth Sign In
- `ampdata` - GraphQL Query
- `ampstorage` - Upload File

### **AWS CDK Go v2**
- `cdkstack` - Stack completo
- `cdklambda` - Lambda Function
- `cdkapi` - API Gateway
- `cdkdynamo` - DynamoDB Table

### **AWS SDK Go v2**
- `sdks3` - S3 Client
- `sdkdynamo` - DynamoDB PutItem
- `sdklambda` - Lambda Invoke

---

## 🌐 Chrome Integration

### Terminal Aliases (zsh)

- `chrome-debug` - Lanzar Chrome con remote debugging (puerto 9222)
- `chrome-open <url>` - Abrir URL en Chrome
- `chrome-tabs` - Listar tabs abiertos
- `chrome-exec "<js>"` - Ejecutar JavaScript en tab activo
- `chrome-close` - Cerrar tab activo
- `chrome-reload` - Recargar página actual

### Claude Desktop + MCP

**MCP Server:** `chrome-devtools` (puerto 9222)

**Comandos en Claude:**
- "Inspecciona la página en http://localhost:3000"
- "Ejecuta console.log('test') en la página"
- "Toma un screenshot de la aplicación"
- "Analiza el performance de esta página"
- "Muestra los console logs"

**Configuración:** `~/Library/Application Support/Claude/claude_desktop_config.json`

### Debugging con DAP

**Los keymaps de DAP ya configurados funcionan con Chrome:**

- `\db` - Toggle breakpoint (funciona en TypeScript/JavaScript)
- `\dc` - Start/Continue debugging (adjunta a Chrome en puerto 9222)
- `\du` - Toggle DAP UI (muestra variables, console, call stack)
- `F10` - Step Over
- `F11` - Step Into

**Workflow:** Ver `CHROME_WORKFLOW.md` para workflows completos

---

**📚 Documentación Relacionada:**
- Chrome Workflow: `CHROME_WORKFLOW.md`
- Neovim Setup: `README.md`
- Next.js Workflow: `NEXTJS_WORKFLOW.md`
- AWS Workflow: `AWS_WORKFLOW.md`
