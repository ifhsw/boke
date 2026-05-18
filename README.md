[2026-05-16-personal-blog-design.md](https://github.com/user-attachments/files/27965741/2026-05-16-personal-blog-design.md)
# 个人博客设计文档

## 概述

个人博客，支持混合型内容（技术、随笔等），温暖文艺风格。管理员通过后台写文章和管理内容，读者可注册登录并评论。部署在自有 Linux 服务器（宝塔面板）。

## 技术栈

- **Next.js 14+ (App Router)** — 前后端一体框架
- **SQLite** (via Prisma ORM) — 轻量数据库
- **NextAuth.js (Auth.js v5)** — 认证系统
- **Tailwind CSS** — 样式（温暖文艺风格）
- **MDX** — 文章内容格式，支持代码高亮和数学公式
- **PM2** — 进程管理
- **Nginx** — 反向代理（宝塔面板）

## 页面结构

### 前台页面

| 页面   | 路由                | 说明                       |
| ---- | ----------------- | ------------------------ |
| 首页   | `/`               | 最新文章卡片列表 + 侧边栏（标签云、最近文章） |
| 频道页  | `/tech`, `/essay` | 按分类浏览文章                  |
| 文章详情 | `/post/[slug]`    | MDX 渲染 + 评论区             |
| 标签页  | `/tag/[name]`     | 按标签筛选文章                  |
| 搜索页  | `/search?q=`      | 服务端全文搜索（标题+内容）           |
| 归档页  | `/archive`        | 按年份时间线浏览                 |
| 关于页  | `/about`          | 个人介绍（管理员可编辑 Markdown）    |
| 问答页  | `/qa`             | 匿名提问箱，支持 Markdown + 图片上传 |
| 账号管理 | `/account`        | 用户自助：修改密码、管理自己的提问      |
| 登录   | `/login`          | 读者和管理员登录                 |
| 注册   | `/register`       | 读者注册                     |

### 后台页面（管理员）

| 页面   | 路由                | 说明                        |
| ---- | ----------------- | ------------------------- |
| 仪表盘  | `/admin`          | 文章数、评论数、读者数统计             |
| 文章管理 | `/admin/posts`    | 写/编辑/删除文章，Markdown 编辑器 + 图片上传 + 实时预览 |
| 评论管理 | `/admin/comments` | 审核/删除评论                   |
| 用户管理 | `/admin/users`    | 查看用户信息、重置密码、删除用户（级联清理）     |
| 站点设置 | `/admin/settings` | 背景图片、关于页内容、管理员密码修改        |

### 响应式

- 桌面端：内容 + 侧边栏双栏布局
- 移动端：单栏，侧边栏收起

## 数据模型

### User（用户）

| 字段           | 类型                   | 说明        |
| ------------ | -------------------- | --------- |
| id           | String (UUID)        | 主键        |
| username     | String               | 用户名，唯一    |
| email        | String               | 邮箱，唯一     |
| passwordHash | String               | bcrypt 哈希 |
| role         | Enum (ADMIN, READER) | 角色        |
| avatar       | String?              | 头像 URL    |
| createdAt    | DateTime             | 创建时间      |

### Post（文章）

| 字段         | 类型                      | 说明          |
| ---------- | ----------------------- | ----------- |
| id         | String (UUID)           | 主键          |
| title      | String                  | 标题          |
| slug       | String                  | URL 友好标识，唯一 |
| excerpt    | String?                 | 摘要          |
| content    | String                  | MDX 内容      |
| coverImage | String?                 | 封面图 URL     |
| category   | Enum (TECH, ESSAY)      | 分类          |
| status     | Enum (DRAFT, PUBLISHED) | 状态          |
| authorId   | String (FK)             | 作者          |
| createdAt  | DateTime                | 创建时间        |
| updatedAt  | DateTime                | 更新时间        |

### Tag（标签）

| 字段   | 类型            | 说明     |
| ---- | ------------- | ------ |
| id   | String (UUID) | 主键     |
| name | String        | 标签名，唯一 |

### PostTag（文章-标签关联）

| 字段     | 类型          | 说明    |
| ------ | ----------- | ----- |
| postId | String (FK) | 文章 ID |
| tagId  | String (FK) | 标签 ID |

联合主键 (postId, tagId)。

### Comment（评论）

| 字段        | 类型            | 说明            |
| --------- | ------------- | ------------- |
| id        | String (UUID) | 主键            |
| content   | String        | 评论内容          |
| postId    | String (FK)   | 所属文章          |
| userId    | String (FK)   | 评论者           |
| parentId  | String? (FK)  | 父评论 ID，支持嵌套回复 |
| createdAt | DateTime      | 创建时间          |

### SiteSetting（站点设置）

| 字段    | 类型     | 说明         |
| ----- | ------ | ---------- |
| key   | String | 主键，设置键名    |
| value | String | 设置值（文本/URL） |

当前使用的键：`background`（网页背景图 URL）、`about_content`（关于页 Markdown 内容）。

### Question（匿名提问）

| 字段        | 类型            | 说明              |
| --------- | ------------- | --------------- |
| id        | String (UUID) | 主键              |
| content   | String        | 提问内容（Markdown）  |
| userId    | String (FK)   | 提问者（对其他用户隐藏）    |
| reply     | String?       | 管理员回复（Markdown）  |
| repliedAt | DateTime?     | 回复时间            |
| createdAt | DateTime      | 创建时间            |

权限：用户只能看到自己的提问和管理员的回复、可删除自己的提问。管理员可看到全部提问及真实身份。

## 认证与安全

### 认证方案

- NextAuth.js (Auth.js v5) + Credentials Provider（邮箱密码登录）
- 密码使用 bcrypt 哈希存储
- JWT + Session 双模式：JWT 用于 API 鉴权，Session 用于页面状态
- 管理员角色通过数据库 role 字段区分
- 中间件保护 `/admin/*` 路由，仅 ADMIN 角色可访问

### 管理员权限

- 查看、创建、编辑、删除文章
- 审核、删除评论
- 查看所有用户完整信息（邮箱、注册时间、评论历史等）
- 重置任意用户密码
- 删除用户（不能删除自己和其他管理员，级联清理文章/评论/提问）
- 修改自己的密码
- 管理站点设置（背景图、关于页内容）
- 回复匿名提问（前台直接操作）
- 管理标签

### 用户自助

- 修改自己的密码（需验证当前密码）
- 查看和删除自己的匿名提问
- 在 `/account` 账号管理页集中管理

### 安全措施

- CSRF 防护（Next.js 内置）
- XSS 防护（React 自动转义 + MDX 渲染时 sanitize）
- Rate limiting：登录/注册接口限流
- 环境变量管理敏感配置（数据库路径、JWT 密钥等）

## 内容格式支持

- Markdown 基础语法
- 代码高亮（Shiki）
- LaTeX 数学公式
- 图片上传（API 上传至 `public/uploads/`，支持 JPEG/PNG/GIF/WebP/SVG，最大 10MB，需登录）
- 文章编辑器支持预览模式、光标位置插入图片
- 匿名提问支持 Markdown + 图片/文件上传
- 中文标题兼容：纯中文标题自动使用时间戳作为 slug 后备
- 关于页面管理员可编辑（Markdown 编辑器 + 预览）

## 部署与运维

### 服务器环境

- Linux + 宝塔面板
- Node.js 18+ 运行环境
- PM2 管理 Next.js 进程（开机自启、自动重启）
- Nginx 反向代理 → Next.js（端口 3000）
- Let's Encrypt SSL 证书（宝塔一键申请）

### 备份策略

- SQLite 数据库文件每日自动备份（宝塔计划任务）
- 文章内容存数据库，不依赖文件系统

### CI/CD（可选）

- GitHub 仓库 push → GitHub Actions 构建 → SSH 部署到服务器
- 或手动 git pull + pm2 restart

## 视觉风格

   主色调：冷灰、墨黑、苍蓝，低饱和高对比

&#x20;

- 氛围：孤绝、肃杀、静谧、宿命感
- 元素：玄龙、古桥、残雪、古刹、阴云
- 笔触：写意留白，线条凌厉，自带颗粒质感
- 圆角卡片布局
- 柔和阴影
- 中文字体优化

## 语言

仅中文界面。


# 玄桥博客 — 服务器部署指南

## 前置条件

| 项目 | 要求 |
|------|------|
| 服务器 | CentOS 7+ / RHEL 7+（推荐 CentOS 7 + 宝塔） |
| 面板 | 宝塔面板 已安装 |
| Node.js | 22 LTS (Next.js 16 要求) |
| 数据库 | SQLite (无需额外安装) |
| 反向代理 | Nginx (宝塔自带) |
| 进程管理 | PM2 |
| 域名 | 已解析到服务器 IP |

---

## 第一步：服务器环境准备

### 1.1 宝塔面板安装 Node.js

```
宝塔面板 → 软件商店 → 搜索 "Node.js 版本管理器" → 安装
→ 安装 Node 22.x LTS
```

### 1.2 安装 PM2（全局）

```bash
npm install -g pm2
```

### 1.3 安装 Git（如果没有）

```bash
# CentOS 7
sudo yum install git -y

# CentOS 8+ / RHEL 8+
sudo dnf install git -y
```

---

## 第二步：上传代码到服务器

### 方式 A：Git 克隆（推荐）

```bash
# SSH 到服务器
cd /www/wwwroot

# 克隆主仓库（含 blog 子模块）
git clone --recurse-submodules https://github.com/ifhsw/boke.git

cd boke/blog
```

> **说明**：主仓库 `boke` 包含 blog 作为 git 子模块，实际应用代码在 `boke/blog/` 目录下。

### 方式 B：宝塔面板上传

```
宝塔面板 → 文件 → /www/wwwroot/ → 新建目录 boke/blog
→ 将本地 blog 文件夹压缩为 .zip 上传
→ 解压到 /www/wwwroot/boke/blog/
```

注意：**不要上传 `.env`、`prisma/dev.db`、`node_modules/`、`.next/`**

---

## 第三步：创建生产环境配置

### 3.1 创建 `.env` 文件

```bash
cd /www/wwwroot/boke/blog
vi .env
```

> 提示：如果不想用 vi，也可以用 `cat` 直接写入：
> ```bash
> cat > .env << 'EOF'
> DATABASE_URL="file:./dev.db"
> AUTH_SECRET="粘贴生成的密钥"
> AUTH_URL="https://你的域名"
> UPLOAD_DIR="./public/uploads"
> EOF
> ```

内容：

```env
DATABASE_URL="file:./dev.db"
AUTH_SECRET="替换为随机密钥"
AUTH_URL="https://你的域名"
UPLOAD_DIR="./public/uploads"
SEED_ADMIN_PASSWORD="设置一个强密码用于管理员账号"
```

### 3.2 生成 AUTH_SECRET

```bash
# 方式1：用 openssl 生成随机字符串
openssl rand -base64 32

# 方式2：用 node 生成
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

把输出的字符串填入 `.env` 的 `AUTH_SECRET`。

### 3.3 创建 `ecosystem.config.cjs`（PM2 配置）

```bash
vi ecosystem.config.cjs
```

内容：

```js
module.exports = {
  apps: [
    {
      name: "blog",
      script: "node_modules/.bin/next",
      args: "start",
      cwd: "/www/wwwroot/boke/blog",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
      },
      instances: 1,
      exec_mode: "fork",
      autorestart: true,
      max_memory_restart: "500M",
    },
  ],
};
```

---

## 第四步：安装依赖 & 构建

```bash
cd /www/wwwroot/boke/blog

