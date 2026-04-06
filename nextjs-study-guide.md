# Next.js 16 Enterprise Bootcamp
## Complete Study Guide

---

## 📚 What You Now Know

This bootcamp covered **production-grade fullstack Next.js 16 development**:

### ✅ Core Concepts
- **React Server Components (RSCs)** – render on server by default, zero JS
- **Client Components** – interactive UI in browser with `'use client'`
- **App Router** – file-based routing, nested layouts, route groups
- **Four rendering strategies** – SSG, SSR, ISR, CSR
- **Streaming & Suspense** – load pages fast, hydrate progressively
- **Caching hierarchy** – browser, edge, Next.js data cache, request memoization
- **Data fetching** – direct DB access on server, fetch with `next:` options

### ✅ Architecture Patterns
- **API Routes** (Route Handlers) – `/app/api/*/route.ts`
- **Middleware/Proxy** – network boundary, auth enforcement
- **Protected routes** – server-side auth checks, RBAC
- **Database integration** – Prisma ORM, migrations, schema design
- **Dynamic routes** – `[slug]`, `[id]`, `generateStaticParams()`, `generateMetadata()`

### ✅ Enterprise Features
- **ISR (Incremental Static Regeneration)** – pre-generate + on-demand revalidation
- **On-demand revalidation** – `revalidatePath()`, `revalidateTag()`
- **JWT authentication** – secure tokens, httpOnly cookies
- **Role-Based Access Control (RBAC)** – middleware enforcement
- **Error handling** – error.tsx, not-found.tsx, error boundaries
- **Performance optimization** – Turbopack, image optimization, code splitting

---

## 🎯 Next.js 16 Cheat Sheet

### File Structure & Routes

```
app/
├── page.tsx                    → GET /
├── layout.tsx                  → Root layout
├── error.tsx                   → Error boundary
├── not-found.tsx              → 404 handler
│
├── (auth)/                     → Route group (no URL segment)
│   ├── login/page.tsx         → GET /login
│   └── signup/page.tsx        → GET /signup
│
├── (protected)/                → Protected section
│   ├── layout.tsx             → Auth guard wrapper
│   ├── dashboard/page.tsx     → GET /dashboard
│   └── [workspace]/
│       └── page.tsx           → GET /[workspace]
│
├── api/
│   ├── auth/
│   │   ├── route.ts           → POST /api/auth
│   │   └── logout/route.ts    → POST /api/auth/logout
│   └── users/
│       └── [id]/route.ts      → GET/PUT /api/users/:id
│
└── middleware.ts → proxy.ts    → Network boundary (Next.js 16)
```

---

## 🔧 Quick Code Snippets

### Server Component with Data Fetching

```typescript
// App/products/page.tsx - This is a Server Component by default
import { db } from '@/lib/db';

export default async function ProductsPage() {
  // Direct database access (zero bundle impact)
  const products = await db.product.findMany();

  return (
    <div>
      {products.map((p) => (
        <div key={p.id}>{p.name}</div>
      ))}
    </div>
  );
}
```

### Client Component for Interactivity

```typescript
// 'use client'; marks this as a Client Component
'use client';

import { useState } from 'react';

export default function CartButton() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Add to cart: {count}
    </button>
  );
}
```

### API Route Handler

```typescript
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function GET(req: NextRequest) {
  const products = await db.product.findMany();
  return NextResponse.json({ products });
}

export async function POST(req: NextRequest) {
  const { name, price } = await req.json();
  const product = await db.product.create({
    data: { name, price },
  });
  return NextResponse.json(product, { status: 201 });
}
```

### Dynamic Route with ISR

```typescript
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  // Pre-generate for top 100 products
  const products = await db.product.findMany({ take: 100 });
  return products.map((p) => ({ id: p.id }));
}

export async function generateMetadata({ params }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  return {
    title: product?.name,
    description: product?.description,
  };
}

export default async function ProductPage({ params }) {
  const product = await db.product.findUnique({
    where: { id: params.id },
  });

  if (!product) notFound();

  return <div>{product.name}</div>;
}

export const revalidate = 3600; // ISR: revalidate every hour
```

### On-Demand Revalidation

```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(req) {
  const { path, tag } = await req.json();

  if (path) revalidatePath(path);
  if (tag) revalidateTag(tag);

  return Response.json({ revalidated: true });
}
```

### Streaming with Suspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

function Skeleton() {
  return <div className="h-48 bg-gray-200 animate-pulse" />;
}

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<Skeleton />}>
        <DataComponent /> {/* Async component */}
      </Suspense>
    </div>
  );
}

