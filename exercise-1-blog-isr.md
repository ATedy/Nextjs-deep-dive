# Exercise 1: Build a Blog with ISR

**Goal**: Create a blog where posts are pre-generated at build-time, but revalidated on-demand when edited.

**Concepts**: Dynamic routes, `generateStaticParams()`, `revalidateTag()`, Server Components.

---

## Step 1: Database Schema

**File: `prisma/schema.prisma`** (add this to existing schema)

```prisma
model BlogPost {
  id        String   @id @default(cuid())
  slug      String   @unique
  title     String
  content   String   @db.Text
  author    String
  image     String?
  published Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([slug])
  @@index([published])
}
```

**Run migrations**:
```bash
npx prisma migrate dev --name add_blog_posts
```

---

## Step 2: Seed the Database

**File: `prisma/seed.ts`**

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.blogPost.createMany({
    data: [
      {
        slug: 'getting-started-nextjs',
        title: 'Getting Started with Next.js 16',
        content: 'Next.js 16 introduces Turbopack for blazing-fast builds...',
        author: 'Emmanuel Doe',
        published: true,
      },
      {
        slug: 'server-components-explained',
        title: 'Understanding React Server Components',
        content: 'Server Components allow you to render React on the server...',
        author: 'Emmanuel Doe',
        published: true,
      },
      {
        slug: 'caching-strategies',
        title: 'Caching Strategies in Production',
        content: 'The use cache directive provides composable caching...',
        author: 'Emmanuel Doe',
        published: true,
      },
    ],
  });

  console.log('Seeded blog posts!');
}

main()
  .then(() => prisma.$disconnect())
  .catch((e) => {
    console.error(e);
    process.exit(1);
  });
```

**Run seed**:
```bash
npx prisma db seed
```

---

## Step 3: Create Blog Post Page (Dynamic Route)

**File: `app/blog/[slug]/page.tsx`** - ISR with revalidation

```typescript
import { notFound } from 'next/navigation';
import { db } from '@/lib/db';

type Props = {
  params: { slug: string };
};

export async function generateStaticParams() {
  // Pre-generate routes for all published posts at build-time
  const posts = await db.blogPost.findMany({
    where: { published: true },
    select: { slug: true },
  });

  return posts.map((post) => ({
    slug: post.slug,
  }));
}

export async function generateMetadata({ params }: Props) {
  const post = await db.blogPost.findUnique({
    where: { slug: params.slug },
  });

  if (!post) return { title: 'Not found' };

  return {
    title: post.title,
    description: post.content.substring(0, 160),
    openGraph: {
      title: post.title,
      description: post.content.substring(0, 160),
      type: 'article',
      authors: [post.author],
    },
  };
}

export default async function BlogPostPage({ params }: Props) {
  const post = await db.blogPost.findUnique({
    where: { slug: params.slug },
  });

  if (!post || !post.published) {
    notFound();
  }

  return (
    <article className="max-w-2xl mx-auto py-8 px-4">
      {post.image && (
        <img
          src={post.image}
          alt={post.title}
          className="w-full h-96 object-cover rounded-lg mb-6"
        />
      )}

      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-2">{post.title}</h1>
        <p className="text-gray-600">
          By {post.author} on {new Date(post.createdAt).toLocaleDateString()}
        </p>
      </header>

      <div className="prose prose-lg max-w-none">
        {post.content.split('\n\n').map((para, i) => (
          <p key={i} className="mb-4">
            {para}
          </p>
        ))}
      </div>

      {/* ISR Revalidation Hint */}
      <div className="mt-12 text-sm text-gray-500 border-t pt-4">
        <p>
          ✓ This page was generated at build-time. Edits trigger on-demand
          revalidation.
        </p>
      </div>
    </article>
  );
}

