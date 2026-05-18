# 代码审计报告

> 审计日期：2026-05-18
> 范围：`C:\Users\shi\Desktop\book\blog`

---

## 一、敏感信息

### 1.1 `.env` — 弱密钥（未跟踪，磁盘残留）

**路径：** `.env` / `.next/standalone/.env`
**内容：** `AUTH_SECRET="dev-secret-change-in-production"`

两个文件都被 `.gitignore` 排除（`.env*`），未被提交到仓库。但磁盘上保留的是占位密钥，生产部署前需更换为强随机字符串。

**建议：** 使用 `openssl rand -base64 32` 生成强密钥更新 `.env`。

---

### 1.2 `prisma/seed.ts:10` — 种子脚本中的硬编码默认密码

```ts
const adminPassword = process.env.SEED_ADMIN_PASSWORD || "admin123";
```

如果未设置环境变量 `SEED_ADMIN_PASSWORD`，将使用弱密码 `admin123` 创建管理员账户。

**建议：** 改为未设置环境变量时直接退出或生成随机密码：

```ts
const adminPassword = process.env.SEED_ADMIN_PASSWORD;
if (!adminPassword) {
  console.error("SEED_ADMIN_PASSWORD not set, aborting seed.");
  process.exit(1);
}
```

---

### 1.3 `prisma/seed.ts:14` — 虚构种子邮箱

```ts
email: "admin@blog.com"
```

`blog.com` 非本项目所有，但仅作为种子测试数据，无实际风险。

---

## 二、未使用的图片文件（共 2,044 KB）

这些文件已被 Git 跟踪但代码中无任何引用，占用仓库空间。

| 文件 | 大小 | 说明 |
|------|------|------|
| `public/bg-wallpaper.gif` | 912 KB | 早期背景图，已由数据库 `SiteSetting` 方案取代 |
| `public/bg-wallpaper.jpeg` | 436 KB | 同上 |
| `public/preview.jpg` | 696 KB | 早期预览图，无引用 |

**建议：** 删除并从 Git 历史中移除：

```bash
git rm public/bg-wallpaper.gif public/bg-wallpaper.jpeg public/preview.jpg
```

---

## 三、`public/uploads/` 运行时文件被 Git 跟踪

`public/uploads/` 下有 15 个用户上传文件（约 3.3 MB），均被 Git 跟踪。这些是运行时产生的用户内容，不应纳入版本控制。

**建议：** 将上传目录加入 `.gitignore`，只保留 `.gitkeep`：

```
public/uploads/*
!public/uploads/.gitkeep
```

---

## 四、`console.error` 生产环境泄露风险

| 文件 | 行号 | 内容 |
|------|------|------|
| `src/app/error.tsx` | 14 | `console.error("Page error:", error)` |
| `src/app/api/upload/route.ts` | 48 | `console.error("Upload error:", error)` |

生产环境中完整的 `error` 对象可能泄露内部路径或数据库信息。

**建议：** 生产构建时仅输出简要信息或使用结构化日志（如 `process.env.NODE_ENV === "production"` 分支）。

---

## 五、已确认安全的部分

- **无 API 密钥泄露** — 未发现 `sk-`、`api_key` 等格式的密钥
- **无真实邮箱泄露** — 仅种子脚本中的虚构测试邮箱
- **无数据库凭证泄露** — 使用 SQLite 本地文件 `file:./dev.db`，无网络暴露
- **无硬编码 JWT/Token** — 认证全部由 NextAuth.js 管理
- **`.gitignore` 覆盖正确** — `.env*`、`/.next/`、`*.db`、`*.pem` 均已排除
- **无注释掉的废弃代码** — 代码整洁，所有注释均为合理的文档说明
- **`22.jpg` 已不存在** — 文件及所有引用均已清除

---

## 六、整改优先级

| 优先级 | 项目 | 操作 |
|--------|------|------|
| **高** | `public/bg-wallpaper.*` + `preview.jpg` | `git rm` 删除约 2MB 无用文件 |
| **高** | `public/uploads/*` | 加入 `.gitignore` |
| **高** | `prisma/seed.ts` 硬编码密码 | 移除默认值，强制读取环境变量 |
| **中** | `console.error` 生产日志 | 加环境判断，避免泄露完整错误对象 |
| **中** | `public/uploads/` 已跟踪文件 | 清理已提交的上传文件 |
| **低** | `.env` 占位密钥 | 部署前使用强随机密钥 |
| **低** | `.next/standalone/.env` | 确认构建产物目录不会意外发布 |
