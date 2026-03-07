---
name: nextjs-app-router-patterns
description: Master Next.js 14+ App Router with Server Components, streaming, parallel routes, and advanced data fetching. Use when building Next.js applications, implementing SSR/SSG, or optimizing React Server Components.
---

# Next.js App Router Patterns (AI-Commerce v2.0)

Comprehensive patterns for Next.js 14+ App Router architecture, tailored specifically to prevent architecture leakage in the `amazon-ai` monorepo.

## 🔴 The Golden Rule for AI-Commerce Frontend

**ALL data fetching MUST go through `frontend/services/`.** No exceptions.

Server Components, Client Components, Server Actions, Route Handlers — regardless of the rendering mode, **every HTTP call to the backend** passes through the centralized Axios service layer (`services/http.ts`).

### ❌ Prohibited Patterns

```typescript
// ❌ NEVER: Direct fetch in a component
const res = await fetch('https://api.example.com/products')

// ❌ NEVER: Raw axios import in a component
import axios from 'axios'
const { data } = await axios.get('/products')

// ❌ NEVER: Direct DB access from the frontend (Server Actions, Route Handlers, etc.)
await db.product.findMany()
await db.cart.upsert(...)

// ❌ NEVER: Calling backend MCP or agent logic from frontend code
import { agentGraph } from 'backend/agents/graph'
```

### ✅ Required Pattern

```typescript
// ✅ ALWAYS: Import the service, call the method
import { productService } from '@/services/product.service'
const products = await productService.getProducts()

// ✅ ALWAYS: For agent interactions
import { agentService } from '@/services/agent.service'
const response = await agentService.sendMessage(message)
```

---

## When to Use This Skill

- Building new Next.js pages/components in the AI-Commerce frontend
- Implementing Server Components and streaming
- Setting up parallel and intercepting routes
- Optimizing data fetching and caching
- Building full-stack features with Server Actions

## Core Concepts

### 1. Rendering Modes

| Mode                  | Where        | When to Use                               |
| --------------------- | ------------ | ----------------------------------------- |
| **Server Components** | Server only  | Data fetching, heavy computation, secrets |
| **Client Components** | Browser      | Interactivity, hooks, browser APIs        |
| **Static**            | Build time   | Content that rarely changes               |
| **Dynamic**           | Request time | Personalized or real-time data            |
| **Streaming**         | Progressive  | Large pages, slow data sources            |

### 2. File Conventions

```
app/
├── layout.tsx       # Shared UI wrapper
├── page.tsx         # Route UI
├── loading.tsx      # Loading UI (Suspense)
├── error.tsx        # Error boundary
├── not-found.tsx    # 404 UI
├── template.tsx     # Re-mounted layout
├── default.tsx      # Parallel route fallback
└── opengraph-image.tsx  # OG image generation
```

### 3. Service Layer Architecture

```
frontend/
├── services/
│   ├── http.ts                # Axios instance, interceptors, base URL
│   ├── product.service.ts     # getProducts(), getProductById(), searchProducts()
│   ├── cart.service.ts        # getCart(), addToCart(), removeFromCart()
│   ├── agent.service.ts       # sendMessage(), streamAgentResponse()
│   └── order.service.ts       # createOrder(), getOrderById()
├── types/
│   ├── product.ts             # Product, ProductFilters, ProductResponse
│   ├── cart.ts                # Cart, CartItem
│   └── agent.ts               # AgentMessage, AgentResponse
```

## Quick Start

```typescript
// services/http.ts — Axios base instance (SINGLE source of HTTP config)
import axios from 'axios'

const http = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api',
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
})

// Auth token interceptor
http.interceptors.request.use((config) => {
  const token = typeof window !== 'undefined'
    ? localStorage.getItem('auth_token')
    : null
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Global error handler
http.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle token expiration
    }
    return Promise.reject(error)
  }
)

export default http
```

```typescript
// services/product.service.ts
import http from './http'
import type { Product, ProductFilters } from '@/types/product'

export const productService = {
  async getProducts(filters?: ProductFilters): Promise<Product[]> {
    const { data } = await http.get('/products', { params: filters })
    return data
  },

  async getProductById(id: string): Promise<Product> {
    const { data } = await http.get(`/products/${id}`)
    return data
  },

  async searchProducts(query: string): Promise<Product[]> {
    const { data } = await http.get('/products/search', { params: { q: query } })
    return data
  },
}
```

```typescript
// app/layout.tsx
import { Inter } from 'next/font/google'
import { Providers } from './providers'

const inter = Inter({ subsets: ['latin'] })

export const metadata = {
  title: { default: 'AI-Commerce', template: '%s | AI-Commerce' },
  description: 'AI-Powered E-Commerce Platform',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

```typescript
// app/page.tsx — Server Component fetching via services
import { productService } from '@/services/product.service'
import { ProductGrid } from '@/components/products/ProductGrid'

