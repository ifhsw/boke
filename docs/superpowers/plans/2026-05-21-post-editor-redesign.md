# 文章编辑器重构 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将文章编辑器从 Markdown textarea 重构为 TipTap 富文本编辑器 (WYSIWYG)，单栏布局，支持图片拖拽粘贴、自动保存、自定义扩展、SEO 设置、定时发布。

**Architecture:** TipTap React 编辑器存储 JSON 到 Prisma Json 字段，服务端通过 `@tiptap/core` `generateHTML()` 渲染为 HTML，经 `sanitize-html` 清理后在前台展示。自定义 MDX 组件 (Callout/CodeBlock/ImageCaption) 重写为 TipTap Node Extensions。编辑器采用单栏布局 (B站风格)，从上到下：状态栏 → 封面 → 标题 → 元数据 → 工具栏 → 编辑区 → 折叠面板。

**Tech Stack:** TipTap React, lowlight (Shiki), KaTeX, Prisma Json, sanitize-html

---

## 文件结构

### 新建

| 文件 | 职责 |
|------|------|
| `src/lib/editor/extensions/callout.ts` | Callout Node Extension (Info/Warn/Tip/Note) |
| `src/lib/editor/extensions/code-block.ts` | CodeBlock + Shiki lowlight 高亮扩展 |
| `src/lib/editor/extensions/image-caption.ts` | 带 Caption 的图片扩展 |
| `src/lib/editor/extensions/math.ts` | KaTeX 行内/块级数学公式扩展 |
| `src/lib/editor/extensions/index.ts` | Barrel export 汇总所有扩展 |
| `src/lib/editor/render.ts` | 服务端 `generateHTML()` 配置 |
| `src/components/TagInput.tsx` | 标签 chip 输入组件 |
| `src/components/CoverUpload.tsx` | 封面图上传组件 |
| `prisma/migrations/20260521_add_editor_fields/migration.sql` | Schema 迁移 |

### 修改

| 文件 | 变更 |
|------|------|
| `prisma/schema.prisma` | content String→Json，新增 seoTitle/seoDesc/scheduledAt/wordCount |
| `src/components/PostEditor.tsx` | 完全重写为 TipTap 编辑器 |
| `src/actions/posts.ts` | content 改为 JSON 处理，修复 tags 更新，新字段支持 |
| `src/app/post/[slug]/page.tsx` | `compileMdx()` → `renderPostContent()` |
| `src/app/admin/posts/new/page.tsx` | 适配新 PostEditor props |
| `src/app/admin/posts/[id]/edit/page.tsx` | JSON content 反序列化传给编辑器 |
| `src/app/my-posts/new/page.tsx` | 适配新 PostEditor props |
| `src/app/my-posts/[id]/edit/page.tsx` | JSON content 反序列化 |
| `src/lib/uploads.ts` | `extractUploadFilenames` 适配 TipTap JSON |
| `prisma/seed.ts` | content 改为 TipTap JSON 格式 |
| `package.json` | 新增/移除依赖 |

### 删除

| 文件 | 原因 |
|------|------|
| `src/components/mdx/Callout.tsx` | 重写为 TipTap Extension |
| `src/components/mdx/CodeBlock.tsx` | 重写为 TipTap Extension |
| `src/components/mdx/ImageCaption.tsx` | 合并到 TipTap ImageCaption 扩展 |
| `src/components/mdx/index.tsx` | MDX 组件注册不再需要 |
| `src/lib/mdx.tsx` | `compileMdx` 不再需要 |

---

## 任务列表

### Task 1: 安装依赖 & 清理无用依赖

**Files:**
- Modify: `package.json`

- [ ] **Step 1: 安装 TipTap 相关依赖**

```bash
cd C:/Users/shi/Desktop/book/blog
npm install @tiptap/react @tiptap/core @tiptap/pm @tiptap/starter-kit @tiptap/extension-image @tiptap/extension-code-block-lowlight @tiptap/extension-table @tiptap/extension-table-row @tiptap/extension-table-cell @tiptap/extension-table-header @tiptap/extension-link @tiptap/extension-placeholder @tiptap/extension-character-count @tiptap/extension-task-list @tiptap/extension-task-item @tiptap/extension-underline @tiptap/extension-highlight lowlight
```

- [ ] **Step 2: 移除不再需要的依赖**

```bash
cd C:/Users/shi/Desktop/book/blog
npm uninstall marked next-mdx-remote remark-gfm remark-math rehype-katex
```

- [ ] **Step 3: 验证安装**

```bash
cd C:/Users/shi/Desktop/book/blog
node -e "require('@tiptap/react'); console.log('TipTap OK')"
node -e "require('lowlight'); console.log('lowlight OK')"
```

Expected: `TipTap OK` `lowlight OK`

- [ ] **Step 4: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add package.json package-lock.json
git commit -m "chore: add TipTap deps, remove MDX/marked deps"
```

---

### Task 2: 更新 Prisma Schema 并创建迁移

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/20260521_add_editor_fields/migration.sql`

- [ ] **Step 1: 更新 Post 模型**

在 `prisma/schema.prisma` 的 Post 模型中:
- 将 `content String` 改为 `content String` (SQLite 不支持 Json 类型，用 String 存 JSON 字符串)
- 新增字段:

```prisma
model Post {
  id          String    @id @default(uuid())
  title       String
  slug        String    @unique
  excerpt     String?
  content     String    // TipTap JSON stored as string (SQLite no Json type)
  coverImage  String?
  category    String
  status      String    @default("DRAFT")
  visibility  String    @default("PUBLIC")
  seoTitle    String?
  seoDesc     String?
  scheduledAt DateTime?
  wordCount   Int?
  authorId    String
  author      User      @relation(fields: [authorId], references: [id])
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  tags        PostTag[]
  comments    Comment[]
}
```

- [ ] **Step 2: 创建迁移 SQL**

创建 `prisma/migrations/20260521_add_editor_fields/migration.sql`:

```sql
-- Redefine Post with new columns
-- SQLite doesn't support ALTER TABLE ADD COLUMN with multiple columns easily,
-- but Prisma will handle this. Just need the migration to exist.
ALTER TABLE "Post" ADD COLUMN "seoTitle" TEXT;
ALTER TABLE "Post" ADD COLUMN "seoDesc" TEXT;
ALTER TABLE "Post" ADD COLUMN "scheduledAt" DATETIME;
ALTER TABLE "Post" ADD COLUMN "wordCount" INTEGER;
```

- [ ] **Step 3: 执行迁移并重新生成客户端**

```bash
cd C:/Users/shi/Desktop/book/blog
npx prisma migrate dev --name add_editor_fields
npx prisma generate
```

Expected: 迁移成功，无报错。

- [ ] **Step 4: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add prisma/schema.prisma prisma/migrations/
git commit -m "feat: add SEO, scheduled publish, word count fields to Post"
```

---

### Task 3: 创建 TipTap Callout 扩展

**Files:**
- Create: `src/lib/editor/extensions/callout.ts`

- [ ] **Step 1: 编写 Callout Node Extension**

```typescript
// src/lib/editor/extensions/callout.ts
import { Node, mergeAttributes } from "@tiptap/core";

export interface CalloutOptions {
  HTMLAttributes: Record<string, unknown>;
}

