# 文章编辑器重构设计

> 日期：2026-05-21
> 状态：已确认
> 目标：参考 Halo / B站 / 掘金，重构文章写作体验为 TipTap 富文本编辑器

---

## 一、架构变更

### 1.1 编辑器替换

| 层面 | 改前 | 改后 |
|------|------|------|
| 编辑器 | `<textarea>` + `marked` 预览 | `@tiptap/react` 组件 |
| 存储格式 | Markdown 字符串 | TipTap JSON (Prisma Json) |
| 服务端渲染 | `compileMdx()` → MDX 组件 | `generateHTML()` → sanitized HTML |
| 自定义组件 | MDX Callout/CodeBlock/ImageCaption | TipTap Node Extensions |

### 1.2 依赖变更

**新增:**
- `@tiptap/react` — React 编辑器组件
- `@tiptap/starter-kit` — 基础扩展 (bold, italic, heading, code, blockquote, list, hr)
- `@tiptap/extension-image` — 图片节点
- `@tiptap/extension-code-block-lowlight` — 代码块 + Shiki 高亮
- `@tiptap/extension-table` + `@tiptap/extension-table-row` + `@tiptap/extension-table-cell` + `@tiptap/extension-table-header` — 表格
- `@tiptap/extension-link` — 链接
- `@tiptap/extension-placeholder` — 占位文字
- `@tiptap/extension-character-count` — 字数统计
- `@tiptap/extension-task-list` + `@tiptap/extension-task-item` — 任务列表
- `@tiptap/extension-underline` — 下划线
- `@tiptap/extension-highlight` — 高亮标记
- `lowlight` — 语法高亮语言注册

**可移除 (不再需要):**
- `marked` — 不再用 Markdown 渲染 (前端预览和 fallback)
- `next-mdx-remote` — 不再编译 MDX
- `remark-gfm`, `remark-math`, `rehype-katex` — MDX 插件
- `sanitize-html` — 保留，服务端渲染 HTML 仍需清理

### 1.3 Prisma Schema 变更

Post 模型字段变更:

```
content     String   → content     Json      // TipTap JSON { type: "doc", content: [...] }
// 已有但启用:
coverImage  String?  // 之前无 UI，现在编辑器支持设置
// 新增:
seoTitle    String?  // SEO 标题
seoDesc     String?  // SEO 描述
scheduledAt DateTime? // 定时发布时间
wordCount   Int?     // 字数 (发布时计算)
```

Slug 生成逻辑: 保留 `slugify` 自动生成 + 手动编辑覆盖。标题变更时 slug 同步更新（仅当 slug 未被手动修改过）。

---

## 二、编辑器页面布局 (单栏)

```
┌────────────────────────────────────┐
│  顶部状态栏 (sticky)                 │
│  [保存状态灯]          [存草稿] [发布] │
├────────────────────────────────────┤
│        封面图上传区 (虚线框)          │
├────────────────────────────────────┤
│        标题 (无边框 textarea)        │
├────────────────────────────────────┤
│  元数据行: 分类 | 标签(chip) | 可见性  │
├────────────────────────────────────┤
│        工具栏 (sticky)              │
│  B I S | H1 H2 H3 | " > | ...     │
├────────────────────────────────────┤
│                                    │
│        编辑区 (TipTap)              │
│        最小高度 400px               │
│        支持 / 斜杠命令              │
│                                    │
├────────────────────────────────────┤
│  ▶ SEO 设置 (折叠)                  │
│    自定义 Slug / SEO标题 / SEO描述   │
├────────────────────────────────────┤
│  ▶ 定时发布 (折叠)                   │
│    日期时间选择器                    │
└────────────────────────────────────┘
```

**布局要点:**
- 顶部状态栏 sticky，始终可见
- 封面图区：点击或拖拽上传，上传后显示预览 + 删除按钮
- 标题：无边框 textarea，自动撑高，placeholder "文章标题..."
- 元数据行：分类下拉 + 标签 chip 组件 + 可见性切换
- 工具栏 sticky 在编辑区上方，分组排列
- 编辑区：TipTap 编辑器，最小高度 400px
- SEO 和定时发布默认折叠，减少视觉干扰

---

## 三、功能清单

### 3.1 输入交互

