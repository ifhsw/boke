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

| 页面 | 路由 | 说明 |
|------|------|------|
| 首页 | `/` | 最新文章卡片列表 + 侧边栏（标签云、最近文章） |
| 频道页 | `/tech`, `/essay` | 按分类浏览文章 |
| 文章详情 | `/post/[slug]` | MDX 渲染 + 评论区 |
| 归档页 | `/archive` | 按年份时间线浏览 |
| 关于页 | `/about` | 个人介绍 |
| 登录 | `/login` | 读者和管理员登录 |
| 注册 | `/register` | 读者注册 |

### 后台页面（管理员）

| 页面 | 路由 | 说明 |
|------|------|------|
| 仪表盘 | `/admin` | 文章数、评论数、读者数统计 |
| 文章管理 | `/admin/posts` | 写/编辑/删除文章，MDX 编辑器 + 实时预览 |
| 评论管理 | `/admin/comments` | 审核/删除评论 |
| 用户管理 | `/admin/users` | 查看所有读者完整信息（邮箱、注册时间、评论历史等） |

### 响应式

- 桌面端：内容 + 侧边栏双栏布局
- 移动端：单栏，侧边栏收起

## 数据模型

### User（用户）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String (UUID) | 主键 |
| username | String | 用户名，唯一 |
| email | String | 邮箱，唯一 |
| passwordHash | String | bcrypt 哈希 |
| role | Enum (ADMIN, READER) | 角色 |
| avatar | String? | 头像 URL |
| createdAt | DateTime | 创建时间 |

### Post（文章）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String (UUID) | 主键 |
| title | String | 标题 |
| slug | String | URL 友好标识，唯一 |
| excerpt | String? | 摘要 |
| content | String | MDX 内容 |
| coverImage | String? | 封面图 URL |
| category | Enum (TECH, ESSAY) | 分类 |
| status | Enum (DRAFT, PUBLISHED) | 状态 |
| authorId | String (FK) | 作者 |
| createdAt | DateTime | 创建时间 |
| updatedAt | DateTime | 更新时间 |

### Tag（标签）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String (UUID) | 主键 |
| name | String | 标签名，唯一 |

### PostTag（文章-标签关联）

| 字段 | 类型 | 说明 |
|------|------|------|
| postId | String (FK) | 文章 ID |
| tagId | String (FK) | 标签 ID |

联合主键 (postId, tagId)。

### Comment（评论）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String (UUID) | 主键 |
| content | String | 评论内容 |
| postId | String (FK) | 所属文章 |
| userId | String (FK) | 评论者 |
| parentId | String? (FK) | 父评论 ID，支持嵌套回复 |
| createdAt | DateTime | 创建时间 |

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
- 管理标签

### 安全措施

- CSRF 防护（Next.js 内置）
- XSS 防护（React 自动转义 + MDX 渲染时 sanitize）
- Rate limiting：登录/注册接口限流
- 环境变量管理敏感配置（数据库路径、JWT 密钥等）

## 内容格式支持

- Markdown 基础语法
- 代码高亮（Shiki）
- LaTeX 数学公式
- 图片（本地存储和外部链接）

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

温暖文艺风格：
- 暖色调（琥珀色、棕色系为主色调）
- 圆角卡片布局
- 柔和阴影
- 舒适亲切的阅读氛围
- 中文字体优化

## 语言

仅中文界面。
