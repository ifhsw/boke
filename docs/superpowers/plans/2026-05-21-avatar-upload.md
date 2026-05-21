# 用户头像上传与显示 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用户可在 `/account` 页面上传并裁剪头像（1:1），全站 NavBar、文章作者卡片、用户主页、评论统一显示头像。

**Architecture:** `react-easy-crop` 裁剪 + Canvas 生成 Blob + 现有 `/api/upload` 上传。创建通用 `Avatar` 组件处理条件渲染（有 URL 显示图片，无 URL 显示字母圆圈），创建 `AvatarCropModal` 封装裁剪流程。`ProfileEditForm` 集成头像上传，通过 `updateProfile` action 保存 URL。Auth session 传递 avatar 给 NavBar/Header 使用。

**Tech Stack:** react-easy-crop, Canvas API, 现有 /api/upload

---

## 文件结构

### 新建

| 文件 | 职责 |
|------|------|
| `src/components/Avatar.tsx` | 通用头像组件：有 URL 显示图片，无 URL 显示字母圆圈 |
| `src/components/AvatarCropModal.tsx` | 裁剪弹窗：react-easy-crop + Canvas 裁剪 + 上传 |

### 修改

| 文件 | 变更 |
|------|------|
| `src/components/ProfileEditForm.tsx` | 头像区域触发裁剪弹窗，保存 avatar URL |
| `src/actions/profile.ts` | `updateProfile` 增加 `avatar` 字段 |
| `src/lib/auth.ts` | session callback 传入 `avatar` |
| `src/app/account/page.tsx` | select 加 `avatar`，传给 ProfileEditForm |
| `src/components/Header.tsx` | 传递 `avatar` 给 NavBar |
| `src/components/NavBar.tsx` | 使用 `<Avatar>` 显示头像 |
| `src/app/post/[slug]/page.tsx` | 作者卡片使用 `<Avatar>` |
| `src/app/user/[username]/page.tsx` | 用户主页使用 `<Avatar>` |
| `src/components/CommentItem.tsx` | 评论列表使用 `<Avatar>` |

---

### Task 1: 安装 react-easy-crop

**Files:**
- Modify: `package.json`

- [ ] **Step 1: 安装依赖**

```bash
cd C:/Users/shi/Desktop/book/blog
npm install react-easy-crop
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add package.json package-lock.json
git commit -m "chore: add react-easy-crop dependency"
```

---

### Task 2: 创建 Avatar 通用组件

**Files:**
- Create: `src/components/Avatar.tsx`

- [ ] **Step 1: 编写 Avatar 组件**

```typescript
// src/components/Avatar.tsx
interface AvatarProps {
  src?: string | null;
  name: string;
  size?: "sm" | "md" | "lg";
}

const sizeMap = {
  sm: "w-8 h-8 text-xs",
  md: "w-10 h-10 text-sm",
  lg: "w-20 h-20 text-2xl",
};

export function Avatar({ src, name, size = "md" }: AvatarProps) {
  const sizeClass = sizeMap[size];

  if (src) {
    return (
      <img
        src={src}
        alt={name}
        className={`${sizeClass} rounded-full object-cover shrink-0`}
      />
    );
  }

  return (
    <div
      className={`${sizeClass} rounded-full bg-gradient-to-br from-primary-400 to-primary-600 flex items-center justify-center text-white font-bold shrink-0`}
    >
      {name.charAt(0).toUpperCase()}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/Avatar.tsx
git commit -m "feat: add Avatar component"
```

---

### Task 3: 创建 AvatarCropModal 裁剪弹窗

**Files:**
- Create: `src/components/AvatarCropModal.tsx`

- [ ] **Step 1: 编写 AvatarCropModal**

