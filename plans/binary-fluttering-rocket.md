# 影视智能体视频生成工作流重构方案

## Context

当前影视智能体的 6 个技能（剧本、分镜、绘图、视频生成、拍摄规划、任务查询）各自独立运行，无法串联成完整视频制作管线。用户希望实现两种模式：**快速模式**（模糊输入→低成本自动生成）和**管线模式**（故事→分镜→角色→逐镜生成，每步确认）。

Seedance 2.0 已支持：文生视频、图生视频（首帧/首尾帧控制）、`@character:<id>` 角色一致性引用、`return_last_frame` 链式生成、多镜头单次生成。这些能力为管线模式提供了技术基础。

## 用户流程分析

用户提出的流程（故事梗概→丰富→确认→分镜→确认→角色照片→二次创作→确认→逐镜生成）**整体合理**，与业界"Director-First"范式一致。需要调整的点：

1. **新增"角色图生成"环节应在分镜确认之前或并行**：角色外观确定后才能写分镜的画面描述，否则分镜中的角色描述无法与角色图对应。
2. **新增"最终合成"环节**：所有分镜生成后需要拼接成完整视频，加入转场和音频。
3. **快速模式不应是完全独立的路径**：快速模式应复用管线模式的底层能力，只是跳过用户确认环节。

## 推荐架构

### 核心思路：引入 Project 状态机 + Pipeline 编排器

```
用户输入 → SkillRouter（意图识别）
              ├─ 简单意图 → 快速模式（现有 video_gen，增强后自动补全提示词）
              └─ 复杂意图 → Pipeline 模式（新建 Project，走状态机流程）
```

### 新增文件

| 文件 | 职责 |
|------|------|
| `backend/film-agent/ProjectManager.py` | 项目状态机、SQLite 持久化、阶段流转 |
| `backend/film-agent/skills/character_sheet.py` | 角色图生成技能（真人照片→img2img→角色设定图） |
| `backend/film-agent/skills/pipeline.py` | 管线编排技能（协调各阶段 LLM 调用和状态流转） |
| `backend/film-agent/pipeline_project.db` | 项目数据库（自动创建） |

### 修改文件

| 文件 | 变更内容 |
|------|----------|
| `backend/film-agent/main.py` | 新增 `/api/project/*` 端点族（~12 个端点） |
| `backend/film-agent/SkillRouter.py` | 新增管线意图识别，路由到 pipeline 技能 |
| `backend/film-agent/llm_client.py` | 新增角色图生成函数、增强视频生成支持角色引用参数 |
| `backend/film-agent/index.html` | 新增管线模式 UI（步骤向导、确认卡片、进度条） |
| `backend/film-agent/skills/__init__.py` | 注册新技能 |
| `backend/film-agent/skills/video_gen.py` | 增强：支持传入角色引用、首帧图、时长等参数 |
| `backend/film-agent/skills/storyboard.py` | 增强：输出增加角色列表和场景列表，用于一致性校验 |

### 数据库设计

```sql
-- 项目主表
CREATE TABLE pipeline_projects (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    title TEXT,
    synopsis TEXT,                    -- 用户原始输入
    expanded_story TEXT,              -- AI 丰富后的完整故事（JSON）
    status TEXT DEFAULT 'init',       -- 状态机当前阶段
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 角色表
CREATE TABLE project_characters (
    id TEXT PRIMARY KEY,
    project_id TEXT REFERENCES pipeline_projects(id) ON DELETE CASCADE,
    name TEXT,
    role TEXT,                        -- 主角/配角/群演
    description TEXT,                 -- 角色文字描述（锁定，所有分镜共用）
    original_photos TEXT,             -- JSON: 用户上传的原图 base64/URL
    stylized_sheet_url TEXT,          -- img2img 生成的角色设定图
    seedance_char_id TEXT,            -- Seedance @character:<id>
    confirmed BOOLEAN DEFAULT 0,
    created_at TIMESTAMP
);

-- 分镜表
CREATE TABLE project_shots (
    id TEXT PRIMARY KEY,
    project_id TEXT REFERENCES pipeline_projects(id) ON DELETE CASCADE,
    shot_number INTEGER,
    scene_id TEXT,                    -- 场景标识（同场景多镜共享）
    location TEXT,                    -- 场景地点（锁定，同场景复用）
    visual_prompt TEXT,               -- 视频生成优化提示词
    character_ids TEXT,               -- JSON: 本镜出现的角色 ID 列表
    duration_sec INTEGER DEFAULT 5,
    camera_movement TEXT,
    transition_from_prev TEXT,        -- 与上一镜的转场
    first_frame_task_id TEXT,         -- 首帧图的异步任务 ID
    first_frame_url TEXT,             -- 首帧图 URL（用于 I2V）
    last_frame_url TEXT,              -- 尾帧图 URL（用于链式生成）
    video_task_id TEXT,               -- 视频生成异步任务 ID
    video_url TEXT,                   -- 最终视频 URL
    status TEXT DEFAULT 'pending',    -- pending/draft/confirmed/generating/done/failed
    created_at TIMESTAMP
);
```