# 1. 安装依赖
npm install

# 2. 初始化数据库（生成 dev.db 和表结构）
npx prisma migrate deploy

# 3. 导入初始数据（管理员账号 + 示例文章）
npx prisma db seed

# 4. 构建生产版本
npm run build
```

构建成功后输出类似：

```
✓ Generating static pages using 19 workers (14/14)
Route (app)
┌ ƒ /
├ ƒ /post/[slug]
...
```

---

## 第五步：启动应用（PM2）

```bash
# 启动
pm2 start ecosystem.config.cjs

# 查看状态
pm2 status

# 查看日志
pm2 logs blog

# 设置开机自启
pm2 save
pm2 startup
# CentOS 会输出类似：sudo env PATH=$PATH:/usr/bin pm2 startup ...
# 复制输出的命令并执行
```

确认应用运行在 `http://localhost:3000`：

```bash
curl http://localhost:3000
```

---

## 第六步：配置 Nginx 反向代理

### 6.1 宝塔面板创建站点

```
宝塔面板 → 网站 → 添加站点
  - 域名：填入你的域名
  - 根目录：/www/wwwroot/boke/blog/public
  - PHP 版本：纯静态
```

### 6.2 修改 Nginx 配置

```
宝塔面板 → 网站 → 点击域名 → 配置文件
```