```typescript
// src/components/AvatarCropModal.tsx
"use client";

import { useState, useRef, useCallback } from "react";
import Cropper, { Area } from "react-easy-crop";

interface AvatarCropModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSave: (url: string) => void;
}

export function AvatarCropModal({ isOpen, onClose, onSave }: AvatarCropModalProps) {
  const [imageSrc, setImageSrc] = useState<string | null>(null);
  const [crop, setCrop] = useState({ x: 0, y: 0 });
  const [zoom, setZoom] = useState(1);
  const [croppedAreaPixels, setCroppedAreaPixels] = useState<Area | null>(null);
  const [uploading, setUploading] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const onCropComplete = useCallback((_: Area, croppedAreaPixels: Area) => {
    setCroppedAreaPixels(croppedAreaPixels);
  }, []);

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => setImageSrc(reader.result as string);
    reader.readAsDataURL(file);
  };

  const createCroppedImage = async (): Promise<Blob | null> => {
    if (!imageSrc || !croppedAreaPixels) return null;
    const image = new Image();
    image.src = imageSrc;
    await new Promise<void>((resolve) => { image.onload = () => resolve(); });
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    if (!ctx) return null;
    const { x, y, width, height } = croppedAreaPixels;
    canvas.width = width;
    canvas.height = height;
    ctx.drawImage(image, x, y, width, height, 0, 0, width, height);
    return new Promise((resolve) => canvas.toBlob((b) => resolve(b), "image/jpeg", 0.9));
  };

  const handleSave = async () => {
    const blob = await createCroppedImage();
    if (!blob) return;
    setUploading(true);
    try {
      const fd = new FormData();
      fd.append("file", blob, "avatar.jpg");
      const res = await fetch("/api/upload", { method: "POST", body: fd });
      const data = await res.json();
      if (data.success && data.url) {
        onSave(data.url);
        setImageSrc(null);
        onClose();
      } else {
        alert(data.error || "上传失败");
      }
    } catch {
      alert("上传失败");
    } finally {
      setUploading(false);
    }
  };

  const handleClose = () => {
    setImageSrc(null);
    onClose();
  };

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm"
         onClick={(e) => { if (e.target === e.currentTarget) handleClose(); }}>
      <div className="bg-white rounded-2xl shadow-xl w-full max-w-md mx-4 overflow-hidden">
        <div className="px-6 py-4 border-b border-primary-200/20 flex items-center justify-between">
          <h3 className="font-semibold text-primary-800">裁剪头像</h3>
          <button onClick={handleClose} className="text-primary-400 hover:text-primary-600 text-lg leading-none">&times;</button>
        </div>

        {!imageSrc ? (
          <div className="p-6 text-center">
            <div
              onClick={() => fileInputRef.current?.click()}
              className="border-2 border-dashed border-primary-200/40 rounded-xl p-10 cursor-pointer hover:border-primary-300/60 transition-colors"
            >
              <span className="text-4xl text-primary-300">📷</span>
              <p className="mt-3 text-sm text-primary-500/50">点击选择图片</p>
              <p className="text-xs text-primary-400/30 mt-1">JPG / PNG / WebP</p>
            </div>
            <input ref={fileInputRef} type="file" accept="image/*" onChange={handleFileChange} className="hidden" />
          </div>
        ) : (
          <>
            <div className="relative w-full h-72 bg-gray-900">
              <Cropper
                image={imageSrc}
                crop={crop}
                zoom={zoom}
                aspect={1}
                cropShape="round"
                onCropChange={setCrop}
                onZoomChange={setZoom}
                onCropComplete={onCropComplete}
              />
            </div>
            <div className="px-6 py-3">
              <label className="text-xs text-primary-400/60">缩放</label>
              <input
                type="range"
                min={1}
                max={3}
                step={0.01}
                value={zoom}
                onChange={(e) => setZoom(parseFloat(e.target.value))}
                className="w-full mt-1"
              />
            </div>
            <div className="px-6 pb-5 flex gap-3">
              <button onClick={handleClose} className="flex-1 py-2 text-sm rounded-lg border border-primary-200/40 text-primary-600 hover:bg-primary-50 transition-colors">
                取消
              </button>
              <button onClick={handleSave} disabled={uploading} className="flex-1 py-2 text-sm rounded-lg bg-primary-900 text-white hover:bg-primary-800 transition-colors disabled:opacity-50">
                {uploading ? "上传中..." : "确认"}
              </button>
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/AvatarCropModal.tsx
git commit -m "feat: add AvatarCropModal with react-easy-crop"
```