async function DataComponent() {
  // This is async; Suspense shows skeleton while loading
  const data = await fetch('...').then(r => r.json());
  return <div>{data}</div>;
}
```

### Protected Route with Auth

```typescript
// app/(protected)/layout.tsx
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth';

export default async function ProtectedLayout({ children }) {
  const session = await getSession();
  
  if (!session) {
    redirect('/login');
  }

  return <>{children}</>;
}
```

### Authentication with JWT

```typescript
// lib/auth.ts
import { SignJWT, jwtVerify } from 'jose';
import { cookies } from 'next/headers';

const secret = new TextEncoder().encode(process.env.JWT_SECRET);

export async function generateToken(userId: string) {
  const token = await new SignJWT({ userId })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('7d')
    .sign(secret);
  
  const c = await cookies();
  c.set('token', token, { httpOnly: true, secure: true });
}

export async function verifyToken() {
  const c = await cookies();
  const token = c.get('token')?.value;
  if (!token) return null;

  try {
    const verified = await jwtVerify(token, secret);
    return verified.payload;
  } catch {
    return null;
  }
}
```

### Middleware (proxy.ts in Next.js 16)

```typescript
// proxy.ts
import { NextRequest, NextResponse } from 'next/server';

export function proxy(request: NextRequest) {
  const path = request.nextUrl.pathname;

  // Check auth for protected routes
  if (path.startsWith('/dashboard')) {
    const token = request.cookies.get('token');
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image).*?)'],
};
```

---

## 📊 Rendering Strategy Decision Tree

```
Does your page content change frequently (every minute)?
  ├─ YES → SSR (Server-Side Rendering)
  │  └─ Render on every request, always fresh
  │  └─ Use for: User dashboards, real-time data
  │
  └─ NO, changes weekly/monthly?
     ├─ YES, but some routes need pre-generation → ISR
     │  └─ Pre-generate at build, revalidate on-demand
     │  └─ Use for: Product pages, blog posts
     │
     └─ NO, static content → SSG
        └─ Generate once at build-time, serve from CDN
        └─ Use for: Marketing pages, docs
```

---

## 🔐 Security Checklist

- [ ] JWT tokens in httpOnly cookies (not localStorage)
- [ ] CSRF protection via sameSite cookies
- [ ] XSS prevention: Content Security Policy headers
- [ ] SQL injection: Use parameterized queries (Prisma ORM handles this)
- [ ] CORS: Configure proper origins in headers
- [ ] Authentication checks in middleware/proxy.ts
- [ ] Rate limiting on API routes
- [ ] Environment variables (.env.local, never committed)
- [ ] Validate/sanitize all user input
- [ ] HTTPS only in production (secure cookie flag)

---

## ⚡ Performance Best Practices

### 1. Image Optimization
```typescript
import Image from 'next/image';

// Automatically optimized: lazy load, responsive, modern formats
<Image
  src="/product.jpg"
  alt="Product"
  width={300}
  height={300}
  priority // Only for above-the-fold
/>
```

### 2. Code Splitting
```typescript
// Automatic: Route-level code splitting
// Each route only loads its own JS

// Manual: Dynamic imports for large libraries
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('@/components/Heavy'), {
  loading: () => <div>Loading...</div>,
});
```

### 3. Caching Strategy
```typescript
// fetch() with caching options
const data = await fetch('https://api.example.com/data', {
  next: { 
    revalidate: 3600, // Cache for 1 hour
    tags: ['data'], // For on-demand revalidation
  },
});

// Or use 'use cache' directive (Next.js 16)
'use cache';
export async function getExpensiveData() {
  return await db.query(...);
}
```

### 4. Compression
```typescript
// next.config.ts
const nextConfig = {
  compress: true, // Gzip compression
};
```

### 5. Monitoring
```typescript
// Use Web Vitals to track performance
import { reportWebVitals } from 'next/web-vitals';

reportWebVitals((metric) => {
  console.log(metric);
  // Send to analytics
});
```

---

## 🚀 Deployment Checklist

### Before Deploying
- [ ] All environment variables set (.env.production)
- [ ] Build succeeds: `npm run build`
- [ ] No TypeScript errors: `npx tsc --noEmit`
- [ ] Database migrations applied
- [ ] API keys and secrets in environment (not in code)
- [ ] CORS headers configured
- [ ] Image domains configured in `next.config.ts`

### Vercel Deployment (Recommended)
```bash
npm install -g vercel
vercel login
vercel
# Follow prompts, set env vars via dashboard
```

### Self-Hosted (AWS, DigitalOcean, etc.)
```bash
npm run build
npm start
# Or: NODE_ENV=production node .next/standalone/server.js
```

### Docker Deployment
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
ENV NODE_ENV=production
CMD ["npm", "start"]
EXPOSE 3000
```

