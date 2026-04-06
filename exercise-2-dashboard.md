# Exercise 2: Build a Real-Time Analytics Dashboard

**Goal**: Create a dashboard that streams data to the client, shows loading states, and has interactive client components.

**Concepts**: Streaming, Suspense, Server Components + Client Components, real-time data, React hooks.

---

## Step 1: Database Schema

**File: `prisma/schema.prisma`** (add to existing)

```prisma
model Analytics {
  id        String   @id @default(cuid())
  eventType String   // 'page_view', 'click', 'signup', etc.
  userId    String?
  sessionId String
  metadata  Json     // { browser, device, country, etc. }
  timestamp DateTime @default(now())

  @@index([timestamp])
  @@index([eventType])
  @@index([sessionId])
}
```

```bash
npx prisma migrate dev --name add_analytics
```

---

## Step 2: Create Analytics Server Functions

**File: `lib/analytics.ts`** - Query functions with caching

```typescript
import { cache } from 'react';
import { db } from './db';

// Memoized query within a single request
export const getAnalyticsSummary = cache(async () => {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  // Get counts for today
  const [pageViews, signups, clicks] = await Promise.all([
    db.analytics.count({
      where: {
        eventType: 'page_view',
        timestamp: { gte: today },
      },
    }),
    db.analytics.count({
      where: {
        eventType: 'signup',
        timestamp: { gte: today },
      },
    }),
    db.analytics.count({
      where: {
        eventType: 'click',
        timestamp: { gte: today },
      },
    }),
  ]);

  return { pageViews, signups, clicks };
});

// Simulate slow database query for streaming demo
export const getChartData = cache(async () => {
  // In real app: aggregate from database
  // For demo: simulate with delay
  await new Promise((resolve) => setTimeout(resolve, 2000));

  const data = Array.from({ length: 12 }, (_, i) => ({
    month: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'][i],
    users: Math.floor(Math.random() * 5000) + 1000,
    revenue: Math.floor(Math.random() * 50000) + 10000,
  }));

  return data;
});

// Get top pages
export const getTopPages = cache(async () => {
  await new Promise((resolve) => setTimeout(resolve, 1500));

  return [
    { path: '/', views: 2450, conversionRate: 12.5 },
    { path: '/features', views: 1830, conversionRate: 9.2 },
    { path: '/pricing', views: 1200, conversionRate: 8.7 },
    { path: '/blog', views: 945, conversionRate: 5.3 },
  ];
});

// Track event (server action)
'use server';
export async function trackEvent(
  eventType: string,
  sessionId: string,
  metadata?: Record<string, any>
) {
  await db.analytics.create({
    data: {
      eventType,
      sessionId,
      metadata: metadata || {},
    },
  });
}
```

---

## Step 3: Create Summary Card Component (Server)

**File: `components/dashboard/SummaryCard.tsx`** - Server Component

```typescript
import { getAnalyticsSummary } from '@/lib/analytics';

type MetricType = 'pageViews' | 'signups' | 'clicks';

const icons: Record<MetricType, string> = {
  pageViews: '👁️',
  signups: '🎉',
  clicks: '🖱️',
};

const titles: Record<MetricType, string> = {
  pageViews: 'Page Views',
  signups: 'New Signups',
  clicks: 'Total Clicks',
};

export async function SummaryCard({ metric }: { metric: MetricType }) {
  const summary = await getAnalyticsSummary();

  const value = summary[metric];
  const previousValue = Math.floor(value * 0.85); // Mock previous day
  const change = (((value - previousValue) / previousValue) * 100).toFixed(1);

  return (
    <div className="bg-white border rounded-lg p-6">
      <div className="flex items-center justify-between">
        <div>
          <p className="text-gray-600 text-sm font-medium">{titles[metric]}</p>
          <p className="text-3xl font-bold mt-2">{value.toLocaleString()}</p>
          <p className={`text-sm mt-2 ${Number(change) > 0 ? 'text-green-600' : 'text-red-600'}`}>
            {change > 0 ? '+' : ''}{change}% vs yesterday
          </p>
        </div>
        <div className="text-4xl">{icons[metric]}</div>
      </div>
    </div>
  );
}
```

