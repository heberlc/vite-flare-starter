# Plan de Desarrollo — Catálogo de Florería

> **Tipo:** Single-client (una florería)
> **Arquitectura:** 2 proyectos separados (Astro público + Vite Flare Starter admin)
> **Plan Cloudflare:** Free
> **Estimación total:** ~2.5 semanas

---

## Arquitectura: Dos Proyectos, Una Base de Datos

### ¿Por qué dos proyectos?

| Necesidad | Solución |
|-----------|----------|
| Catálogo con SEO (cada producto indexable en Google y con preview en WhatsApp) | **Astro SSG** → genera HTML estático por producto |
| Admin profesional con tablas, formularios, modales | **Vite Flare Starter** → SPA React con shadcn/ui completo |

**¿Por qué no un solo Astro con React islands para el admin?**
- Muchas islands cargan más JS que un SPA (cada una carga su propio bundle)
- No comparten estado entre sí (cada island es independiente)
- Pierdes el layout/sidebar/componentes de shadcn/ui que el template ya trae

**¿Por qué no un solo React SPA para todo (como la peluquería)?**
- React SPA envía HTML vacío al browser → Google no indexa los productos
- WhatsApp no puede mostrar preview de un link SPA (necesita meta tags `og:image` en HTML server-rendered)
- La florería NECESITA SEO por producto, la peluquería no

### Esquema

```
floreria.com (Astro → Cloudflare Pages)
├── /                          → Home + productos destacados
├── /catalogo                  → Todos los productos (SEO)
├── /catalogo/ramos            → Por categoría (SEO)
├── /producto/ramo-rosas-rojas → Detalle con og:image (SEO + WhatsApp preview)
└── 🛒 Carrito Nanostores → WhatsApp

admin.floreria.com (Vite Flare Starter → Cloudflare Worker)
├── /dashboard                 → Métricas de ventas
├── /dashboard/products        → CRUD de productos
├── /dashboard/sales           → Registro de ventas
└── /dashboard/settings        → Configuración

         ┌─────────────┐
         │ Cloudflare   │
         │ D1 (shared)  │ ← ambos proyectos conectan a la misma DB
         │ R2 (shared)  │ ← ambos proyectos usan el mismo bucket de fotos
         └─────────────┘
```

---

## Límites del Plan FREE — Viabilidad

### Dos Workers/Pages en plan free

| Recurso | Límite Free | Uso combinado | ¿Alcanza? |
|---------|-------------|--------------|-----------|
| **Workers** | 100,000 requests/día (total cuenta) | Catálogo: ~3,000-10,000 + Admin: ~500 | ✅ Sobra |
| **Pages deploys** | 500/mes | ~30-50/mes | ✅ Sobra |
| **D1 storage** | 500 MB (total cuenta) | ~10 MB (productos + ventas) | ✅ Sobra |
| **D1 reads** | 5M/día | ~5,000-20,000/día | ✅ Sobra |
| **D1 writes** | 100K/día | ~50-100/día | ✅ Sobra |
| **R2 storage** | 10 GB | Fotos de productos: ~500 MB | ✅ Sobra |
| **Workers CPU** | 10ms/request | Queries simples: ~2-4ms | ✅ Sobra |

**✅ Todo cabe en el plan free sin problema.**

---

## Módulos Incluidos vs Descartados

### ✅ Incluidos

| Módulo | Proyecto | Descripción |
|--------|----------|-------------|
| **Catálogo público (SSG)** | Astro | Páginas estáticas por producto con SEO |
| **Carrito (Nanostores)** | Astro | Estado en browser → mensaje WhatsApp |
| **CRUD Productos** | Admin (VFS) | Crear, editar, toggle stock, subir foto |
| **Registro de Ventas** | Admin (VFS) | Registro manual de ventas |
| **Auth admin** | Admin (VFS) | Login con better-auth (ya incluido) |
| **Activity Log** | Admin (VFS) | Historial de cambios en productos |
| **Files/R2** | Admin (VFS) | Upload de fotos de productos |
| **Theme System** | Admin (VFS) | Branding del admin |

### ❌ Descartados del Vite Flare Starter

| Módulo | Razón |
|--------|-------|
| **Workers AI / Chat** | No necesario |
| **API Tokens** | No hay integraciones externas |
| **Notifications** | No necesario (pedidos van por WhatsApp) |
| **Feature Flags** | Overkill para este proyecto |
| **Component Showcase / Style Guide** | Solo desarrollo |
| **Google OAuth** | Admin usa solo email/password |
| **Landing Page del template** | Se reemplaza por Astro |

---

## Proyecto 1: Catálogo Astro (Público)

### Setup