export default async function HomePage() {
  const products = await productService.getProducts()

  return (
    <main>
      <h1>Products</h1>
      <ProductGrid products={products} />
    </main>
  )
}
```

## Patterns

### Pattern 1: Server Components with Data Fetching via Services

```typescript
// app/products/page.tsx
import { Suspense } from 'react'
import { ProductList, ProductListSkeleton } from '@/components/products'
import { FilterSidebar } from '@/components/filters'

interface SearchParams {
  category?: string
  sort?: 'price' | 'name' | 'date'
  page?: string
}

export default async function ProductsPage({
  searchParams,
}: {
  searchParams: Promise<SearchParams>
}) {
  const params = await searchParams

  return (
    <div className="flex gap-8">
      <FilterSidebar />
      <Suspense
        key={JSON.stringify(params)}
        fallback={<ProductListSkeleton />}
      >
        <ProductList
          category={params.category}
          sort={params.sort}
          page={Number(params.page) || 1}
        />
      </Suspense>
    </div>
  )
}

// components/products/ProductList.tsx — Server Component
import { productService } from '@/services/product.service'
import type { ProductFilters } from '@/types/product'

export async function ProductList({ category, sort, page }: ProductFilters) {
  // ✅ Data fetching through service layer
  const { products, totalPages } = await productService.getProducts({
    category,
    sort,
    page,
  })

  return (
    <div>
      <div className="grid grid-cols-3 gap-4">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
      <Pagination currentPage={page} totalPages={totalPages} />
    </div>
  )
}
```

### Pattern 2: Client Components with 'use client'

```typescript
// components/products/AddToCartButton.tsx
'use client'

import { useState, useTransition } from 'react'
import { cartService } from '@/services/cart.service'

export function AddToCartButton({ productId }: { productId: string }) {
  const [isPending, startTransition] = useTransition()
  const [error, setError] = useState<string | null>(null)

  const handleClick = () => {
    setError(null)
    startTransition(async () => {
      try {
        // ✅ Client component calls the service, which calls the backend API
        await cartService.addToCart(productId)
      } catch (err) {
        setError('Failed to add item to cart')
      }
    })
  }

  return (
    <div>
      <button
        onClick={handleClick}
        disabled={isPending}
        className="btn-primary"
      >
        {isPending ? 'Adding...' : 'Add to Cart'}
      </button>
      {error && <p className="text-red-500 text-sm">{error}</p>}
    </div>
  )
}
```

### Pattern 3: Server Actions (Mutations via Services)

```typescript
// app/actions/cart.ts
"use server";

import { revalidateTag } from "next/cache";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { cartService } from "@/services/cart.service";
import { orderService } from "@/services/order.service";

export async function addToCart(productId: string) {
  const cookieStore = await cookies();
  const sessionId = cookieStore.get("session")?.value;

  if (!sessionId) {
    redirect("/login");
  }

  try {
    // ✅ Server Action delegates to the service layer → backend API
    await cartService.addToCart(productId, { sessionId });
    revalidateTag("cart");
    return { success: true };
  } catch (error) {
    return { error: "Failed to add item to cart" };
  }
}

export async function checkout(formData: FormData) {
  const address = formData.get("address") as string;
  const payment = formData.get("payment") as string;

  if (!address || !payment) {
    return { error: "Missing required fields" };
  }

  // ✅ Delegates to order service → backend API handles the domain logic
  const order = await orderService.createOrder({ address, payment });

  redirect(`/orders/${order.id}/confirmation`);
}
```

### Pattern 4: Parallel Routes

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return (
    <div className="dashboard-grid">
      <main>{children}</main>
      <aside className="analytics-panel">{analytics}</aside>
      <aside className="team-panel">{team}</aside>
    </div>
  )
}

// app/dashboard/@analytics/page.tsx
import { analyticsService } from '@/services/analytics.service'

export default async function AnalyticsSlot() {
  // ✅ Each parallel slot fetches via its own service
  const stats = await analyticsService.getDashboardStats()
  return <AnalyticsChart data={stats} />
}

// app/dashboard/@analytics/loading.tsx
export default function AnalyticsLoading() {
  return <ChartSkeleton />
}

// app/dashboard/@team/page.tsx
import { teamService } from '@/services/team.service'

export default async function TeamSlot() {
  const members = await teamService.getMembers()
  return <TeamList members={members} />
}
```

### Pattern 5: Intercepting Routes (Modal Pattern)

