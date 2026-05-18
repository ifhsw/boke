# 代码审计报告

> 审计日期：2026-05-18
> 范围：`C:\Users\shi\Desktop\book\blog`
> 状态：**全部问题已修复**

---

## 一、敏感信息

### 1.1 `.env` — 弱密钥（已确认，未跟踪）

**路径：** `.env` / `.next/standalone/.env`
**内容：** `AUTH_SECRET="dev-secret-change-in-production"`

两个文件都被 `.gitignore` 排除（`.env*`），未被提交到仓库。

**状态：** 无需修改代码。部署时使用 `openssl rand -base64 32` 生成强密钥替换 `.env`。

---

### 1.2 `prisma/seed.ts:10` — 种子脚本硬编码密码（已修复）

**原代码：**
```ts
const adminPassword = process.env.SEED_ADMIN_PASSWORD || "admin123";
```

**修复后：**
```ts
const adminPassword = process.env.SEED_ADMIN_PASSWORD;
if (!adminPassword) {
  console.error("SEED_ADMIN_PASSWORD environment variable is required to seed the database.");
  process.exit(1);
}
```

未设置环境变量时直接报错退出，不再使用弱默认密码。

---

### 1.3 `prisma/seed.ts:14` — 虚构种子邮箱（无需处理）

```ts
email: "admin@blog.com"
```

`blog.com` 非本项目所有，仅作为种子测试数据，无实际风险。

---

## 二、未使用的图片文件（已删除）

| 文件 | 大小 | 状态 |
|------|------|------|
| `public/bg-wallpaper.gif` | 912 KB | 已 `git rm` |
| `public/bg-wallpaper.jpeg` | 436 KB | 已 `git rm` |
| `public/preview.jpg` | 696 KB | 已 `git rm` |

**节省：** 2,044 KB 仓库空间。

---

## 三、`public/uploads/` 运行时文件（已修复）

`.gitignore` 已添加：

```
public/uploads/*
!public/uploads/.gitkeep
```

上传文件不再被 Git 跟踪。

---

## 四、`console.error` 生产环境泄露（已修复）

| 文件 | 修复 |
|------|------|
| `src/app/error.tsx:14` | 加 `NODE_ENV !== "production"` 判断 |
| `src/app/api/upload/route.ts:48` | 加 `NODE_ENV !== "production"` 判断 |

生产环境不再输出完整错误对象。

---

## 五、已确认安全的部分（无需处理）

- **无 API 密钥泄露** — 未发现 `sk-`、`api_key` 等格式的密钥
- **无真实邮箱泄露** — 仅种子脚本中的虚构测试邮箱
- **无数据库凭证泄露** — 使用 SQLite 本地文件，无网络暴露
- **无硬编码 JWT/Token** — 认证全部由 NextAuth.js 管理
- **`.gitignore` 覆盖正确** — `.env*`、`/.next/`、`*.db`、`*.pem` 均已排除
- **无注释掉的废弃代码** — 代码整洁
- **`22.jpg` 已不存在** — 文件及所有引用均已清除

---

## 六、整改记录

| 优先级 | 项目 | 操作 | 状态 |
|--------|------|------|------|
| 高 | `public/bg-wallpaper.*` + `preview.jpg` | `git rm` 删除 | 已完成 |
| 高 | `public/uploads/*` | 加入 `.gitignore` | 已完成 |
| 高 | `prisma/seed.ts` 硬编码密码 | 移除默认值 | 已完成 |
| 中 | `console.error` 生产日志 | 加 NODE_ENV 判断 | 已完成 |
| 中 | `public/uploads/` 已跟踪文件 | 已不被跟踪 | 已完成 |
| 低 | `.env` 占位密钥 | 部署时替换 | 部署时处理 |
| 低 | `.next/standalone/.env` | 构建产物不发布 | 无需处理 |