---

### Task 4: 更新 ProfileEditForm 集成头像

**Files:**
- Modify: `src/components/ProfileEditForm.tsx`

- [ ] **Step 1: 重写 ProfileEditForm 添加头像区域**

完整替换 `src/components/ProfileEditForm.tsx`:

```typescript
// src/components/ProfileEditForm.tsx
"use client";

import { useState } from "react";
import { updateProfile } from "@/actions/profile";
import { Avatar } from "@/components/Avatar";
import { AvatarCropModal } from "@/components/AvatarCropModal";

interface Props {
  initialData: {
    bio: string;
    website: string;
    location: string;
    github: string;
    twitter: string;
    avatar?: string | null;
  };
  username: string;
}

export function ProfileEditForm({ initialData, username }: Props) {
  const [saved, setSaved] = useState(false);
  const [error, setError] = useState("");
  const [avatarUrl, setAvatarUrl] = useState<string | null>(initialData.avatar || null);
  const [cropOpen, setCropOpen] = useState(false);

  return (
    <form
      action={async (formData) => {
        setSaved(false);
        setError("");
        if (avatarUrl) formData.set("avatar", avatarUrl);
        const result = await updateProfile(formData);
        if (result.success) {
          setSaved(true);
          setTimeout(() => setSaved(false), 2000);
        } else {
          setError(result.error || "保存失败");
        }
      }}
      className="space-y-4"
    >
      {/* Avatar */}
      <div>
        <label className="block text-sm font-medium text-primary-600/60 mb-2">头像</label>
        <button
          type="button"
          onClick={() => setCropOpen(true)}
          className="group relative rounded-full overflow-hidden ring-2 ring-primary-200/50 hover:ring-primary-400/50 transition-all"
        >
          <Avatar src={avatarUrl} name={username} size="lg" />
          <div className="absolute inset-0 bg-black/30 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center">
            <span className="text-white text-xs font-medium">更换</span>
          </div>
        </button>
      </div>

      <AvatarCropModal
        isOpen={cropOpen}
        onClose={() => setCropOpen(false)}
        onSave={(url) => { setAvatarUrl(url); setCropOpen(false); }}
      />

      <div>
        <label className="block text-sm font-medium text-primary-600/60 mb-1">个人简介</label>
        <textarea
          name="bio"
          defaultValue={initialData.bio}
          rows={3}
          className="input-field min-h-[80px] resize-y"
          placeholder="介绍一下自己..."
        />
      </div>

      <div>
        <label className="block text-sm font-medium text-primary-600/60 mb-1">位置</label>
        <input
          name="location"
          defaultValue={initialData.location}
          className="input-field"
          placeholder="例如：北京"
        />
      </div>

      <div>
        <label className="block text-sm font-medium text-primary-600/60 mb-1">个人网站</label>
        <input
          name="website"
          defaultValue={initialData.website}
          className="input-field"
          placeholder="例如：example.com"
        />
      </div>

      <div className="flex gap-4">
        <div className="flex-1">
          <label className="block text-sm font-medium text-primary-600/60 mb-1">GitHub</label>
          <input
            name="github"
            defaultValue={initialData.github}
            className="input-field"
            placeholder="GitHub 用户名"
          />
        </div>
        <div className="flex-1">
          <label className="block text-sm font-medium text-primary-600/60 mb-1">Twitter</label>
          <input
            name="twitter"
            defaultValue={initialData.twitter}
            className="input-field"
            placeholder="Twitter 用户名"
          />
        </div>
      </div>

      <div className="flex items-center gap-3">
        <button type="submit" className="btn-primary text-sm">
          保存资料
        </button>
        {saved && <span className="text-xs text-green-500">保存成功</span>}
        {error && <span className="text-xs text-red-500">{error}</span>}
      </div>
    </form>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/ProfileEditForm.tsx
git commit -m "feat: integrate avatar upload in ProfileEditForm"
```

