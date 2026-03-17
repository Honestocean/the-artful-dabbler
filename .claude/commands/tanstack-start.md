# TanStack Start Reference

You are an expert on TanStack Start, the full-stack React framework. Use this reference when building features, adding routes, configuring the project, or answering questions about TanStack Start.

## Overview

TanStack Start is a type-safe, client-first, full-stack React framework built on top of TanStack Router, Vite, and Nitro. It provides SSR, streaming, server functions, and file-based routing out of the box.

## Project Structure

```
├── package.json
├── tsconfig.json
├── vite.config.ts
├── public/
└── src/
    ├── router.tsx              # Router factory
    ├── routeTree.gen.ts        # Auto-generated route tree (DO NOT EDIT)
    ├── components/             # Shared components
    ├── routes/                 # File-based routes
    │   ├── __root.tsx          # Root layout (shell, head, scripts)
    │   ├── index.tsx           # Home route (/)
    │   ├── posts.tsx           # Layout route for /posts/*
    │   ├── posts.index.tsx     # /posts
    │   └── posts.$postId.tsx   # /posts/:postId (dynamic param)
    ├── styles/
    └── utils/
```

## Key Configuration Files

### vite.config.ts
```ts
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'
import viteReact from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import { nitro } from 'nitro/vite'

export default defineConfig({
  server: { port: 3000 },
  resolve: { tsconfigPaths: true },
  plugins: [
    tailwindcss(),
    tanstackStart({ srcDirectory: 'src' }),
    viteReact(),
    nitro(),
  ],
})
```

### tsconfig.json
- Uses `"moduleResolution": "Bundler"`, `"jsx": "react-jsx"`, `"target": "ES2022"`
- Path alias: `"~/*": ["./src/*"]`

## Core Dependencies
- `@tanstack/react-start` - The framework
- `@tanstack/react-router` - File-based routing
- `@tanstack/react-router-devtools` - Dev tools
- `react`, `react-dom` - React 19+
- `zod` - Schema validation
- `nitro` - Server runtime (dev dependency)
- `vite` - Build tool (dev dependency)
- `tailwindcss`, `@tailwindcss/vite` - Styling (dev dependency)

## Scripts
```json
{
  "dev": "vite dev",
  "build": "vite build && tsc --noEmit",
  "preview": "vite preview",
  "start": "node .output/server/index.mjs"
}
```

## Routing

### File-Based Routing
Routes live in `src/routes/`. The route tree is auto-generated in `src/routeTree.gen.ts` — never edit this file manually.

**File naming conventions:**
- `index.tsx` → `/`
- `about.tsx` → `/about`
- `posts.tsx` → Layout wrapper for `/posts/*`
- `posts.index.tsx` → `/posts`
- `posts.$postId.tsx` → `/posts/:postId` (dynamic segment)
- `posts_.$postId.deep.tsx` → `/posts/:postId/deep` (flat route, escapes layout nesting)
- `_pathlessLayout.tsx` → Pathless layout (wraps children without adding a URL segment)
- `__root.tsx` → Root layout (wraps entire app)
- `api/` directory → API routes

### Creating a Route
```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/my-route')({
  component: MyComponent,
  // Optional:
  loader: async () => { /* fetch data */ },
  errorComponent: MyErrorComponent,
  pendingComponent: () => <div>Loading...</div>,
  notFoundComponent: () => <div>Not found</div>,
})

function MyComponent() {
  const data = Route.useLoaderData()
  return <div>{/* ... */}</div>
}
```

### Root Route (__root.tsx)
The root route defines the HTML shell, head content, global styles, and navigation:
```tsx
import { HeadContent, Link, Scripts, createRootRoute } from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
    ],
    links: [
      { rel: 'stylesheet', href: appCss },
    ],
  }),
  errorComponent: DefaultCatchBoundary,
  notFoundComponent: () => <NotFound />,
  shellComponent: RootDocument,
})

function RootDocument({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>
        {/* Navigation */}
        {children}
        <Scripts />
      </body>
    </html>
  )
}
```

### Dynamic Route Parameters
```tsx
// File: posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    return fetchPost(params.postId)
  },
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()
  return <div>{post.title}</div>
}
```

### Layout Routes
A layout route wraps child routes without adding a URL segment when using pathless layouts:
```tsx
// File: posts.tsx — wraps /posts/* routes
export const Route = createFileRoute('/posts')({
  component: PostsLayout,
})

function PostsLayout() {
  return (
    <div>
      <h2>Posts</h2>
      <Outlet />
    </div>
  )
}
```

## Navigation

### Link Component
```tsx
import { Link } from '@tanstack/react-router'

<Link to="/posts" activeProps={{ className: 'font-bold' }}>Posts</Link>
<Link to="/posts/$postId" params={{ postId: '123' }}>Post 123</Link>
```

### Programmatic Navigation
```tsx
const navigate = useNavigate()
navigate({ to: '/posts/$postId', params: { postId: '123' } })
```

## Data Loading

### Loaders
Loaders run before the route renders. They run on the server during SSR and on the client during navigation.
```tsx
export const Route = createFileRoute('/posts')({
  loader: async () => {
    const posts = await fetchPosts()
    return { posts }
  },
  component: PostsComponent,
})

function PostsComponent() {
  const { posts } = Route.useLoaderData()
}
```

