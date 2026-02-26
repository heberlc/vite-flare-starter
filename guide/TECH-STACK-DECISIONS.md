# Decisiones Tecnológicas — Florería Picaflor

> **Fecha:** 2026-02-26
> **Arquitectura:** Turborepo monorepo — Astro SSR (catálogo público) + VFS (admin)
> **Basado en:** Análisis del template Vite Flare Starter v0.12.0 + PLAN-FLORERIA.md

---

## Arquitectura Definitiva

```
floreria-picaflor/                         ← Turborepo monorepo
├── apps/
│   ├── catalog/                           ← Astro SSR → Cloudflare Pages/Worker
│   │   ├── src/pages/                     ← Páginas con SEO + OG tags
│   │   ├── src/components/                ← Componentes Astro + React islands
│   │   ├── src/layouts/                   ← Layout público
│   │   ├── astro.config.mjs
│   │   └── wrangler.jsonc                 ← Bindings: DB (D1), PRODUCTS (R2)
│   │
│   └── admin/                             ← VFS (React + Hono) → Cloudflare Worker
│       ├── src/client/                    ← React SPA (shadcn/ui)
│       ├── src/server/                    ← API Hono
│       ├── src/shared/                    ← Schemas Zod
│       └── wrangler.jsonc                 ← Bindings: DB (D1), PRODUCTS (R2), AVATARS (R2)
│
├── packages/
│   └── shared/                            ← Código compartido
│       ├── src/
│       │   ├── db/schema.ts               ← Drizzle schemas (products, sales)
│       │   ├── schemas/                   ← Zod validations
│       │   ├── types.ts                   ← TypeScript types
│       │   └── constants.ts               ← Categorías, WhatsApp, etc.
│       ├── drizzle/                       ← Migraciones D1
│       └── package.json
│
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

### ¿Por qué Turborepo?

| Beneficio | Detalle |
|-----------|---------|
| **Schema compartido** | `import { products } from '@floreria/shared'` en ambas apps |
| **Un solo `turbo dev`** | Levanta ambas apps en paralelo |
| **Un solo `turbo deploy`** | Despliega ambas apps + migraciones |
| **Tipos siempre en sync** | Un cambio en `packages/shared` se refleja en ambas apps |
| **Migraciones centralizadas** | Viven en `packages/shared/drizzle/` |
| **Cache de builds** | Turbo cachea lo que no cambió |

---

## Stack por App

### `apps/catalog` — Catálogo Público (Astro SSR)

| Tecnología | Versión | Propósito |
|-----------|---------|-----------|
| **Astro** | latest | Framework SSR para páginas con SEO |
| **@astrojs/cloudflare** | latest | Adaptador para Cloudflare Pages/Workers |
| **@astrojs/react** | latest | React islands para carrito (2 componentes) |
| **React** | ^19 | Solo para islands (AddToCart, CartDrawer) |
| **Drizzle ORM** | ^0.44 | Acceso a D1 para leer productos |
| **Tailwind CSS** | v4 | Estilos (mismo sistema que admin) |
| **nanostores** | latest | Estado del carrito (compartido entre islands) |
| **@nanostores/react** | latest | Binding React para nanostores |

**No incluir en catalog:** shadcn/ui, TanStack Query, React Hook Form, better-auth, Hono

### `apps/admin` — Panel Admin (VFS)

| Tecnología | Versión | Propósito |
|-----------|---------|-----------|
| **React 19** | ^19.2.0 | SPA completa |
| **Vite** | ^7.2.2 | Build tool |
| **Hono** | ^4.10.6 | API backend |
| **Drizzle ORM** | ^0.44.7 | Acceso a D1 |
| **better-auth** | ^1.3.34 | Autenticación email/password |
| **Tailwind CSS v4** | ^4.1.17 | Estilos |
| **shadcn/ui (Radix)** | — | Componentes UI profesionales |
| **TanStack Query** | ^5.90.10 | Data fetching + cache |
| **React Hook Form** | ^7.66.0 | Formularios |
| **Zod** | ^3.25.76 | Validación |
| **react-router-dom** | ^7.9.6 | Routing SPA |
| **lucide-react** | ^0.553.0 | Iconos |
| **sonner** | ^2.0.7 | Toasts |
| **date-fns** | ^4.1.0 | Formateo de fechas |
| **react-dropzone** | ^14.3.8 | Upload de fotos |
| **@cloudflare/vite-plugin** | ^1.14.2 | Build CF Worker |
| **Biome** | — | Linting/formatting |
| **Vitest** | ^3.2.4 | Testing |

### `packages/shared` — Código Compartido

| Tecnología | Propósito |
|-----------|-----------|
| **Drizzle ORM** | Definición de tablas (products, sales) |
| **Zod** | Schemas de validación compartidos |
| **TypeScript** | Tipos compartidos |
| **drizzle-kit** | Generación de migraciones |

### Infraestructura Cloudflare (compartida)

| Recurso | Binding | Nombre | Usado por |
|---------|---------|--------|-----------|
| **D1** | `DB` | `floreria-db` | catalog + admin |
| **R2** | `PRODUCTS` | `floreria-products` | catalog (lectura) + admin (escritura) |
| **R2** | `AVATARS` | `floreria-avatars` | admin solamente |

---

## Módulos VFS: Mantener vs Eliminar

### ✅ SE MANTIENE en Admin

| Módulo Server | Módulo Client | Propósito |
|---------------|---------------|-----------|
| `auth` | `auth` | Login del admin |
| `settings` | `settings` | Perfil usuario admin |
| `admin` | `admin` | Panel admin de usuarios |
| `activity` | `activity` | Historial de cambios |
| `files` | `files` | Upload de fotos a R2 |
| `organization` | `organization` | Config de la florería |
| `feature-flags` | — | Mantener módulo (no crear flags nuevos) |

### ❌ SE ELIMINA del Admin

| Módulo/Dependencia | Razón |
|--------------------|-------|
| `chat` (server + client) | No hay funcionalidad AI |
| `api-tokens` (server + client) | No hay integraciones externas |
| `notifications` (server + client) | Pedidos van por WhatsApp |
| `src/server/lib/ai/` | No hay AI |
| `LandingPage` + `PublicLayout` | El público va por Astro |
| `StyleGuidePage` + `ComponentsPage` | Se desactiva con feature flags |
| `voice-agents.ts` + `voice-variables.ts` | No aplica |

### ❌ DEPENDENCIAS A DESINSTALAR del Admin

```
@cloudflare/ai-utils
@sentry/cloudflare
@sentry/react
cmdk
react-markdown
remark-gfm
resend
ua-parser-js
zod-to-json-schema
```

### ❌ BINDINGS A ELIMINAR en Admin

```jsonc
// Eliminar de wrangler.jsonc del admin:
"ai": { "binding": "AI" }   ← No hay AI
```

---

## Límites Plan FREE — Verificación

| Recurso | Límite Free | Uso estimado (ambas apps) | Estado |
|---------|-------------|---------------------------|--------|
| Workers requests | 100,000/día total | catalog ~10,000 + admin ~500 | ✅ Sobra |
| D1 storage | 500 MB | ~15 MB | ✅ Sobra |
| D1 reads | 5M/día | ~20,000/día | ✅ Sobra |
| D1 writes | 100K/día | ~100/día | ✅ Sobra |
| R2 storage | 10 GB | ~1 GB (fotos) | ✅ Sobra |
| R2 Class A ops | 1M/mes | ~1,000/mes (uploads) | ✅ Sobra |
| R2 Class B ops | 10M/mes | ~50,000/mes (reads) | ✅ Sobra |
| Pages deploys | 500/mes | ~30-50/mes | ✅ Sobra |