| 交互 | 行为 |
|------|------|
| Markdown 快捷输入 | `#` 空格 → H1, `##` → H2, `**` → 加粗, `` ` `` → 行内代码, `>` → 引用 (TipTap 内置) |
| 斜杠命令 / | 行首输入 `/` 弹出菜单: H1-H3, 图片, 代码块, 表格, Callout, 数学公式, 分割线, 列表 |
| 浮动菜单 (Bubble Menu) | 选中文字浮现: 加粗, 斜体, 删除线, 链接, 行内代码 |
| 快捷键 | Ctrl+B 加粗, Ctrl+I 斜体, Ctrl+K 链接, Ctrl+Z 撤销, Ctrl+S 手动保存 |

### 3.2 图片处理

- **拖拽上传**: 拖到编辑区 → 上传 `/api/upload` → 插入节点，上传中显示 loading
- **粘贴截图**: Ctrl+V → 检测剪贴板图片 → 自动上传 → 插入
- **工具栏按钮**: 点击 → 文件选择器 → 上传 → 插入
- **封面图**: 独立上传区，与编辑器共用 `/api/upload` 接口
- **图片节点交互**: 点击选中 → 设置标题(caption) / 删除

### 3.3 自动保存

| 机制 | 说明 |
|------|------|
| 本地草稿 (localStorage) | 内容变化后 3s 防抖 → 存储编辑器 JSON。刷新后可恢复 |
| 服务端草稿 | 每 30s 自动保存 (status=DRAFT)，或 Ctrl+S 手动保存 |
| 状态指示灯 | 顶部栏绿点 + "已保存" / "保存中..." / "未保存" |
| 草稿恢复 | 新建文章时检测 localStorage，弹窗 "检测到未保存的草稿，是否恢复？" |

### 3.4 自定义 TipTap 扩展 (替代原 MDX 组件)

| 扩展 | 说明 | 渲染 |
|------|------|------|
| Callout | Info/Warn/Tip/Note 4 种类型，/ 菜单或工具栏插入 | 左边框 + 图标 + 背景色卡片 |
| CodeBlock | 代码块 + 语言选择器 + Shiki (lowlight) 高亮 | macOS三点 + 文件名栏 + 深色代码 |
| ImageCaption | 图片 + 描述文字 | 圆角图片 + 下方居中描述 |
| Math (KaTeX) | 行内 $...$ + 块级 $$...$$ | KaTeX 渲染 |

### 3.5 标签输入

- 替换逗号分隔 input 为 chip 组件
- 输入 + 回车/逗号确认标签
- 每个标签显示 chip 带 × 删除
- 自动补全已有标签 (从数据库)

### 3.6 定时发布

- `scheduledAt` 字段，设值后状态保持 DRAFT
- 非管理员投稿强制 DRAFT，不支持定时发布
- 管理员可直接设定 PUBLISHED + 未来时间
- 前端显示 "将于 YYYY-MM-DD HH:mm 发布"

### 3.7 修复的 Bug

- **updatePost 不更新标签**: 现在通过 TipTap JSON 存储，无需单独处理标签关联？保持标签独立于 content，updatePost 需要同步处理 PostTag 关系。

### 3.8 字数统计

- TipTap character-count 扩展实时显示字数
- 发布时计算最终字数写入 `wordCount`

---

## 四、服务端渲染

文章详情页不再使用 MDX 编译，改为:

```
存储: TipTap JSON (Prisma Json)
渲染: @tiptap/core generateHTML(json, extensions) → HTML 字符串
安全: sanitize-html 清理 HTML
显示: dangerouslySetInnerHTML + 自定义 CSS (prose 风格)
```

代码高亮: 服务端生成 HTML 时，CodeBlock 扩展通过 lowlight 内联 Shiki 主题样式。

Callout/ImageCaption: 自定义扩展的 renderHTML 输出对应的 HTML 结构，详情页 CSS 匹配样式。

---

## 五、影响范围

### 需要修改的文件

| 文件 | 变更 |
|------|------|
| `src/components/PostEditor.tsx` | 完全重写，TipTap 替换 textarea |
| `src/app/admin/posts/new/page.tsx` | 适配新 PostEditor props |
| `src/app/admin/posts/[id]/edit/page.tsx` | 适配新数据格式 |
| `src/app/my-posts/new/page.tsx` | 适配新 PostEditor |
| `src/app/my-posts/[id]/edit/page.tsx` | 适配新数据格式 |
| `src/actions/posts.ts` | content 字段类型变更，修复 tags 更新 |
| `src/app/post/[slug]/page.tsx` | MDX 渲染 → generateHTML 渲染 |
| `src/lib/mdx.tsx` | 删除或重构为 HTML 渲染工具 |
| `prisma/schema.prisma` | content 改为 Json，新增字段 |

### 需要新建的文件

| 文件 | 用途 |
|------|------|
| `src/lib/editor/extensions/` | TipTap 自定义扩展 (Callout, CodeBlock, ImageCaption, Math) |
| `src/lib/editor/render.ts` | 服务端 generateHTML 配置 |
| `src/lib/editor/toolbar.ts` | 工具栏配置 |
| `src/components/TagInput.tsx` | 标签 chip 输入组件 |
| `src/components/CoverUpload.tsx` | 封面图上传组件 |

### 需要删除/归档的内容

| 文件 | 原因 |
|------|------|
| `src/components/mdx/Callout.tsx` | 重写为 TipTap Node Extension |
| `src/components/mdx/CodeBlock.tsx` | 重写为 TipTap Node Extension |
| `src/components/mdx/ImageCaption.tsx` | 合并到 TipTap Image 扩展 |
| `src/components/mdx/index.tsx` | MDX 组件注册不再需要 |

---

## 六、不做的

- **多用户协作**: 个人博客不需要
- **修订历史/版本对比**: 复杂度高，暂不做
- **文章导入/导出**: 不做 Markdown 导入
- **移动端编辑器**: 管理后台桌面端优先
- **旧文章迁移**: 用户确认可直接删除
