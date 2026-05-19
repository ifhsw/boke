# Personal Blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a full-stack personal blog with admin management, reader accounts, comments, and MDX content.

**Architecture:** Next.js 15 App Router full-stack application. Server Components for data fetching, Server Actions for mutations, NextAuth.js for authentication, Prisma/SQLite for persistence, Tailwind CSS for styling (warm literary aesthetic).

**Tech Stack:** Next.js 15, TypeScript, Prisma 6, SQLite, NextAuth.js 5 (Auth.js), Tailwind CSS, MDX, Shiki, KaTeX, Vitest, React Testing Library

---

### Task 1: Project Scaffolding

**Files:**
- Create: all project scaffolding via create-next-app

- [ ] **Step 1: Create Next.js project**

```bash
cd /c/Users/shi/Desktop/book
npx create-next-app@latest blog --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-turbopack
```

- [ ] **Step 2: Install core dependencies**

```bash
cd /c/Users/shi/Desktop/book/blog
npm install prisma @prisma/client next-auth@beta @auth/prisma-adapter bcryptjs slugify
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom jsdom
```

- [ ] **Step 3: Install MDX and content dependencies**

```bash
npm install next-mdx-remote shiki katex rehype-katex remark-math remark-gfm
```

- [ ] **Step 4: Initialize Prisma with SQLite**

```bash
npx prisma init --datasource-provider sqlite
```

- [ ] **Step 5: Create .env file**

Create `blog/.env`:
```
DATABASE_URL="file:./dev.db"
AUTH_SECRET="dev-secret-change-in-production"
AUTH_URL="http://localhost:3000"
UPLOAD_DIR="./public/uploads"
```

Create `blog/.env.example`:
```
DATABASE_URL="file:./dev.db"
AUTH_SECRET="generate-a-random-secret"
AUTH_URL="http://localhost:3000"
UPLOAD_DIR="./public/uploads"
```

- [ ] **Step 6: Configure Tailwind with warm literary theme colors**

Modify `blog/tailwind.config.ts`:
```ts
import type { Config } from "tailwindcss";

const config: Config = {
  content: ["./src/**/*.{js,ts,jsx,tsx,mdx}"],
  theme: {
    extend: {
      colors: {
        warm: {
          bg: "#fef7ed",
          card: "#ffffff",
          border: "#f5d0a9",
          accent: "#b45309",
          text: "#78350f",
          muted: "#92400e",
          link: "#d97706",
        },
      },
      borderRadius: {
        card: "0.75rem",
      },
      boxShadow: {
        card: "0 1px 3px rgba(0,0,0,0.08)",
      },
    },
  },
  plugins: [],
};
export default config;
```

- [ ] **Step 7: Verify setup**

```bash
npm run dev
```
Expected: Next.js dev server starts on localhost:3000, default page renders.

- [ ] **Step 8: Commit**

```bash
cd /c/Users/shi/Desktop/book/blog
git add -A && git commit -m "feat: scaffold Next.js project with Prisma, Tailwind, and dependencies"
```

---

### Task 2: Database Schema

**Files:**
- Create: `prisma/schema.prisma`
- Create: `prisma/seed.ts`

- [ ] **Step 1: Write Prisma schema**

Write `prisma/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id           String    @id @default(uuid())
  username     String    @unique
  email        String    @unique
  passwordHash String
  role         String    @default("READER")
  avatar       String?
  createdAt    DateTime  @default(now())
  posts        Post[]
  comments     Comment[]
}

model Post {
  id         String    @id @default(uuid())
  title      String
  slug       String    @unique
  excerpt    String?
  content    String
  coverImage String?
  category   String
  status     String    @default("DRAFT")
  authorId   String
  author     User      @relation(fields: [authorId], references: [id])
  createdAt  DateTime  @default(now())
  updatedAt  DateTime  @updatedAt
  tags       PostTag[]
  comments   Comment[]
}

model Tag {
  id    String    @id @default(uuid())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  postId String
  tagId  String
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag    Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
}

model Comment {
  id        String    @id @default(uuid())
  content   String
  postId    String
  post      Post      @relation(fields: [postId], references: [id], onDelete: Cascade)
  userId    String
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  parentId  String?
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentReplies")
  createdAt DateTime  @default(now())
}
```

- [ ] **Step 2: Run initial migration**

```bash
npx prisma migrate dev --name init
```
Expected: Migration creates all tables in `prisma/dev.db`.

- [ ] **Step 3: Write seed script**

Write `prisma/seed.ts`:
```ts
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
  const passwordHash = await bcrypt.hash("admin123", 10);

  const admin = await prisma.user.upsert({
    where: { email: "admin@blog.com" },
    update: {},
    create: {
      username: "admin",
      email: "admin@blog.com",
      passwordHash,
      role: "ADMIN",
    },
  });

  const tagTech = await prisma.tag.create({ data: { name: "React" } });
  const tagLife = await prisma.tag.create({ data: { name: "生活" } });

  const post = await prisma.post.create({
    data: {
      title: "欢迎来到我的博客",
      slug: "hello-world",
      excerpt: "这是我的第一篇文章",
      content: `## 你好\n\n这是第一篇博客文章。欢迎来访。`,
      category: "ESSAY",
      status: "PUBLISHED",
      authorId: admin.id,
      tags: { create: [{ tagId: tagLife.id }] },
    },
  });

  console.log("Seed complete:", { admin: admin.email, post: post.title });
}

main()
  .catch((e) => { console.error(e); process.exit(1); })
  .finally(() => prisma.$disconnect());
```

- [ ] **Step 4: Configure seed in package.json**

In `package.json`, add:
```json
"prisma": {
  "seed": "tsx prisma/seed.ts"
}
```

- [ ] **Step 5: Run seed**

```bash
npx prisma db seed
```
Expected: Output shows "Seed complete: ..."

- [ ] **Step 6: Commit**

```bash
git add prisma/schema.prisma prisma/seed.ts prisma/migrations package.json
git commit -m "feat: add database schema and seed data"
```

---

### Task 3: Prisma Client Singleton

**Files:**
- Create: `src/lib/prisma.ts`

- [ ] **Step 1: Write Prisma client singleton**

Write `src/lib/prisma.ts`:
```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

- [ ] **Step 2: Commit**

```bash
git add src/lib/prisma.ts && git commit -m "feat: add Prisma client singleton"
```

---

### Task 4: Authentication Configuration

**Files:**
- Create: `src/lib/auth.ts`
- Create: `src/app/api/auth/[...nextauth]/route.ts`
- Modify: `src/app/layout.tsx`

- [ ] **Step 1: Write NextAuth configuration**

Write `src/lib/auth.ts`:
```ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";
import { prisma } from "./prisma";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Credentials({
      name: "credentials",
      credentials: {
        email: { label: "邮箱", type: "email" },
        password: { label: "密码", type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;
        const user = await prisma.user.findUnique({
          where: { email: credentials.email as string },
        });
        if (!user) return null;
        const valid = await bcrypt.compare(
          credentials.password as string,
          user.passwordHash
        );
        if (!valid) return null;
        return {
          id: user.id,
          email: user.email,
          name: user.username,
          role: user.role,
        };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = (user as any).role;
        token.id = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        (session.user as any).role = token.role;
        (session.user as any).id = token.id;
      }
      return session;
    },
  },
  pages: {
    signIn: "/login",
  },
  session: { strategy: "jwt" },
});
```

