# LoginView — GIF 背景 + 左侧卡片

## Context

用户提供了 `frontend/public/login-bg.gif`（38MB）作为登录页背景。需求极简：GIF 全屏背景 + 左侧一个登录卡片，无其他装饰。

## ⚠️ 性能警告

38MB 的 GIF 作为网页背景是严重问题：
- 首次加载可能需要 20-60 秒（取决于网速）
- 用户看到白屏/黑屏等待，跳出率极高
- 移动端流量消耗巨大

**强烈建议**：将 GIF 转为 MP4/WebM 视频格式（预计可压缩到 2-5MB），用 `<video autoplay loop muted playsinline>` 替代。如需帮助转换请告知。

以下方案同时覆盖两种实现。

---

## 设计方案

```
┌──────────────────────────────────────────────┐
│                                              │
│  ┌────────────────┐                          │
│  │                │                          │
│  │  鲸 ⬡         │      GIF / 视频背景        │
│  │  小鲸鱼·智能体  │      background-size:     │
│  │                │      cover                │
│  │  账号 _______  │                          │
│  │  密码 _______  │                          │
│  │  [ 登 录 ]    │                          │
│  │                │                          │
│  │  注册新账号 →   │                          │
│  └────────────────┘                          │
│       ↑ left-16                             │
│                                              │
└──────────────────────────────────────────────┘
```

- **背景**：全屏 GIF/视频，`background-size: cover` 或 `<video>` 标签
- **卡片**：白色半透明 `bg-white/85 backdrop-blur-md`，宽 380px，定位左侧 `left-[10%]`，垂直居中
- **内容**：与方案 2 相同的品牌 + 表单
- **移动端**：卡片居中全宽

## 实现方式

### GIF 方案（当前，不推荐）

```html
<div class="fixed inset-0" style="background: url('/login-bg.gif') center/cover no-repeat" />
```

### 视频方案（推荐）

将 `login-bg.gif` 转为 MP4，替换为：

```html
<video autoplay loop muted playsinline class="fixed inset-0 size-full object-cover">
  <source src="/login-bg.mp4" type="video/mp4" />
</video>
```

## 改动文件

- `frontend/src/views/LoginView.vue`
- `frontend/src/views/RegisterView.vue`（同步）

## 验证

`npm run build`
