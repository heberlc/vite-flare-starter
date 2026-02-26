# Plan de Desarrollo — Florería Picaflor

> **Versión:** 3.0 (arquitectura Turborepo definitiva)
> **Fecha:** 2026-02-26
> **Arquitectura:** Turborepo monorepo — Astro SSR + VFS
> **Plan Cloudflare:** Free
> **Estimación total:** ~3.5 semanas (6 fases)
> **Decisiones tecnológicas:** ver [TECH-STACK-DECISIONS.md](./TECH-STACK-DECISIONS.md)

---

## Resumen Ejecutivo

Se construye un sistema para la florería usando un **Turborepo monorepo** con dos aplicaciones y un paquete compartido:

| App | Tecnología | Deploy target | Función |
|-----|-----------|---------------|---------|
| `apps/catalog` | Astro SSR | Cloudflare Pages/Worker | Catálogo público con SEO |
| `apps/admin` | React + Hono (VFS) | Cloudflare Worker | Panel admin |
| `packages/shared` | Drizzle + Zod | — | Schema y tipos compartidos |

---

## Fase 0: Setup Turborepo + Limpieza VFS (~3 días)

> Migrar el proyecto actual a estructura monorepo y limpiar el admin.

### 0.1 Crear Estructura Monorepo

```
floreria-picaflor/
├── apps/
│   ├── admin/         ← Mover el VFS actual aquí
│   └── catalog/       ← Nuevo proyecto Astro
├── packages/
│   └── shared/        ← Extraer schemas compartidos
├── turbo.json
├── pnpm-workspace.yaml
└── package.json       ← Root (sin deps, solo scripts turbo)
```

- [ ] Crear directorio raíz `floreria-picaflor/`
- [ ] Crear `pnpm-workspace.yaml`:
  ```yaml
  packages:
    - "apps/*"
    - "packages/*"
  ```
- [ ] Crear `turbo.json` con pipelines: `dev`, `build`, `deploy`, `lint`, `test`
- [ ] Crear `package.json` raíz con scripts turbo
- [ ] Mover el proyecto VFS actual a `apps/admin/`
- [ ] Verificar que `apps/admin` arranca con `pnpm dev`

### 0.2 Crear `packages/shared`

- [ ] Crear `packages/shared/package.json` con nombre `@floreria/shared`
- [ ] Crear `packages/shared/tsconfig.json`
- [ ] Mover schemas de productos y ventas (nuevos) a `packages/shared/src/db/`
- [ ] Crear `packages/shared/src/schemas/` con Zod validations compartidos
- [ ] Crear `packages/shared/src/types.ts` con tipos compartidos
- [ ] Crear `packages/shared/src/constants.ts` (categorías, WhatsApp number)
- [ ] Mover `drizzle/` migraciones a `packages/shared/drizzle/`
- [ ] Configurar exports en `package.json` del shared

### 0.3 Infraestructura Cloudflare

- [ ] Crear D1 database: `wrangler d1 create floreria-db`
- [ ] Crear R2 bucket: `wrangler r2 bucket create floreria-products`
- [ ] Crear R2 bucket: `wrangler r2 bucket create floreria-avatars`
- [ ] Actualizar `apps/admin/wrangler.jsonc`:
  - `name` → `floreria-admin`
  - `database_name` → `floreria-db` + nuevo `database_id`
  - Agregar binding `PRODUCTS` → `floreria-products`
  - Renombrar binding `AVATARS` → `floreria-avatars`
  - Eliminar binding `AI`
- [ ] Actualizar scripts de migración en `apps/admin/package.json`

### 0.4 Limpieza de Módulos Admin (Server)

- [ ] Eliminar `src/server/modules/chat/`
- [ ] Eliminar `src/server/modules/api-tokens/`
- [ ] Eliminar `src/server/modules/notifications/`
- [ ] Eliminar `src/server/lib/ai/`
- [ ] Limpiar `src/server/index.ts`: eliminar rutas e imports de chat, api-tokens, notifications, AI
- [ ] Limpiar type `Env`: eliminar `AI`, `GOOGLE_CLIENT_*`, `AI_GATEWAY_*`, `CF_AIG_*`
- [ ] Agregar `PRODUCTS: R2Bucket` al type `Env`
- [ ] Eliminar exports de `apiTokens`, `userNotifications*` de `src/server/db/schema.ts`