```typescript
// File structure for product quick-view modal
// app/
// ├── @modal/
// │   ├── (.)products/[id]/page.tsx  # Intercept
// │   └── default.tsx
// ├── products/
// │   └── [id]/page.tsx              # Full page
// └── layout.tsx

// app/@modal/(.)products/[id]/page.tsx
import { Modal } from '@/components/Modal'
import { ProductDetail } from '@/components/ProductDetail'
import { productService } from '@/services/product.service'

export default async function ProductModal({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  // ✅ Uses service layer
  const product = await productService.getProductById(id)

  return (
    <Modal>
      <ProductDetail product={product} />
    </Modal>
  )
}

// app/products/[id]/page.tsx — Full page version
import { productService } from '@/services/product.service'

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const product = await productService.getProductById(id)

  return (
    <div className="product-page">
      <ProductDetail product={product} />
      <RelatedProducts productId={id} />
    </div>
  )
}

// app/layout.tsx
export default function RootLayout({
  children,
  modal,
}: {
  children: React.ReactNode
  modal: React.ReactNode
}) {
  return (
    <html>
      <body>
        {children}
        {modal}
      </body>
    </html>
  )
}
```

### Pattern 6: Streaming with Suspense

```typescript
// app/product/[id]/page.tsx
import { Suspense } from 'react'
import { productService } from '@/services/product.service'

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params

  // This data loads first (blocking) — via service
  const product = await productService.getProductById(id)

  return (
    <div>
      {/* Immediate render */}
      <ProductHeader product={product} />

      {/* Stream in reviews */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />
      </Suspense>

      {/* Stream in AI recommendations */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={id} />
      </Suspense>
    </div>
  )
}

// These components fetch their own data via services
import { reviewService } from '@/services/review.service'
import { productService } from '@/services/product.service'

async function Reviews({ productId }: { productId: string }) {
  const reviews = await reviewService.getByProduct(productId)
  return <ReviewList reviews={reviews} />
}

async function Recommendations({ productId }: { productId: string }) {
  // ✅ Recommendations come from the backend (which internally uses the AI agent)
  const products = await productService.getRecommendations(productId)
  return <ProductCarousel products={products} />
}
```

### Pattern 7: Metadata and SEO

```typescript
// app/products/[slug]/page.tsx
import { Metadata } from 'next'
import { notFound } from 'next/navigation'
import { productService } from '@/services/product.service'

type Props = {
  params: Promise<{ slug: string }>
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params
  const product = await productService.getProductBySlug(slug)

  if (!product) return {}

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [{ url: product.image, width: 1200, height: 630 }],
    },
    twitter: {
      card: 'summary_large_image',
      title: product.name,
      description: product.description,
      images: [product.image],
    },
  }
}

export default async function ProductPage({ params }: Props) {
  const { slug } = await params
  const product = await productService.getProductBySlug(slug)

  if (!product) notFound()

  return <ProductDetail product={product} />
}
```

## Caching Strategies

In AI-Commerce, caching is handled at the **Axios service layer** level using response interceptors, SWR, or React Query, **not** via the native `fetch` cache API.

### Client-Side Caching with SWR

```typescript
// hooks/useProducts.ts
'use client'

import useSWR from 'swr'
import { productService } from '@/services/product.service'

export function useProducts(filters?: ProductFilters) {
  const { data, error, isLoading, mutate } = useSWR(
    ['products', filters],
    () => productService.getProducts(filters),
    {
      revalidateOnFocus: false,
      dedupingInterval: 60000, // Dedupe requests within 60s
    }
  )

  return { products: data, error, isLoading, refresh: mutate }
}
```

### Server-Side Revalidation

```typescript
// When using Server Actions that mutate data, revalidate cache tags:
"use server";

import { revalidateTag, revalidatePath } from "next/cache";
import { productService } from "@/services/product.service";

export async function updateProduct(id: string, data: ProductData) {
  await productService.updateProduct(id, data);
  revalidateTag("products");
  revalidatePath("/products");
}
```

## Best Practices

### Do's

- **Start with Server Components** — Add 'use client' only when needed
- **Always use `services/`** — Every HTTP call goes through the Axios service layer
- **Use Suspense boundaries** — Enable streaming for slow data
- **Leverage parallel routes** — Independent loading states
- **Use Server Actions for mutations** — But always delegate to `services/`
- **Type everything** — Mirror backend Pydantic schemas in `frontend/types/`

### Don'ts

- **Don't use `fetch` directly** — Use the Axios service layer from `services/http.ts`
- **Don't access DB from frontend** — No `db.*`, no Prisma, no SQLAlchemy in Next.js
- **Don't use hooks in Server Components** — No useState, useEffect
- **Don't bypass the service proxy** — No raw `axios.get()` in components
- **Don't over-nest layouts** — Each layout adds to the component tree
- **Don't ignore loading states** — Always provide `loading.tsx` or Suspense

## Resources

- [Next.js App Router Documentation](https://nextjs.org/docs/app)
- [SWR Documentation](https://swr.vercel.app/)
- [Axios Documentation](https://axios-http.com/)