```bash
npm create astro@latest floreria-catalogo -- --template minimal
cd floreria-catalogo
npx astro add cloudflare
npx astro add react          # para islands del carrito
pnpm add nanostores @nanostores/react
pnpm add drizzle-orm
pnpm add -D drizzle-kit wrangler
```

### Páginas (SSG con SEO)

```
src/pages/
├── index.astro              → Home: hero + destacados + categorías
├── catalogo/
│   ├── index.astro          → Grid de todos los productos
│   └── [category].astro     → Filtrado por categoría
└── producto/
    └── [slug].astro         → Detalle del producto
```

### SEO por producto (clave para WhatsApp preview)

```astro
<!-- producto/[slug].astro -->
---
const product = await getProductBySlug(slug)
---
<head>
  <title>{product.name} | Florería</title>
  <meta name="description" content={product.description} />
  <meta property="og:title" content={product.name} />
  <meta property="og:image" content={product.image} />
  <meta property="og:description" content={product.description} />
  <meta property="og:type" content="product" />
  <meta property="product:price:amount" content={product.price / 100} />
</head>
```

**Resultado:** Al compartir `floreria.com/producto/ramo-rosas-rojas` en WhatsApp → se ve preview con foto, nombre y descripción.

### Carrito (Nanostores — solo 2 islands en el público)

```typescript
// src/stores/cart.ts
import { map, computed } from 'nanostores'

export type CartItem = {
  id: string
  name: string
  price: number
  quantity: number
  image: string
}

export const $cart = map<Record<string, CartItem>>({})

export const $cartTotal = computed($cart, (cart) =>
  Object.values(cart).reduce((sum, item) => sum + item.price * item.quantity, 0)
)

export const $cartCount = computed($cart, (cart) =>
  Object.values(cart).reduce((sum, item) => sum + item.quantity, 0)
)

export function addToCart(product: CartItem) {
  const current = $cart.get()[product.id]
  $cart.setKey(product.id, {
    ...product,
    quantity: (current?.quantity || 0) + 1,
  })
}

export function removeFromCart(id: string) {
  const cart = { ...$cart.get() }
  delete cart[id]
  $cart.set(cart)
}
```

### WhatsApp checkout

```typescript
// src/lib/whatsapp.ts
const WHATSAPP_NUMBER = '51999999999'

export function openWhatsAppCheckout(items: CartItem[]) {
  let msg = '🌸 *Nuevo Pedido*\n\n'
  items.forEach((item, i) => {
    msg += `${i + 1}. ${item.name} x${item.quantity} - S/${(item.price * item.quantity / 100).toFixed(2)}\n`
  })
  const total = items.reduce((s, i) => s + i.price * i.quantity, 0)
  msg += `\n💰 *Total: S/${(total / 100).toFixed(2)}*`
  msg += `\n\n📍 ¿Entrega o recojo en tienda?`
  window.open(`https://wa.me/${WHATSAPP_NUMBER}?text=${encodeURIComponent(msg)}`, '_blank')
}
```

### React Islands (solo 2, mínimo JS)

| Componente | Dónde aparece | Función |
|-----------|--------------|---------|
| `AddToCartButton.tsx` | Cada card de producto | Agregar al carrito |
| `CartDrawer.tsx` | Header (global) | Ver carrito + checkout WhatsApp |

---

## Proyecto 2: Admin con Vite Flare Starter

### Schema compartido (misma D1)

```typescript
// Productos (usado por ambos proyectos)
export const products = sqliteTable('products', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  description: text('description'),
  category: text('category').notNull(),
  price: integer('price').notNull(),       // centavos
  image: text('image'),                    // URL en R2
  images: text('images'),                  // JSON array de URLs
  inStock: integer('in_stock').default(1), // 1=disponible, 0=agotado
  featured: integer('featured').default(0),
  sortOrder: integer('sort_order').default(0),
  createdAt: integer('created_at'),
  updatedAt: integer('updated_at'),
})