替换为：

```nginx
server {
    listen 80;
    server_name 你的域名;

    # 上传的图片/文件
    location /uploads/ {
        alias /www/wwwroot/boke/blog/public/uploads/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 静态资源缓存
    location /_next/static/ {
        alias /www/wwwroot/boke/blog/.next/static/;
        expires 365d;
        add_header Cache-Control "public, immutable";
    }

    # 反向代理到 Next.js
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

保存后重载 Nginx。

---

## 第七步：配置 SSL（HTTPS）

```
宝塔面板 → 网站 → 点击域名 → SSL
→ Let's Encrypt → 申请证书 → 勾选 "强制 HTTPS"
```

---

## 第八步：防火墙 & 安全

### 8.1 宝塔防火墙

```
宝塔面板 → 安全 → 防火墙
  - 放行端口：80 (HTTP)、443 (HTTPS)
  - 不放行端口：3000（Next.js 只监听本地）
```

### 8.2 SELinux（CentOS/RHEL 特有）

宝塔面板通常会处理 SELinux，但如果遇到 Nginx 502 或上传失败：

```bash
# 查看 SELinux 状态
getenforce

# 临时关闭（重启失效）
sudo setenforce 0

# 永久关闭（不推荐，建议配置策略而非关闭）
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

> 宝塔面板用户一般无需手动处理，面板已自动配置。

