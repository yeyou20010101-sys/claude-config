# 智能体应用页面前端迁移方案：Vue 3 + TypeScript + Tailwind CSS

## 背景

项目前端目前是双轨并行架构：

| | 轨道 A：Vue SPA（控制台） | 轨道 B：Agent Shell（聊天页） |
|---|---|---|
| 技术栈 | ✅ Vue 3 + TS + Tailwind CSS 4 | ❌ 原生 JS + 纯 CSS + HTML 模板 |
| 代码量 | ~6,200 行 | ~8,900 行（JS ~7,800 + CSS ~1,100） |
| 构建 | Vite | Python 脚本字符串替换 |
| 复用度 | 组件化良好 | 4 个 agent 大量重复代码（~70% 重复） |

**目标**：将轨道 B 迁移到 Vue 3 + TS + Tailwind，统一技术栈，消除双轨维护负担。

---

## 核心架构决策

### 1. Agent 聊天页作为 Vue SPA 路由（非独立 HTML 页面）

- 新路由：`/agent/health`、`/agent/film`、`/agent/learn`、`/agent/finance`
- 使用独立的 `AgentLayout.vue`（三栏布局），不复用 ConsoleLayout
- 后端不再提供 HTML 页面（`/health/ui` 等），仅提供 API
- API 调用通过 Vite/Nginx 代理到各 agent 后端，路径不变（`/health/api/chat/stream` 等）
- 旧的 `/agent/:name` 路由改为重定向到新路由

### 2. 最大程度复用共享逻辑

- 4 个 agent 的 ~70% 重复代码提取为 Vue composable 和共享组件
- film-chat.js 内联的 SSE 处理（~290 行）合并到 `useSSEStream`，与另 3 个 agent 统一
- 每个 agent 的 View 层只包含差异逻辑（~100-200 行 vs 原来的 500-2200 行）

### 3. 渐进迁移，每阶段可独立验证

- 新旧页面共存（Vite 代理 `/health/ui` 指向旧 HTML，`/agent/health` 指向新 Vue 页面）
- 每个 agent 迁移完成后即可测试，不影响其他 agent
- 全部迁移完成后再清理旧代码

---

## 组件树

```
AgentLayout.vue                    ← 三栏 flex 布局（替代 agent-layout.css）
├── AgentSidebar.vue               ← 左侧 agent 导航（替代 agent-sidebar.js）
├── ChatArea.vue                   ← 消息列表容器（核心中间栏）
│   ├── WelcomeCard.vue            ← 欢迎消息 + 建议卡片
│   ├── MessageBubble.vue (v-for)  ← 单条消息气泡（用户/agent）
│   │   ├── MediaResults.vue       ← 图片/视频渲染（film 专有）
│   │   ├── TaskProgress.vue       ← 任务进度条 + 轮询（film 专有）
│   │   ├── SuggestionChips.vue    ← 回复后的建议芯片
│   │   └── MessageActions.vue     ← 复制 + 朗读按钮
│   └── ScrollBottomButton         ← 回到底部按钮
├── AgentInput.vue                 ← 输入栏（替代 agent-input.css）
│   ├── QuickChips.vue             ← 技能芯片 + TTS/附件开关
│   ├── SkillModeHint.vue          ← 当前技能模式提示
│   ├── PreviewRow.vue             ← 图片/文件预览行
│   └── VoiceInput.vue             ← 按住说话按钮
└── ChatHistorySidebar.vue         ← 右侧历史会话（替代 agent-history.js）
    └── SessionList                ← 会话列表（含删除/新建）
```

---

## Composable 设计

| Composable | 替代的原生 JS | 行数估算 | 职责 |
|---|---|---|---|
| `useChat.ts` | 各 agent chat 脚本的核心逻辑 | ~200 | 消息发送/停止、技能切换、输入模式、状态管理 |
| `useSSEStream.ts` | `agent-stream-tts.js` + film 内联 SSE | ~250 | SSE 流解析、token 渲染、TTS 分发、回调注入 |
| `useTTS.ts` | `tts-player.js` | ~300 | AudioContext 管理、队列播放、按钮绑定 |
| `useTTSCache.ts` | `tts-audio-cache.js` | ~120 | IndexedDB 音频缓存 CRUD |
| `useVoiceInput.ts` | `voice-input.js` | ~200 | WebSocket ASR + PCM 录音回退 |
| `useChatHistory.ts` | `agent-history.js` | ~180 | 会话列表 CRUD、切换会话 |
| `useTaskPolling.ts` | film-chat.js 任务轮询逻辑 | ~150 | 进度条动画 + 定时轮询 + sessionStorage 持久化 |
| `useAgentConfig.ts` | `window.AGENT_SHELL_CONFIG` | ~60 | 配置 provider/inject，类型安全 |
| `useMessageUtils.ts` | `agent-chat-utils.js` | ~50 | HTTP 错误解析、JSON 泄漏检测 |

---

## 路由变更

**新增路由**（`frontend/src/router/index.ts`）：

```typescript
// 智能体聊天页（使用 AgentLayout，需认证）
{ path: '/agent/health',  name: 'agent-health',  component: () => import('@/views/agent/HealthChatView.vue'),  meta: { auth: true } },
{ path: '/agent/film',    name: 'agent-film',    component: () => import('@/views/agent/FilmChatView.vue'),    meta: { auth: true } },
{ path: '/agent/learn',   name: 'agent-learn',   component: () => import('@/views/agent/LearnChatView.vue'),   meta: { auth: true } },
{ path: '/agent/finance', name: 'agent-finance', component: () => import('@/views/agent/FinanceChatView.vue'), meta: { auth: true } },
```