// Ventas (solo admin)
export const sales = sqliteTable('sales', {
  id: text('id').primaryKey(),
  productId: text('product_id').references(() => products.id),
  productName: text('product_name').notNull(),
  productPrice: integer('product_price').notNull(),
  quantity: integer('quantity').notNull().default(1),
  total: integer('total').notNull(),
  clientName: text('client_name'),
  clientPhone: text('client_phone'),
  notes: text('notes'),
  source: text('source').default('whatsapp'), // "whatsapp" | "tienda"
  createdAt: integer('created_at'),
})
```

### Índices D1

```sql
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_stock ON products(in_stock);
CREATE INDEX idx_sales_date ON sales(created_at);
CREATE INDEX idx_sales_product ON sales(product_id);
```

### Módulos a crear en el admin

#### `products` module

**Rutas API:**
- `GET /api/products` → listar (con filtros: categoría, stock)
- `POST /api/products` → crear
- `PUT /api/products/:id` → editar
- `PATCH /api/products/:id/stock` → toggle stock
- `DELETE /api/products/:id` → eliminar

**Frontend:**
- Tabla de productos con columnas: foto, nombre, categoría, precio, stock (switch), acciones
- Formulario crear/editar con upload de imagen a R2
- Filtros por categoría y estado de stock

#### `sales` module

**Rutas API:**
- `GET /api/sales` → listar ventas (paginadas)
- `POST /api/sales` → registrar venta
- `GET /api/sales/stats` → métricas (hoy, semana, mes)

**Frontend:**
- Formulario de registro: seleccionar producto(s), cantidad, cliente, origen
- Tabla de ventas con fecha, producto, monto
- Dashboard con métricas: ventas hoy, esta semana, ingresos del mes

### Limpieza del template

- Eliminar módulo Chat (AI) de sidebar y rutas
- Ocultar API Tokens: `VITE_FEATURE_API_TOKENS=false`
- Eliminar landing page (el público va por Astro)
- Renombrar app: `APP_NAME=Florería Admin`
- Redirect `/` → `/dashboard`

---

## Flujo Completo

```
CLIENTE:
1. Entra a floreria.com → ve productos con fotos y precios
2. Agrega productos al carrito (Nanostores)
3. Click "Enviar pedido por WhatsApp"
4. Se abre WhatsApp con mensaje detallado
5. Coordina con la florería (pago, entrega)

ADMIN:
1. Entra a admin.floreria.com → login
2. Agrega/edita productos con fotos
3. Marca productos como "sin stock" cuando se agotan
4. Después de confirmar pedido por WhatsApp → registra la venta
5. Ve métricas de ventas en el dashboard
```

---

## Optimización para Plan FREE

### Catálogo Astro
- **Build estático (SSG)** — las páginas se generan en build, no en cada request. Cloudflare las sirve desde CDN gratis sin invocar Workers
- **Solo 2 React islands** — mínimo JS para el carrito
- **Imágenes optimizadas** — usar Astro `<Image>` para WebP/AVIF automático
- **Rebuild al modificar productos** — cuando el admin cambia un producto, trigger un nuevo deploy del catálogo (Cloudflare Pages deploy hook)

### Admin Vite Flare Starter
- **Paginar siempre** — máximo 50 productos por página
- **Índices D1** — en slug, category, in_stock, created_at
- **No SELECT * sin WHERE** — siempre filtrar
- **Comprimir imágenes** antes de subir a R2 (max 500 KB)

---

## Deploy

### Catálogo Astro
```bash
# Cloudflare Pages (estático)
pnpm build
npx wrangler pages deploy dist/
```

### Admin Vite Flare Starter
```bash
# Cloudflare Worker
pnpm db:migrate:remote
pnpm deploy
```

### Conectar ambos a la misma D1
En `wrangler.jsonc` de ambos proyectos, usar el mismo `database_id`:
```jsonc
"d1_databases": [{
  "binding": "DB",
  "database_name": "floreria-db",
  "database_id": "mismo-id-en-ambos-proyectos"
}]
```

---

## Checklist de Implementación

### Proyecto 1: Catálogo Astro
- [ ] Setup Astro + Cloudflare adapter + React
- [ ] Schema de productos + D1 (compartido)
- [ ] Home page (hero + productos destacados)
- [ ] Catálogo con filtro por categorías
- [ ] Página de producto con SEO completo (og:image para WhatsApp)
- [ ] Nanostores carrito (add, remove, count, total)
- [ ] AddToCartButton + CartDrawer (React islands)
- [ ] Checkout: generar mensaje WhatsApp

### Proyecto 2: Admin Vite Flare Starter
- [ ] Fork/clonar template + limpiar módulos innecesarios
- [ ] Schema de productos + ventas (mismo D1)
- [ ] Módulo products: CRUD + toggle stock + upload foto R2
- [ ] Módulo sales: registro + lista + métricas
- [ ] Dashboard personalizado (métricas de ventas + accesos rápidos)
- [ ] Eliminar Chat AI, ocultar API Tokens
- [ ] Branding (logo, tema, APP_NAME)

### Infraestructura
- [ ] Crear D1 database `floreria-db` (compartida)
- [ ] Crear R2 bucket `floreria-images` (compartido)
- [ ] Deploy catálogo en Cloudflare Pages
- [ ] Deploy admin en Cloudflare Worker
- [ ] Configurar deploy hook (rebuild catálogo cuando admin modifica productos)
