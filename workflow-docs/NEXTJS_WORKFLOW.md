# 🚀 Next.js 15 + TypeScript - Workflow Hiper Productivo

## 📋 Índice
1. [Snippets Disponibles](#snippets-disponibles)
2. [Autocompletado Inteligente](#autocompletado-inteligente)
3. [Workflow Hiper Productivo](#workflow-hiper-productivo)
4. [LSP Features Específicos](#lsp-features-específicos)

---

## 📝 Snippets Disponibles

### **Server Components (Next.js 15 App Router)**

#### `nsc` - Next.js Server Component
```typescript
'use server'

interface ComponentNameProps {
  params: { id: string }
}

export default async function ComponentName({ params }: ComponentNameProps) {
  // Server-side logic

  return (
    <div>
      // JSX
    </div>
  )
}
```

**Uso:**
1. Insert mode → escribe `nsc`
2. Tab → Completa snippet
3. Escribe nombre del componente
4. Tab → Define props
5. Tab → Lógica server-side
6. Tab → JSX

---

### **Client Components**

#### `ncc` - Next.js Client Component
```typescript
'use client'

import { useState } from 'react'

interface ComponentNameProps {
  // props
}

export default function ComponentName({ }: ComponentNameProps) {
  const [state, setState] = useState()

  return (
    <div>
      // JSX
    </div>
  )
}
```

---

### **Server Actions**

#### `nsa` - Next.js Server Action
```typescript
'use server'

export async function actionName(formData: FormData) {
  // Server action logic

  return { success: true }
}
```

**Caso de uso:** Formularios, mutaciones, revalidación de cache

---

### **Route Handlers (API Routes)**

#### `nget` - GET Route Handler
```typescript
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  try {
    // Logic

    return NextResponse.json({ data: result })
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    )
  }
}
```

#### `npost` - POST Route Handler
Similar a GET pero con `request.json()` para leer body

---

### **Metadata (SEO)**

#### `nmeta` - Static Metadata
```typescript
import { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'Page Title',
  description: 'Page description',
}
```

#### `ndmeta` - Dynamic Metadata (con params)
```typescript
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const data = // fetch data

  return {
    title: `${data.title}`,
    description: 'Description',
  }
}
```

---

### **React Hooks**

| Snippet | Descripción |
|---------|-------------|
| `us` | `useState<T>()` |
| `ue` | `useEffect(() => {}, [])` |
| `ucb` | `useCallback(() => {}, [])` |
| `um` | `useMemo(() => {}, [])` |

---

### **TypeScript Utilities**

| Snippet | Descripción |
|---------|-------------|
| `int` | Interface definition |
| `typ` | Type definition |
| `afn` | Async function with types |
| `trycatch` | Try-catch block |

---

## 🧠 Autocompletado Inteligente

### **En Insert Mode:**

1. **Props destructuring:**
   ```tsx
   function Component({ | })  // Presiona Ctrl+Space
   // → Autocompleta con props del type
   ```

2. **Import automático:**
   ```tsx
   use  // Presiona Ctrl+x Ctrl+o
   // → Autocompleta "useState" + auto-import
   ```

3. **Signature help:**
   ```tsx
   fetch(  // Presiona Ctrl+k
   // → Muestra: fetch(input: RequestInfo, init?: RequestInit)
   ```

4. **Inlay hints (automático):**
   ```tsx
   onClick={handleClick}
          ^^^^^^^^
          (onClick: MouseEventHandler<HTMLButtonElement>)
   ```

---

## 🔥 Workflow Hiper Productivo

### **1. Crear Page (App Router)**

```vim
# En app/users/page.tsx
1. Insert mode
2. nsc<Tab>          # Snippet Server Component
3. UsersPage<Tab>    # Nombre
4. <Tab>             # Skip props si no hay
5. const users = await fetchUsers()<Tab>
6. <ul>{users.map(...)}</ul>
```

**Tiempo: ~10 segundos** ✨

---

### **2. Crear Client Component con Estado**

```vim
# En components/Counter.tsx
1. Insert mode
2. ncc<Tab>          # Snippet Client Component
3. Counter<Tab>
4. count: number<Tab>  # Props
5. us<Tab>           # useState snippet
6. count, Count, number, 0  # Completa state
7. Escribir JSX con autocompletado
```

**Tiempo: ~15 segundos** ✨

---

### **3. Agregar Server Action a Formulario**

```vim
# En app/actions.ts
1. nsa<Tab>          # Snippet Server Action
2. createUser<Tab>
3. formData<Tab>
4. Escribir lógica:
   const name = formData.get('name')
   await db.user.create({ data: { name } })
   revalidatePath('/users')

# En component:
<form action={createUser}>
  # ts_ls autocompleta action disponible
</form>
```

**Tiempo: ~20 segundos** ✨

---

### **4. API Route Completa**

```vim
# En app/api/users/route.ts
1. npost<Tab>        # Snippet POST handler
2. const { name } = body<Tab>
3. const user = await db.user.create({ data: { name } })<Tab>
4. user              # Return data
```

**Tiempo: ~15 segundos** ✨

---

### **5. SEO Dinámico**

```vim
# En app/blog/[slug]/page.tsx
1. ndmeta<Tab>       # Dynamic metadata
2. slug<Tab>
3. post, slug<Tab>
4. const post = await getPost(slug)<Tab>
5. ${post.title}<Tab>
6. post.excerpt
```

**Tiempo: ~12 segundos** ✨

---

## 🎯 LSP Features Específicos (TypeScript)

### **Auto-import Inteligente**

```tsx
// Escribe componente no importado
<Button />
  ^^^^
  # \ca → "Add import from '@/components/ui/button'"
```

### **Type Checking en Tiempo Real**

```tsx
const user: User = { name: 'John' }
                   ^^^^^^^^^^^^^^^^
// Error inline: Property 'email' is missing
// ]d para saltar al error
// \ca → "Add missing properties"
```

### **Refactoring Avanzado**

```tsx
// Selecciona JSX en Visual mode
<div className="card">
  <h2>{title}</h2>
  <p>{description}</p>
</div>

# Visual mode → \ca
# → "Extract to component"
# → Crea nuevo componente con props correctos
```

### **Organizar Imports Automáticamente**

```tsx
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { useState } from 'react'
import type { User } from '@/types'

# \ca → "Organize imports"
# ↓ Resultado:

import type { User } from '@/types'
import { useState } from 'react'
import { z } from 'zod'

import { Button } from '@/components/ui/button'
```

---

## 🔄 Workflow Completo: Feature Nueva

### **Crear Feature de A-Z (Ejemplo: Sistema de Posts)**

```vim
# 1. Server Component (app/posts/page.tsx)
nsc<Tab> → PostsPage
const posts = await db.post.findMany()
<PostList posts={posts} />

# 2. Client Component (components/PostList.tsx)
ncc<Tab> → PostList
posts: Post[]
us<Tab> → filter, setFilter, string, ''
Render lista con filtro

# 3. Server Action (app/posts/actions.ts)
nsa<Tab> → createPost
formData logic + revalidatePath

# 4. API Route (app/api/posts/route.ts)
nget<Tab> + npost<Tab>
CRUD completo

# 5. Metadata (app/posts/page.tsx - arriba del todo)
nmeta<Tab> → "Posts | My App", "Browse all posts"
```

**Tiempo total: ~3 minutos** 🚀

---

## ⚡ Atajos Clave para Next.js

| Atajo | Acción |
|-------|--------|
| `\ca` | Auto-import, organize imports, extract component |
| `gd` | Ir a definición (tipos, componentes, funciones) |
| `gt` | Ir a definición de tipo |
| `\rn` | Renombrar (actualiza imports/exports) |
| `]d` | Siguiente error TypeScript |
| `K` | Ver props/types del componente |
| `\f` | Formatear con Prettier (vía ts_ls) |

---

## 💡 Tips Productivos

1. **Usa `\ld`** para ver todos los componentes del archivo (navegación rápida)
2. **Usa `gr`** para ver dónde se usa un componente
3. **Usa `gi`** para ver implementaciones de interfaces
4. **Combina snippets:** `ncc<Tab>` + `us<Tab>` + `ue<Tab>` = Component completo
5. **Inlay hints:** Verás tipos inline automáticamente (parámetros, retornos)
6. **Auto-complete en JSX:** `<div cl` → `<Tab>` → `className`

---

## 🎨 Ejemplo Real: Formulario Completo

```vim
# components/CreatePostForm.tsx
1. ncc<Tab> → CreatePostForm<Tab><Tab>
2. us<Tab> → pending, setPending, boolean, false
3. Escribir JSX:

<form action={createPost} onSubmit={() => setPending(true)}>
  <input name="title" />  # Autocompleta atributos
  <textarea name="content" />
  <button disabled={pending}>
    {pending ? 'Creating...' : 'Create'}
  </button>
</form>

# app/posts/actions.ts (en otra ventana con split)
4. <C-x>| → Split vertical
5. :e app/posts/actions.ts
6. nsa<Tab> → createPost<Tab>formData<Tab>
7. Lógica:
   const title = formData.get('title') as string
   await db.post.create({ data: { title, ... } })
   revalidatePath('/posts')
```

**Resultado:** Formulario funcional con Server Action en ~2 minutos ⚡

---

## 🧪 Testing Workflow

```vim
# Crear test para componente
1. \ff → Buscar archivo
2. Counter.tsx<Enter>
3. :vs Counter.test.tsx  # Split vertical
4. Escribir test con autocomplete de tipos
5. gd en "Counter" → Salta a implementación
6. <C-o> → Volver al test
```

---

## 🏆 Métricas de Productividad

| Tarea | Sin Snippets | Con Workflow | Ganancia |
|-------|--------------|--------------|----------|
| Server Component | ~45s | ~10s | **4.5x** |
| Client Component + State | ~60s | ~15s | **4x** |
| Server Action | ~50s | ~20s | **2.5x** |
| API Route | ~70s | ~15s | **4.6x** |
| Metadata SEO | ~30s | ~12s | **2.5x** |

**Promedio: 3.6x más rápido** 🚀