### 0.5 Limpieza de Módulos Admin (Client)

- [ ] Eliminar `src/client/modules/chat/`
- [ ] Eliminar `src/client/modules/api-tokens/`
- [ ] Eliminar rutas de chat y api-tokens de `src/client/App.tsx`
- [ ] Eliminar `LandingPage` y `PublicLayout`
- [ ] Redirect `/` → `/dashboard` (el admin no tiene landing)
- [ ] Limpiar sidebar: eliminar links a chat, api-tokens

### 0.6 Limpieza de Config y Dependencias

- [ ] Eliminar `src/shared/config/voice-agents.ts`
- [ ] Eliminar `src/shared/config/voice-variables.ts`
- [ ] Eliminar `src/shared/schemas/api-token.schema.ts`
- [ ] Desinstalar dependencias innecesarias:
  ```bash
  pnpm remove @cloudflare/ai-utils @sentry/cloudflare @sentry/react cmdk react-markdown remark-gfm resend ua-parser-js zod-to-json-schema
  ```
- [ ] Configurar `.dev.vars`:
  ```
  VITE_APP_NAME=Florería Picaflor
  VITE_APP_ID=floreria-picaflor
  ENABLE_EMAIL_LOGIN=true
  ENABLE_EMAIL_SIGNUP=false
  VITE_FEATURE_API_TOKENS=false
  ```
- [ ] Actualizar `index.html` con branding
- [ ] Verificar: `pnpm build` compila sin errores
- [ ] Verificar: `pnpm test` pasa (ajustar tests que fallen por eliminaciones)

---

## Fase 1: Schema Compartido y API de Productos (~3 días)

> Crear el schema de productos en shared y el CRUD completo en admin.

### 1.1 Schema de Productos (`packages/shared`)

- [ ] Crear `packages/shared/src/db/products.ts`:
  ```
  id, name, slug, description, category, price (centavos),
  image (URL R2), images (JSON array), inStock, featured,
  sortOrder, createdAt, updatedAt
  ```
- [ ] Crear `packages/shared/src/schemas/product.schema.ts` (Zod)
- [ ] Crear `packages/shared/src/constants.ts`:
  - Categorías: `ramos`, `arreglos`, `plantas`, `complementos`
  - Moneda: soles (S/)
- [ ] Exportar todo desde `packages/shared/src/index.ts`
- [ ] Generar migración: `pnpm db:generate:named "add_products_table"`
- [ ] Aplicar migración local: `pnpm db:migrate:local`

### 1.2 API de Productos (`apps/admin` — Server)

- [ ] Crear `src/server/modules/products/routes.ts`:
  - `GET /api/products` — listar con filtros (category, stock, featured, paginado)
  - `GET /api/products/:id` — obtener uno
  - `POST /api/products` — crear (auth required)
  - `PUT /api/products/:id` — actualizar (auth required)
  - `PATCH /api/products/:id/stock` — toggle stock (auth required)
  - `DELETE /api/products/:id` — eliminar (auth required)
- [ ] Crear `src/server/modules/products/db/schema.ts` — re-exportar desde `@floreria/shared`
- [ ] Registrar rutas en `src/server/index.ts`
- [ ] Agregar activity logging en create/update/delete
- [ ] Crear rutas públicas sin auth (para que Astro las consuma):
  - `GET /api/public/products` — lista pública (solo in_stock=1)
  - `GET /api/public/products/:slug` — producto por slug
  - `GET /api/public/categories` — lista de categorías con count

### 1.3 Upload de Fotos de Productos

- [ ] Crear `POST /api/products/:id/image` — upload foto principal a R2 `PRODUCTS`
- [ ] Crear `DELETE /api/products/:id/image` — eliminar foto de R2
- [ ] Servir fotos: `GET /api/public/image/:key` — leer de R2 con cache headers
- [ ] Validar: solo imágenes (jpg, png, webp), max 2MB
- [ ] Comprimir en frontend antes de upload

