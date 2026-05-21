# 用户头像上传与显示 设计文档

> 日期：2026-05-21
> 状态：已确认
> 目标：用户可在 `/account` 页面上传并裁剪头像，全站统一显示头像

---

## 一、功能概述

在账号管理页 (`/account`) 添加头像上传功能，支持手动裁剪（1:1），上传后全站统一显示——替换现有的静态字母圆圈。

### 用户流程

```
点击头像区 → 选择图片文件 → 裁剪弹窗 (1:1, 拖拽调整) → 确认裁剪 → 上传 → 保存 URL
```

### 不做

- 头像压缩优化（使用 Canvas 裁剪即可）
- 默认头像 / Gravatar 集成
- 管理员替用户上传头像
- 背景图改动

---

## 二、技术方案

### 2.1 裁剪库

使用 `react-easy-crop`：
- 轻量级 React 裁剪组件
- 支持拖拽移动、滚轮缩放
- 提供裁剪区域坐标，配合 Canvas 生成最终图片

### 2.2 新增依赖

- `react-easy-crop`

### 2.3 新增文件

| 文件 | 职责 |
|------|------|
| `src/components/Avatar.tsx` | 通用头像组件：有 URL 显示图片，无 URL 显示字母圆圈 |
| `src/components/AvatarCropModal.tsx` | 裁剪弹窗：react-easy-crop + Canvas 裁剪 + 上传 |

### 2.4 修改文件

| 文件 | 变更 |
|------|------|
| `src/components/ProfileEditForm.tsx` | 头像区域触发裁剪弹窗，保存 avatar URL |
| `src/actions/profile.ts` | `updateProfile` 增加 `avatar` 字段 |
| `src/lib/auth.ts` | session callback 传入 `avatar` |
| `src/components/NavBar.tsx` | 登陆后使用 `<Avatar>` 显示头像 |
| `src/app/post/[slug]/page.tsx` | 作者卡片使用 `<Avatar>` |
| `src/app/user/[username]/page.tsx` | 用户主页使用 `<Avatar>` |
| `src/components/CommentItem.tsx` | 评论列表使用 `<Avatar>` |

---

## 三、组件设计

### 3.1 Avatar 组件

```tsx
interface AvatarProps {
  src?: string | null;
  name: string;
  size?: "sm" | "md" | "lg";
}

// sm: 32px, md: 40px, lg: 80px
// 有 src → <img className="rounded-full object-cover" />
// 无 src → 渐变圆形 + 首字母
```

使用场景：
| 位置 | size |
|------|------|
| NavBar | sm |
| 文章作者卡片 | md |
| 用户主页 | lg |
| 评论列表 | sm |

### 3.2 AvatarCropModal 组件

```tsx
interface AvatarCropModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSave: (url: string) => void;
}
```

内部流程：
1. 用户点击 → `<input type="file">` 选择图片
2. 图片加载 → 显示 `react-easy-crop` 裁剪界面（1:1 固定比例）
3. 用户拖拽/缩放调整裁剪区域
4. 确认 → Canvas 生成裁剪后图片 (Blob)
5. 上传到 `/api/upload` → 获得 URL
6. 回调 `onSave(url)` → 父组件保存

### 3.3 ProfileEditForm 集成

在现有字段（bio, website, location, github, twitter）上方添加头像区域：
- 显示当前 `<Avatar>`（如果有）
- 点击触发 `<AvatarCropModal>`
- 保存时连同 `avatar` URL 一起提交到 `updateProfile`

---

## 四、后端变更

### 4.1 updateProfile action

`src/actions/profile.ts` 的 update 数据中增加 `avatar` 字段：

```typescript
// 从 formData 获取 avatar URL
const avatar = (formData.get("avatar") as string) || null;

// 更新
await prisma.user.update({
  where: { id: userId },
  data: { bio, website, location, github, twitter, avatar },
});
```

### 4.2 Auth Session

`src/lib/auth.ts` 中 JWT callback 和 session callback 传入 avatar：

```typescript
// jwt callback
if (user) {
  token.avatar = user.avatar;
}

// session callback
session.user.avatar = token.avatar;
```

---

## 五、组件树变更

```
ProfileEditForm
├── <Avatar />                          ← 新增：当前头像预览（点击触发裁剪）
├── <AvatarCropModal />                 ← 新增：裁剪弹窗
├── <input name="avatar" hidden />      ← 新增：隐藏字段传递 URL
└── ... 现有字段 (bio, website, ...)    ← 不变
```

全站显示：
```
NavBar / post page / user page / comments
├── <Avatar src={user.avatar} name={user.username} />
```

---

## 六、Avatar 组件统一渲染规则

```
if (src && src.length > 0)
  → <img src={src} className="rounded-full object-cover w-X h-X" />
else
  → <div className="rounded-full bg-gradient w-X h-X flex items-center justify-center text-white font-bold">
      {name.charAt(0).toUpperCase()}
    </div>
```

尺寸映射：`sm` = `w-8 h-8 text-xs`, `md` = `w-10 h-10 text-sm`, `lg` = `w-20 h-20 text-2xl`