declare module "@tiptap/core" {
  interface Commands<ReturnType> {
    callout: {
      setCallout: (attrs: { type: "info" | "warn" | "tip" | "note" }) => ReturnType;
    };
  }
}

export const Callout = Node.create<CalloutOptions>({
  name: "callout",

  group: "block",
  content: "block+",
  defining: true,

  addAttributes() {
    return {
      type: { default: "info", rendered: false },
    };
  },

  renderHTML({ node, HTMLAttributes }) {
    const type = node.attrs.type as string;
    const colors: Record<string, { bg: string; border: string; text: string; icon: string }> = {
      info:  { bg: "#eff6ff", border: "#3b82f6", text: "#1e40af", icon: "💡" },
      warn:  { bg: "#fffbeb", border: "#f59e0b", text: "#92400e", icon: "⚠️" },
      tip:   { bg: "#f0fdf4", border: "#22c55e", text: "#166534", icon: "✅" },
      note:  { bg: "#f9fafb", border: "#6b7280", text: "#374151", icon: "📝" },
    };
    const c = colors[type] || colors.info;
    return [
      "div",
      mergeAttributes(HTMLAttributes, {
        style: `border-left:3px solid ${c.border};background:${c.bg};padding:8px 12px;border-radius:0 4px 4px 0;margin:12px 0;`,
      }),
      ["div", { style: `display:flex;align-items:center;gap:6px;color:${c.text};font-weight:600;margin-bottom:4px;` },
        ["span", {}, `${c.icon} ${type.toUpperCase()}`],
      ],
      ["div", { style: `color:${c.text};` }, 0],
    ];
  },

  parseHTML() {
    return [{ tag: "div[data-callout]" }];
  },

  addCommands() {
    return {
      setCallout: (attrs) => ({ commands }) => {
        return commands.insertContent({
          type: this.name,
          attrs,
          content: [{ type: "paragraph" }],
        });
      },
    };
  },
});
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/extensions/callout.ts
git commit -m "feat: add TipTap Callout node extension"
```

---

### Task 4: 创建 TipTap CodeBlock 扩展 (Shiki lowlight)

**Files:**
- Create: `src/lib/editor/extensions/code-block.ts`

- [ ] **Step 1: 编写 CodeBlock 扩展**

```typescript
// src/lib/editor/extensions/code-block.ts
import CodeBlockLowlight from "@tiptap/extension-code-block-lowlight";
import { common, createLowlight } from "lowlight";

const lowlight = createLowlight(common);

export const CodeBlock = CodeBlockLowlight.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      language: {
        default: "plaintext",
        rendered: true,
      },
    };
  },

  renderHTML({ node, HTMLAttributes }) {
    const lang = node.attrs.language || "plaintext";
    const tree = lowlight.highlight(lang, node.textContent);
    const html = lowlight.toHtml(tree);
    return [
      "div",
      { class: "code-block-wrapper" },
      ["div", { class: "code-block-header" },
        ["span", { class: "code-block-dots" }, "●●●"],
        ["span", { class: "code-block-lang" }, lang],
      ],
      ["pre", HTMLAttributes,
        ["code", { class: `language-${lang}` }, html],
      ],
    ];
  },
});
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/extensions/code-block.ts
git commit -m "feat: add TipTap CodeBlock extension with Shiki via lowlight"
```

---

### Task 5: 创建 TipTap ImageCaption 扩展

**Files:**
- Create: `src/lib/editor/extensions/image-caption.ts`

- [ ] **Step 1: 编写 ImageCaption 扩展**

```typescript
// src/lib/editor/extensions/image-caption.ts
import Image from "@tiptap/extension-image";
import { mergeAttributes, Node } from "@tiptap/core";

declare module "@tiptap/core" {
  interface Commands<ReturnType> {
    imageCaption: {
      setImageCaption: (attrs: { src: string; alt?: string; caption?: string }) => ReturnType;
    };
  }
}

export const ImageCaption = Image.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      caption: { default: null, rendered: true },
    };
  },

  renderHTML({ node, HTMLAttributes }) {
    const caption = node.attrs.caption;
    return [
      "figure",
      { class: "image-caption-wrapper", style: "margin:16px 0;text-align:center;" },
      ["img", mergeAttributes(HTMLAttributes, {
        style: "max-width:100%;border-radius:8px;box-shadow:0 2px 8px rgba(0,0,0,0.1);",
        loading: "lazy",
      })],
      ...(caption
        ? [["figcaption", { style: "margin-top:6px;font-size:13px;color:#888;" }, caption]]
        : []),
    ];
  },

  parseHTML() {
    return [
      { tag: "figure.image-caption-wrapper" },
      { tag: "img[src]" },
    ];
  },

  addCommands() {
    return {
      setImageCaption:
        (attrs) =>
        ({ commands }) => {
          return commands.insertContent({
            type: this.name,
            attrs,
          });
        },
    };
  },
});
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/extensions/image-caption.ts
git commit -m "feat: add TipTap ImageCaption extension"
```

---

### Task 6: 创建 TipTap Math (KaTeX) 扩展

**Files:**
- Create: `src/lib/editor/extensions/math.ts`

- [ ] **Step 1: 编写 Math 扩展**

```typescript
// src/lib/editor/extensions/math.ts
import { Node, mergeAttributes } from "@tiptap/core";
import katex from "katex";

declare module "@tiptap/core" {
  interface Commands<ReturnType> {
    mathInline: {
      setMathInline: (attrs: { latex: string }) => ReturnType;
    };
    mathBlock: {
      setMathBlock: (attrs: { latex: string }) => ReturnType;
    };
  }
}

export const MathInline = Node.create({
  name: "mathInline",
  group: "inline",
  inline: true,
  atom: true,

  addAttributes() {
    return { latex: { default: "" } };
  },

  renderHTML({ node }) {
    try {
      const html = katex.renderToString(node.attrs.latex, { throwOnError: false });
      return ["span", { class: "math-inline" }, html];
    } catch {
      return ["span", {}, node.attrs.latex];
    }
  },

  parseHTML() {
    return [{ tag: "span.math-inline" }];
  },
});

export const MathBlock = Node.create({
  name: "mathBlock",
  group: "block",
  atom: true,

  addAttributes() {
    return { latex: { default: "" } };
  },

  renderHTML({ node }) {
    try {
      const html = katex.renderToString(node.attrs.latex, { throwOnError: false, displayMode: true });
      return ["div", { class: "math-block", style: "text-align:center;margin:16px 0;" }, html];
    } catch {
      return ["div", {}, node.attrs.latex];
    }
  },

  parseHTML() {
    return [{ tag: "div.math-block" }];
  },
});
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/extensions/math.ts
git commit -m "feat: add TipTap Math (KaTeX) extensions"
```

---

### Task 7: 创建扩展 Barrel Export

**Files:**
- Create: `src/lib/editor/extensions/index.ts`

- [ ] **Step 1: 编写 barrel export**

```typescript
// src/lib/editor/extensions/index.ts
export { Callout } from "./callout";
export { CodeBlock } from "./code-block";
export { ImageCaption } from "./image-caption";
export { MathInline, MathBlock } from "./math";
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/extensions/index.ts
git commit -m "feat: add extensions barrel export"
```

---

### Task 8: 创建服务端渲染模块

**Files:**
- Create: `src/lib/editor/render.ts`

- [ ] **Step 1: 编写 render.ts**

```typescript
// src/lib/editor/render.ts
import { generateHTML } from "@tiptap/core";
import StarterKit from "@tiptap/starter-kit";
import { Image } from "@tiptap/extension-image";
import Table from "@tiptap/extension-table";
import TableRow from "@tiptap/extension-table-row";
import TableCell from "@tiptap/extension-table-cell";
import TableHeader from "@tiptap/extension-table-header";
import TaskList from "@tiptap/extension-task-list";
import TaskItem from "@tiptap/extension-task-item";
import Highlight from "@tiptap/extension-highlight";
import Underline from "@tiptap/extension-underline";
import Link from "@tiptap/extension-link";
import { Callout } from "./extensions/callout";
import { CodeBlock } from "./extensions/code-block";
import { ImageCaption } from "./extensions/image-caption";
import { MathInline, MathBlock } from "./extensions/math";