### 1.4 Frontend de Productos (`apps/admin` — Client)

- [ ] Crear `src/client/modules/products/`:
  ```
  products/
  ├── pages/
  │   ├── ProductsPage.tsx         ← Tabla con filtros
  │   └── ProductFormPage.tsx      ← Crear/editar producto
  ├── components/
  │   ├── ProductsTable.tsx        ← DataTable shadcn/ui
  │   ├── ProductForm.tsx          ← React Hook Form + Zod
  │   ├── StockToggle.tsx          ← Switch component
  │   ├── CategoryFilter.tsx       ← Select/tabs de categorías
  │   └── ProductImageUpload.tsx   ← Dropzone con preview
  └── hooks/
      └── useProducts.ts           ← TanStack Query hooks
  ```
- [ ] Agregar rutas en `App.tsx`:
  - `/dashboard/products` → `ProductsPage`
  - `/dashboard/products/new` → `ProductFormPage`
  - `/dashboard/products/:id/edit` → `ProductFormPage`
- [ ] Agregar link "Productos" en sidebar

---

## Fase 2: Ventas y Dashboard (~3 días)

> Sistema de registro de ventas y dashboard con métricas.

### 2.1 Schema de Ventas (`packages/shared`)

- [ ] Crear `packages/shared/src/db/sales.ts`:
  ```
  id, productId, productName, productPrice, quantity, total,
  clientName, clientPhone, notes, source ("whatsapp"|"tienda"),
  createdBy (userId), createdAt
  ```
- [ ] Crear `packages/shared/src/schemas/sale.schema.ts` (Zod)
- [ ] Generar migración: `pnpm db:generate:named "add_sales_table"`
- [ ] Aplicar migración local

### 2.2 API de Ventas (`apps/admin` — Server)

- [ ] Crear `src/server/modules/sales/routes.ts`:
  - `GET /api/sales` — listar ventas paginadas (auth required)
  - `POST /api/sales` — registrar venta (auth required)
  - `GET /api/sales/stats` — métricas: hoy, semana, mes
  - `GET /api/sales/stats/daily` — ventas por día (últimos 30 días, para gráfica)
  - `DELETE /api/sales/:id` — eliminar venta (auth required)
- [ ] Registrar rutas en `src/server/index.ts`
- [ ] Agregar activity logging

### 2.3 Frontend de Ventas (`apps/admin` — Client)

- [ ] Crear `src/client/modules/sales/`:
  ```
  sales/
  ├── pages/
  │   ├── SalesPage.tsx            ← Lista de ventas
  │   └── NewSalePage.tsx          ← Registrar venta
  ├── components/
  │   ├── SalesTable.tsx           ← Tabla con fecha, producto, monto
  │   ├── SaleForm.tsx             ← Seleccionar producto(s), cantidad, cliente
  │   ├── SalesStats.tsx           ← Cards de métricas
  │   └── SalesChart.tsx           ← Gráfica de ventas diarias (simple)
  └── hooks/
      └── useSales.ts             ← TanStack Query hooks
  ```
- [ ] Agregar rutas en `App.tsx`
- [ ] Agregar link "Ventas" en sidebar

### 2.4 Dashboard Personalizado

- [ ] Rediseñar `DashboardPage.tsx`:
  - 3 cards de métricas: ventas hoy | ventas semana | ingresos mes
  - Lista de últimas 5 ventas
  - Productos sin stock o con stock bajo
  - Botones rápidos: "Registrar venta", "Agregar producto"

---

## Fase 3: Catálogo Astro SSR (~4 días)

> Crear la app pública del catálogo con SEO y WhatsApp checkout.

### 3.1 Setup Astro

- [ ] Crear `apps/catalog/`:
  ```bash
  cd apps
  npm create astro@latest catalog -- --template minimal
  cd catalog
  npx astro add cloudflare
  npx astro add react
  npx astro add tailwind
  pnpm add nanostores @nanostores/react
  pnpm add drizzle-orm
  ```