---

## Step 4: Chart Component (Client + Server)

**File: `components/dashboard/ChartCard.tsx`** - Client Component (interactive)

```typescript
'use client';

import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import { useState, useEffect } from 'react';

interface ChartData {
  month: string;
  users: number;
  revenue: number;
}

export function ChartCard({ data }: { data: ChartData[] }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) return <div className="h-96 bg-gray-100 rounded animate-pulse" />;

  return (
    <div className="bg-white border rounded-lg p-6">
      <h2 className="text-lg font-semibold mb-4">User Growth & Revenue</h2>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="month" />
          <YAxis yAxisId="left" />
          <YAxis yAxisId="right" orientation="right" />
          <Tooltip />
          <Legend />
          <Line yAxisId="left" type="monotone" dataKey="users" stroke="#3b82f6" name="Users" />
          <Line yAxisId="right" type="monotone" dataKey="revenue" stroke="#10b981" name="Revenue ($)" />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

**File: `components/dashboard/ChartCardWithSuspense.tsx`** - Wrapper with Suspense

```typescript
import { Suspense } from 'react';
import { getChartData } from '@/lib/analytics';
import { ChartCard } from './ChartCard';

function ChartSkeleton() {
  return <div className="bg-white border rounded-lg p-6 h-96 animate-pulse" />;
}

export function ChartCardWithSuspense() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <ChartCardContent />
    </Suspense>
  );
}

async function ChartCardContent() {
  const data = await getChartData();
  return <ChartCard data={data} />;
}
```

---

## Step 5: Top Pages Table (Server Component)

**File: `components/dashboard/TopPagesTable.tsx`** - Server Component

```typescript
import { getTopPages } from '@/lib/analytics';

