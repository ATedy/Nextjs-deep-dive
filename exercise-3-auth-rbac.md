# Exercise 3: Secure Authentication & Role-Based Authorization

**Goal**: Build enterprise-grade authentication with JWT tokens, role-based access control (RBAC), and secure API routes.

**Concepts**: Middleware, JWT, Server Actions, secure cookies, role-based access.

---

## Step 1: Authentication Library Setup

**File: `lib/auth.ts`** - Core auth functions

```typescript
import { jwtVerify, SignJWT } from 'jose';
import { cookies } from 'next/headers';
import { db } from './db';

const secret = new TextEncoder().encode(process.env.JWT_SECRET || 'dev-secret');

// Types
export interface AuthPayload {
  userId: string;
  email: string;
  role: 'user' | 'admin' | 'moderator';
  iat: number;
  exp: number;
}

// Generate JWT token
export async function generateToken(
  userId: string,
  email: string,
  role: string,
  expiresIn: string = '7d'
) {
  const iat = Math.floor(Date.now() / 1000);
  const exp = iat + (getDaysInSeconds(expiresIn) || 7 * 24 * 60 * 60);

  const token = await new SignJWT({ userId, email, role })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt(iat)
    .setExpirationTime(exp)
    .sign(secret);

  return token;
}

// Verify JWT token
export async function verifyToken(token: string): Promise<AuthPayload | null> {
  try {
    const { payload } = await jwtVerify(token, secret);
    return payload as AuthPayload;
  } catch {
    return null;
  }
}

// Helper: parse duration string to seconds
function getDaysInSeconds(duration: string): number {
  const match = duration.match(/^(\d+)([dhms])$/);
  if (!match) return 0;
  const [, value, unit] = match;
  const num = parseInt(value);
  const multipliers: Record<string, number> = {
    d: 24 * 60 * 60,
    h: 60 * 60,
    m: 60,
    s: 1,
  };
  return num * (multipliers[unit] || 0);
}

// Get session from cookies
export async function getSession(): Promise<AuthPayload | null> {
  const cookieStore = await cookies();
  const token = cookieStore.get('auth_token')?.value;

  if (!token) return null;

  return verifyToken(token);
}

// Set auth cookie
export async function setAuthCookie(token: string, expiresIn: number) {
  const cookieStore = await cookies();
  cookieStore.set({
    name: 'auth_token',
    value: token,
    httpOnly: true, // JS cannot access (prevents XSS)
    secure: process.env.NODE_ENV === 'production', // HTTPS only
    sameSite: 'lax', // CSRF protection
    maxAge: expiresIn,
  });
}

// Logout
export async function logout() {
  const cookieStore = await cookies();
  cookieStore.delete('auth_token');
}

// Check authorization by role
export function requireRole(role: 'user' | 'admin' | 'moderator') {
  return async (fn: Function) => {
    const session = await getSession();
    if (!session) {
      throw new Error('Unauthorized: No session');
    }
    if (session.role !== role && session.role !== 'admin') {
      throw new Error(`Forbidden: Requires ${role} role`);
    }
    return fn();
  };
}
```

---

## Step 2: User Schema & Prisma Model

**File: `prisma/schema.prisma`** (update User model)

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  password  String   // hashed with bcrypt
  
  // Authorization
  role      String   @default("user") // 'user', 'admin', 'moderator'
  isActive  Boolean  @default(true)
  
  // Tracking
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  lastLoginAt DateTime?

  // Relations
  posts     BlogPost[]
  apiKeys   ApiKey[]

  @@index([email])
  @@index([role])
}

model ApiKey {
  id        String   @id @default(cuid())
  key       String   @unique // hashed
  name      String
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  permissions String[] // ['read:posts', 'write:posts', etc.]
  rateLimit  Int      @default(100) // requests per minute
  
  lastUsedAt DateTime?
  expiresAt  DateTime?
  createdAt  DateTime @default(now())

  @@index([userId])
}