### 8.3 文件权限

```bash
chmod 755 /www/wwwroot/boke/blog
chmod 644 /www/wwwroot/boke/blog/.env
chmod 644 /www/wwwroot/boke/blog/prisma/dev.db
chmod -R 755 /www/wwwroot/boke/blog/public/uploads
```

---

## 第九步：数据库备份（宝塔计划任务）

```
宝塔面板 → 计划任务 → 添加
  - 任务名称：博客数据库备份
  - 执行周期：每天 03:00
  - 脚本内容：
```

```bash
#!/bin/bash
SRC="/www/wwwroot/boke/blog/prisma/dev.db"
DST="/www/backup/blog-$(date +%Y%m%d).db"
cp "$SRC" "$DST"
# 保留最近 30 天的备份
find /www/backup/ -name "blog-*.db" -mtime +30 -delete
```

---

## 第十步：更新部署（后续）

### 代码有更新时

```bash
cd /www/wwwroot/boke/blog

# 拉取最新代码
git pull

# 如果子模块有更新
cd /www/wwwroot/boke
git pull
git submodule update --remote blog
cd blog

# 安装新依赖（如果有）
npm install

# 数据库迁移（如果有）
npx prisma migrate deploy

# 重新构建
npm run build

# 重启应用（零停机）
pm2 reload blog
```

---

## 默认管理员账号

种子脚本会使用 `.env` 中 `SEED_ADMIN_PASSWORD` 的值作为管理员密码。

| 字段 | 值 |
|------|-----|
| 邮箱 | `admin@blog.com` |
| 密码 | `.env` 中 `SEED_ADMIN_PASSWORD` 设置的值 |

如果未设置 `SEED_ADMIN_PASSWORD`，种子脚本会报错退出，不会创建默认密码。

管理员登录后可在 `/admin/settings` 或 `/account` 页面修改密码。

---

## 故障排查

### 502 Bad Gateway

```bash
# 1. 检查 Next.js 是否在运行
pm2 status

# 2. 查看错误日志
pm2 logs blog --err

# 3. 检查端口占用
ss -tlnp | grep 3000

# 4. CentOS: 检查 SELinux 是否拦截
# 查看 SELinux 审计日志
sudo ausearch -m avc -ts recent | grep nginx
# 如果 Nginx 连接被 SELinux 拦截，临时关闭验证：
sudo setenforce 0
```

