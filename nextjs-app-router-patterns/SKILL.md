---
name: nextjs-app-router-patterns
description: >-
  Next.js 14+ App Router patterns: Server Components, Client Components,
  streaming, parallel routes, intercepting routes, service layer architecture,
  and caching strategies. Use when building Next.js apps, implementing SSR/SSG,
  or setting up frontend data fetching patterns.
---

# Next.js App Router Patterns

## Golden Rule

**All data fetching MUST go through a centralized service layer** (`services/` or `lib/services/`). No exceptions — regardless of rendering mode (Server Components, Client Components, Server Actions).

### Prohibited

```typescript
// NEVER: Direct fetch in a component
const res = await fetch('/api/users')

// NEVER: Raw axios import in a component
import axios from 'axios'
const { data } = await axios.get('/api/users')

// NEVER: Direct DB access from the frontend
await db.user.findMany()
```

### Required

```typescript
// ALWAYS: Through the service layer
import { userService } from '@/services/user'
const users = await userService.getAll()
```

## Service Layer Architecture

```
lib/
├── api-client.ts           # Axios instance, interceptors, base URL
├── services/
│   ├── auth.ts             # login, register, getProfile
│   ├── users.ts            # getUsers, getUserById
│   └── {domain}.ts         # One file per domain
├── store/                  # Zustand stores (client state)
└── config/env.ts           # NEXT_PUBLIC_* defaults

@types/
├── auth.ts                 # AuthResponse, LoginRequest
├── users.ts                # User, UserCreate
└── {domain}.ts             # One file per domain
```

### Axios Base Instance

```typescript
// lib/api-client.ts
import axios from 'axios'

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:9015/api',
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
})

apiClient.interceptors.request.use((config) => {
  const token = typeof window !== 'undefined'
    ? localStorage.getItem('auth_token') : null
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

export default apiClient
```

### Service Example

```typescript
// lib/services/users.ts
import apiClient from '@/lib/api-client'
import type { User, UserCreate } from '@/types/users'

export const userService = {
  async getAll(): Promise<User[]> {
    const { data } = await apiClient.get('/users')
    return data
  },
  async getById(id: number): Promise<User> {
    const { data } = await apiClient.get(`/users/${id}`)
    return data
  },
  async create(payload: UserCreate): Promise<User> {
    const { data } = await apiClient.post('/users', payload)
    return data
  },
}
```

## File Conventions

```
app/
├── layout.tsx       # Shared UI wrapper (persistent across navigations)
├── page.tsx         # Route UI
├── loading.tsx      # Loading UI (automatic Suspense boundary)
├── error.tsx        # Error boundary
├── not-found.tsx    # 404 UI
└── template.tsx     # Re-mounted on every navigation
```

## Pattern 1: Server Components with Services

```typescript
// app/users/page.tsx — Server Component (default)
import { Suspense } from 'react'
import { userService } from '@/services/users'

export default async function UsersPage() {
  const users = await userService.getAll()
  return (
    <div>
      <h1>Users</h1>
      <Suspense fallback={<UserListSkeleton />}>
        <UserList users={users} />
      </Suspense>
    </div>
  )
}
```

## Pattern 2: Client Components

```typescript
// components/CreateUserForm.tsx
'use client'

import { useState, useTransition } from 'react'
import { userService } from '@/services/users'

export function CreateUserForm() {
  const [isPending, startTransition] = useTransition()

  const handleSubmit = (formData: FormData) => {
    startTransition(async () => {
      await userService.create({
        email: formData.get('email') as string,
        name: formData.get('name') as string,
      })
    })
  }

  return (
    <form action={handleSubmit}>
      <input name="email" type="email" required />
      <input name="name" required />
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  )
}
```

## Pattern 3: Server Actions via Services

```typescript
// app/actions/users.ts
"use server"

import { revalidateTag } from "next/cache"
import { redirect } from "next/navigation"
import { userService } from "@/services/users"

export async function createUser(formData: FormData) {
  const email = formData.get("email") as string
  if (!email) return { error: "Email required" }

  await userService.create({ email, name: formData.get("name") as string })
  revalidateTag("users")
  redirect("/users")
}
```

## Pattern 4: Parallel Routes

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children, analytics, activity,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  activity: React.ReactNode
}) {
  return (
    <div className="grid grid-cols-3 gap-4">
      <main className="col-span-2">{children}</main>
      <aside>{analytics}</aside>
      <aside>{activity}</aside>
    </div>
  )
}

// app/dashboard/@analytics/page.tsx — loads independently
import { dashboardService } from '@/services/dashboard'

export default async function AnalyticsSlot() {
  const stats = await dashboardService.getStats()
  return <StatsChart data={stats} />
}
```

## Pattern 5: Streaming with Suspense

```typescript
// app/users/[id]/page.tsx
import { Suspense } from 'react'
import { userService } from '@/services/users'

export default async function UserPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const user = await userService.getById(Number(id))

  return (
    <div>
      <UserHeader user={user} />
      <Suspense fallback={<ActivitySkeleton />}>
        <UserActivity userId={id} />
      </Suspense>
    </div>
  )
}
```

## Pattern 6: Metadata and SEO

```typescript
import { Metadata } from 'next'
import { userService } from '@/services/users'

export async function generateMetadata({ params }): Promise<Metadata> {
  const { id } = await params
  const user = await userService.getById(Number(id))
  return { title: user.name, description: `Profile of ${user.name}` }
}
```

## Caching Strategies

### Client-Side with TanStack Query

```typescript
'use client'
import { useQuery } from '@tanstack/react-query'
import { userService } from '@/services/users'

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => userService.getAll(),
    staleTime: 60_000,
  })
}
```

### Server-Side Revalidation

```typescript
"use server"
import { revalidateTag, revalidatePath } from "next/cache"

export async function updateUser(id: number, data: UserUpdate) {
  await userService.update(id, data)
  revalidateTag("users")
  revalidatePath("/users")
}
```

## Good Practices

- Start with Server Components — add `'use client'` only when needed
- Every HTTP call goes through `services/`
- Use Suspense boundaries for streaming slow data
- Leverage parallel routes for independent loading states
- Type everything — mirror backend schemas in `@types/`
- Provide `loading.tsx` or Suspense fallbacks for every data fetch

## Bad Practices

- Direct `fetch()` or `axios` imports in components
- Using hooks (`useState`, `useEffect`) in Server Components
- Bypassing the service layer for "quick" API calls
- Over-nesting layouts (each adds to the component tree)
- Ignoring loading and error states
- Returning `any` from service methods

## Checklist

- [ ] Centralized Axios instance in `lib/api-client.ts`
- [ ] One service file per backend domain in `services/`
- [ ] TypeScript interfaces for all API responses in `@types/`
- [ ] No direct `fetch`/`axios` imports in components
- [ ] Server Components by default; `'use client'` only when interactive
- [ ] Suspense boundaries around async data
- [ ] `loading.tsx` and `error.tsx` for every route segment
- [ ] Server Actions delegate to services, never call DB directly