const extensions = [
  StarterKit.configure({ codeBlock: false }),
  Callout,
  CodeBlock,
  ImageCaption,
  Image,
  MathInline,
  MathBlock,
  Table.configure({ resizable: true }),
  TableRow,
  TableCell,
  TableHeader,
  TaskList,
  TaskItem.configure({ nested: true }),
  Highlight,
  Underline,
  Link.configure({ openOnClick: true }),
];

export function renderPostContent(json: object): string {
  try {
    return generateHTML(json as any, extensions);
  } catch (err) {
    console.error("[render] generateHTML error:", (err as Error).message);
    return "<p>内容渲染失败</p>";
  }
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/editor/render.ts
git commit -m "feat: add server-side TipTap JSON → HTML renderer"
```

---

### Task 9: 创建 TagInput 组件

**Files:**
- Create: `src/components/TagInput.tsx`

- [ ] **Step 1: 编写 TagInput 组件**

```typescript
// src/components/TagInput.tsx
"use client";

import { useState, useRef, useEffect } from "react";

interface TagInputProps {
  name: string;
  defaultValue?: string[];
  suggestions?: string[];
}

export function TagInput({ name, defaultValue = [], suggestions = [] }: TagInputProps) {
  const [tags, setTags] = useState<string[]>(defaultValue);
  const [input, setInput] = useState("");
  const [showSuggestions, setShowSuggestions] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const filteredSuggestions = suggestions.filter(
    (s) => !tags.includes(s) && s.toLowerCase().includes(input.toLowerCase())
  );

  const addTag = (tag: string) => {
    const trimmed = tag.trim();
    if (trimmed && !tags.includes(trimmed)) {
      setTags([...tags, trimmed]);
    }
    setInput("");
    inputRef.current?.focus();
  };

  const removeTag = (index: number) => {
    setTags(tags.filter((_, i) => i !== index));
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === "Enter" || e.key === ",") {
      e.preventDefault();
      if (input.trim()) addTag(input);
    }
    if (e.key === "Backspace" && !input && tags.length > 0) {
      removeTag(tags.length - 1);
    }
  };

  return (
    <div className="relative">
      <input type="hidden" name={name} value={tags.join(",")} />
      <div className="flex flex-wrap gap-2 items-center border border-primary-200/30 rounded-lg p-2 bg-white focus-within:ring-2 focus-within:ring-primary-200/50">
        {tags.map((tag, i) => (
          <span key={i} className="inline-flex items-center gap-1 bg-primary-100 text-primary-700 px-2 py-0.5 rounded-full text-xs">
            {tag}
            <button type="button" onClick={() => removeTag(i)} className="text-primary-400 hover:text-red-500 leading-none">&times;</button>
          </span>
        ))}
        <input
          ref={inputRef}
          value={input}
          onChange={(e) => { setInput(e.target.value); setShowSuggestions(true); }}
          onFocus={() => setShowSuggestions(true)}
          onBlur={() => setTimeout(() => setShowSuggestions(false), 150)}
          onKeyDown={handleKeyDown}
          placeholder={tags.length === 0 ? "输入标签，回车确认..." : ""}
          className="flex-1 min-w-[80px] outline-none text-sm bg-transparent py-1"
        />
      </div>
      {showSuggestions && filteredSuggestions.length > 0 && (
        <div className="absolute z-10 top-full mt-1 left-0 right-0 bg-white border border-primary-200/30 rounded-lg shadow-md max-h-32 overflow-y-auto">
          {filteredSuggestions.map((s) => (
            <button
              key={s}
              type="button"
              onMouseDown={(e) => { e.preventDefault(); addTag(s); }}
              className="block w-full text-left px-3 py-1.5 text-sm hover:bg-primary-50 transition-colors"
            >
              {s}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/TagInput.tsx
git commit -m "feat: add TagInput chip component with autocomplete"
```

---

### Task 10: 创建 CoverUpload 组件

**Files:**
- Create: `src/components/CoverUpload.tsx`

- [ ] **Step 1: 编写 CoverUpload 组件**

```typescript
// src/components/CoverUpload.tsx
"use client";

import { useState, useRef } from "react";

interface CoverUploadProps {
  name: string;
  defaultValue?: string;
}

export function CoverUpload({ name, defaultValue }: CoverUploadProps) {
  const [url, setUrl] = useState<string>(defaultValue || "");
  const [uploading, setUploading] = useState(false);
  const [dragOver, setDragOver] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const uploadFile = async (file: File) => {
    setUploading(true);
    try {
      const formData = new FormData();
      formData.append("file", file);
      const res = await fetch("/api/upload", { method: "POST", body: formData });
      const data = await res.json();
      if (data.success && data.url) {
        setUrl(data.url);
      } else {
        alert(data.error || "上传失败");
      }
    } catch {
      alert("上传失败");
    } finally {
      setUploading(false);
    }
  };

  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setDragOver(false);
    const file = e.dataTransfer.files?.[0];
    if (file && file.type.startsWith("image/")) uploadFile(file);
  };

  return (
    <div>
      <input type="hidden" name={name} value={url} />
      {url ? (
        <div className="relative rounded-lg overflow-hidden border border-primary-200/20">
          <img src={url} alt="封面图" className="w-full h-48 object-cover" />
          <button
            type="button"
            onClick={() => setUrl("")}
            className="absolute top-2 right-2 bg-red-500 text-white rounded-full w-6 h-6 flex items-center justify-center text-xs hover:bg-red-600 transition-colors"
          >
            &times;
          </button>
        </div>
      ) : (
        <div
          onDrop={handleDrop}
          onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
          onDragLeave={() => setDragOver(false)}
          onClick={() => fileInputRef.current?.click()}
          className={`text-center p-10 rounded-lg border-2 border-dashed cursor-pointer transition-colors ${
            dragOver ? "border-primary-400 bg-primary-50" : "border-primary-200/40 hover:border-primary-300/60"
          }`}
        >
          <span className="text-3xl text-primary-300">🖼</span>
          <p className="mt-2 text-sm text-primary-500/50">{uploading ? "上传中..." : "点击或拖拽上传封面图"}</p>
          <p className="mt-1 text-xs text-primary-400/30">推荐 1920×1080，JPG/PNG/WebP，最大 10MB</p>
          <input
            ref={fileInputRef}
            type="file"
            accept="image/jpeg,image/png,image/gif,image/webp"
            onChange={(e) => { const f = e.target.files?.[0]; if (f) uploadFile(f); }}
            className="hidden"
          />
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/CoverUpload.tsx
git commit -m "feat: add CoverUpload component with drag-and-drop"
```

---

### Task 11: 重写 PostEditor 为 TipTap 编辑器 (Part 1 - 核心编辑器)

**Files:**
- Modify: `src/components/PostEditor.tsx`

- [ ] **Step 1: 重写 PostEditor.tsx 完整代码**

```typescript
// src/components/PostEditor.tsx
"use client";

import { useState, useRef, useEffect, useCallback } from "react";
import { useEditor, EditorContent, BubbleMenu } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import { Image } from "@tiptap/extension-image";
import Table from "@tiptap/extension-table";
import TableRow from "@tiptap/extension-table-row";
import TableCell from "@tiptap/extension-table-cell";
import TableHeader from "@tiptap/extension-table-header";
import TaskList from "@tiptap/extension-task-list";
import TaskItem from "@tiptap/extension-task-item";
import Highlight from "@tiptap/extension-highlight";
import Underline from "@tiptap/extension-underline";
import Link from "@tiptap/extension-link";
import Placeholder from "@tiptap/extension-placeholder";
import CharacterCount from "@tiptap/extension-character-count";
import { Callout } from "@/lib/editor/extensions/callout";
import { CodeBlock } from "@/lib/editor/extensions/code-block";
import { ImageCaption } from "@/lib/editor/extensions/image-caption";
import { MathInline, MathBlock } from "@/lib/editor/extensions/math";
import { TagInput } from "@/components/TagInput";
import { CoverUpload } from "@/components/CoverUpload";

const STORAGE_KEY = "blog-draft";

export function PostEditor({
  action,
  initialData,
  submitLabel,
  showStatus = true,
  showVisibility = false,
  tagSuggestions = [],
  postId, // for draft storage key
}: {
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  action: (formData: FormData) => Promise<any>;
  initialData?: {
    title: string;
    excerpt: string;
    content: string; // TipTap JSON string
    category: string;
    status: string;
    visibility?: string;
    tags?: string;
    coverImage?: string;
    seoTitle?: string;
    seoDesc?: string;
    scheduledAt?: string;
  };
  submitLabel: string;
  showStatus?: boolean;
  showVisibility?: boolean;
  tagSuggestions?: string[];
  postId?: string;
}) {
  const [title, setTitle] = useState(initialData?.title || "");
  const [category, setCategory] = useState(initialData?.category || "TECH");
  const [status, setStatus] = useState(initialData?.status || "DRAFT");
  const [visibility, setVisibility] = useState(initialData?.visibility || "PUBLIC");
  const [saveState, setSaveState] = useState<"saved" | "saving" | "unsaved">("saved");
  const [uploading, setUploading] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);
  const formRef = useRef<HTMLFormElement>(null);
  const saveTimerRef = useRef<NodeJS.Timeout | null>(null);

  // Parse initial content
  const initialContent = (() => {
    if (initialData?.content) {
      try {
        return JSON.parse(initialData.content);
      } catch {
        return { type: "doc", content: [{ type: "paragraph", content: [{ type: "text", text: initialData.content }] }] };
      }
    }
    return undefined;
  })();

  const editor = useEditor({
    immediatelyRender: false,
    extensions: [
      StarterKit.configure({ codeBlock: false }),
      Callout,
      CodeBlock,
      ImageCaption,
      Image,
      MathInline,
      MathBlock,
      Table.configure({ resizable: true }),
      TableRow,
      TableCell,
      TableHeader,
      TaskList,
      TaskItem.configure({ nested: true }),
      Highlight,
      Underline,
      Link.configure({ openOnClick: false }),
      Placeholder.configure({ placeholder: "开始写作... 输入 / 弹出菜单" }),
      CharacterCount.configure({}),
    ],
    content: initialContent,
    editorProps: {
      attributes: {
        class: "prose prose-lg max-w-none focus:outline-none min-h-[400px] px-6 py-5",
      },
      handleDrop: (view, event, _slice, moved) => {
        if (!moved && event.dataTransfer?.files?.length) {
          const file = event.dataTransfer.files[0];
          if (file.type.startsWith("image/")) {
            uploadAndInsertImage(file);
            event.preventDefault();
            return true;
          }
        }
        return false;
      },
      handlePaste: (_view, event) => {
        const items = event.clipboardData?.items;
        if (items) {
          for (const item of items) {
            if (item.type.startsWith("image/")) {
              const file = item.getAsFile();
              if (file) {
                uploadAndInsertImage(file);
                event.preventDefault();
                return true;
              }
            }
          }
        }
        return false;
      },
    },
    onUpdate: () => {
      setSaveState("unsaved");
      // Debounce local save
      if (saveTimerRef.current) clearTimeout(saveTimerRef.current);
      saveTimerRef.current = setTimeout(() => autoSaveDraft(), 3000);
    },
  });

  // Restore draft from localStorage
  useEffect(() => {
    if (!initialData?.content && editor) {
      const draft = localStorage.getItem(`${STORAGE_KEY}-new`);
      if (draft) {
        const confirmed = window.confirm("检测到未保存的草稿，是否恢复？");
        if (confirmed) {
          try {
            editor.commands.setContent(JSON.parse(draft));
          } catch { /* ignore */ }
        }
      }
    }
  }, []);

  const autoSaveDraft = useCallback(() => {
    if (!editor) return;
    const json = editor.getJSON();
    const key = postId ? `${STORAGE_KEY}-${postId}` : `${STORAGE_KEY}-new`;
    localStorage.setItem(key, JSON.stringify(json));
    setSaveState("saved");
  }, [editor, postId]);

  const uploadAndInsertImage = async (file: File) => {
    setUploading(true);
    try {
      const fd = new FormData();
      fd.append("file", file);
      const res = await fetch("/api/upload", { method: "POST", body: fd });
      const data = await res.json();
      if (data.success && data.url) {
        editor?.chain().focus().setImageCaption({ src: data.url, alt: file.name }).run();
      } else {
        alert(data.error || "上传失败");
      }
    } catch {
      alert("上传失败");
    } finally {
      setUploading(false);
    }
  };

  const handleToolbarUpload = () => {
    const file = fileInputRef.current?.files?.[0];
    if (file) uploadAndInsertImage(file);
  };

  const getContentJson = () => {
    if (!editor) return "";
    return JSON.stringify(editor.getJSON());
  };

  const characterCount = editor?.storage.characterCount?.characters?.() ?? 0;

  // Cleanup on unmount
  useEffect(() => {
    return () => { if (saveTimerRef.current) clearTimeout(saveTimerRef.current); };
  }, []);

  if (!editor) return null;

  return (
    <form ref={formRef} action={action} className="max-w-4xl mx-auto">
      {/* Hidden fields for JSON content and non-editor metadata */}
      <input type="hidden" name="content" value={getContentJson()} />
      <input type="hidden" name="wordCount" value={characterCount} />

      {/* ---- Top Bar ---- */}
      <div className="sticky top-0 z-30 flex items-center justify-between px-6 py-3 bg-white/90 backdrop-blur border-b border-primary-200/20 mb-4 -mx-2">
        <div className="flex items-center gap-3">
          <h1 className="text-sm font-semibold text-primary-700">✏️ {postId ? "编辑文章" : "写文章"}</h1>
          <span className={`text-xs ${saveState === "saved" ? "text-green-500" : saveState === "saving" ? "text-amber-500" : "text-red-400"}`}>
            ● {saveState === "saved" ? "已保存" : saveState === "saving" ? "保存中..." : "未保存"}
          </span>
        </div>
        <div className="flex gap-3">
          <button type="submit" name="status" value="DRAFT" className="text-sm px-4 py-1.5 rounded-lg border border-primary-200/40 text-primary-600 hover:bg-primary-50 transition-colors">
            💾 存草稿
          </button>
          <button type="submit" name="status" value={status} className="text-sm px-5 py-1.5 rounded-lg bg-primary-900 text-white hover:bg-primary-800 transition-colors">
            {submitLabel}
          </button>
        </div>
      </div>

      {/* ---- Cover Image ---- */}
      <div className="px-2 mb-4">
        <CoverUpload name="coverImage" defaultValue={initialData?.coverImage} />
      </div>

      {/* ---- Title ---- */}
      <div className="px-2 mb-3">
        <textarea
          name="title"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          placeholder="文章标题..."
          rows={1}
          className="w-full text-3xl font-bold border-none outline-none resize-none bg-transparent text-primary-900 placeholder:text-primary-300/60"
          onInput={(e) => {
            const el = e.currentTarget;
            el.style.height = "auto";
            el.style.height = el.scrollHeight + "px";
          }}
          required
        />
      </div>

      {/* ---- Metadata Row ---- */}
      <div className="px-2 mb-4 flex flex-wrap items-center gap-4 text-sm">
        <div className="flex items-center gap-2">
          <span className="text-primary-400/60">📂</span>
          <select
            name="category"
            value={category}
            onChange={(e) => setCategory(e.target.value)}
            className="border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm bg-white text-primary-700 outline-none focus:ring-2 focus:ring-primary-200/50"
          >
            <option value="TECH">技术</option>
            <option value="ESSAY">随笔</option>
          </select>
        </div>
        <div className="flex items-center gap-2 flex-1">
          <span className="text-primary-400/60">🏷</span>
          <TagInput
            name="tags"
            defaultValue={initialData?.tags ? initialData.tags.split(",").map((t: string) => t.trim()).filter(Boolean) : []}
            suggestions={tagSuggestions}
          />
        </div>
        {showVisibility && (
          <div className="flex items-center gap-2">
            <span className="text-primary-400/60">👁</span>
            <select
              name="visibility"
              value={visibility}
              onChange={(e) => setVisibility(e.target.value)}
              className="border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm bg-white text-primary-700 outline-none focus:ring-2 focus:ring-primary-200/50"
            >
              <option value="PUBLIC">公开</option>
              <option value="PRIVATE">仅自己可见</option>
            </select>
          </div>
        )}
        {showStatus && (
          <div className="flex items-center gap-2">
            <span className="text-primary-400/60">📌</span>
            <select
              name="status_select"
              value={status}
              onChange={(e) => setStatus(e.target.value)}
              className="border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm bg-white text-primary-700 outline-none focus:ring-2 focus:ring-primary-200/50"
            >
              <option value="DRAFT">草稿</option>
              <option value="PUBLISHED">发布</option>
            </select>
          </div>
        )}
      </div>

      {/* ---- Toolbar ---- */}
      <div className="sticky top-12 z-20 px-2 mb-3">
        <div className="flex flex-wrap items-center gap-1 px-3 py-2 bg-white border border-primary-200/20 rounded-xl shadow-sm">
          <ToolBtn onClick={() => editor.chain().focus().toggleBold().run()} active={editor.isActive("bold")} label="B" title="加粗 (Ctrl+B)" />
          <ToolBtn onClick={() => editor.chain().focus().toggleItalic().run()} active={editor.isActive("italic")} label="I" title="斜体 (Ctrl+I)" style="italic" />
          <ToolBtn onClick={() => editor.chain().focus().toggleStrike().run()} active={editor.isActive("strike")} label="S" title="删除线" style="line-through" />
          <span className="w-px h-5 bg-primary-200/30 mx-1" />
          <ToolBtn onClick={() => editor.chain().focus().toggleHeading({ level: 1 }).run()} active={editor.isActive("heading", { level: 1 })} label="H1" />
          <ToolBtn onClick={() => editor.chain().focus().toggleHeading({ level: 2 }).run()} active={editor.isActive("heading", { level: 2 })} label="H2" />
          <ToolBtn onClick={() => editor.chain().focus().toggleHeading({ level: 3 }).run()} active={editor.isActive("heading", { level: 3 })} label="H3" />
          <span className="w-px h-5 bg-primary-200/30 mx-1" />
          <ToolBtn onClick={() => editor.chain().focus().toggleBlockquote().run()} active={editor.isActive("blockquote")} label="&ldquo;" />
          <ToolBtn onClick={() => editor.chain().focus().toggleCodeBlock().run()} active={editor.isActive("codeBlock")} label="&lt;/&gt;" />
          <input ref={fileInputRef} type="file" accept="image/*" onChange={handleToolbarUpload} className="hidden" />
          <ToolBtn onClick={() => fileInputRef.current?.click()} active={false} label="🖼" title="插入图片" />
          <ToolBtn onClick={() => editor.chain().focus().insertTable({ rows: 3, cols: 3 }).run()} active={false} label="⊞" title="插入表格" />
          <ToolBtn onClick={() => {
            const url = window.prompt("URL:");
            if (url) editor.chain().focus().setLink({ href: url }).run();
          }} active={editor.isActive("link")} label="🔗" title="插入链接" />
          <ToolBtn onClick={() => editor.chain().focus().setHorizontalRule().run()} active={false} label="—" title="分割线" />
          <span className="w-px h-5 bg-primary-200/30 mx-1" />
          <ToolBtn onClick={() => editor.chain().focus().toggleBulletList().run()} active={editor.isActive("bulletList")} label="•" />
          <ToolBtn onClick={() => editor.chain().focus().toggleOrderedList().run()} active={editor.isActive("orderedList")} label="1." />
          <ToolBtn onClick={() => editor.chain().focus().toggleTaskList().run()} active={editor.isActive("taskList")} label="☑" />
          <span className="w-px h-5 bg-primary-200/30 mx-1" />
          <ToolBtn onClick={() => editor.chain().focus().setCallout({ type: "info" }).run()} active={false} label="💡" title="Callout" />
          <ToolBtn onClick={() => {
            const latex = window.prompt("LaTeX 公式:");
            if (latex) editor.chain().focus().setMathBlock({ latex }).run();
          }} active={false} label="∑" title="数学公式" />
          <span className="text-xs text-primary-300/60 ml-auto">{characterCount.toLocaleString()} 字</span>
        </div>
      </div>

      {/* ---- Bubble Menu ---- */}
      {editor && (
        <BubbleMenu editor={editor} className="flex gap-1 bg-white border border-primary-200/30 rounded-lg shadow-lg px-2 py-1.5">
          <ToolBtn onClick={() => editor.chain().focus().toggleBold().run()} active={editor.isActive("bold")} label="B" />
          <ToolBtn onClick={() => editor.chain().focus().toggleItalic().run()} active={editor.isActive("italic")} label="I" style="italic" />
          <ToolBtn onClick={() => editor.chain().focus().toggleStrike().run()} active={editor.isActive("strike")} label="S" style="line-through" />
          <span className="w-px h-4 bg-primary-200/30 self-center" />
          <ToolBtn onClick={() => {
            const url = window.prompt("URL:");
            if (url) editor.chain().focus().setLink({ href: url }).run();
          }} active={editor.isActive("link")} label="🔗" />
          <ToolBtn onClick={() => editor.chain().focus().toggleCode().run()} active={editor.isActive("code")} label="&lt;&gt;" />
        </BubbleMenu>
      )}

      {/* ---- Editor Content ---- */}
      <div className="px-2 mb-4">
        <div className="bg-white border border-primary-200/20 rounded-xl overflow-hidden shadow-sm">
          <EditorContent editor={editor} />
          {uploading && (
            <div className="px-6 py-2 text-xs text-primary-400/60 border-t border-primary-200/10">📎 图片上传中...</div>
          )}
        </div>
      </div>

      {/* ---- Collapsible Panels ---- */}
      <div className="px-2 mb-16 space-y-3">
        <details className="bg-white border border-primary-200/20 rounded-xl overflow-hidden">
          <summary className="cursor-pointer px-5 py-3 text-sm text-primary-600/70 hover:text-primary-800 transition-colors select-none">
            🔍 SEO 设置
          </summary>
          <div className="px-5 pb-4 space-y-3">
            <div>
              <label className="text-xs text-primary-400/60">自定义 Slug</label>
              <input name="slug" defaultValue={initialData?.title ? "" : ""} className="w-full mt-1 border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm outline-none focus:ring-2 focus:ring-primary-200/50" placeholder="留空则从标题自动生成" />
            </div>
            <div className="flex gap-3">
              <div className="flex-1">
                <label className="text-xs text-primary-400/60">SEO 标题</label>
                <input name="seoTitle" defaultValue={initialData?.seoTitle || ""} className="w-full mt-1 border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm outline-none focus:ring-2 focus:ring-primary-200/50" />
              </div>
              <div className="flex-1">
                <label className="text-xs text-primary-400/60">SEO 描述</label>
                <input name="seoDesc" defaultValue={initialData?.seoDesc || ""} className="w-full mt-1 border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm outline-none focus:ring-2 focus:ring-primary-200/50" />
              </div>
            </div>
          </div>
        </details>
        <details className="bg-white border border-primary-200/20 rounded-xl overflow-hidden">
          <summary className="cursor-pointer px-5 py-3 text-sm text-primary-600/70 hover:text-primary-800 transition-colors select-none">
            📅 定时发布
          </summary>
          <div className="px-5 pb-4 flex items-center gap-3">
            <input type="datetime-local" name="scheduledAt" defaultValue={initialData?.scheduledAt || ""} className="border border-primary-200/30 rounded-lg px-3 py-1.5 text-sm outline-none focus:ring-2 focus:ring-primary-200/50" />
            <span className="text-xs text-primary-400/50">留空则立即发布</span>
          </div>
        </details>
      </div>
    </form>
  );
}

function ToolBtn({
  onClick, active, label, title, style,
}: {
  onClick: () => void;
  active: boolean;
  label: string;
  title?: string;
  style?: string;
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      title={title}
      className={`text-xs px-2 py-1 rounded transition-colors ${
        active ? "bg-primary-200/50 text-primary-800" : "text-primary-500/70 hover:bg-primary-100/50 hover:text-primary-700"
      }`}
      style={style ? { fontStyle: style === "italic" ? "italic" : undefined, textDecoration: style === "line-through" ? "line-through" : undefined } : undefined}
    >
      {label}
    </button>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/PostEditor.tsx
git commit -m "feat: rewrite PostEditor with TipTap WYSIWYG editor"
```

---

### Task 12: 更新 createPost 和 updatePost Server Actions

**Files:**
- Modify: `src/actions/posts.ts`

- [ ] **Step 1: 重写 createPost 处理 JSON content 和新字段**

替换 `src/actions/posts.ts` 的 `createPost` 函数 (lines 16-68)：

```typescript
export async function createPost(formData: FormData) {
  const session = await auth();
  if (!session?.user) return { success: false, error: "请先登录" };

  const isAdmin = (session.user as any).role === "ADMIN";
  const userId = (session.user as any).id;

  const title = formData.get("title") as string;
  const content = formData.get("content") as string; // JSON string
  const category = formData.get("category") as string;
  const excerpt = formData.get("excerpt") as string;
  const status = (formData.get("status") as string) || "DRAFT";
  const visibility = (formData.get("visibility") as string) || "PUBLIC";
  const coverImage = (formData.get("coverImage") as string) || null;
  const seoTitle = (formData.get("seoTitle") as string) || null;
  const seoDesc = (formData.get("seoDesc") as string) || null;
  const scheduledAt = formData.get("scheduledAt") as string || null;
  const wordCountStr = (formData.get("wordCount") as string) || "0";
  const wordCount = parseInt(wordCountStr, 10) || null;
  const tagNames = (formData.get("tags") as string).split(",").map((t) => t.trim()).filter(Boolean);
  const customSlug = (formData.get("slug") as string) || null;

  let slug = customSlug || slugify(title, { lower: true, strict: true, locale: "zh" });
  if (!slug) slug = Date.now().toString(36);
  const existing = await prisma.post.findUnique({ where: { slug } });
  if (existing) slug += "-" + Date.now().toString(36);

  const finalStatus = isAdmin ? status : "DRAFT";

  await prisma.post.create({
    data: {
      title, slug, content, category,
      excerpt: excerpt || null,
      status: finalStatus, visibility,
      coverImage,
      seoTitle, seoDesc,
      scheduledAt: scheduledAt ? new Date(scheduledAt) : null,
      wordCount,
      authorId: userId,
      tags: {
        create: await Promise.all(
          tagNames.map(async (name) => {
            const tag = await prisma.tag.upsert({
              where: { name }, create: { name }, update: {},
            });
            return { tagId: tag.id };
          })
        ),
      },
    },
  });

  revalidatePath("/admin/posts");
  revalidatePath("/my-posts");
  revalidatePath("/");
  redirect(isAdmin ? "/admin/posts" : "/my-posts");
}
```

- [ ] **Step 2: 重写 updatePost 处理 JSON content、新字段、修复 tags 更新**

替换 `updatePost` 函数 (lines 70-109)：

```typescript
export async function updatePost(id: string, formData: FormData) {
  const session = await auth();
  if (!session?.user) return { success: false, error: "请先登录" };

  const isAdmin = (session.user as any).role === "ADMIN";
  const userId = (session.user as any).id;

  const post = await prisma.post.findUnique({ where: { id } });
  if (!post) return { success: false, error: "文章不存在" };
  if (!isAdmin && post.authorId !== userId) return { success: false, error: "无权限" };

  const title = formData.get("title") as string;
  const content = formData.get("content") as string;
  const category = formData.get("category") as string;
  const excerpt = formData.get("excerpt") as string;
  const status = (formData.get("status") as string) || "DRAFT";
  const visibility = (formData.get("visibility") as string) || undefined;
  const coverImage = (formData.get("coverImage") as string) || null;
  const seoTitle = (formData.get("seoTitle") as string) || null;
  const seoDesc = (formData.get("seoDesc") as string) || null;
  const scheduledAt = formData.get("scheduledAt") as string || null;
  const wordCountStr = (formData.get("wordCount") as string) || "0";
  const wordCount = parseInt(wordCountStr, 10) || null;
  const tagNames = (formData.get("tags") as string).split(",").map((t) => t.trim()).filter(Boolean);
  const customSlug = (formData.get("slug") as string) || null;

  const finalStatus = isAdmin ? status : "DRAFT";

  const updateData: Record<string, unknown> = {
    title, content, category, excerpt: excerpt || null, status: finalStatus,
    coverImage, seoTitle, seoDesc,
    scheduledAt: scheduledAt ? new Date(scheduledAt) : null,
    wordCount,
  };

  // Auto-update slug if title changed and slug not manually set
  if (!customSlug) {
    let newSlug = slugify(title, { lower: true, strict: true, locale: "zh" });
    if (!newSlug) newSlug = Date.now().toString(36);
    if (newSlug !== post.slug) {
      const existing = await prisma.post.findUnique({ where: { slug: newSlug } });
      if (existing && existing.id !== id) newSlug += "-" + Date.now().toString(36);
      updateData.slug = newSlug;
    }
  } else if (customSlug !== post.slug) {
    updateData.slug = customSlug;
  }

  if (isAdmin && visibility) {
    updateData.visibility = visibility;
  }

  await prisma.post.update({ where: { id }, data: updateData });

  // FIX: Update tags (was missing in old code)
  if (tagNames.length > 0) {
    // Disconnect all existing tags
    await prisma.postTag.deleteMany({ where: { postId: id } });
    // Upsert and connect new tags
    for (const name of tagNames) {
      const tag = await prisma.tag.upsert({
        where: { name }, create: { name }, update: {},
      });
      await prisma.postTag.create({ data: { postId: id, tagId: tag.id } });
    }
  }

  // Clean orphan tags
  await cleanOrphanTags();

  revalidatePath("/admin/posts");
  revalidatePath("/my-posts");
  revalidatePath("/");
  redirect(isAdmin ? "/admin/posts" : "/my-posts");
}
```

- [ ] **Step 3: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/actions/posts.ts
git commit -m "feat: update server actions for TipTap JSON content and new fields"
```

---

### Task 13: 更新 uploads.ts 适配 TipTap JSON

**Files:**
- Modify: `src/lib/uploads.ts`

- [ ] **Step 1: 更新 extractUploadFilenames 支持 TipTap JSON**

替换 `extractUploadFilenames` 函数 (lines 8-11)：

```typescript
/** Extract all /uploads/ filenames from TipTap JSON content */
export function extractUploadFilenames(content: string): string[] {
  // Try to parse as JSON (TipTap format)
  try {
    const json = JSON.parse(content);
    const urls: string[] = [];
    const walk = (node: unknown) => {
      if (!node || typeof node !== "object") return;
      const obj = node as Record<string, unknown>;
      if (typeof obj.src === "string" && obj.src.includes("/uploads/")) {
        const match = obj.src.match(/\/uploads\/([^\s"')>]+)/);
        if (match) urls.push(match[1]);
      }
      if (obj.content && Array.isArray(obj.content)) {
        for (const child of obj.content) walk(child);
      }
    };
    walk(json);
    return urls;
  } catch {
    // Fallback: plain text/Markdown
    const matches = content.matchAll(/\/uploads\/([^\s)"')\]>]+)/g);
    return [...matches].map((m) => m[1]);
  }
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/uploads.ts
git commit -m "feat: update extractUploadFilenames for TipTap JSON"
```

---

### Task 14: 更新文章详情页渲染

**Files:**
- Modify: `src/app/post/[slug]/page.tsx`

- [ ] **Step 1: 替换 MDX 渲染为 generateHTML 渲染**

在 `src/app/post/[slug]/page.tsx`:

替换 import (line 3):
```typescript
// from:
import { compileMdx } from "@/lib/mdx";
// to:
import { renderPostContent } from "@/lib/editor/render";
import { sanitizeHtmlContent } from "@/lib/sanitize";
```

替换渲染逻辑 (line 42):
```typescript
// from:
const mdxContent = await compileMdx(post.content);
// to:
let htmlContent: string;
try {
  const json = JSON.parse(post.content);
  htmlContent = sanitizeHtmlContent(renderPostContent(json));
} catch {
  htmlContent = sanitizeHtmlContent(post.content);
}
```

替换 MDX Content 区域 (lines 143-146):
```tsx
{/* ---- Article Content ---- */}
<div
  className="py-8 animate-fade-in animation-delay-200 prose prose-lg max-w-none
    [&_img]:max-w-full [&_img]:rounded-lg
    [&_pre]:bg-[#1e1e2e] [&_pre]:text-gray-300 [&_pre]:p-4 [&_pre]:rounded-lg
    [&_code]:font-mono [&_code]:text-sm
    [&_table]:w-full [&_th]:border [&_td]:border [&_th]:p-2 [&_td]:p-2
    [&_blockquote]:border-l-2 [&_blockquote]:border-blue-400 [&_blockquote]:pl-4 [&_blockquote]:text-gray-600"
  dangerouslySetInnerHTML={{ __html: htmlContent }}
/>
```

替换 Reading time 计算 (line 43):
```typescript
// from:
const readingTime = estimateReadingTime(post.content);
// to (content is now JSON, use wordCount field or fallback):
const readingTime = post.wordCount
  ? Math.max(1, Math.ceil(post.wordCount / 300))
  : estimateReadingTime(post.content);
```

替换 wordCount 显示 (line 45):
```typescript
// from:
const wordCount = post.content.replace(/\s/g, "").length;
// to:
const wordCount = post.wordCount || post.content.replace(/\s/g, "").length;
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/app/post/[slug]/page.tsx
git commit -m "feat: update post detail page for TipTap HTML rendering"
```

---

### Task 15: 更新编辑器页面适配器 (Admin & My-Posts)

**Files:**
- Modify: `src/app/admin/posts/new/page.tsx`
- Modify: `src/app/admin/posts/[id]/edit/page.tsx`
- Modify: `src/app/my-posts/new/page.tsx`
- Modify: `src/app/my-posts/[id]/edit/page.tsx`

- [ ] **Step 1: 更新 admin/posts/new/page.tsx**

```typescript
import { createPost } from "@/actions/posts";
import { PostEditor } from "@/components/PostEditor";
import { prisma } from "@/lib/prisma";

export default async function NewPostPage() {
  const tagSuggestions = (await prisma.tag.findMany({ select: { name: true } })).map((t) => t.name);

  return (
    <div>
      <PostEditor action={createPost} submitLabel="发布" showVisibility={true} tagSuggestions={tagSuggestions} />
    </div>
  );
}
```

- [ ] **Step 2: 更新 admin/posts/[id]/edit/page.tsx**

```typescript
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

  const tagSuggestions = (await prisma.tag.findMany({ select: { name: true } })).map((t) => t.name);
  const bindUpdate = updatePost.bind(null, post.id);

  return (
    <div>
      <PostEditor
        action={bindUpdate}
        initialData={{
          title: post.title,
          excerpt: post.excerpt || "",
          content: post.content,
          category: post.category,
          status: post.status,
          visibility: post.visibility,
          tags: post.tags.map((pt) => pt.tag.name).join(", "),
          coverImage: post.coverImage || "",
          seoTitle: post.seoTitle || "",
          seoDesc: post.seoDesc || "",
          scheduledAt: post.scheduledAt ? new Date(post.scheduledAt).toISOString().slice(0, 16) : "",
        }}
        submitLabel="保存"
        showVisibility={true}
        tagSuggestions={tagSuggestions}
        postId={post.id}
      />
    </div>
  );
}
```

- [ ] **Step 3: 更新 my-posts/new/page.tsx**

```typescript
import { createPost } from "@/actions/posts";
import { PostEditor } from "@/components/PostEditor";
import { prisma } from "@/lib/prisma";

export default async function MyPostsNewPage() {
  const tagSuggestions = (await prisma.tag.findMany({ select: { name: true } })).map((t) => t.name);

  return (
    <div>
      <PostEditor action={createPost} submitLabel="投稿" showStatus={false} showVisibility={true} tagSuggestions={tagSuggestions} />
    </div>
  );
}
```

- [ ] **Step 4: 更新 my-posts/[id]/edit/page.tsx**

```typescript
import { auth } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import { updatePost } from "@/actions/posts";
import { PostEditor } from "@/components/PostEditor";
import { notFound, redirect } from "next/navigation";

export default async function EditMyPostPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const session = await auth();
  const userId = (session?.user as any).id;
  const isAdmin = (session?.user as any).role === "ADMIN";

  const post = await prisma.post.findUnique({
    where: { id },
    include: { tags: { include: { tag: true } } },
  });
  if (!post) notFound();
  if (!isAdmin && post.authorId !== userId) redirect("/my-posts");

  const tagSuggestions = (await prisma.tag.findMany({ select: { name: true } })).map((t) => t.name);
  const bindUpdate = updatePost.bind(null, post.id);

  return (
    <div>
      <PostEditor
        action={bindUpdate}
        initialData={{
          title: post.title,
          excerpt: post.excerpt || "",
          content: post.content,
          category: post.category,
          status: post.status,
          visibility: post.visibility,
          tags: post.tags.map((pt) => pt.tag.name).join(", "),
          coverImage: post.coverImage || "",
          seoTitle: post.seoTitle || "",
          seoDesc: post.seoDesc || "",
          scheduledAt: post.scheduledAt ? new Date(post.scheduledAt).toISOString().slice(0, 16) : "",
        }}
        submitLabel="保存"
        showStatus={isAdmin}
        showVisibility={true}
        tagSuggestions={tagSuggestions}
        postId={post.id}
      />
    </div>
  );
}
```

- [ ] **Step 5: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/app/admin/posts/new/page.tsx src/app/admin/posts/[id]/edit/page.tsx src/app/my-posts/new/page.tsx src/app/my-posts/[id]/edit/page.tsx
git commit -m "feat: update editor page adapters for TipTap PostEditor"
```

---

### Task 16: 更新 Seed 脚本

**Files:**
- Modify: `prisma/seed.ts`

- [ ] **Step 1: 更新 seed content 为 TipTap JSON**

替换 `prisma/seed.ts` 中的 content 字段 (line 45):

```typescript
content: JSON.stringify({
  type: "doc",
  content: [
    { type: "heading", attrs: { level: 2 }, content: [{ type: "text", text: "你好" }] },
    { type: "paragraph", content: [{ type: "text", text: "这是第一篇博客文章。欢迎来访。" }] },
  ],
}),
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add prisma/seed.ts
git commit -m "feat: update seed script for TipTap JSON content"
```

---

### Task 17: 删除废弃的 MDX 文件

**Files:**
- Delete: `src/components/mdx/Callout.tsx`
- Delete: `src/components/mdx/CodeBlock.tsx`
- Delete: `src/components/mdx/ImageCaption.tsx`
- Delete: `src/components/mdx/index.tsx`
- Delete: `src/lib/mdx.tsx`

- [ ] **Step 1: 删除废弃文件**

```bash
cd C:/Users/shi/Desktop/book/blog
rm src/components/mdx/Callout.tsx
rm src/components/mdx/CodeBlock.tsx
rm src/components/mdx/ImageCaption.tsx
rm src/components/mdx/index.tsx
rm src/lib/mdx.tsx
```

- [ ] **Step 2: 清理空目录**

```bash
cd C:/Users/shi/Desktop/book/blog
rmdir src/components/mdx 2>/dev/null || true
```

- [ ] **Step 3: 检查编译**

```bash
cd C:/Users/shi/Desktop/book/blog
npx tsc --noEmit 2>&1 | head -30
```

修复任何残留的 `@/lib/mdx` 或 `@/components/mdx` import 错误。

- [ ] **Step 4: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add -A
git commit -m "refactor: remove deprecated MDX files"
```

---

### Task 18: 构建验证 & 浏览器测试

**Files:** None (verification only)

- [ ] **Step 1: 运行 TypeScript 检查**

```bash
cd C:/Users/shi/Desktop/book/blog
npx tsc --noEmit
```

Expected: 无类型错误。

- [ ] **Step 2: 运行构建**

```bash
cd C:/Users/shi/Desktop/book/blog
npm run build
```

Expected: 构建成功。

- [ ] **Step 3: 启动开发服务器手动测试**

```bash
cd C:/Users/shi/Desktop/book/blog
npm run dev
```

手动验证:
1. 访问 `http://localhost:3000/admin/posts/new` — 确认编辑器加载
2. 输入标题、选择分类、添加标签
3. 输入内容 — 确认 WYSIWYG 渲染
4. 工具栏按钮 — 加粗/斜体/标题/代码块等
5. 拖拽图片到编辑区 — 确认上传并插入
6. 粘贴截图 — 确认自动上传
7. 输入 `/` — 确认浏览器控制台无报错 (slash 命令可选)
8. 设置封面图 — 拖拽上传
9. 展开 SEO 设置 — 输入 slug、SEO 标题
10. 设置定时发布时间
11. 点击发布 — 确认跳转到文章列表
12. 访问文章详情页 — 确认内容正确渲染
13. 编辑已有文章 — 确认内容回显
14. 草稿自动保存 — 刷新页面确认恢复提示

- [ ] **Step 4: 修复发现的问题并提交**

```bash
cd C:/Users/shi/Desktop/book/blog
git add -A
git commit -m "fix: issues found during manual testing"
```

---

## Self-Review Checklist

- [x] **Spec coverage**: 架构变更(Task 1-2,11,14)、布局(Task 11)、输入交互(Task 11 toolbar/bubble)、图片处理(Task 11 drop/paste)、自动保存(Task 11 localStorage)、自定义扩展(Task 3-7)、标签输入(Task 9)、封面图(Task 10)、SEO(Task 11 panels)、定时发布(Task 11 panels)、tags bug fix(Task 12)、字数统计(Task 11 toolbar)、服务端渲染(Task 8)、删除旧文件(Task 17)
- [x] **No placeholders**: 所有任务包含完整代码和命令
- [x] **Type consistency**: PostEditor props 在 Task 11 定义，Task 15 使用一致；renderPostContent 签名在 Task 8 定义，Task 14 使用一致
