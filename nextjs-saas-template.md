# Next.js 16 Fullstack SaaS Architecture Template

## Project Structure

```
saas-app/
├── app/
│   ├── layout.tsx                      # Root layout, fonts, providers
│   ├── page.tsx                        # Landing page
│   ├── (auth)/
│   │   ├── login/page.tsx             # Login form
│   │   ├── signup/page.tsx            # Signup form
│   │   └── forgot-password/page.tsx   # Password reset
│   ├── (protected)/
│   │   ├── layout.tsx                 # Auth guard layout
│   │   ├── dashboard/page.tsx         # User dashboard
│   │   ├── settings/page.tsx          # User settings
│   │   └── [workspace]/
│   │       └── page.tsx               # Workspace detail
│   ├── api/
│   │   ├── auth/
│   │   │   ├── route.ts               # Auth API (signup, login)
│   │   │   ├── verify-email/route.ts  # Email verification
│   │   │   └── logout/route.ts        # Logout
│   │   ├── users/
│   │   │   └── [id]/route.ts          # User CRUD
│   │   ├── workspaces/
│   │   │   └── route.ts               # Workspace CRUD
│   │   └── webhooks/
│   │       └── stripe/route.ts        # Stripe webhook
│   └── error.tsx                      # Error boundary
├── lib/
│   ├── db.ts                          # Database client (Prisma)
│   ├── auth.ts                        # Authentication utils
│   ├── utils.ts                       # Helper functions
│   └── types.ts                       # TypeScript types
├── components/
│   ├── ui/                            # Reusable UI components
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   └── Card.tsx
│   ├── auth/
│   │   └── AuthGuard.tsx              # Client component for auth checks
│   └── dashboard/
│       ├── UserNav.tsx
│       └── WorkspaceSelector.tsx
├── middleware.ts → proxy.ts           # Next.js 16 network boundary
├── next.config.ts                     # Build configuration
├── prisma/
│   ├── schema.prisma                  # Database schema
│   └── migrations/                    # Database migrations
├── .env.local                         # Secrets (never commit)
├── tsconfig.json                      # TypeScript config
└── package.json
```

---

## 1. ROOT LAYOUT (App Shell)

**File: `app/layout.tsx`** - Wraps all pages, loads global styles, sets up providers

```typescript
import type { Metadata } from 'next';
import { Geist_Mono } from 'next/font/google';
import './globals.css';
import { SessionProvider } from 'next-auth/react'; // Or your auth library

const geistMono = Geist_Mono({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'SaaS App',
  description: 'Enterprise SaaS platform',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={geistMono.className}>
        <SessionProvider> {/* Auth provider wraps entire app */}
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

---

## 2. AUTHENTICATION ROUTE GROUP

**File: `app/(auth)/login/page.tsx`** - Server Component with embedded Client Component form

```typescript
'use server'; // Optional explicit server marker

import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth';
import LoginForm from '@/components/auth/LoginForm'; // Client component

export default async function LoginPage() {
  // Server-side: Check if already logged in
  const session = await getSession();
  if (session) {
    redirect('/dashboard');
  }

  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="w-full max-w-md">
        <h1 className="text-3xl font-bold mb-6">Log in</h1>
        <LoginForm /> {/* Client component handles form submission */}
      </div>
    </div>
  );
}
```

**File: `components/auth/LoginForm.tsx`** - Client Component

```typescript
'use client';

import { FormEvent, useState } from 'react';
import { useRouter } from 'next/navigation';

export default function LoginForm() {
  const router = useRouter();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    try {
      const res = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
      });

      if (!res.ok) {
        const { error: msg } = await res.json();
        setError(msg || 'Login failed');
        return;
      }

      // Success: redirect to dashboard
      router.push('/dashboard');
      router.refresh(); // Refresh server components
    } catch (err) {
      setError('Network error');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="email"
        placeholder="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        required
      />
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <button
        type="submit"
        disabled={loading}
        className="w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-700"
      >
        {loading ? 'Logging in...' : 'Log in'}
      </button>
    </form>
  );
}
```

---

## 3. API ROUTE (Route Handler)

**File: `app/api/auth/route.ts`** - Server-only, handles auth logic

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { hash, verify } from '@/lib/crypto';
import { db } from '@/lib/db';
import { createSession } from '@/lib/auth';

export async function POST(req: NextRequest) {
  try {
    const { email, password } = await req.json();

    // 1. Find user in database
    const user = await db.user.findUnique({ where: { email } });
    if (!user) {
      return NextResponse.json(
        { error: 'Invalid credentials' },
        { status: 401 }
      );
    }

    // 2. Verify password
    const passwordValid = await verify(password, user.passwordHash);
    if (!passwordValid) {
      return NextResponse.json(
        { error: 'Invalid credentials' },
        { status: 401 }
      );
    }

    // 3. Create session (store in database or JWT)
    const session = await createSession(user.id);

    // 4. Return session cookie + response
    const response = NextResponse.json(
      { message: 'Logged in', user: { id: user.id, email: user.email } },
      { status: 200 }
    );

    response.cookies.set({
      name: 'session',
      value: session.token,
      httpOnly: true, // Secure: cannot be accessed from JS
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 7, // 7 days
    });

    return response;
  } catch (error) {
    console.error('Auth error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

---

## 4. PROTECTED ROUTE WITH AUTH GUARD

**File: `app/(protected)/layout.tsx`** - Wraps protected routes, verifies auth on server

```typescript
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth';

