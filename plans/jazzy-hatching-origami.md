# Landing 页面重构 + 路由守卫改造 开发计划

## 背景

当前项目存在以下问题：
1. Landing 页面 (`/landing`) 是独立静态 HTML，不在 Vue SPA 内，导致路由守卫无法保护从 landing 跳转到后端服务的链路
2. 用户必须先登录才能看到 Portal，再通过 Portal 进入 landing，流程冗余
3. 缺少"我的管理"统一入口和"法务"模块占位
4. landing 页面气泡没有点击反馈特效
5. landing 作为首页却在 URL 中有 `/landing` 路径，不够直观

**目标**：将 landing 页面改造为 Vue SPA 的公开首页，通过路由守卫实现"浏览公开、使用需登录"的体验。

---

## 核心架构决策

**将 landing 页面从静态 HTML 转为 Vue 组件 (`LandingView.vue`)**，挂载在根路由 `/`（无需认证）。Agent 气泡链接指向 auth-gated 的 Vue 路由，由路由守卫统一拦截未登录用户 → 跳转登录页（带 `redirect` 参数）→ 登录成功后回到目标页面。

---

## 分步实施计划

### 第 1 步：创建 LandingView.vue（公开首页）

**文件**：`frontend/agent-workbench/src/views/LandingView.vue`（新建）

将 `public/landing/index.html` 的设计迁移为 Vue SFC：
- 深色渐变背景 (`#0f0c29 → #302b63 → #24243e`)
- 环境光晕 (与 PortalView 一致的 ambient orbs)
- 毛玻璃卡片容器 (`glass-card`)
- 浮动鲸鱼 emoji + 渐变标题 "小鲸鱼智能体"
- 副标题 "选择一个智能体开始对话"
- Canvas 粒子背景（加载 `particles.js` + `whale.js` + `background.js`）

**气泡布局**（2 列网格，移动端单列）：

| 气泡 | 图标 | 颜色主题 | 链接目标 | 状态 |
|---|---|---|---|---|
| 健康管理 | 🏥 | 绿色 `--color-health` | `/#/agent/health` | 正常 |
| 影视创作 | 🎬 | 紫色 `--color-film` | `/#/agent/film` | 正常 |
| 知识学习 | 📚 | 蓝色 `--color-learn` | `/#/agent/learn` | 正常 |
| 财务分析 | 💰 | 琥珀色 `--color-finance` | `/#/agent/finance` | 正常 |
| 法务智能体 | ⚖️ | 灰色/ slate | 无（点击弹出"即将上线"提示）| 占位 |
| 我的管理 | 📊 | 靛蓝色 (indigo) | `/#/dashboard` | 正常 |

每个气泡设计：
- 毛玻璃背景 + 边框，悬停上浮 + 发光边框
- 左侧 emoji 图标，右侧标题 + 英文标签
- `router-link` 组件（法务除外）

**复用参考**：
- 设计风格对齐 [PortalView.vue](frontend/agent-workbench/src/views/PortalView.vue)（渐变、光晕、毛玻璃、动画）
- 颜色变量复用 [theme.css](public/assets/css/theme.css) 现有 `--color-*` 变量
- Canvas 背景复用 `public/assets/js/` 下的 particles.js、whale.js、background.js

---

### 第 2 步：创建 AgentRedirectView.vue（认证网关）

**文件**：`frontend/agent-workbench/src/views/AgentRedirectView.vue`（新建）

一个轻量级 Vue 组件，作用是：
1. 路由设置为 `meta: { auth: true }` — 路由守卫自动拦截未登录用户
2. 已登录用户进入后，根据路由参数重定向到对应后端 Agent UI

```
路由: /agent/:name
- /agent/health  → window.location.href = '/health/ui'
- /agent/film    → window.location.href = '/film/ui'
- /agent/learn   → window.location.href = '/learn/ui'
- /agent/finance → window.location.href = '/finance/ui'
```

使用 `onMounted` 执行重定向，展示一个简短的加载过渡（鲸鱼动画 + "正在进入..." 文字）。

---

### 第 3 步：改造路由守卫 + LoginView

**文件**：`frontend/agent-workbench/src/router/index.ts`（修改）