- [ ] **Step 2: Write API route for auth**

Create `src/app/api/auth/[...nextauth]/route.ts`:
```ts
import { handlers } from "@/lib/auth";

export const { GET, POST } = handlers;
```

- [ ] **Step 3: Wrap root layout with SessionProvider**

Write `src/app/layout.tsx`:
```tsx
import type { Metadata } from "next";
import "./globals.css";
import { SessionProvider } from "next-auth/react";

export const metadata: Metadata = {
  title: "我的博客",
  description: "记录技术与生活",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="zh-CN">
      <body className="bg-warm-bg text-warm-text min-h-screen">
        <SessionProvider>{children}</SessionProvider>
      </body>
    </html>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add src/lib/auth.ts src/app/api/auth/ src/app/layout.tsx
git commit -m "feat: add NextAuth credentials authentication"
```

---

### Task 5: Auth Server Actions

**Files:**
- Create: `src/actions/auth.ts`

- [ ] **Step 1: Write auth server actions**

Write `src/actions/auth.ts`:
```ts
"use server";

import { signIn, signOut } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import bcrypt from "bcryptjs";
import { AuthError } from "next-auth";

export async function loginAction(formData: FormData) {
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;
  try {
    await signIn("credentials", { email, password, redirect: false });
    return { success: true };
  } catch (error) {
    if (error instanceof AuthError) {
      return { success: false, error: "邮箱或密码错误" };
    }
    throw error;
  }
}

export async function registerAction(formData: FormData) {
  const username = formData.get("username") as string;
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;

  if (!username || !email || !password) {
    return { success: false, error: "请填写所有字段" };
  }
  if (password.length < 6) {
    return { success: false, error: "密码至少6位" };
  }

  const existing = await prisma.user.findFirst({
    where: { OR: [{ email }, { username }] },
  });
  if (existing) {
    return { success: false, error: "邮箱或用户名已存在" };
  }

  const passwordHash = await bcrypt.hash(password, 10);
  await prisma.user.create({
    data: { username, email, passwordHash, role: "READER" },
  });

  return { success: true };
}

export async function logoutAction() {
  await signOut({ redirect: false });
}
```

- [ ] **Step 2: Commit**

```bash
git add src/actions/auth.ts && git commit -m "feat: add auth server actions (login, register, logout)"
```

---

### Task 6: Global Styles

**Files:**
- Modify: `src/app/globals.css`

- [ ] **Step 1: Write global styles with warm literary theme**

Write `src/app/globals.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@import url('https://fonts.googleapis.com/css2?family=Noto+Serif+SC:wght@400;600;700&family=Inter:wght@400;500;600&display=swap');

@layer base {
  body {
    font-family: 'Inter', 'Noto Serif SC', sans-serif;
  }
  article {
    font-family: 'Noto Serif SC', serif;
    line-height: 1.8;
    font-size: 1.05rem;
  }
  article h1, article h2, article h3 {
    font-family: 'Inter', sans-serif;
    font-weight: 700;
    margin-top: 1.5em;
    margin-bottom: 0.5em;
  }
  article h2 { font-size: 1.5rem; color: #78350f; }
  article h3 { font-size: 1.25rem; color: #92400e; }
  article p { margin-bottom: 1em; }
  article code {
    background: #fef3c7;
    padding: 0.15em 0.4em;
    border-radius: 0.25rem;
    font-size: 0.9em;
    font-family: 'Fira Code', monospace;
  }
  article pre {
    background: #1e1e1e;
    color: #d4d4d4;
    padding: 1.25rem;
    border-radius: 0.75rem;
    overflow-x: auto;
    margin: 1rem 0;
    font-size: 0.9rem;
  }
  article pre code {
    background: none;
    padding: 0;
    color: inherit;
  }
  article blockquote {
    border-left: 3px solid #f5d0a9;
    padding-left: 1rem;
    color: #92400e;
    font-style: italic;
    margin: 1rem 0;
  }
  article a {
    color: #d97706;
    text-decoration: underline;
  }
  article img {
    border-radius: 0.75rem;
    max-width: 100%;
    margin: 1rem 0;
  }
}

@layer components {
  .btn-primary {
    @apply bg-warm-accent text-white px-6 py-2 rounded-lg font-medium
           hover:bg-warm-muted transition-colors disabled:opacity-50;
  }
  .btn-secondary {
    @apply border border-warm-border text-warm-text px-6 py-2 rounded-lg font-medium
           hover:bg-warm-bg transition-colors;
  }
  .card {
    @apply bg-warm-card rounded-card shadow-card p-6;
  }
  .input-field {
    @apply w-full border border-warm-border rounded-lg px-4 py-2
           focus:outline-none focus:ring-2 focus:ring-warm-accent
           bg-white text-warm-text placeholder:text-warm-muted/50;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add src/app/globals.css && git commit -m "feat: add warm literary global styles"
```

---

### Task 7: Shared Layout Components

**Files:**
- Create: `src/components/Header.tsx`
- Create: `src/components/Footer.tsx`
- Create: `src/components/Sidebar.tsx`
- Create: `src/components/TagCloud.tsx`
- Create: `src/components/PostCard.tsx`

- [ ] **Step 1: Write Header component**

Write `src/components/Header.tsx`:
```tsx
import Link from "next/link";
import { auth } from "@/lib/auth";
import { LogoutButton } from "./LogoutButton";

export async function Header() {
  const session = await auth();
  const isAdmin = (session?.user as any)?.role === "ADMIN";

  return (
    <header className="border-b-2 border-warm-border bg-warm-card">
      <div className="max-w-5xl mx-auto px-4 py-4 flex items-center justify-between">
        <Link href="/" className="text-xl font-bold text-warm-accent">
          📖 我的博客
        </Link>
        <nav className="flex items-center gap-6 text-sm">
          <Link href="/" className="text-warm-muted hover:text-warm-accent transition-colors">
            首页
          </Link>
          <Link href="/tech" className="text-warm-muted hover:text-warm-accent transition-colors">
            技术
          </Link>
          <Link href="/essay" className="text-warm-muted hover:text-warm-accent transition-colors">
            随笔
          </Link>
          <Link href="/archive" className="text-warm-muted hover:text-warm-accent transition-colors">
            归档
          </Link>
          <Link href="/about" className="text-warm-muted hover:text-warm-accent transition-colors">
            关于
          </Link>
          {isAdmin && (
            <Link href="/admin" className="text-warm-accent font-medium hover:underline">
              管理
            </Link>
          )}
          {session ? (
            <span className="text-warm-muted">
              {session.user?.name}
              <LogoutButton />
            </span>
          ) : (
            <Link href="/login" className="btn-primary text-sm">
              登录
            </Link>
          )}
        </nav>
      </div>
    </header>
  );
}
```

- [ ] **Step 2: Write LogoutButton (client component)**

Write `src/components/LogoutButton.tsx`:
```tsx
"use client";

import { logoutAction } from "@/actions/auth";

export function LogoutButton() {
  return (
    <button
      onClick={() => logoutAction()}
      className="ml-2 text-sm text-warm-link hover:underline"
    >
      退出
    </button>
  );
}
```