export default async function ProtectedLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  // Server-side auth check: runs on every request
  const session = await getSession();
  if (!session) {
    redirect('/login');
  }

  return (
    <div className="flex">
      <aside className="w-64 bg-gray-900 text-white p-4">
        <nav className="space-y-2">
          <a href="/dashboard" className="block hover:bg-gray-800 p-2">
            Dashboard
          </a>
          <a href="/settings" className="block hover:bg-gray-800 p-2">
            Settings
          </a>
          <a href="/api/auth/logout" className="block hover:bg-gray-800 p-2">
            Logout
          </a>
        </nav>
      </aside>
      <main className="flex-1 p-6">{children}</main>
    </div>
  );
}
```

**File: `app/(protected)/dashboard/page.tsx`** - Server Component with live data

```typescript
import { getSession } from '@/lib/auth';
import { db } from '@/lib/db';

export default async function DashboardPage() {
  const session = await getSession();
  
  // Fetch user's workspaces (server-side, direct DB access)
  const workspaces = await db.workspace.findMany({
    where: { ownerId: session.userId },
    include: { _count: { select: { members: true } } },
  });

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Dashboard</h1>
      <div className="grid grid-cols-3 gap-4">
        {workspaces.map((workspace) => (
          <div key={workspace.id} className="border p-4 rounded">
            <h2 className="font-semibold">{workspace.name}</h2>
            <p className="text-gray-600">
              {workspace._count.members} members
            </p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 5. DYNAMIC ROUTES & PARAMETERS

**File: `app/(protected)/[workspace]/page.tsx`** - Dynamic segment

```typescript
import { notFound } from 'next/navigation';
import { getSession } from '@/lib/auth';
import { db } from '@/lib/db';

type Props = {
  params: { workspace: string };
  searchParams: { [key: string]: string | string[] };
};

export async function generateStaticParams() {
  // Pre-generate routes for top 100 workspaces (ISR)
  const workspaces = await db.workspace.findMany({ take: 100 });
  return workspaces.map((ws) => ({ workspace: ws.slug }));
}

export default async function WorkspacePage({ params }: Props) {
  const session = await getSession();
  
  // Find workspace by slug
  const workspace = await db.workspace.findUnique({
    where: { slug: params.workspace },
    include: { owner: true, members: true },
  });

  if (!workspace) {
    notFound();
  }

  // Check authorization
  const isMember = workspace.members.some((m) => m.id === session.userId) ||
    workspace.ownerId === session.userId;
  
  if (!isMember) {
    throw new Error('Unauthorized');
  }

  return (
    <div>
      <h1 className="text-3xl font-bold">{workspace.name}</h1>
      <p className="text-gray-600">Owner: {workspace.owner.email}</p>
      <p className="text-sm">Members: {workspace.members.length}</p>
    </div>
  );
}
```

---

## 6. CACHING & DATA REVALIDATION

**File: `lib/queries.ts`** - Cached queries with revalidation

```typescript
import { cache } from 'react';
import { db } from './db';

// Memoize expensive queries within a single request
export const getWorkspace = cache(async (id: string) => {
  return await db.workspace.findUnique({
    where: { id },
    include: { members: true, projects: true },
  });
});

// Server action: revalidate on-demand
'use server';
import { revalidateTag } from 'next/cache';

export async function updateWorkspaceName(id: string, name: string) {
  await db.workspace.update({
    where: { id },
    data: { name },
  });

  // Revalidate all routes that use this workspace
  revalidateTag(`workspace-${id}`);
}
```

**File: `app/(protected)/[workspace]/settings/page.tsx`** - Using revalidation

```typescript
'use client';

import { useState } from 'react';
import { updateWorkspaceName } from '@/lib/queries';

export default function WorkspaceSettingsPage({ workspace }: { workspace: any }) {
  const [name, setName] = useState(workspace.name);
  const [saving, setSaving] = useState(false);

  const handleSave = async () => {
    setSaving(true);
    await updateWorkspaceName(workspace.id, name);
    setSaving(false);
  };

  return (
    <form onSubmit={(e) => { e.preventDefault(); handleSave(); }}>
      <input
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <button type="submit" disabled={saving}>
        {saving ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

## 7. STREAMING & SUSPENSE

**File: `app/(protected)/dashboard/page.tsx`** - Load fast, stream data

```typescript
import { Suspense } from 'react';
import { getSession } from '@/lib/auth';
import WorkspaceList from '@/components/dashboard/WorkspaceList';

function WorkspaceSkeleton() {
  return <div className="bg-gray-200 h-48 rounded animate-pulse" />;
}

export default async function DashboardPage() {
  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Dashboard</h1>
      
      {/* Suspense boundary: show skeleton while loading */}
      <Suspense fallback={<WorkspaceSkeleton />}>
        <WorkspaceList />
      </Suspense>
    </div>
  );
}
```

**File: `components/dashboard/WorkspaceList.tsx`** - Async component (Server Component)

```typescript
import { db } from '@/lib/db';
import { getSession } from '@/lib/auth';

export default async function WorkspaceList() {
  // This runs in parallel with Suspense—page loads immediately
  const session = await getSession();
  const workspaces = await db.workspace.findMany({
    where: { ownerId: session.userId },
  });

  return (
    <div className="grid grid-cols-3 gap-4">
      {workspaces.map((ws) => (
        <div key={ws.id} className="border p-4 rounded">
          {ws.name}
        </div>
      ))}
    </div>
  );
}
```

---

## 8. MIDDLEWARE (PROXY) - Next.js 16

**File: `proxy.ts`** - Network boundary (replaces middleware.ts)

```typescript
import { NextRequest, NextResponse } from 'next/server';

export function proxy(request: NextRequest) {
  // 1. Check auth on protected routes
  const path = request.nextUrl.pathname;
  
  if (path.startsWith('/dashboard') || path.startsWith('/api')) {
    const session = request.cookies.get('session');
    
    if (!session) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // 2. Add custom headers
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-request-id', crypto.randomUUID());
  requestHeaders.set('x-user-id', request.cookies.get('session')?.value || '');

  // 3. Log requests (server-side)
  console.log(`${request.method} ${path}`);

  return NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  });
}

export const config = {
  matcher: [
    // Run proxy on all routes except static files
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

---

## 9. DATABASE INTEGRATION (Prisma)

**File: `prisma/schema.prisma`** - Database schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  passwordHash  String
  name          String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  workspaces    Workspace[]
  sessions      Session[]

  @@index([email])
}

model Session {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime
  createdAt DateTime @default(now())

  @@index([userId])
}

model Workspace {
  id        String   @id @default(cuid())
  name      String
  slug      String   @unique
  ownerId   String
  owner     User     @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  members   User[]   @relation("WorkspaceMembers")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  projects  Project[]

  @@index([ownerId])
  @@index([slug])
}

model Project {
  id          String    @id @default(cuid())
  name        String
  workspaceId String
  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([workspaceId])
}
```

**File: `lib/db.ts`** - Database client

```typescript
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const db =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: ['error'], // Only log errors in production
  });

if (process.env.NODE_ENV !== 'production')
  globalForPrisma.prisma = db;
```

---

## 10. ENVIRONMENT VARIABLES

**File: `.env.local`** (never commit)

```env
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/saas_db"

# Auth
JWT_SECRET="your-secret-key-min-32-chars"
SESSION_DURATION=604800

# External APIs
STRIPE_API_KEY="sk_live_..."
SENDGRID_API_KEY="SG..."

# App
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NODE_ENV="development"
```

**File: `lib/types.ts`** - TypeScript types

```typescript
export interface Session {
  userId: string;
  email: string;
  expiresAt: Date;
}

export interface User {
  id: string;
  email: string;
  name?: string;
}

export interface Workspace {
  id: string;
  name: string;
  slug: string;
  ownerId: string;
}
```

---

## 11. PRODUCTION DEPLOYMENT CONFIG

**File: `next.config.ts`**

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  typescript: {
    // Fail build on type errors
    tsconfigPath: './tsconfig.json',
  },
  
  // Caching strategies
  onDemandEntries: {
    maxInactiveAge: 25 * 1000, // Keep page in memory for 25s
    pagesBufferLength: 5,
  },

  // Image optimization
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
      },
    ],
  },

  // Environment variables
  env: {
    NEXT_PUBLIC_APP_VERSION: '1.0.0',
  },

  // Headers for security
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        {
          key: 'X-Content-Type-Options',
          value: 'nosniff',
        },
        {
          key: 'X-Frame-Options',
          value: 'DENY',
        },
      ],
    },
  ],
};

export default nextConfig;
```

---

## Quick Start Commands

```bash
# Initialize Next.js 16 with TypeScript
npx create-next-app@latest saas-app --typescript --tailwind

# Install dependencies
cd saas-app
npm install @prisma/client bcryptjs next-auth
npm install -D prisma

# Setup database
npx prisma init
# Edit .env.local with your DATABASE_URL
npx prisma migrate dev --name init

# Run dev server
npm run dev
# Open http://localhost:3000

# Build for production
npm run build
npm start
```