- [ ] Configurar `astro.config.mjs`:
  - Output: `server` (SSR)
  - Adapter: `@astrojs/cloudflare`
  - Integrations: react, tailwind
- [ ] Crear `apps/catalog/wrangler.jsonc`:
  ```jsonc
  {
    "name": "floreria-catalog",
    "d1_databases": [{
      "binding": "DB",
      "database_name": "floreria-db",
      "database_id": "mismo-id-que-admin"
    }],
    "r2_buckets": [{
      "binding": "PRODUCTS",
      "bucket_name": "floreria-products"
    }]
  }
  ```
- [ ] Agregar `@floreria/shared` como dependencia
- [ ] Verificar que `pnpm dev` arranca el catálogo

### 3.2 Layout y Componentes Base

- [ ] Crear `src/layouts/BaseLayout.astro`:
  - Header con logo, navegación, botón carrito
  - Footer con WhatsApp, dirección, horario
  - Meta tags SEO base
- [ ] Crear `src/components/ProductCard.astro`
- [ ] Crear `src/components/CategoryNav.astro`
- [ ] Crear `src/components/SEOHead.astro` (meta tags reutilizables)

### 3.3 Páginas con SEO

- [ ] Crear `src/pages/index.astro`:
  - Hero con branding de la florería
  - Grid de productos destacados (`featured=1`)
  - Navegación por categorías
  - CTA "Ver catálogo completo"
- [ ] Crear `src/pages/catalogo/index.astro`:
  - Grid de todos los productos disponibles
  - Filtro por categorías
  - `<title>Catálogo | Florería Picaflor</title>`
- [ ] Crear `src/pages/catalogo/[category].astro`:
  - Productos filtrados por categoría
  - `<title>Ramos | Catálogo | Florería Picaflor</title>`
- [ ] Crear `src/pages/producto/[slug].astro`:
  - Foto grande, nombre, descripción, precio
  - Meta tags OG completos:
    ```html
    <meta property="og:title" content="{product.name}" />
    <meta property="og:image" content="{product.image}" />
    <meta property="og:description" content="{product.description}" />
    <meta property="og:type" content="product" />
    <meta property="product:price:amount" content="{price}" />
    ```
  - Botón "Agregar al carrito"
  - Botón "Pedir por WhatsApp" (directo, sin carrito)

### 3.4 Carrito + Checkout WhatsApp (React Islands)

- [ ] Crear `src/stores/cart.ts` (nanostores):
  - `$cart` — map de items
  - `$cartTotal` — computed total
  - `$cartCount` — computed count
  - `addToCart()`, `removeFromCart()`, `clearCart()`
- [ ] Crear `src/components/react/AddToCartButton.tsx` (React island)
- [ ] Crear `src/components/react/CartDrawer.tsx` (React island):
  - Lista de items con cantidad y subtotal
  - Total
  - Botón "Enviar pedido por WhatsApp"
- [ ] Crear `src/lib/whatsapp.ts`:
  - Genera mensaje formateado con emoji
  - Abre `wa.me/{numero}?text={mensaje}`

### 3.5 Acceso a Datos

- [ ] Crear `src/lib/db.ts` — helper para acceder a D1 via Astro runtime
- [ ] Crear funciones de queries:
  - `getProducts(filters)` — lista paginada
  - `getProductBySlug(slug)` — producto individual
  - `getCategories()` — categorías con count
  - `getFeaturedProducts()` — productos destacados

### 3.6 Diseño Visual

- [ ] Diseño mobile-first responsive
- [ ] Colores y tipografía consistentes con la florería
- [ ] Badge "Agotado" para productos sin stock (visibles pero no comprables)
- [ ] Animaciones sutiles en hover de cards
- [ ] Optimización de imágenes con Astro `<Image>`

---

## Fase 4: Configuración de Florería y Conexión (~2 días)

> Adaptar organization settings y conectar ambas apps.

### 4.1 Adaptar Organization Settings

- [ ] Agregar campos a la tabla/schema de organization:
  - `whatsappNumber` — número de WhatsApp para pedidos
  - `address` — dirección de la tienda
  - `schedule` — horario de atención
  - `instagramUrl` — link a Instagram (opcional)
  - `facebookUrl` — link a Facebook (opcional)