- [ ] **Step 3: Write Footer component**

Write `src/components/Footer.tsx`:
```tsx
export function Footer() {
  return (
    <footer className="border-t border-warm-border mt-16 py-8 text-center text-sm text-warm-muted">
      <p>&copy; {new Date().getFullYear()} 我的博客 · 用 ❤️ 和 Next.js 构建</p>
    </footer>
  );
}
```

- [ ] **Step 4: Write PostCard component**

Write `src/components/PostCard.tsx`:
```tsx
import Link from "next/link";

interface PostCardProps {
  title: string;
  slug: string;
  excerpt: string | null;
  category: string;
  createdAt: Date;
  tags: { tag: { name: string } }[];
}

export function PostCard({ title, slug, excerpt, category, createdAt, tags }: PostCardProps) {
  const categoryLabel = category === "TECH" ? "技术" : "随笔";
  return (
    <Link href={`/post/${slug}`} className="card block hover:shadow-md transition-shadow">
      <div className="flex items-center gap-2 text-xs mb-2">
        <span className="text-warm-link font-medium">{categoryLabel}</span>
        <span className="text-warm-muted/60">{new Date(createdAt).toLocaleDateString("zh-CN")}</span>
      </div>
      <h2 className="text-lg font-semibold text-warm-text mb-1">{title}</h2>
      {excerpt && <p className="text-sm text-warm-muted line-clamp-2">{excerpt}</p>}
      {tags.length > 0 && (
        <div className="flex gap-2 mt-3">
          {tags.map((pt) => (
            <span key={pt.tag.name} className="text-xs bg-warm-bg text-warm-muted px-2 py-0.5 rounded-full">
              {pt.tag.name}
            </span>
          ))}
        </div>
      )}
    </Link>
  );
}
```

- [ ] **Step 5: Write Sidebar component**

Write `src/components/Sidebar.tsx`:
```tsx
import { prisma } from "@/lib/prisma";

export async function Sidebar() {
  const recentPosts = await prisma.post.findMany({
    where: { status: "PUBLISHED" },
    orderBy: { createdAt: "desc" },
    take: 5,
    select: { title: true, slug: true },
  });

  const tags = await prisma.tag.findMany({
    include: { _count: { select: { posts: true } } },
  });

  return (
    <aside className="space-y-6">
      <div className="card">
        <h3 className="font-semibold text-warm-text mb-3">最近文章</h3>
        <ul className="space-y-2 text-sm">
          {recentPosts.map((p) => (
            <li key={p.slug}>
              <a href={`/post/${p.slug}`} className="text-warm-muted hover:text-warm-accent transition-colors">
                {p.title}
              </a>
            </li>
          ))}
        </ul>
      </div>
      <div className="card">
        <h3 className="font-semibold text-warm-text mb-3">标签</h3>
        <div className="flex flex-wrap gap-2">
          {tags.map((t) => (
            <span key={t.id} className="text-xs bg-warm-bg text-warm-muted px-2 py-1 rounded-full">
              {t.name} ({t._count.posts})
            </span>
          ))}
        </div>
      </div>
    </aside>
  );
}
```

- [ ] **Step 6: Update root layout to include Header and Footer**

Modify `src/app/layout.tsx`:
```tsx
import type { Metadata } from "next";
import "./globals.css";
import { SessionProvider } from "next-auth/react";
import { Header } from "@/components/Header";
import { Footer } from "@/components/Footer";

export const metadata: Metadata = {
  title: "我的博客",
  description: "记录技术与生活",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="zh-CN">
      <body className="bg-warm-bg text-warm-text min-h-screen flex flex-col">
        <SessionProvider>
          <Header />
          {children}
          <Footer />
        </SessionProvider>
      </body>
    </html>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add src/components/ src/app/layout.tsx
git commit -m "feat: add shared layout components (Header, Footer, Sidebar, PostCard)"
```

---

### Task 8: Home Page

**Files:**
- Modify: `src/app/page.tsx`

- [ ] **Step 1: Write home page**

Write `src/app/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { PostCard } from "@/components/PostCard";
import { Sidebar } from "@/components/Sidebar";

export default async function HomePage() {
  const posts = await prisma.post.findMany({
    where: { status: "PUBLISHED" },
    orderBy: { createdAt: "desc" },
    include: { tags: { include: { tag: true } } },
  });

  return (
    <main className="max-w-5xl mx-auto px-4 py-8 flex-1">
      <div className="flex flex-col lg:flex-row gap-8">
        <div className="flex-1 space-y-6">
          {posts.length === 0 ? (
            <p className="text-center text-warm-muted py-20">还没有文章，敬请期待。</p>
          ) : (
            posts.map((post) => <PostCard key={post.id} {...post} />)
          )}
        </div>
        <div className="w-full lg:w-72">
          <Sidebar />
        </div>
      </div>
    </main>
  );
}
```

- [ ] **Step 2: Start dev server and verify homepage renders**

```bash
npm run dev
```
Visit `http://localhost:3000` — expected: header, post card "欢迎来到我的博客", sidebar with recent posts and tags, footer.

- [ ] **Step 3: Commit**

```bash
git add src/app/page.tsx && git commit -m "feat: add home page with post list and sidebar"
```

---

### Task 9: Login & Register Pages

**Files:**
- Create: `src/app/login/page.tsx`
- Create: `src/app/register/page.tsx`

- [ ] **Step 1: Write login page**

Write `src/app/login/page.tsx`:
```tsx
"use client";

import { loginAction } from "@/actions/auth";
import { useRouter } from "next/navigation";
import { useState } from "react";
import Link from "next/link";

export default function LoginPage() {
  const router = useRouter();
  const [error, setError] = useState("");

  async function handleSubmit(formData: FormData) {
    const result = await loginAction(formData);
    if (result.success) {
      router.push("/");
      router.refresh();
    } else {
      setError(result.error || "登录失败");
    }
  }

  return (
    <main className="max-w-md mx-auto px-4 py-20">
      <h1 className="text-2xl font-bold text-warm-text text-center mb-8">登录</h1>
      <form action={handleSubmit} className="card space-y-4">
        {error && <p className="text-red-500 text-sm text-center">{error}</p>}
        <div>
          <label className="block text-sm font-medium text-warm-muted mb-1">邮箱</label>
          <input name="email" type="email" className="input-field" required />
        </div>
        <div>
          <label className="block text-sm font-medium text-warm-muted mb-1">密码</label>
          <input name="password" type="password" className="input-field" required />
        </div>
        <button type="submit" className="btn-primary w-full">登录</button>
        <p className="text-sm text-center text-warm-muted">
          没有账号？<Link href="/register" className="text-warm-link hover:underline">注册</Link>
        </p>
      </form>
    </main>
  );
}
```

- [ ] **Step 2: Write register page**