### 种子脚本报错：SEED_ADMIN_PASSWORD

```
SEED_ADMIN_PASSWORD environment variable is required
```

在 `.env` 中添加：

```bash
echo 'SEED_ADMIN_PASSWORD="你的强密码"' >> /www/wwwroot/boke/blog/.env
```

然后重新执行 `npx prisma db seed`。

### 页面无法加载 / API 报错

```bash
# 确认 .env 中的 AUTH_URL 为 https://你的域名
cat /www/wwwroot/boke/blog/.env

# 确认 AUTH_SECRET 已设置（不能是占位值）
grep AUTH_SECRET /www/wwwroot/boke/blog/.env

# 重启应用
pm2 restart blog
```

### 数据库被锁定

```bash
# SQLite 单写锁问题，重启应用即可
pm2 restart blog
```

### 上传图片失败

```bash
# 确认 uploads 目录存在且可写
ls -la /www/wwwroot/boke/blog/public/uploads/
chmod 755 /www/wwwroot/boke/blog/public/uploads/
```

---

## 目录结构（服务器上）

```
/www/wwwroot/boke/
├── .gitmodules            # 子模块配置
├── docs/                  # 文档
└── blog/                  # ← 应用代码（子模块）
    ├── .env               # 环境变量（敏感，不要提交 git）
    ├── .env.example       # 环境变量示例
    ├── .next/             # 构建产物（不要手动修改）
    ├── .gitignore
    ├── ecosystem.config.cjs  # PM2 配置
    ├── next.config.ts
    ├── package.json
    ├── node_modules/      # 依赖
    ├── prisma/
    │   ├── dev.db         # SQLite 数据库文件（需备份）
    │   ├── migrations/    # 数据库迁移文件
    │   ├── schema.prisma  # 数据模型
    │   └── seed.ts
    ├── public/
    │   ├── uploads/       # 上传的图片/文件（运行时生成）
    │   └── favicon.ico
    ├── src/
    │   ├── actions/       # Server Actions
    │   ├── app/           # App Router 页面
    │   │   ├── account/   # 用户账号管理
    │   │   ├── admin/     # 后台管理
    │   │   ├── about/     # 关于页
    │   │   ├── api/       # API 路由
    │   │   ├── login/     # 登录
    │   │   ├── post/      # 文章详情
    │   │   ├── qa/        # 匿名提问
    │   │   ├── register/  # 注册
    │   │   ├── search/    # 搜索
    │   │   ├── tag/       # 标签筛选
    │   │   ├── tech/      # 技术频道
    │   │   ├── essay/     # 随笔频道
    │   │   └── archive/   # 归档
    │   ├── components/    # React 组件
    │   ├── lib/           # 工具库
    │   └── app/globals.css  # 全局样式
    └── .env.example
```

## 页面功能清单

部署完成后，访问 `https://你的域名` 可使用的功能：

| 路由 | 功能 | 权限 |
|------|------|------|
| `/` | 首页，文章列表 + 侧边栏 | 公开 |
| `/tech`, `/essay` | 技术/随笔频道 | 公开 |
| `/post/[slug]` | 文章详情 + 评论 | 公开 |
| `/tag/[name]` | 按标签筛选文章 | 公开 |
| `/search?q=` | 全文搜索 | 公开 |
| `/archive` | 按年份归档 | 公开 |
| `/about` | 关于页（管理员可编辑） | 公开 |
| `/qa` | 匿名提问（支持 Markdown+图片） | 需登录 |
| `/account` | 账号管理（改密、管理提问） | 需登录 |
| `/admin` | 后台仪表盘 | 仅管理员 |
| `/admin/posts` | 文章管理（CRUD） | 仅管理员 |
| `/admin/comments` | 评论管理 | 仅管理员 |
| `/admin/users` | 用户管理（重置密码、删除用户） | 仅管理员 |
| `/admin/settings` | 站点设置（背景、关于页、修改密码） | 仅管理员 |

