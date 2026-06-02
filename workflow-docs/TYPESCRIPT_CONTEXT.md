# TypeScript - Autocompletado con Contexto Completo

## ✅ Configuración Aplicada

El autocompletado ahora incluye **contexto completo**:

### **1. Proyecto (tus archivos)**
- ✅ Componentes en `app/`, `src/`, `components/`
- ✅ Utilidades en `lib/`, `utils/`
- ✅ Types e interfaces propias
- ✅ Hooks personalizados

### **2. node_modules (librerías de terceros)**
- ✅ `react` → `useState`, `useEffect`, `FC`, etc.
- ✅ `next/navigation` → `useRouter`, `redirect`, `notFound`
- ✅ `next/server` → `NextRequest`, `NextResponse`
- ✅ AWS Amplify v6 → `generateClient`, `signIn`, `uploadData`
- ✅ Cualquier librería en tu `package.json`

---

## 🔧 Requisitos en tu Proyecto Next.js

Para que funcione al 100%, tu proyecto Next.js debe tener `tsconfig.json` configurado correctamente:

### **tsconfig.json recomendado para Next.js v15**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./components/*"],
      "@/lib/*": ["./lib/*"],
      "@/utils/*": ["./utils/*"]
    }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    "app/**/*",
    "src/**/*",
    "components/**/*",
    "lib/**/*"
  ],
  "exclude": ["node_modules"]
}
```

### **Si usas AWS Amplify v6, agregar:**

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/amplify/*": ["./amplify/*"]
    }
  },
  "include": [
    "amplify/**/*"
  ]
}
```

---

## 🚀 Cómo Funciona el Autocompletado

### **Ejemplo 1: Imports de node_modules**

```typescript
// Escribe:
import { use

// Autocompleta:
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { useSearchParams } from 'next/navigation'
```

### **Ejemplo 2: Imports de tu proyecto**

```typescript
// Escribe (asumiendo que tienes @/components/Header.tsx):
import { H

// Autocompleta:
import { Header } from '@/components/Header'
```

### **Ejemplo 3: Props de componentes (node_modules)**

```typescript
// Cursor en:
<Link href="" |

// Autocompleta:
// - prefetch
// - replace
// - scroll
// - shallow
// (Props de next/link)
```

### **Ejemplo 4: Funciones con parámetros completos**

```typescript
// Escribe:
useState

// Autocompleta con snippet:
const [state, setState] = useState<string>('')
//                                  ^cursor aquí para editar tipo
```

### **Ejemplo 5: AWS Amplify v6**

```typescript
// Escribe:
import { gene

// Autocompleta:
import { generateClient } from 'aws-amplify/data'
```

---

## 🎯 Features Activadas

### **Auto-imports al guardar (BufWritePre)**

Cuando guardas un archivo `.ts/.tsx`:
1. ✅ Agrega imports faltantes automáticamente
2. ✅ Organiza imports (agrupa react, next, third-party, proyecto)
3. ✅ Elimina imports no usados

**Si no quieres esto**, comenta las líneas 42-48 en `~/.config/nvim/lua/config/typescript.lua`:

```lua
-- Auto-imports al guardar (opcional, comenta si no quieres)
-- vim.api.nvim_create_autocmd("BufWritePre", {
--   buffer = bufnr,
--   callback = function()
--     vim.cmd("TSToolsAddMissingImports sync")
--     vim.cmd("TSToolsOrganizeImports sync")
--   end,
-- })
```

### **Trigger Characters**

El autocompletado se activa automáticamente cuando escribes:
- `.` → Métodos/propiedades
- `"` o `'` → Paths de imports
- `/` → Paths de archivos
- `@` → Alias de paths (@/components)
- `<` → Componentes JSX
- `#` → Private fields

---

## 🔍 Comandos TypeScript Disponibles

| Comando | Keymap | Descripción |
|---------|--------|-------------|
| `:TSToolsOrganizeImports` | `\to` | Organiza imports |
| `:TSToolsAddMissingImports` | `\ti` | Agrega imports faltantes |
| `:TSToolsRemoveUnused` | `\tu` | Elimina imports no usados |
| `:TSToolsSortImports` | `\ts` | Ordena imports alfabéticamente |
| `:TSToolsFixAll` | `\tf` | Fix todos los errores auto-arreglables |

---

## 🧪 Verificar que Funciona

### **Test 1: Verificar LSP activo**

```vim
:LspInfo

# Debería mostrar:
# Client: typescript-tools (id: 1, bufnr: [1])
```

### **Test 2: Autocompletado de node_modules**

Abre cualquier `.tsx`:
```typescript
import { use|  // Presiona Ctrl+x Ctrl+o
```

Debería sugerir: `useState`, `useEffect`, `useCallback`, etc.

### **Test 3: Autocompletado de proyecto**

```typescript
import { | } from '@/  // Presiona Ctrl+x Ctrl+o
```

Debería sugerir tus carpetas: `components`, `lib`, `utils`, etc.

### **Test 4: Props con contexto**

```typescript
const MyComponent = () => {
  const router = useRouter()

  router.|  // Presiona Ctrl+x Ctrl+o
}
```

Debería sugerir: `push`, `replace`, `refresh`, `back`, `forward`, `prefetch`

---

## ⚡ Optimización de Performance

TypeScript puede ser lento en proyectos grandes. Si notas lag:

### **Opción 1: Excluir node_modules pesados**

En `tsconfig.json`:
```json
{
  "exclude": [
    "node_modules",
    ".next",
    "out",
    "dist"
  ]
}
```

### **Opción 2: Incrementar memoria de TypeScript**

Crea `.vscode/settings.json` (también lo lee Neovim):
```json
{
  "typescript.tsserver.maxTsServerMemory": 8192
}
```

### **Opción 3: Usar workspace particionado**

Para monorepos grandes, usa `tsconfig.base.json` + `tsconfig.json` por paquete.

---

## 🐛 Troubleshooting

### **Problema: No sugiere nada de node_modules**

**Solución:**
1. Verifica que exista `node_modules/`:
   ```bash
   ls node_modules/react
   ```
2. Reinicia LSP:
   ```vim
   :LspRestart
   ```
3. Verifica `package.json` tiene las dependencias:
   ```bash
   cat package.json | grep dependencies
   ```

### **Problema: No sugiere archivos de mi proyecto**

**Solución:**
1. Verifica `tsconfig.json` incluye tus carpetas:
   ```json
   "include": ["app/**/*", "components/**/*"]
   ```
2. Reinicia LSP:
   ```vim
   :LspRestart
   ```

### **Problema: Lag al escribir**

**Solución:**
1. Deshabilita diagnósticos en insert mode (ya configurado):
   ```lua
   publish_diagnostic_on = "insert_leave"
   ```
2. Reduce inlay hints:
   ```lua
   includeInlayParameterNameHints = "literals"  -- Solo literales
   ```

---

## 📚 Recursos

- **TypeScript Compiler Options**: https://www.typescriptlang.org/tsconfig
- **Next.js TypeScript**: https://nextjs.org/docs/app/building-your-application/configuring/typescript
- **typescript-tools.nvim**: https://github.com/pmizio/typescript-tools.nvim

---

**Configurado:** 2025-10-08
**Contexto:** Proyecto + node_modules
**Optimizado para:** Next.js v15 + AWS Amplify v6