路由表变更：
```typescript
routes: [
  // 公开路由
  { path: '/',           name: 'landing',  component: LandingView,       meta: { title: '小鲸鱼智能体' } },
  { path: '/login',      name: 'login',    component: LoginView,          meta: { title: '登录', guest: true } },
  { path: '/register',   name: 'register', component: RegisterView,       meta: { title: '注册', guest: true } },
  
  // Agent 认证网关
  { path: '/agent/:name', name: 'agent-redirect', component: AgentRedirectView, meta: { auth: true } },
  
  // 旧 portal 重定向
  { path: '/portal', redirect: '/' },
  
  // 控制台（需要认证）
  { path: '/console', component: ConsoleLayout, meta: { auth: true }, redirect: '/dashboard',
    children: [ /* ...现有 9 个子路由，路径不变 */ ]
  },
]
```

**关键**：控制台路由前缀从 `/` 改为 `/console`，避免与 landing 的 `/` 冲突。

> 或者更简单的方案：控制台保持 `/` 不变，landing 设为独立路由。Vue Router 按定义顺序匹配，把 landing 放在前面，`/` 精确匹配 landing，其余走控制台布局。但这样 `/dashboard` 会同时匹配 `/`（landing 是 exact `/`）和控制台子路由，需要调整。

**推荐方案**：不改变现有路由结构，利用 Vue Router 的优先级规则：
- `/` (landing) 放在 routes 数组最前面，作为 exact match
- 控制台路由 `path: '/'` 使用 `redirect` 但添加条件——实际上 Vue Router 会先匹配到 `/` 就停下来了

**最简方案（实际采用）**：
- Landing 页面路由：`/` ，不设置 `meta.auth`
- 现有控制台路由：`/` 改为 `/app`，所有子路由前缀变为 `/app/dashboard`、`/app/workbench` 等
- `/portal` 重定向到 `/`
- 更新 `AppSidebar.vue`、`PortalView.vue` 中的硬编码路径
- 这样改动最少，路由结构最清晰

```
/                → LandingView (公开)
/login           → LoginView (访客)
/register        → RegisterView (访客)
/agent/:name     → AgentRedirectView (需认证)
/app             → ConsoleLayout (需认证)
  /app/dashboard
  /app/workbench
  /app/devices
  ...
```

**`beforeEach` 守卫改造**：
```typescript
router.beforeEach((to, _from) => {
  const { isLoggedIn } = useAuth()
  
  // 需要认证但未登录 → 带 redirect 参数跳转登录
  if (to.meta.auth && !isLoggedIn.value) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  
  // 已登录用户访问访客页面 → 跳首页
  if (to.meta.guest && isLoggedIn.value) return '/'
})
```

**LoginView.vue 改造**：
- 读取 `route.query.redirect` 参数
- 登录成功后：优先跳转 `redirect` 目标，否则跳转 `/`
- 注册页也支持 `redirect` 透传（注册成功后去登录页，登录后回到目标）

**文件**：`frontend/agent-workbench/src/views/LoginView.vue`（修改）
```typescript
const redirect = (route.query.redirect as string) || '/'
// 登录成功后
router.replace(redirect)
```

**文件**：`frontend/agent-workbench/src/views/RegisterView.vue`（修改）
- 注册成功后跳转登录时透传 `redirect` 参数

---

### 第 4 步：添加点击特效

**文件**：`frontend/agent-workbench/src/views/LandingView.vue` 内

每个气泡点击时触发：
1. **CSS `:active` 缩放**：`transform: scale(0.94)` + `transition: transform 0.1s`
2. **涟漪波纹**：点击位置扩散出一个圆形波纹（使用 `@keyframes ripple`，从 `scale(0)` + `opacity(1)` → `scale(4)` + `opacity(0)`，持续 0.6s）
3. **法务气泡**：点击弹出 toast/气泡提示 "即将上线，敬请期待"

实现方式：定义一个 `handleClick(e: MouseEvent)` 函数，在气泡容器上绑定 `@click`。创建临时 DOM 元素（ripple span），绝对定位在点击坐标，动画结束后自动移除。

也可使用一个通用的 Vue composable：`composables/useRipple.ts`

---

### 第 5 步：更新关联文件和清理

**修改文件清单**：