**修改现有路由**：
- `/agent/:name`（AgentRedirectView）→ 改为重定向到 `/agent/:name` 新路由
- `/chat/:agentId` → 重定向到 `/agent/:name`

**路由层级**：Agent 聊天页**不嵌套**在 `/app`（ConsoleLayout）下，因为它们使用完全不同的三栏布局。通过 `AgentLayout.vue` 直接在 view 组件中渲染。

---

## 配置迁移

JSON 配置文件（`frontend/shells/configs/*.json`）迁移为 TypeScript：

```
frontend/src/config/agents/
├── film.ts      ← 来自 shells/configs/film.json
├── health.ts    ← 来自 shells/configs/health.json
├── learn.ts     ← 来自 shells/configs/learn.json
└── finance.ts   ← 来自 shells/configs/finance.json
```

通过 `provideAgentConfig(agentId)` / `injectAgentConfig()` 在组件树中传递，替代 `window.AGENT_SHELL_CONFIG` 全局变量。

---

## 样式迁移策略

~1,100 行纯 CSS → Tailwind utility classes：

- **布局**（flex、宽高、间距）→ Tailwind utilities（`flex`, `h-screen`, `w-[200px]` 等）
- **颜色变量**（`--color-page` 等）→ Tailwind theme 扩展 + CSS 变量（agent 主题色按配置动态设置）
- **按钮/输入框** → Tailwind + 少量 component style
- **动画**（气泡进入、加载点、打字光标、脉冲录音等 9 个 keyframes）→ 保留在 `<style>` 块或 `style.css` 中，Tailwind 无法完全替代

---

## 迁移阶段

### 阶段 1：基础设施（3-4 天）

构建所有共享 composable 和组件，建立模式。

**新建 30 个文件**：
- 9 个 composable（`useChat`, `useSSEStream`, `useTTS`, `useTTSCache`, `useVoiceInput`, `useChatHistory`, `useTaskPolling`, `useAgentConfig`, `useMessageUtils`）
- 15 个组件（`AgentLayout`, `AgentSidebar`, `ChatHistorySidebar`, `ChatArea`, `MessageBubble`, `AgentInput`, `QuickChips`, `SkillModeHint`, `PreviewRow`, `VoiceInput`, `MessageActions`, `ToastContainer`, `WelcomeCard`, `SuggestionChips`, `MediaResults`, `TaskProgress`）
- 4 个 agent 配置文件
- 4 个 API service 文件（`chatApi.ts`, `ttsApi.ts`, `historyApi.ts`, `mediaApi.ts`）

**修改**：
- `style.css`（添加动画 keyframes）
- `agentConfigs.ts`（扩展配置检索）

### 阶段 2：简单 Agent — Health（2 天）

- 创建 `HealthChatView.vue`（~150 行）
- 接入所有 composable，实现完整的发送→流式渲染→TTS 链路
- 健康专有：食物图片上传、结构化数据渲染
- **验证**：`/agent/health` 与 `/health/ui`（旧版）功能一致

### 阶段 3：相同模式 Agent — Learn + Finance（2 天）

- 创建 `LearnChatView.vue` 和 `FinanceChatView.vue`
- 与 Health 几乎相同的组件组合，仅差异在配置和专有功能
- 财务专有：PDF 上传
- **验证**：两个 agent 与旧版功能一致

### 阶段 4：复杂 Agent — Film（3-4 天）

- 创建 `FilmChatView.vue`（~300 行）
- 接入 `useTaskPolling`（任务进度轮询）
- 图片压缩、参考图管理（最多 3 张）、视频首尾帧、虚拟角色化
- SSE 回调注入（`done` 事件触发任务轮询/媒体渲染）
- Session anchors（会话素材锚点）
- **验证**：`/agent/film` 与 `/film/ui`（旧版）功能一致

### 阶段 5：清理（1 天）

**删除 ~22 个旧文件**：
- `frontend/public/assets/js/` 下所有 JS（保留 `pcm-processor.js`）
- `frontend/public/assets/css/` 下所有 CSS
- `frontend/public/assets/voice-input.js`
- `frontend/shells/` 整个目录（模板 + JSON 配置）
- `scripts/build_agent_shells.py`
- `frontend/src/views/AgentRedirectView.vue`
- 各 `backend/*-agent/index.html`

**修改**：
- `package.json`：移除 `build:shells` 脚本
- `router/index.ts`：移除 `/agent/:name` 重定向路由

---

## 总计

| | 新建 | 修改 | 删除 |
|---|---|---|---|
| 文件数 | 36 | 7 | ~22 |
| 工期 | — | — | **11-13 天** |

---

## 关键风险与缓解

| 风险 | 缓解 |
|---|---|
| TTS AudioContext 在路由切换时泄漏 | composable 在 `onUnmounted` 中清理，TTS session 显式停止 |
| WebSocket 语音输入连接泄漏 | `useVoiceInput` 在 `onUnmounted` 中关闭 WS |
| film agent SSE 处理复杂度高 | `useSSEStream` 设计为回调注入模式，film 专有逻辑通过 `StreamContext` 回调注入 |
| 书签 URL 兼容 | 保留 `/agent/:name` 路由做重定向 |
| 新旧页面共存期用户混淆 | 旧页面添加"即将升级"横幅，全部迁移后删除 |

---

## 验证方法

1. **开发环境逐 agent 验证**：每个阶段完成后，在新旧页面间对比功能
2. **核心流程回归**：发送消息 → 流式渲染 → TTS 播报 → 历史会话切换 → 语音输入
3. **Film 专有流程**：文生图 → 图生视频 → 任务轮询 → 进度条 → 结果渲染
4. **边界场景**：网络错误、中止生成、快速连续发送、页面刷新恢复任务
5. **运行 `npm run build` 确认构建通过**