export async function TopPagesTable() {
  const pages = await getTopPages();

  return (
    <div className="bg-white border rounded-lg overflow-hidden">
      <div className="px-6 py-4 border-b">
        <h2 className="text-lg font-semibold">Top Pages</h2>
      </div>
      <table className="w-full text-sm">
        <thead className="bg-gray-50 border-b">
          <tr>
            <th className="text-left px-6 py-3 font-medium text-gray-700">Page</th>
            <th className="text-right px-6 py-3 font-medium text-gray-700">Views</th>
            <th className="text-right px-6 py-3 font-medium text-gray-700">Conversion</th>
          </tr>
        </thead>
        <tbody>
          {pages.map((page) => (
            <tr key={page.path} className="border-b hover:bg-gray-50">
              <td className="px-6 py-3 font-mono text-blue-600">{page.path}</td>
              <td className="text-right px-6 py-3">{page.views.toLocaleString()}</td>
              <td className="text-right px-6 py-3">{page.conversionRate.toFixed(1)}%</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

## Step 6: Dashboard Page with Streaming

**File: `app/(protected)/analytics/page.tsx`** - Main dashboard

```typescript
import { Suspense } from 'react';
import { SummaryCard } from '@/components/dashboard/SummaryCard';
import { ChartCardWithSuspense } from '@/components/dashboard/ChartCardWithSuspense';
import { TopPagesTable } from '@/components/dashboard/TopPagesTable';

// Loading skeletons
function SummaryCardSkeleton() {
  return <div className="bg-white border rounded-lg p-6 h-32 animate-pulse" />;
}

function TopPagesSkeleton() {
  return <div className="bg-white border rounded-lg h-96 animate-pulse" />;
}

export default function AnalyticsDashboard() {
  return (
    <div className="space-y-8">
      <div>
        <h1 className="text-4xl font-bold">Analytics</h1>
        <p className="text-gray-600 mt-1">Real-time metrics and insights</p>
      </div>

      {/* Summary Cards - 3 columns with Suspense */}
      <div className="grid grid-cols-3 gap-6">
        <Suspense fallback={<SummaryCardSkeleton />}>
          <SummaryCard metric="pageViews" />
        </Suspense>

        <Suspense fallback={<SummaryCardSkeleton />}>
          <SummaryCard metric="signups" />
        </Suspense>

        <Suspense fallback={<SummaryCardSkeleton />}>
          <SummaryCard metric="clicks" />
        </Suspense>
      </div>

      {/* Chart - Streaming loads while other content renders */}
      <ChartCardWithSuspense />

      {/* Top Pages Table */}
      <Suspense fallback={<TopPagesSkeleton />}>
        <TopPagesTable />
      </Suspense>

      {/* Client Component: Real-time event tracker */}
      <EventTracker />
    </div>
  );
}

// Client Component: Real-time event tracking (hydrated in browser)
'use client';

import { useEffect, useState } from 'react';
import { trackEvent } from '@/lib/analytics';

function EventTracker() {
  const [events, setEvents] = useState(0);

  useEffect(() => {
    // Track page view on load
    const sessionId = crypto.randomUUID();
    trackEvent('page_view', sessionId, {
      page: '/analytics',
      timestamp: new Date().toISOString(),
    });

    // Track clicks
    const handleClick = () => {
      setEvents((e) => e + 1);
      trackEvent('click', sessionId, {
        page: '/analytics',
        timestamp: new Date().toISOString(),
      });
    };

    window.addEventListener('click', handleClick);
    return () => window.removeEventListener('click', handleClick);
  }, []);

  return (
    <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 text-sm">
      <p className="text-blue-900">
        📊 Events tracked this session: <strong>{events}</strong>
      </p>
    </div>
  );
}
```

---

## Step 7: Enable Streaming (Next.js Config)

**File: `next.config.ts`** (optional: streaming is default in App Router)

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Streaming is enabled by default in App Router
  // If using experimental.ppr in Next.js 15, remove it for Next.js 16
  experimental: {
    // Next.js 16 uses Cache Components instead
  },

  // Enable compression for faster streaming
  compress: true,

  // SWR (Stale-While-Revalidate) cache headers
  onDemandEntries: {
    maxInactiveAge: 25 * 1000,
    pagesBufferLength: 5,
  },
};

export default nextConfig;
```

---

## Step 8: Test the Dashboard

**Run development server**:
```bash
npm run dev
```

**Visit dashboard**:
```
http://localhost:3000/analytics
```

**What you'll see**:
1. ✅ Page loads instantly (shell renders)
2. ⏳ Summary cards load first (cached)
3. ⏳ Chart starts rendering (2s delay)
4. ⏳ Top pages table finishes loading (1.5s)
5. ✅ Page shows loading skeletons during streaming

---

## Advanced: Real-Time Updates with WebSocket

**File: `app/api/analytics/stream/route.ts`** - Server-Sent Events

```typescript
import { NextRequest } from 'next/server';
import { db } from '@/lib/db';

export async function GET(req: NextRequest) {
  // Create ReadableStream for Server-Sent Events
  const stream = new ReadableStream({
    async start(controller) {
      try {
        const encoder = new TextEncoder();
        let lastTimestamp = new Date();

        // Send events every 5 seconds
        const interval = setInterval(async () => {
          const newEvents = await db.analytics.findMany({
            where: {
              timestamp: { gt: lastTimestamp },
            },
            take: 10,
          });

          if (newEvents.length > 0) {
            const data = `data: ${JSON.stringify(newEvents)}\n\n`;
            controller.enqueue(encoder.encode(data));
            lastTimestamp = new Date();
          }
        }, 5000);

        // Cleanup on disconnect
        req.signal.addEventListener('abort', () => {
          clearInterval(interval);
          controller.close();
        });
      } catch (error) {
        controller.error(error);
      }
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

---

## What You Learned

✅ **Streaming & Suspense** (load page shell first, stream data)  
✅ **Server Components** (direct DB queries, zero JS)  
✅ **Client Components** (interactivity, event tracking)  
✅ **Mixed rendering** (combine server + client seamlessly)  
✅ **Performance** (Suspense boundaries show loading states)  
✅ **Real-time data** (Server-Sent Events)  

---

## Next Challenge

**Add**: 
- Filters (date range, event type dropdown)
- Export analytics as CSV
- Custom alerts (send email when pageviews drop)
- A/B testing dashboard