| 文件 | 操作 | 说明 |
|---|---|---|
| `frontend/agent-workbench/src/router/index.ts` | 修改 | 添加新路由，改造守卫，控制台前缀变更 |
| `frontend/agent-workbench/src/views/LandingView.vue` | 新建 | 公开首页 |
| `frontend/agent-workbench/src/views/AgentRedirectView.vue` | 新建 | Agent 认证网关 |
| `frontend/agent-workbench/src/views/LoginView.vue` | 修改 | 支持 redirect 参数 |
| `frontend/agent-workbench/src/views/RegisterView.vue` | 修改 | 透传 redirect 参数 |
| `frontend/agent-workbench/src/views/PortalView.vue` | 修改 | 更新"智能体应用"链接为 `/` |
| `frontend/agent-workbench/src/components/console/AppSidebar.vue` | 修改 | 更新导航链接（如果控制台路由前缀变更） |
| `frontend/agent-workbench/src/composables/useRipple.ts` | 新建(可选) | 涟漪特效 composable |
| `public/assets/css/theme.css` | 修改 | 添加法务(`--color-legal`)和管理(`--color-manage`)颜色变量 |
| `deploy/nginx/nginx.conf` | 修改 | `/landing` 301 重定向到 `/`，移除 `/landing/` location |
| `public/landing/index.html` | 删除 | 不再需要静态 landing |
| `public/portal/index.html` | 修改 | 重定向目标从 `/#/portal` 改为 `/#/` |

**AppSidebar.vue 路径更新**（如果控制台前缀改为 `/app`）：
- `to="/dashboard"` → `to="/app/dashboard"`
- `to="/workbench"` → `to="/app/workbench"`
- 等等，共 9 个导航链接

---

### 第 6 步：Nginx 配置更新

```nginx
# 旧的 landing 重定向到首页
location = /landing {
    return 301 /;
}
location /landing/ {
    return 301 /;
}

# portal 重定向到首页  
location = /portal {
    return 301 /;
}
location /portal/ {
    return 301 /;
}

# 根路径服务 Vue SPA（不变）
location / {
    root "D:/200_Work/xiaojingyu/frontend/agent-workbench/dist";
    try_files $uri $uri/ /index.html;
}
```

---

## 路由汇总（改造后）

| 路径 | 认证 | 说明 |
|---|---|---|
| `/#/` | 否 | 公开首页（landing），6 个功能气泡 |
| `/#/login` | 访客 | 登录，支持 `?redirect=` |
| `/#/register` | 访客 | 注册 |
| `/#/agent/health` | 是 | 认证后重定向到 `/health/ui` |
| `/#/agent/film` | 是 | 认证后重定向到 `/film/ui` |
| `/#/agent/learn` | 是 | 认证后重定向到 `/learn/ui` |
| `/#/agent/finance` | 是 | 认证后重定向到 `/finance/ui` |
| `/#/app/dashboard` | 是 | 控制台·数据看板 |
| `/#/app/workbench` | 是 | 控制台·我的助手 |
| ... | 是 | 其余控制台页面 |
| `/#/portal` | — | 301 重定向到 `/` |

---

## 验证方案

1. **公开访问测试**：浏览器无痕窗口打开 `https://localhost:9000/`，应看到 landing 页面（6 个气泡），无需登录
2. **认证拦截测试**：点击"健康管理" → 应跳转到登录页，URL 带 `?redirect=%2Fagent%2Fhealth`
3. **登录回跳测试**：输入账号密码登录 → 应自动跳转到 `/health/ui`（健康管理后端界面）
4. **我的管理测试**：返回首页，点击"我的管理" → 如已登录直接进入 dashboard；如未登录则走登录流程后进入 dashboard
5. **法务占位测试**：点击"法务智能体" → 显示"即将上线"提示，不跳转
6. **点击特效测试**：点击任意气泡 → 出现涟漪波纹动画
7. **Portal 兼容测试**：访问 `/#/portal` → 自动跳转到首页
8. **旧 URL 兼容测试**：访问 `/landing` → 301 跳转到 `/`
9. **控制台功能测试**：sidebar 导航、workbench、agent 切换等均正常
10. **构建测试**：`npm run build` 无报错，`npm run dev` 正常启动