// Next.js 16: Revalidate every 1 hour (background refresh)
// Or use on-demand revalidation (next step)
export const revalidate = 3600; // 1 hour in seconds
```

---

## Step 4: Admin Dashboard to Revalidate Posts

**File: `app/api/blog/revalidate/route.ts`** - On-demand revalidation

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { revalidatePath } from 'next/cache';
import { revalidateTag } from 'next/cache';

// Secret token to prevent unauthorized revalidation
const REVALIDATE_SECRET = process.env.REVALIDATE_SECRET || 'your-secret';

export async function POST(req: NextRequest) {
  // 1. Verify request is authorized
  const token = req.headers.get('authorization')?.replace('Bearer ', '');
  if (token !== REVALIDATE_SECRET) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }

  // 2. Get slug from request body
  const { slug } = await req.json();
  if (!slug) {
    return NextResponse.json(
      { error: 'slug is required' },
      { status: 400 }
    );
  }

  // 3. Revalidate the specific post route
  revalidatePath(`/blog/${slug}`, 'page');
  revalidateTag(`blog-${slug}`); // For tagged revalidation

  return NextResponse.json({
    revalidated: true,
    slug,
    message: `Blog post /${slug} revalidated`,
  });
}
```

**File: `app/api/blog/[id]/route.ts`** - Update post and trigger revalidation

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function PUT(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  // Verify admin auth (implement your own)
  const adminToken = req.headers.get('authorization');
  if (!adminToken) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { title, content, published } = await req.json();

  // Update database
  const post = await db.blogPost.update({
    where: { id: params.id },
    data: { title, content, published },
  });

  // Trigger revalidation
  revalidatePath(`/blog/${post.slug}`, 'page');

  return NextResponse.json(post);
}
```

---

## Step 5: Blog List Page (SSG)

**File: `app/blog/page.tsx`** - List all posts

```typescript
import Link from 'next/link';
import { db } from '@/lib/db';

export default async function BlogPage() {
  const posts = await db.blogPost.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="max-w-2xl mx-auto py-8 px-4">
      <h1 className="text-4xl font-bold mb-8">Blog</h1>

      <div className="space-y-6">
        {posts.map((post) => (
          <article
            key={post.id}
            className="border p-6 rounded-lg hover:bg-gray-50 transition"
          >
            <Link href={`/blog/${post.slug}`}>
              <h2 className="text-2xl font-semibold mb-2 hover:text-blue-600">
                {post.title}
              </h2>
            </Link>

            <p className="text-gray-600 mb-3">
              {post.content.substring(0, 150)}...
            </p>

            <div className="text-sm text-gray-500 flex justify-between">
              <span>By {post.author}</span>
              <span>{new Date(post.createdAt).toLocaleDateString()}</span>
            </div>

            <Link
              href={`/blog/${post.slug}`}
              className="text-blue-600 hover:underline text-sm mt-2 inline-block"
            >
              Read more →
            </Link>
          </article>
        ))}
      </div>

      {posts.length === 0 && (
        <p className="text-gray-500 text-center py-12">
          No blog posts published yet.
        </p>
      )}
    </div>
  );
}

// This page is SSG: generated once at build-time, served from CDN
export const revalidate = 86400; // 24 hours
```

---

## Step 6: Test ISR

**In development**:
```bash
npm run dev
# Visit http://localhost:3000/blog
# Visit http://localhost:3000/blog/getting-started-nextjs
```

**Manually trigger revalidation** (for testing):
```bash
curl -X POST http://localhost:3000/api/blog/revalidate \
  -H "Authorization: Bearer your-secret" \
  -H "Content-Type: application/json" \
  -d '{"slug": "getting-started-nextjs"}'
```

**Build for production**:
```bash
npm run build
npm start
# Visit http://localhost:3000/blog/[slug]
# Edit a post in the database
# Trigger revalidation
# Visit again → new content!
```

---

## What You Learned

✅ **Dynamic routes** (`[slug]/page.tsx`)  
✅ **Static generation** (`generateStaticParams()`)  
✅ **ISR** (pre-render + background revalidation)  
✅ **On-demand revalidation** (`revalidatePath`, `revalidateTag`)  
✅ **SEO with metadata** (`generateMetadata()`)  
✅ **Server Components** (direct DB access, zero JS)  

---

## Next Challenge

**Add**: Comments to blog posts, markdown rendering, search functionality, related posts.