### Preloading
Default preload strategy is `'intent'` (preloads on hover/focus). Configure in router:
```tsx
const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
})
```

## Server Functions

Server functions run exclusively on the server and can be called from the client:
```tsx
import { createServerFn } from '@tanstack/react-start'

const getServerTime = createServerFn().handler(async () => {
  return new Date().toISOString()
})

// Use in a component or loader
const time = await getServerTime()
```

### With Validation (Zod)
```tsx
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'

const createPost = createServerFn()
  .validator(z.object({
    title: z.string(),
    body: z.string(),
  }))
  .handler(async ({ data }) => {
    // data is typed { title: string, body: string }
    return db.posts.create(data)
  })

// Call it
await createPost({ data: { title: 'Hello', body: 'World' } })
```

## API Routes

API routes live in `src/routes/api/`:
```tsx
// File: src/routes/api/posts.ts
import { json } from '@tanstack/react-start'
import { createAPIFileRoute } from '@tanstack/react-start/api'

export const APIRoute = createAPIFileRoute('/api/posts')({
  GET: async () => {
    const posts = await fetchPosts()
    return json(posts)
  },
  POST: async ({ request }) => {
    const body = await request.json()
    const post = await createPost(body)
    return json(post, { status: 201 })
  },
})
```

## Head Management / SEO

Managed via the `head` function in route definitions:
```tsx
export const Route = createFileRoute('/about')({
  head: () => ({
    meta: [
      { title: 'About Us' },
      { name: 'description', content: 'Learn about us' },
      { name: 'og:title', content: 'About Us' },
    ],
    links: [
      { rel: 'canonical', href: 'https://example.com/about' },
    ],
  }),
  component: AboutPage,
})
```

## Error Handling

### Error Boundaries
```tsx
export const Route = createFileRoute('/risky')({
  errorComponent: ({ error }) => (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
    </div>
  ),
  component: RiskyComponent,
})
```

### Global Error/NotFound
Set defaults in the router:
```tsx
const router = createRouter({
  routeTree,
  defaultErrorComponent: DefaultCatchBoundary,
  defaultNotFoundComponent: () => <NotFound />,
})
```

## Middleware

Middleware can be added to routes for auth, logging, etc.:
```tsx
import { createMiddleware } from '@tanstack/react-start'

const authMiddleware = createMiddleware().server(async ({ next }) => {
  const user = await getUser()
  if (!user) throw redirect({ to: '/login' })
  return next({ context: { user } })
})

export const Route = createFileRoute('/dashboard')({
  beforeLoad: authMiddleware,
  component: DashboardComponent,
})
```

## Redirect

```tsx
import { redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/old-page')({
  beforeLoad: () => {
    throw redirect({ to: '/new-page' })
  },
})
```

## Search Params

```tsx
import { z } from 'zod'

const searchSchema = z.object({
  page: z.number().default(1),
  filter: z.string().optional(),
})

export const Route = createFileRoute('/posts')({
  validateSearch: searchSchema,
  component: PostsComponent,
})

function PostsComponent() {
  const { page, filter } = Route.useSearch()
  const navigate = useNavigate()

  // Update search params
  navigate({ search: { page: 2 } })
}
```

## Deferred Data / Streaming

```tsx
import { defer } from '@tanstack/react-router'

export const Route = createFileRoute('/deferred')({
  loader: async () => {
    const criticalData = await fetchCritical()
    const slowDataPromise = fetchSlowData() // Don't await

    return {
      criticalData,
      slowData: defer(slowDataPromise),
    }
  },
  component: DeferredComponent,
})

function DeferredComponent() {
  const { criticalData, slowData } = Route.useLoaderData()
  return (
    <div>
      <div>{criticalData}</div>
      <Suspense fallback={<div>Loading slow data...</div>}>
        <Await promise={slowData}>
          {(data) => <div>{data}</div>}
        </Await>
      </Suspense>
    </div>
  )
}
```

## Deployment

### Build
```bash
npm run build
```

### Start Production Server
```bash
npm run start
# Runs: node .output/server/index.mjs
```

### Deployment Targets
Nitro supports many deployment targets. Configure in vite.config.ts:
```ts
nitro({
  preset: 'node-server',    // Default
  // preset: 'vercel',
  // preset: 'netlify',
  // preset: 'cloudflare-pages',
})
```

## Node.js Requirement
TanStack Start requires **Node.js >= 22.12.0**.

## Common Patterns

### Protected Routes
```tsx
const authMiddleware = createMiddleware().server(async ({ next }) => {
  const session = await getSession()
  if (!session) throw redirect({ to: '/login' })
  return next({ context: { session } })
})
```

### Data Fetching with Server Functions
```tsx
const getPosts = createServerFn().handler(async () => {
  return db.posts.findMany()
})

export const Route = createFileRoute('/posts')({
  loader: () => getPosts(),
  component: PostsPage,
})
```

### Form Handling
```tsx
const createPost = createServerFn()
  .validator(z.object({ title: z.string(), body: z.string() }))
  .handler(async ({ data }) => {
    await db.posts.create({ data })
    throw redirect({ to: '/posts' })
  })
```