- [ ] Actualizar formulario de organization en admin
- [ ] Crear endpoint público: `GET /api/public/config` — datos de la florería

### 4.2 Conectar Catálogo con Config

- [ ] El catálogo Astro consume `GET /api/public/config` del admin (o lee directo de D1)
- [ ] Usar datos en footer, página de contacto, botón WhatsApp

---

## Fase 5: Polish y Deploy (~2 días)

> Optimizar, asegurar y desplegar.

### 5.1 Optimización

- [ ] Comprimir imágenes antes de upload (frontend, WebP, max 500KB)
- [ ] Paginación en listas (máximo 50 items)
- [ ] Cache headers en catálogo público (`Cache-Control: public, max-age=300`)
- [ ] Astro `<Image>` para lazy loading y formatos modernos

### 5.2 Seguridad

- [ ] Todas las rutas de escritura requieren auth en admin
- [ ] Rutas públicas son solo lectura
- [ ] Rate limiting en uploads
- [ ] Validar tipos de archivo (solo imágenes)
- [ ] Sanitizar inputs de texto (prevenir XSS)

### 5.3 Deploy Admin

- [ ] Aplicar migraciones remotas: `pnpm db:migrate:remote`
- [ ] Configurar secrets admin:
  ```bash
  echo "secret" | npx wrangler secret put BETTER_AUTH_SECRET
  echo "https://admin.floreria-picaflor.com" | npx wrangler secret put BETTER_AUTH_URL
  echo "http://localhost:5173,https://admin.floreria-picaflor.com" | npx wrangler secret put TRUSTED_ORIGINS
  echo "admin@floreria.com" | npx wrangler secret put ADMIN_EMAILS
  ```
- [ ] Deploy: `pnpm deploy` desde `apps/admin/`

### 5.4 Deploy Catálogo

- [ ] Configurar Cloudflare Pages o Worker para `apps/catalog/`
- [ ] Deploy: `pnpm build && npx wrangler pages deploy` desde `apps/catalog/`
- [ ] Verificar SEO: compartir URL de producto en WhatsApp → debe mostrar preview
- [ ] Verificar catálogo en móvil

### 5.5 Verificación Final

- [ ] Flujo completo cliente: ver catálogo → agregar al carrito → enviar pedido WhatsApp
- [ ] Flujo completo admin: login → agregar producto con foto → registrar venta → ver dashboard
- [ ] SEO: meta tags OG en cada página de producto
- [ ] WhatsApp preview: compartir link → se ve foto, nombre, descripción
- [ ] Responsive: catálogo funciona en móvil

---

## Flujo Completo del Sistema

```
CLIENTE:
1. Visita floreria-picaflor.com              → Astro SSR (catálogo)
2. Navega por categorías, ve productos       → D1 read (products)
3. Agrega productos al carrito               → nanostores (localStorage)
4. Click "Enviar pedido por WhatsApp"        → wa.me con mensaje
5. Coordina pago y entrega en WhatsApp       → humano

ADMIN:
1. Visita admin.floreria-picaflor.com       → VFS (React SPA)
2. Login con email/password                  → better-auth
3. Agrega productos con fotos               → D1 write + R2 upload
4. Marca stock agotado                       → D1 write (in_stock=0)
5. Registra venta confirmada                 → D1 write (sales)
6. Ve métricas en dashboard                  → D1 read (sales stats)

SINCRONIZACIÓN:
- Admin escribe en D1/R2 ← mismos recursos
- Catálogo lee de D1/R2  ← reflejo instantáneo (SSR)
- No hay deploy hooks ni rebuilds necesarios
```

---

## Dominios Sugeridos (Cloudflare Workers/Pages)

| App | Dominio | Tipo |
|-----|---------|------|
| Catálogo | `floreria-picaflor.com` o `floreria-catalog.{subdomain}.workers.dev` | Pages/Worker |
| Admin | `admin.floreria-picaflor.com` o `floreria-admin.{subdomain}.workers.dev` | Worker |