---

## 🎓 Questions & Answers

### Q: What's the difference between Server and Client Components?

**A**: Server Components render on the server, sending HTML to the browser. They can access databases and secrets directly. Client Components render in the browser and handle interactivity. By default, all components are Server Components unless marked with `'use client'`.

---

### Q: How does ISR work?

**A**: ISR combines static generation with background revalidation. At build time, you generate routes with `generateStaticParams()`. The page is served from cache until the revalidation period expires. After expiration, the page regenerates in the background while serving the stale version. You can also trigger revalidation on-demand with `revalidatePath()` or `revalidateTag()`.

---

### Q: What's the benefit of Server Components over traditional APIs?

**A**: Server Components eliminate client-server waterfalls. You can fetch data on the server and render directly in your component without an extra round-trip. The rendered HTML is sent to the client, reducing JavaScript bundle size. No N+1 query problems because you control exactly what's fetched.

---

### Q: How do you handle authentication in Next.js?

**A**: Use JWT tokens stored in secure httpOnly cookies. On the server, verify tokens in middleware (proxy.ts) and check `getSession()` in protected routes/API handlers. On the client, use Client Components to access session context for UI changes. Never trust client-side auth alone.

---

### Q: What's the difference between `revalidate` and `revalidateTag()`?

**A**: `revalidate` is a time-based strategy (e.g., revalidate every 1 hour). `revalidateTag()` is event-based—you trigger revalidation manually when data changes. Use revalidate for slowly-changing content (blogs), and revalidateTag for frequently-updated content (product inventory).

---

### Q: How do you optimize images in Next.js?

**A**: Use the `<Image>` component instead of `<img>`. It automatically:
- Optimizes formats (WebP, AVIF)
- Lazy-loads below the fold
- Resizes for different viewports
- Prevents layout shift (width/height required)

---

### Q: What's streaming and why use it?

**A**: Streaming sends HTML chunks to the browser as they're ready, instead of waiting for the entire page to render. Combined with Suspense, you can show skeleton loaders for slow components while fast data renders immediately. This improves perceived performance and Time to First Byte (TTFB).

---

## 📈 From This Bootcamp to Your First Job

### What Employers Want to See

1. **Real Projects**: Blog, e-commerce, SaaS dashboard (add to GitHub, deploy live)
2. **Database Integration**: Show you can use Prisma, migrations, schemas
3. **Authentication**: Implement login/signup with tokens, sessions
4. **Performance**: Demonstrate caching, ISR, image optimization
5. **Testing**: Unit tests (Jest), integration tests (Playwright)
6. **Deployment**: Show your app running on Vercel or AWS

### Project Ideas for Portfolio

✅ **Tier 1 (Beginner)**
- Blog with ISR (this bootcamp Exercise 1)
- Portfolio site with dark mode, Mdx blog posts
- Contact form with email sending (Resend, SendGrid)

✅ **Tier 2 (Intermediate)**
- SaaS dashboard with real-time data (Exercise 2)
- E-commerce product listing (filters, search, cart)
- Community forum with authentication

✅ **Tier 3 (Advanced)**
- Multi-tenant SaaS with workspaces (Asli Store, StayEase level)
- Real-time collaboration app (WebSocket)
- Admin panel with advanced RBAC

---

## 🔗 Resources & Documentation

- **Next.js Docs**: https://nextjs.org/docs
- **React Docs**: https://react.dev
- **Prisma Docs**: https://www.prisma.io/docs
- **Tailwind CSS**: https://tailwindcss.com/docs
- **Next Auth.js**: https://next-auth.js.org
- **Vercel Docs**: https://vercel.com/docs

---

## ✨ Summary: Your Next.js Superpower

You now understand:
- How to structure fullstack apps with Next.js
- When to use Server vs Client Components
- How to fetch data efficiently without N+1 problems
- How to build secure authentication systems
- How to optimize performance with caching and streaming
- How to deploy to production

**Next step**: Build a real project. Start small (blog), deploy it, iterate.


## Commit This to Memory

```
SSR:   On-demand rendering          (Fresh data, slower response)
SSG:   Build-time rendering         (Static pages, CDN cached)
ISR:   SSG + Background revalidate  (Best of both)
CSR:   Browser rendering            (Heavy JS, slowest TTI)

Server Components: Render on server (zero JS, direct DB access)
Client Components: Render in browser (interactive, use hooks)

Auth Flow:
  Login → JWT Token → httpOnly Cookie → Middleware checks
  → Verified → Access protected resource

Performance:
  Stream data → Show skeleton → Hydrate → Interactive
```

Good luck! You're ready. 🚀