model BlogPost {
  id        String   @id @default(cuid())
  title     String
  content   String   @db.Text
  published Boolean  @default(false)
  
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([authorId])
}
```

```bash
npx prisma migrate dev --name add_auth_rbac
```

---

## Step 3: Sign-Up & Login APIs

**File: `app/api/auth/signup/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { hash } from 'bcryptjs';
import { db } from '@/lib/db';
import { generateToken, setAuthCookie } from '@/lib/auth';

export async function POST(req: NextRequest) {
  try {
    const { email, password, name } = await req.json();

    // Validate input
    if (!email || !password || password.length < 8) {
      return NextResponse.json(
        { error: 'Invalid email or password (min 8 chars)' },
        { status: 400 }
      );
    }

    // Check if user exists
    const existing = await db.user.findUnique({ where: { email } });
    if (existing) {
      return NextResponse.json(
        { error: 'Email already registered' },
        { status: 409 }
      );
    }

    // Hash password
    const passwordHash = await hash(password, 12);

    // Create user
    const user = await db.user.create({
      data: {
        email,
        name: name || email.split('@')[0],
        password: passwordHash,
        role: 'user', // Default role
      },
    });

    // Generate token (7 days)
    const token = await generateToken(user.id, user.email, user.role, '7d');
    const expiresIn = 7 * 24 * 60 * 60; // 7 days in seconds

    // Set secure cookie
    await setAuthCookie(token, expiresIn);

    return NextResponse.json(
      {
        message: 'Account created',
        user: {
          id: user.id,
          email: user.email,
          role: user.role,
        },
      },
      { status: 201 }
    );
  } catch (error) {
    console.error('Signup error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

**File: `app/api/auth/login/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { compare } from 'bcryptjs';
import { db } from '@/lib/db';
import { generateToken, setAuthCookie } from '@/lib/auth';

export async function POST(req: NextRequest) {
  try {
    const { email, password } = await req.json();

    // Find user
    const user = await db.user.findUnique({ where: { email } });
    if (!user || !user.isActive) {
      return NextResponse.json(
        { error: 'Invalid credentials' },
        { status: 401 }
      );
    }

    // Verify password
    const passwordValid = await compare(password, user.password);
    if (!passwordValid) {
      return NextResponse.json(
        { error: 'Invalid credentials' },
        { status: 401 }
      );
    }

    // Update last login
    await db.user.update({
      where: { id: user.id },
      data: { lastLoginAt: new Date() },
    });

    // Generate token
    const token = await generateToken(user.id, user.email, user.role, '7d');
    const expiresIn = 7 * 24 * 60 * 60;

    await setAuthCookie(token, expiresIn);

    return NextResponse.json({
      message: 'Logged in',
      user: {
        id: user.id,
        email: user.email,
        role: user.role,
      },
    });
  } catch (error) {
    console.error('Login error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

---

## Step 4: Protected API Routes (Admin Only)

**File: `app/api/admin/users/route.ts`** - List all users (admin only)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getSession } from '@/lib/auth';
import { db } from '@/lib/db';

export async function GET(req: NextRequest) {
  // Check auth
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Check role
  if (session.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden: Admin only' }, { status: 403 });
  }

  // Fetch users (excluding passwords)
  const users = await db.user.findMany({
    select: {
      id: true,
      email: true,
      name: true,
      role: true,
      isActive: true,
      lastLoginAt: true,
      createdAt: true,
    },
    orderBy: { createdAt: 'desc' },
  });

  return NextResponse.json({ users });
}
```

**File: `app/api/admin/users/[id]/role/route.ts`** - Update user role (admin only)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getSession } from '@/lib/auth';
import { db } from '@/lib/db';

export async function PATCH(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  if (session.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  const { role } = await req.json();
  if (!['user', 'admin', 'moderator'].includes(role)) {
    return NextResponse.json({ error: 'Invalid role' }, { status: 400 });
  }

  const user = await db.user.update({
    where: { id: params.id },
    data: { role },
    select: {
      id: true,
      email: true,
      role: true,
    },
  });

  return NextResponse.json(user);
}
```

---

## Step 5: Middleware/Proxy for Auth Checks

**File: `proxy.ts`** - Network boundary (replaces middleware.ts in Next.js 16)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { jwtVerify } from 'jose';

const secret = new TextEncoder().encode(process.env.JWT_SECRET || 'dev-secret');

export async function proxy(request: NextRequest) {
  const path = request.nextUrl.pathname;

  // 1. Public routes (no auth needed)
  if (path === '/' || path === '/login' || path === '/signup') {
    return NextResponse.next();
  }

  // 2. Admin routes
  if (path.startsWith('/api/admin') || path.startsWith('/admin')) {
    const token = request.cookies.get('auth_token')?.value;

    if (!token) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    try {
      const { payload } = await jwtVerify(token, secret);
      
      if (payload.role !== 'admin') {
        return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
      }
    } catch {
      return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
    }
  }

  // 3. Protected routes (any authenticated user)
  if (path.startsWith('/dashboard') || path.startsWith('/api/user')) {
    const token = request.cookies.get('auth_token')?.value;

    if (!token) {
      if (request.nextUrl.pathname.startsWith('/api')) {
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
      }
      return NextResponse.redirect(new URL('/login', request.url));
    }

    try {
      await jwtVerify(token, secret);
    } catch {
      if (request.nextUrl.pathname.startsWith('/api')) {
        return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
      }
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // 4. Rate limiting (basic)
  const ip = request.ip || 'unknown';
  const key = `rate:${ip}:${Date.now() / 60000}`;
  
  // In production: use Redis
  // For now: just log
  console.log(`[${new Date().toISOString()}] ${request.method} ${path} from ${ip}`);

  return NextResponse.next();
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

---

## Step 6: Protected UI Component

**File: `components/auth/ProtectedRoute.tsx`** - Client-side guard

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';

interface AuthPayload {
  userId: string;
  email: string;
  role: 'user' | 'admin' | 'moderator';
}

interface Props {
  children: React.ReactNode;
  requiredRole?: 'user' | 'admin' | 'moderator';
}

export function ProtectedRoute({ children, requiredRole }: Props) {
  const router = useRouter();
  const [session, setSession] = useState<AuthPayload | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Fetch session from API
    const checkAuth = async () => {
      try {
        const res = await fetch('/api/auth/me');
        if (res.ok) {
          const data = await res.json();
          setSession(data);

          // Check role
          if (requiredRole && data.role !== requiredRole && data.role !== 'admin') {
            router.push('/unauthorized');
          }
        } else {
          router.push('/login');
        }
      } catch {
        router.push('/login');
      } finally {
        setLoading(false);
      }
    };

    checkAuth();
  }, [router, requiredRole]);

  if (loading) return <div className="text-center py-12">Loading...</div>;
  if (!session) return null;

  return <>{children}</>;
}
```

---

## Step 7: Create Admin Panel

**File: `app/(admin)/admin/users/page.tsx`**

```typescript
import { getSession } from '@/lib/auth';
import { redirect } from 'next/navigation';
import UsersList from '@/components/admin/UsersList';

export default async function AdminUsersPage() {
  const session = await getSession();

  if (!session) {
    redirect('/login');
  }

  if (session.role !== 'admin') {
    throw new Error('Forbidden');
  }

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">User Management</h1>
      <UsersList />
    </div>
  );
}
```

---

## Step 8: Test Authentication

**Sign up**:
```bash
curl -X POST http://localhost:3000/api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "SecurePass123", "name": "John Doe"}'
```

**Login**:
```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"email": "user@example.com", "password": "SecurePass123"}'
```

**Access protected route** (with auth cookie):
```bash
curl -X GET http://localhost:3000/api/admin/users \
  -b cookies.txt
```

---

## What You Learned

✅ **JWT authentication** (secure tokens)  
✅ **Password hashing** (bcryptjs)  
✅ **Secure cookies** (httpOnly, sameSite)  
✅ **Role-based access control** (RBAC)  
✅ **Middleware/Proxy** (auth enforcement)  
✅ **Protected API routes** (admin endpoints)  
✅ **Protected UI** (client-side guards)  

---

## Next Challenge

**Add**:
- Multi-factor authentication (2FA)
- OAuth/Google Sign-In
- Session management dashboard (revoke tokens)
- Audit logging (track admin actions)
- IP whitelisting for admin routes