Write `src/app/register/page.tsx`:
```tsx
"use client";

import { registerAction } from "@/actions/auth";
import { useRouter } from "next/navigation";
import { useState } from "react";
import Link from "next/link";

export default function RegisterPage() {
  const router = useRouter();
  const [error, setError] = useState("");

  async function handleSubmit(formData: FormData) {
    const result = await registerAction(formData);
    if (result.success) {
      router.push("/login");
    } else {
      setError(result.error || "注册失败");
    }
  }

  return (
    <main className="max-w-md mx-auto px-4 py-20">
      <h1 className="text-2xl font-bold text-warm-text text-center mb-8">注册</h1>
      <form action={handleSubmit} className="card space-y-4">
        {error && <p className="text-red-500 text-sm text-center">{error}</p>}
        <div>
          <label className="block text-sm font-medium text-warm-muted mb-1">用户名</label>
          <input name="username" type="text" className="input-field" required />
        </div>
        <div>
          <label className="block text-sm font-medium text-warm-muted mb-1">邮箱</label>
          <input name="email" type="email" className="input-field" required />
        </div>
        <div>
          <label className="block text-sm font-medium text-warm-muted mb-1">密码</label>
          <input name="password" type="password" className="input-field" minLength={6} required />
        </div>
        <button type="submit" className="btn-primary w-full">注册</button>
        <p className="text-sm text-center text-warm-muted">
          已有账号？<Link href="/login" className="text-warm-link hover:underline">登录</Link>
        </p>
      </form>
    </main>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add src/app/login/page.tsx src/app/register/page.tsx
git commit -m "feat: add login and register pages"
```

---

### Task 10: Post Detail Page & MDX Renderer

**Files:**
- Create: `src/app/post/[slug]/page.tsx`
- Create: `src/lib/mdx.ts`
- Create: `src/components/MdxRenderer.tsx`

- [ ] **Step 1: Write MDX processing utility**

Write `src/lib/mdx.ts`:
```ts
import { compileMDX } from "next-mdx-remote/rsc";
import remarkGfm from "remark-gfm";
import remarkMath from "remark-math";
import rehypeKatex from "rehype-katex";

export async function compileMdx(source: string) {
  const { content } = await compileMDX({
    source,
    options: {
      mdxOptions: {
        remarkPlugins: [remarkGfm, remarkMath],
        rehypePlugins: [rehypeKatex],
      },
      parseFrontmatter: false,
    },
  });
  return content;
}
```

- [ ] **Step 2: Write post detail page**

Write `src/app/post/[slug]/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { compileMdx } from "@/lib/mdx";
import { notFound } from "next/navigation";
import { CommentSection } from "@/components/CommentSection";

export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await prisma.post.findUnique({
    where: { slug },
    include: { author: { select: { username: true } } },
  });

  if (!post || post.status !== "PUBLISHED") notFound();

  const mdxContent = await compileMdx(post.content);

  const categoryLabel = post.category === "TECH" ? "技术" : "随笔";

  return (
    <main className="max-w-3xl mx-auto px-4 py-8">
      <article>
        <header className="mb-8">
          <div className="flex items-center gap-3 text-sm mb-2">
            <span className="text-warm-link font-medium">{categoryLabel}</span>
            <span className="text-warm-muted/60">
              {new Date(post.createdAt).toLocaleDateString("zh-CN")}
            </span>
          </div>
          <h1 className="text-3xl font-bold text-warm-text">{post.title}</h1>
          <p className="text-sm text-warm-muted mt-2">作者：{post.author.username}</p>
        </header>
        <div className="prose">{mdxContent}</div>
      </article>
      <hr className="my-8 border-warm-border" />
      <CommentSection postId={post.id} />
    </main>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add src/lib/mdx.ts src/app/post/ src/components/MdxRenderer.tsx
git commit -m "feat: add post detail page with MDX rendering"
```

---

### Task 11: Comment System

**Files:**
- Create: `src/actions/comments.ts`
- Create: `src/components/CommentSection.tsx`
- Create: `src/components/CommentItem.tsx`

- [ ] **Step 1: Write comment server actions**

Write `src/actions/comments.ts`:
```ts
"use server";

import { prisma } from "@/lib/prisma";
import { auth } from "@/lib/auth";
import { revalidatePath } from "next/cache";

export async function createComment(formData: FormData) {
  const session = await auth();
  if (!session?.user) return { success: false, error: "请先登录" };

  const content = formData.get("content") as string;
  const postId = formData.get("postId") as string;
  const parentId = formData.get("parentId") as string | null;

  if (!content?.trim()) return { success: false, error: "评论不能为空" };

  await prisma.comment.create({
    data: {
      content: content.trim(),
      postId,
      userId: (session.user as any).id,
      parentId: parentId || null,
    },
  });

  revalidatePath("/post/[slug]");
  return { success: true };
}

export async function deleteComment(commentId: string) {
  const session = await auth();
  if (!session?.user || (session.user as any).role !== "ADMIN") {
    return { success: false, error: "无权限" };
  }

  await prisma.comment.delete({ where: { id: commentId } });
  revalidatePath("/post/[slug]");
  return { success: true };
}
```

- [ ] **Step 2: Write CommentSection (client component)**

Write `src/components/CommentSection.tsx`:
```tsx
"use client";

import { createComment, deleteComment } from "@/actions/comments";
import { useSession } from "next-auth/react";
import { useState } from "react";

interface CommentWithUser {
  id: string;
  content: string;
  createdAt: Date;
  user: { username: string; avatar: string | null };
  parentId: string | null;
  replies?: CommentWithUser[];
}

export function CommentSection({
  postId,
  comments: initialComments,
}: {
  postId: string;
  comments: CommentWithUser[];
}) {
  const { data: session } = useSession();
  const isAdmin = (session?.user as any)?.role === "ADMIN";
  const [replyTo, setReplyTo] = useState<string | null>(null);

  function nestComments(comments: CommentWithUser[]): CommentWithUser[] {
    const top = comments.filter((c) => !c.parentId);
    const replies = comments.filter((c) => c.parentId);
    return top.map((c) => ({
      ...c,
      replies: replies.filter((r) => r.parentId === c.id),
    }));
  }

  const nested = nestComments(initialComments);

  return (
    <div>
      <h3 className="text-xl font-bold text-warm-text mb-4">评论 ({initialComments.length})</h3>
      {nested.map((comment) => (
        <CommentItem
          key={comment.id}
          comment={comment}
          isAdmin={isAdmin}
          onReply={setReplyTo}
          replyTo={replyTo}
          postId={postId}
        />
      ))}
      {session ? (
        <form action={createComment} className="mt-6 space-y-3">
          <input type="hidden" name="postId" value={postId} />
          {replyTo && <input type="hidden" name="parentId" value={replyTo} />}
          <textarea
            name="content"
            className="input-field min-h-[100px]"
            placeholder={replyTo ? "写下你的回复..." : "写下你的评论..."}
            required
          />
          <div className="flex gap-2">
            <button type="submit" className="btn-primary text-sm">发表</button>
            {replyTo && (
              <button type="button" className="btn-secondary text-sm" onClick={() => setReplyTo(null)}>
                取消回复
              </button>
            )}
          </div>
        </form>
      ) : (
        <p className="text-sm text-warm-muted mt-4">
          <a href="/login" className="text-warm-link hover:underline">登录</a> 后发表评论
        </p>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Write CommentItem component**

Write `src/components/CommentItem.tsx`:
```tsx
"use client";