---

### Task 5: 更新 updateProfile 和 account 页面

**Files:**
- Modify: `src/actions/profile.ts`
- Modify: `src/app/account/page.tsx`

- [ ] **Step 1: 更新 profile.ts 增加 avatar 字段**

编辑 `src/actions/profile.ts`，在 line 17 后增加 avatar 提取，在 line 21 的 data 中增加 avatar:

```typescript
// 在 line 17 后添加:
const avatar = (formData.get("avatar") as string)?.trim() || null;

// 将 line 21 的 data 改为:
data: { bio, website, location, github, twitter, avatar },
```

- [ ] **Step 2: 更新 account page 传递 avatar 给 ProfileEditForm**

编辑 `src/app/account/page.tsx`:

在 select 中增加（line 29 后）:
```typescript
avatar: true,
```

在 ProfileEditForm 的 initialData 中增加（line 76 后）:
```typescript
avatar: user.avatar,
```

在 ProfileEditForm 组件上增加 `username` prop:
```tsx
<ProfileEditForm
  initialData={{
    bio: user.bio || "",
    website: user.website || "",
    location: user.location || "",
    github: user.github || "",
    twitter: user.twitter || "",
    avatar: user.avatar,
  }}
  username={user.username}
/>
```

- [ ] **Step 3: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/actions/profile.ts src/app/account/page.tsx
git commit -m "feat: add avatar field to updateProfile action and account page"
```

---

### Task 6: 更新 Auth Session 传递 avatar

**Files:**
- Modify: `src/lib/auth.ts`

- [ ] **Step 1: auth.ts 变更**

在 `authorize` 回调的 return (line 26) 中添加 `avatar`:

```typescript
return {
  id: user.id,
  email: user.email,
  name: user.username,
  role: user.role,
  avatar: user.avatar,
};
```

在 `jwt` callback (line 43 后) 中添加:

```typescript
if (user) {
  token.role = (user as any).role;
  token.id = user.id;
  token.avatar = (user as any).avatar;
}
```

在 `session` callback (line 49 后) 中添加:

```typescript
(session.user as any).avatar = token.avatar;
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/lib/auth.ts
git commit -m "feat: pass avatar through auth session"
```

---

### Task 7: 更新 Header 和 NavBar 传递并显示 avatar

**Files:**
- Modify: `src/components/Header.tsx`
- Modify: `src/components/NavBar.tsx`

- [ ] **Step 1: 更新 Header 传递 avatar**

编辑 `src/components/Header.tsx`:

添加:
```typescript
const userAvatar = (session?.user as any)?.avatar || null;
```

NavBar 调用改为:
```tsx
return <NavBar isAdmin={isAdmin} isLoggedIn={isLoggedIn} userName={userName} userAvatar={userAvatar} />;
```

- [ ] **Step 2: 更新 NavBar 接收并显示 avatar**

编辑 `src/components/NavBar.tsx`:

添加 import:
```typescript
import { Avatar } from "@/components/Avatar";
```

在 Props 接口中添加:
```typescript
userAvatar: string | null;
```

解构添加:
```typescript
export function NavBar({ isAdmin, isLoggedIn, userName, userAvatar }: NavBarProps) {
```

替换桌面端用户名区域 (lines 129-138):
```tsx
{userName ? (
  <span className="flex items-center gap-2 text-sm text-primary-600/80">
    <Link href="/account" className="flex items-center gap-2 hover:opacity-80 transition-opacity">
      <Avatar src={userAvatar} name={userName} size="sm" />
      <span className="hidden sm:inline text-xs font-medium text-primary-600/50 hover:text-primary-700 transition-colors">
        {userName}
      </span>
    </Link>
    <LogoutButton />
  </span>
) : (
  <Link href="/login" className="btn-primary text-sm !py-1.5 !px-4 !rounded-lg ml-1">
    登录
  </Link>
)}
```

替换移动端用户区域 (lines 252-262):
```tsx
{userName && (
  <div className="mt-4 pt-4 border-t border-primary-200/30 px-4">
    <Link href="/account" className="flex items-center gap-3 mb-2">
      <Avatar src={userAvatar} name={userName} size="sm" />
      <span className="text-sm text-primary-600/70 hover:text-primary-800 transition-colors">
        {userName}
      </span>
    </Link>
    <LogoutButton />
  </div>
)}
```

- [ ] **Step 3: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/Header.tsx src/components/NavBar.tsx
git commit -m "feat: display avatar in NavBar"
```

---

### Task 8: 更新文章详情页作者卡片使用 Avatar

**Files:**
- Modify: `src/app/post/[slug]/page.tsx`

- [ ] **Step 1: 替换两处字母圆圈为 Avatar**

添加 import:
```typescript
import { Avatar } from "@/components/Avatar";
```

替换作者 row 中的字母圆圈 (约 line 120-122):
```tsx
<Avatar src={post.author.avatar || null} name={post.author.username} size="md" />
```

替换底部作者卡片中的字母圆圈 (约 line 193-195):
```tsx
<Avatar src={post.author.avatar || null} name={post.author.username} size="md" />
```

确保 post 查询中 author 的 select 包含 avatar:
```typescript
author: { select: { username: true, avatar: true } },
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/app/post/[slug]/page.tsx
git commit -m "feat: use Avatar in post detail page author card"
```

---

### Task 9: 更新用户主页使用 Avatar

**Files:**
- Modify: `src/app/user/[username]/page.tsx`

- [ ] **Step 1: 替换字母圆圈为 Avatar**

添加 import:
```typescript
import { Avatar } from "@/components/Avatar";
```

替换 (lines 50-52):
```tsx
<Avatar src={user.avatar} name={user.username} size="lg" />
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/app/user/[username]/page.tsx
git commit -m "feat: use Avatar in user profile page"
```

---

### Task 10: 更新评论列表使用 Avatar

**Files:**
- Modify: `src/components/CommentItem.tsx`

- [ ] **Step 1: 在评论中显示头像**

添加 import:
```typescript
import { Avatar } from "@/components/Avatar";
```

在用户名 Link 前添加 Avatar (line 32 后):
```tsx
<div className="flex items-center gap-2 text-sm mb-1">
  <Avatar src={comment.user.avatar} name={comment.user.username} size="sm" />
  <Link href={`/user/${comment.user.username}`} className="font-medium text-primary-800 hover:text-primary-600 transition-colors">{comment.user.username}</Link>
  <span className="text-primary-600/60 text-xs">
    {new Date(comment.createdAt).toLocaleDateString("zh-CN")}
  </span>
</div>
```

- [ ] **Step 2: Commit**

```bash
cd C:/Users/shi/Desktop/book/blog
git add src/components/CommentItem.tsx
git commit -m "feat: use Avatar in comment list"
```

---

### Task 11: 构建验证

**Files:** None

- [ ] **Step 1: TypeScript 检查**

```bash
cd C:/Users/shi/Desktop/book/blog
npx tsc --noEmit
```

Expected: 无类型错误。

- [ ] **Step 2: 构建**

```bash
cd C:/Users/shi/Desktop/book/blog
npm run build
```

Expected: 构建成功，所有路由编译通过。

- [ ] **Step 3: 修复任何问题并提交**

```bash
cd C:/Users/shi/Desktop/book/blog
git add -A && git commit -m "fix: build issues from avatar feature" || echo "no fixes needed"
```
