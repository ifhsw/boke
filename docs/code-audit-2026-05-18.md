# 代码审计报告

> 审计日期：2026-05-19（更新于 2026-05-19）
> 范围：`C:\Users\shi\Desktop\book\blog`
> 状态：**高危问题已修复，中等问题已记录**
> 新增功能：文章可见性 (visibility) — 已审查通过

---

## 一、XSS 防护（已修复）

### 1.1 `marked.parse()` + `dangerouslySetInnerHTML` 无清理

`marked` 默认不清理 HTML 标签，所有通过 `marked.parse()` + `dangerouslySetInnerHTML` 渲染用户内容的地方都存在存储型 XSS 风险。

**影响范围**：关于页面、问答内容、MDX 回退渲染、客户端编辑器预览

**修复**：安装 `sanitize-html`，创建 `src/lib/sanitize.ts` 统一清理函数，应用到所有 8 个入口点：

| 文件 | 函数/位置 |
|------|-----------|
| `src/app/about/page.tsx` | 关于页面自定义内容 |
| `src/app/account/page.tsx` | 用户提问内容 |
| `src/lib/mdx.tsx` | MDX 编译失败回退 |
| `src/components/QuestionList.tsx` | 提问内容和回复 |
| `src/components/PostEditor.tsx` | 编辑器预览 |
| `src/components/AboutEditor.tsx` | 关于编辑器预览 |
| `src/components/QaForm.tsx` | 提问编辑器预览 |

**清理规则**：允许常见 Markdown 标签（h1-h6, img, a, pre, code, table, etc.），仅允许 `http`/`https`/`mailto` 协议，移除 `<script>`、`onerror` 等事件处理器。

---

## 二、文件上传安全（已修复）

### 2.1 SVG 文件上传（高危）

`src/app/api/upload/route.ts` 允许 `image/svg+xml` 类型上传。SVG 可包含嵌入式 `<script>` 标签和事件处理器。

**修复**：从允许类型列表中移除 `image/svg+xml`。

### 2.2 客户端 MIME 类型可伪造（高危）

上传验证仅依赖客户端声明的 `file.type`（Content-Type 头），未验证文件内容。

**修复**：添加魔术字节验证，检查文件头与声明类型匹配：

| MIME 类型 | 魔术字节 |
|-----------|---------|
| image/jpeg | `FF D8 FF` |
| image/png | `89 50 4E 47` |
| image/gif | `47 49 46` |
| image/webp | `52 49 46 46` |

### 2.3 文件名未清理

文件扩展名直接从用户提供的文件名提取，未做任何过滤。

**修复**：清理扩展名中的非字母数字字符，限制为 `jpg/jpeg/png/gif/webp`。

---

## 三、认证与密钥（已记录）

### 3.1 AUTH_SECRET 弱密钥

`.env` 中 `AUTH_SECRET="dev-secret-change-in-production"` 为占位值，部署时需替换。部署指南已覆盖此步骤。

**状态**：无需修改代码，部署时处理。

---

## 四、其他已记录问题（中低优先级）

### 4.1 无输入长度限制

多个 Server Action 接受无界字符串输入（评论内容、文章内容、个人资料字段等），无最大长度约束。

**建议**：在 action 层添加字符数上限，或在 Prisma schema 层使用 `@db.Text` + 应用层校验。

### 4.2 无速率限制

登录、注册、评论、提问等端点无限流保护。

**建议**：使用 `next-rate-limit` 或反向代理层（Nginx `limit_req`）实现。

### 4.3 `updateSiteSetting` 无键名白名单

**建议**：限制可写入的 setting key 为已知集合 `["background", "about_content"]`。

### 4.4 个人信息字段无格式验证

`website`/`github`/`twitter` 字段接受任意字符串。

**建议**：添加 URL 格式/用户名格式校验。

### 4.5 `lib/uploads.ts` 路径遍历

`startsWith` 检查在大小写不敏感文件系统上可能被绕过。

**建议**：使用 `fs.realpathSync` 进行更严格的路径规范。

---

## 五、已确认安全（无需处理）

- **无 SQL 注入** — 全部使用 Prisma ORM，无 `$queryRaw`/`$executeRaw`
- **文章所有权检查** — `updatePost`/`deletePost` 验证 `isAdmin || authorId === userId`
- **非管理员强制草稿** — `createPost`/`updatePost` 对非 Admin 用户强制 `finalStatus = "DRAFT"`
- **文章可见性控制** — 新增 `visibility` 字段 (PUBLIC/PRIVATE)，仅作者+管理员可查看私密文章；公开页面 (首页/频道/归档/搜索/标签/侧边栏/用户主页) 统一过滤 `visibility: "PUBLIC"`；管理员修改可见性通过 `updatePost` 中的 `isAdmin` 守卫保护
- **密码安全** — bcrypt，盐轮数 10
- **JWT 安全** — 服务端存储 role/id，不依赖客户端
- **CSRF 防护** — Next.js Server Actions 内置
- **删除级联** — `deleteUser` 在 `prisma.$transaction` 中处理文章/评论/提问清理
- **评论纯文本渲染** — `CommentItem.tsx` 使用 `{comment.content}`（React 自动转义）
- **外链 rel 属性** — 用户主页外链使用 `rel="noopener noreferrer"`
- **`.gitignore` 正确** — `.env*`、`/.next/`、`*.db`、`*.pem`、`public/uploads/*` 均已排除
- **无硬编码密钥** — 无 `sk-`、`api_key` 等格式泄露
- **上传文件隔离** — 上传文件进 `public/uploads/`，Nginx 直接 serve，不经过应用逻辑

---

## 六、审计记录

| 优先级 | 项目 | 操作 | 状态 |
|--------|------|------|------|
| 严重 | XSS: marked.parse 无清理 | 安装 sanitize-html，创建统一清理函数 | 已完成 |
| 高 | 上传: SVG 文件可包含脚本 | 从允许列表移除 SVG | 已完成 |
| 高 | 上传: MIME 类型可伪造 | 添加魔术字节验证 | 已完成 |
| 高 | 上传: 文件名未清理 | 添加扩展名过滤 | 已完成 |
| 中 | 无输入长度限制 | 记录待处理 | 待处理 |
| 中 | 无速率限制 | 记录待处理 | 待处理 |
| 中 | updateSiteSetting 无键名白名单 | 记录待处理 | 待处理 |
| 低 | 个人信息字段无格式验证 | 记录待处理 | 待处理 |
| 低 | uploads.ts 路径遍历 | 记录待处理 | 待处理 |