import { deleteComment, createComment } from "@/actions/comments";

interface CommentWithUser {
  id: string;
  content: string;
  createdAt: Date;
  user: { username: string; avatar: string | null };
  parentId: string | null;
  replies?: CommentWithUser[];
}

export function CommentItem({
  comment,
  isAdmin,
  onReply,
  replyTo,
  postId,
  depth = 0,
}: {
  comment: CommentWithUser;
  isAdmin: boolean;
  onReply: (id: string | null) => void;
  replyTo: string | null;
  postId: string;
  depth?: number;
}) {
  return (
    <div className={`${depth > 0 ? "ml-6 border-l-2 border-warm-border pl-4" : ""} mb-4`}>
      <div className="flex items-center gap-2 text-sm mb-1">
        <span className="font-medium text-warm-text">{comment.user.username}</span>
        <span className="text-warm-muted/60 text-xs">
          {new Date(comment.createdAt).toLocaleDateString("zh-CN")}
        </span>
      </div>
      <p className="text-sm text-warm-text mb-2">{comment.content}</p>
      <div className="flex gap-2 text-xs">
        <button onClick={() => onReply(comment.id)} className="text-warm-link hover:underline">
          回复
        </button>
        {isAdmin && (
          <button
            onClick={() => deleteComment(comment.id)}
            className="text-red-500 hover:underline"
          >
            删除
          </button>
        )}
      </div>
      {comment.replies?.map((reply) => (
        <CommentItem
          key={reply.id}
          comment={reply}
          isAdmin={isAdmin}
          onReply={onReply}
          replyTo={replyTo}
          postId={postId}
          depth={depth + 1}
        />
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Update post detail page to include comments**

Modify `src/app/post/[slug]/page.tsx` to fetch and pass comments:
```tsx
// Add this after the post query:
const comments = await prisma.comment.findMany({
  where: { postId: post.id },
  include: { user: { select: { username: true, avatar: true } } },
  orderBy: { createdAt: "asc" },
});

// Update CommentSection usage:
<CommentSection postId={post.id} comments={comments} />
```

- [ ] **Step 5: Commit**

```bash
git add src/actions/comments.ts src/components/CommentSection.tsx src/components/CommentItem.tsx src/app/post/
git commit -m "feat: add comment system with nested replies"
```

---

### Task 12: Channel & Archive & About Pages

**Files:**
- Create: `src/app/tech/page.tsx`
- Create: `src/app/essay/page.tsx`
- Create: `src/app/archive/page.tsx`
- Create: `src/app/about/page.tsx`

- [ ] **Step 1: Write tech channel page**

Write `src/app/tech/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { PostCard } from "@/components/PostCard";

export default async function TechPage() {
  const posts = await prisma.post.findMany({
    where: { status: "PUBLISHED", category: "TECH" },
    orderBy: { createdAt: "desc" },
    include: { tags: { include: { tag: true } } },
  });

  return (
    <main className="max-w-3xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold text-warm-text mb-6">技术</h1>
      {posts.length === 0 ? (
        <p className="text-center text-warm-muted py-20">暂无技术文章。</p>
      ) : (
        <div className="space-y-6">{posts.map((p) => <PostCard key={p.id} {...p} />)}</div>
      )}
    </main>
  );
}
```

- [ ] **Step 2: Write essay channel page**

Write `src/app/essay/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { PostCard } from "@/components/PostCard";

export default async function EssayPage() {
  const posts = await prisma.post.findMany({
    where: { status: "PUBLISHED", category: "ESSAY" },
    orderBy: { createdAt: "desc" },
    include: { tags: { include: { tag: true } } },
  });

  return (
    <main className="max-w-3xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold text-warm-text mb-6">随笔</h1>
      {posts.length === 0 ? (
        <p className="text-center text-warm-muted py-20">暂无随笔文章。</p>
      ) : (
        <div className="space-y-6">{posts.map((p) => <PostCard key={p.id} {...p} />)}</div>
      )}
    </main>
  );
}
```

- [ ] **Step 3: Write archive page**

Write `src/app/archive/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import Link from "next/link";

export default async function ArchivePage() {
  const posts = await prisma.post.findMany({
    where: { status: "PUBLISHED" },
    orderBy: { createdAt: "desc" },
    select: { title: true, slug: true, createdAt: true, category: true },
  });

  const groupedByYear = posts.reduce<Record<string, typeof posts>>((acc, post) => {
    const year = new Date(post.createdAt).getFullYear().toString();
    (acc[year] ||= []).push(post);
    return acc;
  }, {});

  return (
    <main className="max-w-3xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold text-warm-text mb-6">归档</h1>
      {Object.entries(groupedByYear).map(([year, yearPosts]) => (
        <section key={year} className="mb-8">
          <h2 className="text-xl font-semibold text-warm-accent mb-4">{year}</h2>
          <ul className="space-y-2">
            {yearPosts.map((p) => (
              <li key={p.slug} className="flex items-center gap-4 text-sm">
                <span className="text-warm-muted/60 w-16 shrink-0">
                  {new Date(p.createdAt).toLocaleDateString("zh-CN", { month: "short", day: "numeric" })}
                </span>
                <span className="text-xs text-warm-link bg-warm-bg px-2 py-0.5 rounded-full w-10 text-center">
                  {p.category === "TECH" ? "技术" : "随笔"}
                </span>
                <Link href={`/post/${p.slug}`} className="text-warm-text hover:text-warm-accent transition-colors">
                  {p.title}
                </Link>
              </li>
            ))}
          </ul>
        </section>
      ))}
    </main>
  );
}
```

- [ ] **Step 4: Write about page**

Write `src/app/about/page.tsx`:
```tsx
export default function AboutPage() {
  return (
    <main className="max-w-2xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold text-warm-text mb-6">关于我</h1>
      <div className="card space-y-4">
        <p className="text-warm-muted leading-relaxed">
          你好，欢迎来到我的个人博客。这里记录了我的技术探索和生活随想。
        </p>
        <p className="text-warm-muted leading-relaxed">
          技术方面，我关注前端开发、系统设计和开源项目。生活方面，我喜欢阅读、写作和探索新事物。
        </p>
        <p className="text-warm-muted leading-relaxed">
          希望通过这个博客，与你分享我的思考与收获。
        </p>
      </div>
    </main>
  );
}
```

- [ ] **Step 5: Commit**

```bash
git add src/app/tech/ src/app/essay/ src/app/archive/ src/app/about/
git commit -m "feat: add channel, archive, and about pages"
```

---

### Task 13: Admin Layout & Dashboard

**Files:**
- Create: `src/app/admin/layout.tsx`
- Create: `src/components/AdminSidebar.tsx`
- Create: `src/app/admin/page.tsx`

- [ ] **Step 1: Write AdminSidebar**

Write `src/components/AdminSidebar.tsx`:
```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";

const links = [
  { href: "/admin", label: "仪表盘" },
  { href: "/admin/posts", label: "文章管理" },
  { href: "/admin/comments", label: "评论管理" },
  { href: "/admin/users", label: "用户管理" },
];

export function AdminSidebar() {
  const pathname = usePathname();
  return (
    <aside className="w-48 shrink-0">
      <nav className="card space-y-1">
        {links.map((link) => (
          <Link
            key={link.href}
            href={link.href}
            className={`block px-3 py-2 rounded-lg text-sm transition-colors ${
              pathname === link.href
                ? "bg-warm-accent text-white"
                : "text-warm-muted hover:bg-warm-bg"
            }`}
          >
            {link.label}
          </Link>
        ))}
      </nav>
    </aside>
  );
}
```

- [ ] **Step 2: Write admin layout with auth protection**

Write `src/app/admin/layout.tsx`:
```tsx
import { auth } from "@/lib/auth";
import { redirect } from "next/navigation";
import { AdminSidebar } from "@/components/AdminSidebar";

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  if ((session?.user as any)?.role !== "ADMIN") redirect("/login");

  return (
    <main className="max-w-5xl mx-auto px-4 py-8 flex gap-8">
      <AdminSidebar />
      <div className="flex-1">{children}</div>
    </main>
  );
}
```

- [ ] **Step 3: Write admin dashboard**

Write `src/app/admin/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";

export default async function AdminDashboard() {
  const [postCount, commentCount, readerCount, draftCount] = await Promise.all([
    prisma.post.count(),
    prisma.comment.count(),
    prisma.user.count({ where: { role: "READER" } }),
    prisma.post.count({ where: { status: "DRAFT" } }),
  ]);

  const stats = [
    { label: "文章总数", value: postCount },
    { label: "草稿", value: draftCount },
    { label: "评论", value: commentCount },
    { label: "读者", value: readerCount },
  ];

  return (
    <div>
      <h1 className="text-2xl font-bold text-warm-text mb-6">仪表盘</h1>
      <div className="grid grid-cols-2 gap-4">
        {stats.map((s) => (
          <div key={s.label} className="card text-center">
            <div className="text-3xl font-bold text-warm-accent">{s.value}</div>
            <div className="text-sm text-warm-muted mt-1">{s.label}</div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add src/app/admin/layout.tsx src/app/admin/page.tsx src/components/AdminSidebar.tsx
git commit -m "feat: add admin layout, sidebar, and dashboard"
```

---

### Task 14: Admin Post Management

**Files:**
- Create: `src/actions/posts.ts`
- Create: `src/app/admin/posts/page.tsx`
- Create: `src/app/admin/posts/new/page.tsx`
- Create: `src/app/admin/posts/[id]/edit/page.tsx`
- Create: `src/components/PostEditor.tsx`

- [ ] **Step 1: Write post server actions**

Write `src/actions/posts.ts`:
```ts
"use server";

import { prisma } from "@/lib/prisma";
import { auth } from "@/lib/auth";
import { revalidatePath } from "next/cache";
import slugify from "slugify";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  const session = await auth();
  if ((session?.user as any)?.role !== "ADMIN") return { success: false, error: "无权限" };

  const title = formData.get("title") as string;
  const content = formData.get("content") as string;
  const category = formData.get("category") as string;
  const excerpt = formData.get("excerpt") as string;
  const status = formData.get("status") as string;
  const tagNames = (formData.get("tags") as string).split(",").map((t) => t.trim()).filter(Boolean);

  let slug = slugify(title, { lower: true, strict: true, locale: "zh" });
  const existing = await prisma.post.findUnique({ where: { slug } });
  if (existing) slug += "-" + Date.now().toString(36);

  const userId = (session.user as any).id;

  await prisma.post.create({
    data: {
      title,
      slug,
      content,
      category,
      excerpt: excerpt || null,
      status,
      authorId: userId,
      tags: {
        create: await Promise.all(
          tagNames.map(async (name) => {
            const tag = await prisma.tag.upsert({
              where: { name },
              create: { name },
              update: {},
            });
            return { tagId: tag.id };
          })
        ),
      },
    },
  });

  revalidatePath("/admin/posts");
  revalidatePath("/");
  redirect("/admin/posts");
}

export async function updatePost(id: string, formData: FormData) {
  const session = await auth();
  if ((session?.user as any)?.role !== "ADMIN") return { success: false, error: "无权限" };

  const title = formData.get("title") as string;
  const content = formData.get("content") as string;
  const category = formData.get("category") as string;
  const excerpt = formData.get("excerpt") as string;
  const status = formData.get("status") as string;

  await prisma.post.update({
    where: { id },
    data: { title, content, category, excerpt: excerpt || null, status },
  });

  revalidatePath("/admin/posts");
  revalidatePath("/");
  redirect("/admin/posts");
}

export async function deletePost(id: string) {
  const session = await auth();
  if ((session?.user as any)?.role !== "ADMIN") return { success: false, error: "无权限" };

  await prisma.post.delete({ where: { id } });
  revalidatePath("/admin/posts");
  revalidatePath("/");
  return { success: true };
}
```

- [ ] **Step 2: Write PostEditor client component**

Write `src/components/PostEditor.tsx`:
```tsx
"use client";

import { useState } from "react";

interface PostEditorProps {
  action: (formData: FormData) => Promise<void>;
  initialData?: {
    title: string;
    excerpt: string;
    content: string;
    category: string;
    status: string;
    tags?: string;
  };
  submitLabel: string;
}

export function PostEditor({ action, initialData, submitLabel }: PostEditorProps) {
  const [preview, setPreview] = useState(false);
  const [content, setContent] = useState(initialData?.content || "");

  return (
    <form action={action} className="space-y-4">
      <div>
        <label className="block text-sm font-medium text-warm-muted mb-1">标题</label>
        <input name="title" defaultValue={initialData?.title} className="input-field" required />
      </div>
      <div>
        <label className="block text-sm font-medium text-warm-muted mb-1">摘要</label>
        <input name="excerpt" defaultValue={initialData?.excerpt || ""} className="input-field" />
      </div>
      <div>
        <label className="block text-sm font-medium text-warm-muted mb-1">标签（逗号分隔）</label>
        <input name="tags" defaultValue={initialData?.tags || ""} className="input-field" placeholder="React, 生活, 教程" />
      </div>
      <div className="flex gap-4">
        <div className="flex-1">
          <label className="block text-sm font-medium text-warm-muted mb-1">分类</label>
          <select name="category" defaultValue={initialData?.category || "TECH"} className="input-field">
            <option value="TECH">技术</option>
            <option value="ESSAY">随笔</option>
          </select>
        </div>
        <div className="flex-1">
          <label className="block text-sm font-medium text-warm-muted mb-1">状态</label>
          <select name="status" defaultValue={initialData?.status || "DRAFT"} className="input-field">
            <option value="DRAFT">草稿</option>
            <option value="PUBLISHED">发布</option>
          </select>
        </div>
      </div>
      <div>
        <div className="flex justify-between items-center mb-1">
          <label className="text-sm font-medium text-warm-muted">内容 (Markdown)</label>
          <button type="button" onClick={() => setPreview(!preview)} className="text-xs text-warm-link hover:underline">
            切换 {preview ? "编辑" : "预览"}
          </button>
        </div>
        {preview ? (
          <div className="input-field min-h-[300px] prose" dangerouslySetInnerHTML={{ __html: content }} />
        ) : (
          <textarea
            name="content"
            value={content}
            onChange={(e) => setContent(e.target.value)}
            className="input-field min-h-[300px] font-mono text-sm"
            required
          />
        )}
      </div>
      <button type="submit" className="btn-primary">{submitLabel}</button>
    </form>
  );
}
```

- [ ] **Step 3: Write admin posts list page**

Write `src/app/admin/posts/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import Link from "next/link";
import { deletePost } from "@/actions/posts";

export default async function AdminPostsPage() {
  const posts = await prisma.post.findMany({
    orderBy: { createdAt: "desc" },
  });

  return (
    <div>
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold text-warm-text">文章管理</h1>
        <Link href="/admin/posts/new" className="btn-primary text-sm">写文章</Link>
      </div>
      <div className="space-y-3">
        {posts.map((post) => (
          <div key={post.id} className="card flex justify-between items-center">
            <div>
              <Link href={`/post/${post.slug}`} className="font-medium text-warm-text hover:text-warm-accent">
                {post.title}
              </Link>
              <div className="text-xs text-warm-muted mt-1">
                {post.status === "DRAFT" ? "草稿" : "已发布"} · {new Date(post.createdAt).toLocaleDateString("zh-CN")}
              </div>
            </div>
            <div className="flex gap-2">
              <Link href={`/admin/posts/${post.id}/edit`} className="btn-secondary text-xs py-1 px-3">编辑</Link>
              <form action={deletePost.bind(null, post.id)}>
                <button type="submit" className="text-xs text-red-500 hover:underline py-1 px-3">
                  删除
                </button>
              </form>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Write new post page**

Write `src/app/admin/posts/new/page.tsx`:
```tsx
import { createPost } from "@/actions/posts";
import { PostEditor } from "@/components/PostEditor";

export default function NewPostPage() {
  return (
    <div>
      <h1 className="text-2xl font-bold text-warm-text mb-6">写文章</h1>
      <PostEditor action={createPost} submitLabel="发布" />
    </div>
  );
}
```

- [ ] **Step 5: Write edit post page**

Write `src/app/admin/posts/[id]/edit/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { updatePost } from "@/actions/posts";
import { PostEditor } from "@/components/PostEditor";
import { notFound } from "next/navigation";

export default async function EditPostPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const post = await prisma.post.findUnique({
    where: { id },
    include: { tags: { include: { tag: true } } },
  });
  if (!post) notFound();

  const bindUpdate = updatePost.bind(null, post.id);

  return (
    <div>
      <h1 className="text-2xl font-bold text-warm-text mb-6">编辑文章</h1>
      <PostEditor
        action={bindUpdate}
        initialData={{
          title: post.title,
          excerpt: post.excerpt || "",
          content: post.content,
          category: post.category,
          status: post.status,
          tags: post.tags.map((pt) => pt.tag.name).join(", "),
        }}
        submitLabel="保存"
      />
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add src/actions/posts.ts src/components/PostEditor.tsx src/app/admin/posts/
git commit -m "feat: add admin post CRUD with editor"
```

---

### Task 15: Admin Comment & User Management

**Files:**
- Create: `src/app/admin/comments/page.tsx`
- Create: `src/app/admin/users/page.tsx`

- [ ] **Step 1: Write admin comments page**

Write `src/app/admin/comments/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";
import { deleteComment } from "@/actions/comments";

export default async function AdminCommentsPage() {
  const comments = await prisma.comment.findMany({
    orderBy: { createdAt: "desc" },
    include: {
      user: { select: { username: true } },
      post: { select: { title: true, slug: true } },
    },
    take: 50,
  });

  return (
    <div>
      <h1 className="text-2xl font-bold text-warm-text mb-6">评论管理</h1>
      <div className="space-y-3">
        {comments.map((c) => (
          <div key={c.id} className="card flex justify-between items-start gap-4">
            <div className="flex-1">
              <p className="text-sm text-warm-text">{c.content}</p>
              <div className="text-xs text-warm-muted mt-1">
                {c.user.username} · 在 <a href={`/post/${c.post.slug}`} className="text-warm-link hover:underline">{c.post.title}</a> · {new Date(c.createdAt).toLocaleString("zh-CN")}
              </div>
            </div>
            <form action={deleteComment.bind(null, c.id)}>
              <button type="submit" className="text-xs text-red-500 hover:underline">删除</button>
            </form>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Write admin users page**

Write `src/app/admin/users/page.tsx`:
```tsx
import { prisma } from "@/lib/prisma";

export default async function AdminUsersPage() {
  const users = await prisma.user.findMany({
    orderBy: { createdAt: "desc" },
    include: {
      _count: {
        select: { comments: true, posts: true },
      },
    },
  });

  return (
    <div>
      <h1 className="text-2xl font-bold text-warm-text mb-6">用户管理</h1>
      <div className="space-y-3">
        {users.map((u) => (
          <div key={u.id} className="card">
            <div className="flex justify-between items-start">
              <div>
                <span className="font-medium text-warm-text">{u.username}</span>
                <span className={`ml-2 text-xs px-2 py-0.5 rounded-full ${u.role === "ADMIN" ? "bg-warm-accent text-white" : "bg-warm-bg text-warm-muted"}`}>
                  {u.role === "ADMIN" ? "管理员" : "读者"}
                </span>
              </div>
            </div>
            <div className="text-sm text-warm-muted mt-1 space-y-0.5">
              <div>邮箱：{u.email}</div>
              <div>注册时间：{new Date(u.createdAt).toLocaleString("zh-CN")}</div>
              <div>文章数：{u._count.posts} · 评论数：{u._count.comments}</div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add src/app/admin/comments/ src/app/admin/users/
git commit -m "feat: add admin comment and user management pages"
```

---

### Task 16: Deployment Configuration

**Files:**
- Create: `ecosystem.config.cjs`
- Create: `next.config.ts`
- Create: `.gitignore` additions

- [ ] **Step 1: Create PM2 ecosystem file**

Write `ecosystem.config.cjs`:
```js
module.exports = {
  apps: [
    {
      name: "blog",
      script: "node_modules/.bin/next",
      args: "start",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
      },
    },
  ],
};
```

- [ ] **Step 2: Configure Next.js for production**

Write `next.config.ts`:
```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  images: {
    remotePatterns: [],
  },
};

export default nextConfig;
```

- [ ] **Step 3: Ensure .gitignore covers .superpowers**

Modify `.gitignore` — add at end:
```
.superpowers/
```

- [ ] **Step 4: Final verification build**

```bash
npm run build
```
Expected: Build succeeds without errors, producing a standalone output.

- [ ] **Step 5: Commit**

```bash
git add ecosystem.config.cjs next.config.ts .gitignore
git commit -m "feat: add production deployment config"
```

---

### Deployment Instructions

On the Linux server (宝塔面板):

```bash
# 1. Clone repository
git clone <repo-url> /www/wwwroot/blog
cd /www/wwwroot/blog

# 2. Install dependencies and build
npm install
npx prisma migrate deploy
npx prisma db seed
npm run build

# 3. Start with PM2
pm2 start ecosystem.config.cjs
pm2 save
pm2 startup

# 4. Configure Nginx in 宝塔 panel:
# - Create site pointing to localhost:3000 (reverse proxy)
# - Enable SSL via Let's Encrypt
# - Set static file serving: public/

# 5. Database backup (宝塔计划任务, daily):
cp prisma/dev.db /www/backup/blog-$(date +%Y%m%d).db
```

---

### Task 25: Comment System Fixes (Completed)

- [x] **Step 1: Fix recursive nestComments** — Rewrote `nestComments()` to use Map-based recursive approach, supports arbitrary nesting depth
- [x] **Step 2: Reply-to-username indicator** — Reply banner now shows "回复 @username" instead of generic text
- [x] **Step 3: Link usernames to profiles** — All username displays in comments, posts, and admin pages link to `/user/[username]`

### Task 26: User Post Creation (Completed)

**Files:**
- Modify: `src/actions/posts.ts` — Allow logged-in users; non-admin forced DRAFT; ownership checks
- Modify: `src/components/PostEditor.tsx` — Added `showStatus` prop for non-admin mode
- Create: `src/app/my-posts/layout.tsx` — Auth protection (login required)
- Create: `src/app/my-posts/page.tsx` — User's own posts list with edit/delete
- Create: `src/app/my-posts/new/page.tsx` — Write post (auto DRAFT, no status selector)
- Create: `src/app/my-posts/[id]/edit/page.tsx` — Edit own post (ownership check)
- Modify: `src/components/NavBar.tsx` — "写文章" (all users), "我的文章" (non-admin), "管理" (admin)
- Modify: `src/components/Header.tsx` — Pass `isLoggedIn` prop

### Task 27: User Profiles (Completed)

**Files:**
- Modify: `prisma/schema.prisma` — Added bio, website, location, github, twitter fields
- Create: `src/actions/profile.ts` — updateProfile server action
- Create: `src/components/ProfileEditForm.tsx` — Form for bio/website/location/github/twitter
- Create: `src/app/user/[username]/page.tsx` — Public profile with avatar, bio, social links, published posts
- Modify: `src/app/account/page.tsx` — Integrated profile editing + "查看我的主页" link
- Modify: `src/app/post/[slug]/page.tsx` — Author name links to profile
- Modify: `src/app/admin/comments/page.tsx` — Username links to profile
- Modify: `src/app/admin/users/page.tsx` — Username links to profile

### Task 17: Tag Filtering & Search (Completed)

**Files:**
- Create: `src/app/tag/[name]/page.tsx`
- Create: `src/app/search/page.tsx`
- Modify: `src/components/Sidebar.tsx`
- Modify: `src/components/NavBar.tsx`

- [x] **Step 1: Create tag page** — `/tag/[name]` shows all published posts with that tag using PostCard
- [x] **Step 2: Update sidebar tags** — Links from `/archive` to `/tag/[name]`
- [x] **Step 3: Add search input to NavBar** — Inline search form submits to `/search?q=`
- [x] **Step 4: Create search page** — Server-side Prisma `contains` search on title + content

---

### Task 18: Admin Settings — Background & About Editor (Completed)

**Files:**
- Create: `prisma/schema.prisma` — Added `SiteSetting` model
- Create: `src/actions/settings.ts`
- Create: `src/components/BackgroundUploader.tsx`
- Create: `src/components/AboutEditor.tsx`
- Create: `src/app/admin/settings/page.tsx`
- Modify: `src/app/layout.tsx` — Dynamic background via `<style>` tag
- Modify: `src/app/about/page.tsx` — Reads `about_content` from DB
- Modify: `src/components/AdminSidebar.tsx` — Added "站点设置" link

- [x] **Step 1: Add SiteSetting model** — Key-value store for `background` and `about_content`
- [x] **Step 2: Background upload** — Admin uploads wallpaper image, applies to all pages
- [x] **Step 3: About editor** — Markdown editor with preview, image upload, saves to DB
- [x] **Step 4: About page** — Custom content renders below original cards; fallback to defaults

---

### Task 19: Bug Fixes (Completed)

- [x] **Logout refresh** — `LogoutButton.tsx` calls `router.refresh()` after signOut
- [x] **Chinese slug fix** — `slugify` returns empty for pure Chinese titles; fallback to timestamp
- [x] **Background not displaying** — Switched from CSS variable to `<style>` tag injection; added `dynamic = "force-dynamic"`
- [x] **Deleted 22.jpg** — Removed hardcoded 711KB background image from repo

---

### Task 20: Anonymous Q&A System (Completed)

**Files:**
- Create: `prisma/schema.prisma` — Added `Question` model
- Create: `src/actions/questions.ts`
- Create: `src/app/qa/page.tsx`
- Create: `src/components/QaForm.tsx`
- Create: `src/components/QuestionList.tsx`
- Modify: `src/components/NavBar.tsx` — Added "问答" nav link

- [x] **Step 1: Add Question model** — content, userId, reply, repliedAt
- [x] **Step 2: Create server actions** — createQuestion, replyToQuestion, deleteQuestion
- [x] **Step 3: Create QA page** — Role-based: users see own questions, admin sees all with real names
- [x] **Step 4: Rich text support** — Markdown rendering, image/file upload in questions and replies
- [x] **Step 5: Edit replies** — Admin can modify replies after posting

---

### Task 21: Password Management (Completed)

**Files:**
- Modify: `src/actions/auth.ts` — Added `changePassword`, `resetUserPassword`
- Create: `src/components/ChangePasswordForm.tsx`
- Create: `src/components/ResetPasswordButton.tsx`
- Modify: `src/app/admin/settings/page.tsx` — Added password change section
- Modify: `src/app/admin/users/page.tsx` — Added reset password button

- [x] **Step 1: changePassword action** — Admin changes own password, verifies current password
- [x] **Step 2: resetUserPassword action** — Admin resets any user's password (no old password needed)
- [x] **Step 3: Settings page** — Password change card with ChangePasswordForm
- [x] **Step 4: Users page** — ResetPasswordButton per user row

---

### Task 22: User Account Self-Service (Completed)

**Files:**
- Create: `src/app/account/page.tsx`
- Create: `src/components/QuestionDeleteButton.tsx`
- Modify: `src/components/NavBar.tsx` — Username links to `/account`
- Modify: `src/components/QuestionList.tsx` — Delete button on questions

- [x] **Step 1: Create account page** — Show user info, change password, manage questions
- [x] **Step 2: Delete question** — Users delete own, admin deletes any; confirm-then-delete UX
- [x] **Step 3: NavBar link** — Desktop username + mobile drawer link to `/account`

---

### Task 23: Upload API Permission Fix (Completed)

**Files:**
- Modify: `src/app/api/upload/route.ts` — Changed from admin-only to any authenticated user

- [x] Non-admin users can now upload images/files for Q&A questions

---

### Task 24: Delete User (Completed)

**Files:**
- Modify: `src/actions/auth.ts` — Added `deleteUser` action
- Create: `src/components/DeleteUserButton.tsx`
- Modify: `src/app/admin/users/page.tsx` — Added DeleteUserButton + auth checks

- [x] **Step 1: deleteUser action** — Admin-only, cannot delete self or other admins, cascades clean up posts/comments/questions
- [x] **Step 2: DeleteUserButton** — Confirm-then-delete UX, hidden for self and other admins
- [x] **Step 3: Users page** — Shows delete button only for non-admin users who are not the current user
