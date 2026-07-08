# Context
用户希望在未登录首页（`/`，即 [frontend/src/views/LandingView.vue](frontend/src/views/LandingView.vue)）也能切换深色模式；当前只能在登录后控制台顶部切换。现有项目的主题机制已经是全局生效的：应用启动时在 [frontend/src/main.ts](frontend/src/main.ts) 读取 `localStorage['workbench-theme']` 并切换 `document.documentElement.dark`，而登录后页头与设置页又各自复制了一套主题切换逻辑。此次改动的目标是在首页右上角补上切换入口，同时把重复主题逻辑收敛为一处，保证首页、登录后页头、设置页三处状态一致且刷新后仍能保持主题。

# 推荐方案
新增一个轻量主题组合式函数，统一管理：
- 主题模式（`light | dark | auto`）
- 当前实际是否为深色（`isDarkTheme`）
- 初始化主题（挂载前执行，避免首屏闪烁）
- 设置主题 / 单按钮切换主题

然后让首页、控制台页头、设置页都复用这套逻辑：首页只新增一个右上角按钮，不再复制第三份主题代码。

# 关键改动文件
1. `frontend/src/composables/useTheme.ts`
   - 新增轻量共享主题逻辑。
   - 暴露最小 API：`initTheme()`、`useTheme()`。
   - `useTheme()` 返回：`themeMode`、`isDarkTheme`、`setTheme(mode)`、`toggleTheme()`。
   - 内部统一复用现有模式：读写 `localStorage['workbench-theme']`，并通过 `document.documentElement.classList.toggle('dark', ...)` 控制主题。
   - `auto` 模式下沿用系统偏好 `window.matchMedia('(prefers-color-scheme: dark)')`。

2. `frontend/src/main.ts`
   - 用 `initTheme()` 替换当前内联的主题初始化逻辑。
   - 保持“在 `createApp()` 前初始化主题”的时机，避免首页首次加载时亮/暗闪烁。

3. `frontend/src/components/console/AppHeader.vue`
   - 删除本地 `isDarkTheme` 状态、`toggleTheme()` 主题 DOM 操作，以及 `onMounted` 里的重复初始化。
   - 改为直接使用 `useTheme()` 提供的 `isDarkTheme` 和 `toggleTheme()`。
   - 保持当前按钮交互与图标表现不变，但修正与 `auto` 模式可能不一致的问题。

4. `frontend/src/views/SettingsView.vue`
   - 删除本地主题状态与 `toggleTheme(mode)` 实现。
   - 改为使用 `useTheme()` 的 `themeMode` 与 `setTheme(mode)`。
   - 保持“浅色 / 深色 / 自动”三按钮 UI，不改交互，仅改数据来源。

5. `frontend/src/views/LandingView.vue`
   - 引入 `useTheme()`，在页面右上角新增主题切换按钮。
   - 位置：挂在根容器 `landing-page relative` 内，使用绝对定位（如 `absolute top-4 right-4 z-30`，桌面端可略放大边距）。
   - 图标与 aria-label 绑定 `isDarkTheme`，点击调用 `toggleTheme()`。
   - 复用现有站内风格（毛玻璃 / 边框 / backdrop-blur），不新增全局样式抽象。
   - 同时修改拖拽入口保护逻辑：主题按钮增加例如 `data-no-drag` 标记，在 `onPointerDown()` 中与 `.bubble` 一样跳过，避免点击按钮误触发轨道拖拽。

# 可复用的现有逻辑
- 启动时恢复主题的模式： [frontend/src/main.ts](frontend/src/main.ts)
- 登录后页头的单按钮切换交互： [frontend/src/components/console/AppHeader.vue](frontend/src/components/console/AppHeader.vue)
- 设置页的三态主题模式（light/dark/auto）： [frontend/src/views/SettingsView.vue](frontend/src/views/SettingsView.vue)
- 首页本身已具备暗色样式适配，无需再补主题基础样式： [frontend/src/views/LandingView.vue](frontend/src/views/LandingView.vue)

# 实施注意点
- 首页按钮必须跳过现有全局 `pointerdown / pointermove / wheel` 拖拽链路，否则会和轨道滑动交互冲突。
- 单按钮（首页、页头）应基于“当前实际主题”切换，而不是在 `light/dark/auto` 三态间循环；点击后直接落成显式 `light` 或 `dark`，更符合用户直觉。
- 设置页继续保留 `auto`；页头和首页按钮只做快捷切换。
- `useTheme.ts` 内部可顺手加 `window/document` 守卫，保持实现稳健；但不引入 store、不做额外抽象。

# 验证方案
1. 启动前端，打开未登录首页 `/`。
2. 在首页右上角点击主题按钮，确认：
   - 页面立即在浅/深色间切换。
   - `html` 根节点的 `dark` class 同步变化。
   - `localStorage['workbench-theme']` 被正确写入 `light` 或 `dark`。
3. 刷新首页，确认主题保持不变，且无明显首屏闪烁。
4. 登录进入控制台，确认页头按钮图标状态与首页一致，并可继续切换。
5. 进入设置页，分别切换“浅色 / 深色 / 自动”，确认：
   - 当前页面主题即时更新。
   - 返回首页后按钮状态与实际主题一致。
   - `auto` 模式下会跟随系统偏好。
6. 在首页桌面端拖动气泡轨道、移动端点击卡片和主题按钮，确认没有手势冲突。
