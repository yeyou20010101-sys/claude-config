# Context

当前问题不是万象视频配置没保存，也不是 `service_type` 存错，而是**默认创建的 episode 没有绑定图片/视频/音频配置**，同时**工作台生成视频时没有显式传 `config_id`**。这会导致部分旧链路只能依赖 `/videos` 的 episode 回查或全局 active video 配置，表现为当前集视频模型为空、或在特定情况下报 `No active video AI config`。

本次按最小改动实施 B + C：
- **B**：新建 drama 自动创建默认 episode 时，绑定当前 active 的 image/video/audio 配置。
- **C**：工作台发起视频生成时，显式携带当前 episode 的 `lockedVideoConfigId` 作为 `config_id`。

目标是让“默认集配置锁定”和“视频生成请求配置”都与现有 episode 级配置设计保持一致，不引入新的抽象或迁移。

# Recommended Approach

## 1. 扩展 AI 配置返回结构，复用现有 active-config 逻辑
**文件：** `backend/src/services/ai.ts`

- 给 `AIConfig` 增加 `id` 字段。
- 在 `mapConfigRow()` 中把 `row.id` 映射到返回值。
- 保持 `getActiveConfig(serviceType)`、`getConfigById(id)` 现有调用方式不变，只是补充返回信息。

**复用现有逻辑：**
- `getActiveConfig(serviceType)` — `backend/src/services/ai.ts`
- `getConfigById(id)` — `backend/src/services/ai.ts`

**原因：**
这样 `dramas.ts` 可以直接拿到 active config 的 id，不需要重复写三段查询 SQL，是最小且一致的复用方式。

## 2. 在 drama 默认创建 episode 时写入三类配置 id
**文件：** `backend/src/routes/dramas.ts`

- 在 `POST /dramas` 创建默认 episode 之前，分别读取：
  - `getActiveConfig('image')`
  - `getActiveConfig('video')`
  - `getActiveConfig('audio')`
- 在默认 episode 的 `db.insert(schema.episodes).values(...)` 中补上：
  - `imageConfigId`
  - `videoConfigId`
  - `audioConfigId`
- 其余创建逻辑不改，仍保持默认 episode 自动生成行为。

**关键位置：**
- 当前默认 episode 创建逻辑：`backend/src/routes/dramas.ts`
- 参考已有显式保存配置逻辑：`backend/src/routes/episodes.ts`

**注意：**
- 这次不额外收紧 `/dramas` 的校验条件；如果某类 active config 缺失，沿用现有宽松行为，避免扩大改动面。
- 本次不处理历史数据回填，只修复后续新建 drama 的默认 episode。

## 3. 在工作台视频请求中显式传 `config_id`
**文件：** `frontend/app/pages/drama/[id]/episode/[episodeNumber].vue`

- 在 `buildVideoGenParams(sb)` 的初始 `params` 中加入：
  - `config_id: lockedVideoConfigId.value || undefined`
- 继续复用当前页面已有的：
  - `lockedVideoConfigId`
  - `lockedVideoConfigLabel`
- 不改现有 `reference_mode`、首尾帧、多参考图参数拼装逻辑。

**关键位置：**
- `buildVideoGenParams(sb)` — `frontend/app/pages/drama/[id]/episode/[episodeNumber].vue`
- `lockedVideoConfigId` 计算属性 — 同文件

**原因：**
这是最小的前端兜底：即使后端仍会根据 `storyboard_id` 回查 episode 并覆盖 `config_id`，前端请求也能明确表达“当前集锁定的视频配置”。

## 4. 保持 `/videos` 现有优先级不变
**文件：** `backend/src/routes/videos.ts`

- 不调整当前优先级：
  1. 请求体中的 `config_id`
  2. 若有 `storyboard_id`，则用所属 episode 的 `videoConfigId` 覆盖
  3. 最终在 `generateVideo()` 中回退到 `getActiveConfig('video')`

**原因：**
当前设计本来就是“episode 锁定配置优先”。B+C 生效后，前端传入值与 episode 回查值应一致，无需在这次修复中再改优先级，避免引入新的行为变化。

# Files To Modify

1. `backend/src/services/ai.ts`
2. `backend/src/routes/dramas.ts`
3. `frontend/app/pages/drama/[id]/episode/[episodeNumber].vue`

# Verification

## 后端验证
1. 新建一个已有 active image/video/audio 配置的 drama。
2. 调用 `GET /dramas/:id`，确认默认创建的 episode 带有：
   - `image_config_id`
   - `video_config_id`
   - `audio_config_id`
3. 确认这些 id 对应当前 active 配置。

## 前端/接口联调验证
1. 打开某个 episode 工作台。
2. 触发一次视频生成。
3. 在浏览器网络请求中确认 `/videos` 请求体包含 `config_id`，且值等于当前页面的 `lockedVideoConfigId`。
4. 确认同一请求仍保留现有 `storyboard_id`、`reference_mode` 及首尾帧/多参考图参数。

## 端到端行为验证
1. 用新建 drama 的默认 episode 完整走一次分镜视频生成。
2. 确认后端不再因为默认 episode 缺少 `video_config_id` 而走空配置链路。
3. 如有日志/记录可见，确认实际使用的 provider/model 与 episode 锁定的视频配置一致。

## 建议执行的检查
- 后端 TypeScript 检查：`cd backend && npm run typecheck`
- 前端构建检查：`cd frontend && npm run build`