### 状态机设计（支持每个阶段的返工循环）

整体流程方向不可逆，但每个阶段内部支持"草稿↔确认"循环，且上游变更会级联失效下游：

```
INIT ──→ STORY ──→ CHARACTERS ──→ SHOTS ──→ GENERATING ──→ COMPLETED
  ↑        ↑           ↑            ↑
  └── 每个阶段内部:  DRAFT ⇄ CONFIRMED（可反复返工）
```

**每个阶段的内部状态**：

```
NOT_STARTED → DRAFT → CONFIRMED
                ↑         │
                └─────────┘  (驳回/修改意见 → 重新生成草稿)
```

**阶段流转与级联失效规则**：

| 触发事件 | 当前阶段变化 | 下游阶段影响 |
|----------|-------------|-------------|
| 用户确认故事 | STORY: DRAFT→CONFIRMED | CHARACTERS: NOT_STARTED→DRAFT（自动提取角色） |
| 用户驳回故事，提交修改意见 | STORY: CONFIRMED→DRAFT（重新生成） | CHARACTERS → NOT_STARTED, SHOTS → NOT_STARTED（级联失效） |
| 用户确认全部角色 | CHARACTERS: DRAFT→CONFIRMED | SHOTS: NOT_STARTED→DRAFT（自动生成分镜） |
| 用户驳回某个角色，要求重绘 | CHARACTERS: CONFIRMED→DRAFT（仅该角色重绘） | SHOTS → NOT_STARTED（级联失效） |
| 用户确认全部分镜 | SHOTS: DRAFT→CONFIRMED | GENERATING: 进入生成队列 |
| 用户驳回某个分镜，提交修改意见 | SHOTS: CONFIRMED→DRAFT（重新生成） | GENERATING 尚未进入，无影响 |
| 用户修改已确认的故事 | STORY: CONFIRMED→DRAFT | CHARACTERS + SHOTS → NOT_STARTED（级联失效，需重新确认） |
| 某个分镜视频生成失败 | SHOT: GENERATING→FAILED | 仅该镜重试，不影响其他镜 |

**流程说明**：
1. **INIT**: 用户输入故事梗概，创建项目
2. **STORY 阶段**: AI 丰富故事 → 用户审核 → 可多次返工 → 确认后锁定故事文本
3. **CHARACTERS 阶段**: 从故事提取角色 → 用户上传真实照片（可选）→ AI 生成角色设定图 → 用户逐个人物审核确认/驳回重绘 → 全部确认后锁定角色外观
4. **SHOTS 阶段**: 基于锁定的故事+角色生成分镜 → 用户逐镜审核 → 可单独驳回某镜 → 全部确认后锁定分镜方案
5. **GENERATING 阶段**: 批量生成视频 → 单镜失败可单独重试 → 全部完成后进入导出
6. **COMPLETED**: 可选拼接导出最终视频

**注意：角色创建在分镜之前**，因为分镜描述需要引用已确认的角色外观，确保一致性。

### API 设计

```
# 项目管理
POST   /api/project/create              # 创建项目 {synopsis} → {project_id, expanded_story}
GET    /api/project/{id}                # 获取项目详情（含角色、分镜及其确认状态）
GET    /api/projects                    # 列出用户所有项目
DELETE /api/project/{id}                # 删除项目

# 故事阶段
POST   /api/project/{id}/story/generate  # 生成/重新生成故事（initial 或 retry）
POST   /api/project/{id}/story/confirm   # 确认故事，锁定文本
POST   /api/project/{id}/story/reject    # 驳回故事 {feedback} → 故事回到 DRAFT，角色+分镜级联失效

# 角色阶段
POST   /api/project/{id}/characters/extract       # 从故事中提取角色列表
POST   /api/project/{id}/characters/{cid}/upload-photo     # 上传真人照片
POST   /api/project/{id}/characters/{cid}/generate-sheet   # 生成/重新生成角色设定图
POST   /api/project/{id}/characters/{cid}/confirm          # 确认单个角色
POST   /api/project/{id}/characters/{cid}/reject           # 驳回单个角色 {feedback} → 角色回到 DRAFT，分镜级联失效
POST   /api/project/{id}/characters/confirm-all            # 一键确认全部角色

# 分镜阶段
POST   /api/project/{id}/shots/generate     # 生成/重新生成分镜方案
POST   /api/project/{id}/shots/{sid}/confirm # 确认单个分镜
POST   /api/project/{id}/shots/{sid}/reject  # 驳回单个分镜 {feedback} → 该镜回到 DRAFT
POST   /api/project/{id}/shots/confirm-all   # 一键确认全部分镜

# 生成阶段
POST   /api/project/{id}/generate/start     # 启动批量视频生成（异步）
GET    /api/project/{id}/generate/status    # 查询生成进度（每镜状态+总体百分比）
POST   /api/project/{id}/generate/retry-shot/{sid}  # 重试失败的单镜
POST   /api/project/{id}/generate/export    # 拼接导出最终视频

# 快速模式（复用现有 /api/chat）
# 在 SkillRouter 中新增 quick_video 意图，直接调 video_gen 但不走管线
```

**驳回/返工流程示例**（故事阶段）：

```
1. POST /story/generate → 返回 expanded_story（状态: STORY/DRAFT）
2. 用户不满意 → POST /story/reject {feedback: "太短了，加一段冲突情节"}
   → 系统根据 feedback 重新生成（状态保持 STORY/DRAFT）
3. 用户满意 → POST /story/confirm → 状态变为 STORY/CONFIRMED
   → 系统自动进入 CHARACTERS/DRAFT
```

**驳回/返工流程示例**（角色阶段，体现级联失效）：

```
1. 故事已确认 → CHARACTERS/DRAFT，生成角色 A、B 的设定图
2. 用户确认角色 A → POST /characters/A/confirm
3. 用户驳回角色 B → POST /characters/B/reject {feedback: "发型不对，要短发"}
   → 角色 B 重新生成设定图，分镜回到 NOT_STARTED
4. 用户确认新角色 B → POST /characters/B/confirm
5. 全部角色确认 → SHOTS/DRAFT，系统自动生成分镜
```

### 跨镜一致性策略

结合 Seedance 2.0 能力和提示词工程，分层保证一致性：

| 层级 | 手段 | 说明 |
|------|------|------|
| **角色一致性** | `@character:<id>` | 先生成角色设定图注册到 Seedance，所有分镜提示词中引用 |
| **场景一致性** | 锁定场景描述块 | 同场景多镜复用相同的 location/lighting 文本块 |
| **外观锁定** | 固定角色描述 | 角色文字描述（发型、服装、体型）在所有分镜提示词中逐字重复 |
| **镜头衔接** | `return_last_frame` → 下一镜 `first_frame` | 利用 Seedance 首尾帧控制实现无缝衔接 |
| **批量关键帧** | 同场景多镜的首帧图批量生成 | 减少风格漂移 |

### 快速模式实现

当用户输入如"生成一个海边日落的视频"时：
1. SkillRouter 关键词匹配到 video_gen
2. video_gen 的 LLM 提示词优化器自动补全细节（当前已支持此能力）
3. 直接调用 try_video_generation()，使用低成本参数（duration=4s, resolution=480p, service_tier=flex）
4. 返回结果，不走管线确认流程

### 管线模式 UI（index.html 新增）

在现有聊天 UI 基础上新增"项目工作台"面板：
- **步骤指示器**：横向步骤条，高亮当前阶段
- **故事确认卡片**：展示 AI 丰富后的故事，提供"确认"/"修改"/"重新生成"按钮
- **角色管理卡片**：展示提取的角色，提供上传照片入口，展示角色设定图，确认/重绘按钮
- **分镜预览卡片**：网格展示每个分镜的画面描述、时长、运镜，可单独编辑
- **生成进度面板**：总体进度条 + 每个分镜的状态（排队/生成中/完成/失败）

## 实施阶段

### Phase 1: 基础设施（Project 系统）— 预计影响 3 个文件
- 创建 `ProjectManager.py`（DB schema + 状态机 + CRUD）
- 在 `main.py` 添加 project CRUD 端点
- 在 `index.html` 添加项目列表入口

### Phase 2: 管线核心流程 — 预计影响 5 个文件
- 创建 `skills/pipeline.py`（编排器）
- 增强 `skills/storyboard.py`（角色+场景列表输出）
- 增强 `skills/video_gen.py`（接受角色引用、首帧图参数）
- 在 `main.py` 添加管线阶段端点
- 更新 `SkillRouter.py`（管线意图路由）

### Phase 3: 角色管理 — 预计影响 2 个文件
- 创建 `skills/character_sheet.py`（真人照片→img2img→角色设定图）
- 在 `llm_client.py` 添加角色图生成函数

### Phase 4: 生成与一致性 — 预计影响 2 个文件
- 实现逐镜生成引擎（利用 return_last_frame 链式调用）
- 批量生成进度追踪

### Phase 5: UI — 预计影响 1 个文件
- `index.html` 管线模式向导式 UI

## 验证方式

1. **快速模式**：发送"生成一个海边日落视频"→ 确认返回 task_id 且视频正常生成
2. **故事管线**：发送"创作一个关于敦煌壁画的2分钟短剧"→ 确认故事生成→确认→分镜生成→确认
3. **角色一致性**：上传真人照片→确认角色设定图保留了关键特征→逐镜生成的角色外观一致
4. **端到端**：从故事梗概到最终视频的全流程走通
