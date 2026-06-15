# 影视智能体短剧工作流 — 实现方案

## Context

当前影视智能体拥有 6 个独立技能（剧本、分镜、图片生成、视频生成、拍摄规划、媒体查询），但各技能独立运作，无串联编排。需要构建一条自动化短剧生产流水线：**上传真人照片 + 需求描述 → 剧本 → 分镜 → 逐镜图片生成（保持人物一致性）→ 逐镜视频生成 → 拼接成片**。

核心挑战：
1. **人物一致性**：真实照片无法直接用于模型生成（合规限制），需通过图生图/风格迁移保留人物基本特征后作为参考
2. **跨分镜协调**：所有分镜中的人物外貌、场景风格必须保持一致
3. **异步编排**：图片和视频生成为异步任务，需编排等待、并行、进度追踪

## 总体架构

```
用户上传照片 + 需求
       ↓
[1] 人物特征提取 (Vision Model) → 人物档案 (文本描述 + 风格化参考图)
       ↓
[2] 剧本生成 (LLM) → 15秒短剧剧本
       ↓
[3] 分镜生成 (LLM) → 3-5个分镜，含镜头描述、运镜、时长
       ↓
[4] 逐镜图片生成 (img2img / 文本生图) → 保持人物一致性
       ↓
[5] 逐镜视频生成 (i2v / t2v) → 图片转短视频片段
       ↓
[6] 视频拼接 (ffmpeg) → 最终成片
```

每步完成后可中断等待用户确认/修改，再继续。

## 技术方案

### 1. 人物一致性策略（核心难点）

采用**三层递进**方案：

| 层级 | 方法 | 用途 |
|------|------|------|
| L1-人物档案 | Vision model 分析照片 → 结构化外貌描述（发型、脸型、五官、体型、肤色等） | 所有生图 prompt 中的 `[CHARACTER_DESC]` 占位符 |
| L2-风格化参考图 | 用 L1 的文本描述 + 原图做轻量风格迁移 → 生成一张"插画/电影风格"参考图 | 作为后续所有分镜生图的视觉 anchor |
| L3-一致性 Prompt | 生成一个 `CONSISTENCY_TAG`（如"30岁亚洲男性，短黑发，方脸，浓眉，体型中等"）注入每个分镜 prompt | 确保跨分镜人物一致 |

**关键安全处理**：DashScope 的 qwen-image-2.0 可能对真人照片有合规限制。Plan B：如果直接图生图被拒，则用 vision model 提取极详细的外观文字描述，纯文本生图（近似效果）。

### 2. 新增 API 能力（llm_client.py 扩展）

在 `backend/film-agent/llm_client.py` 新增三个方法：

```
try_image_understanding(image_base64, question)   → 调用 QWEN_MODEL (多模态) 分析图片
try_image_to_image(prompt, reference_base64, strength)  → 图生图（带参考图）
try_image_to_video(image_url, prompt, duration)    → 图生视频（以图片为首帧）
```

DashScope API 对应关系：
- 图生图：`POST /services/aigc/multimodal-generation/generation`，`input.messages` 中同时传 `{"image": "base64..."}` 和 `{"text": prompt}`
- 图生视频：`POST /services/aigc/video-generation/video-synthesis`，`input` 中加 `image` 字段（需验证模型名，可能为 `wan2.7-i2v` 系列）
- 图片理解：qwen3.5-omni-plus 本身支持 vision，用现有 `try_text_chat` 传入 image 参数即可

在 `backend/shared/config.py` 新增配置：
```
I2V_MODEL = os.getenv("I2V_MODEL", "wan2.7-i2v-2026-04-25")
```

### 3. 新增工作流技能 + 编排引擎

#### `backend/film-agent/skills/short_drama.py` — ShortDramaSkill

继承 `BaseSkill`，作为工作流入口。`run()` 方法：
1. 解析 payload：`{message, image_base64, character_photo_base64}`
2. 返回 `{workflow_id, current_step, steps_preview}`，启动后台工作流

#### `backend/film-agent/workflow_engine.py` — WorkflowEngine

核心编排器，管理异步流水线：

```
class WorkflowEngine:
    - _workflows: Dict[str, WorkflowState]  (内存中追踪)
    - submit(workflow) → workflow_id
    - get_status(workflow_id) → WorkflowState
    - _run_pipeline(workflow): 按顺序执行各 step
      - 每个 step 完成后更新 WorkflowState
      - 独立分镜的图片/视频生成并行提交
      - 全部完成后触发视频拼接
```

**WorkflowState 数据结构：**
```python
@dataclass
class WorkflowStep:
    name: str           # "character_profile" | "script" | "storyboard" | "shot_images" | "shot_videos" | "composite"
    status: str         # "pending" | "running" | "completed" | "failed"
    progress: float     # 0.0-1.0
    result: dict | None
    error: str | None

@dataclass
class WorkflowState:
    workflow_id: str
    steps: List[WorkflowStep]
    current_step_index: int
    overall_status: str  # "running" | "completed" | "failed" | "awaiting_review"
    created_at: float
    final_video_url: str | None
```

### 4. 新增 API 端点（main.py）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/workflow/short-drama` | 提交短剧工作流（含照片+需求） |
| GET | `/api/workflow/{id}` | 查询工作流状态/进度 |
| POST | `/api/workflow/{id}/modify` | 修改某分镜描述后重新生成 |
| POST | `/api/workflow/{id}/confirm` | 确认分镜，继续后续步骤 |
| DELETE | `/api/workflow/{id}` | 取消/删除工作流 |

### 5. 视频拼接方案

使用 `ffmpeg` 进行视频拼接：

```bash
ffmpeg -f concat -safe 0 -i concat_list.txt -c copy output.mp4
```

实现要点：
- 动态生成 `concat_list.txt`（文件列表）
- 若需过渡效果，用 `xfade` 滤镜添加 crossfade
- 若需背景音乐（后续扩展），用 `-i audio.mp3` 混合
- Windows 下需确保 ffmpeg 在 PATH 中，或打包便携版到项目

在 `backend/film-agent/video_composer.py` 中封装：
```
compose_videos(clip_urls: List[str], output_dir: str, transitions: bool) → output_path
```

### 6. 前端改动

主要改动在 `backend/film-agent/index.html`（film agent 的独立 UI，在 Vue workbench 中以 iframe 加载）：

**新增组件/区域：**
- 照片上传区：拖拽上传 + 预览裁剪（保持 1:1 或 9:16 比例）
- 工作流进度条：带步骤标签的水平进度条（人物提取 → 剧本 → 分镜 → 图片 → 视频 → 成片）
- 分镜审阅面板：卡片列表展示每个分镜（含生成的图片），支持编辑描述、重新生成单个分镜
- 最终视频播放器：播放拼接后的成片，支持下载

**修改文件清单：**
- `backend/film-agent/index.html` — 主 UI 扩展

Vue workbench (`frontend/agent-workbench/`) 侧**暂不改动**，因为 film agent 通过 iframe 独立运行。

### 7. SkillRouter 更新

在 `_ROUTE_ORDER` 最前面新增 `ShortDramaSkill`，关键词：`短剧/拍视频/生成短剧/创作视频/制作短片`

## 实施步骤

### Step 1: 扩展 llm_client.py（API 能力层）
- 新增 `try_image_understanding()` — vision 分析
- 新增 `try_image_to_image()` — 图生图（带参考图）
- 新增 `try_image_to_video()` — 图生视频
- 新增 `download_media_to_local()` — 下载 OSS 资源到本地临时文件

### Step 2: 新增配置 & 依赖
- `shared/config.py` 新增 `I2V_MODEL`
- `film-agent/requirements.txt` 确认包含必要的依赖
- 项目根目录确认/安装 ffmpeg

### Step 3: 实现人物一致性模块
- `backend/film-agent/character_profile.py`
  - `extract_character_description(photo_base64)` → 结构化人物描述
  - `generate_character_reference(photo_base64, description)` → 风格化参考图
  - `build_consistency_tag(description)` → 注入 prompt 的一致性标签

### Step 4: 实现分镜生成增强
- 修改 `skills/storyboard.py`，使其输出的分镜 prompt 自动注入人物一致性标签
- 新增 `skills/shot_generator.py` — 单个分镜的图片+视频生成逻辑

### Step 5: 实现视频拼接器
- `backend/film-agent/video_composer.py`
  - `compose_videos(clip_paths: List[str], output_path: str)` → 调用 ffmpeg 拼接

### Step 6: 实现工作流引擎
- `backend/film-agent/workflow_engine.py` — 完整编排引擎

### Step 7: 实现 ShortDramaSkill
- `backend/film-agent/skills/short_drama.py` — 工作流入口技能

### Step 8: 新增 API 端点
- `main.py` 新增 4 个工作流端点

### Step 9: 前端 UI 改造
- `index.html` 新增工作流面板、照片上传、分镜审阅、视频播放

### Step 10: 注册 & 集成测试
- `SkillRouter.py` 注册新技能
- 端到端测试完整流水线

## 验证方案

1. **单元测试**：测试 `character_profile.py` 的人物描述提取准确性（用不同照片验证）
2. **API 测试**：用 curl 测试各工作流端点的请求/响应
3. **端到端测试**：
   - 准备一张真人照片（开发者本人或志愿者）
   - 上传照片 + 需求"15秒短片，主角在咖啡馆遇见老朋友"
   - 验证完整流水线：剧本 → 分镜 → 逐镜图片 → 逐镜视频 → 成片
   - 检查人物在不同分镜中是否保持一致
4. **边界测试**：无照片输入、极短需求、网络异常重试

## 风险 & 缓解

| 风险 | 缓解措施 |
|------|---------|
| DashScope 不允许真人照片图生图 | Plan B：纯文本描述生图（vision 提取详细描述 → text-to-image） |
| 图生视频模型不存在/不可用 | 降级为文本生视频（t2v），prompt 中详细描述首帧画面 |
| Windows 下 ffmpeg 未安装 | 提供便携版 ffmpeg.exe 放在 `tools/` 目录；安装脚本自动检测 |
| 异步任务超时/失败 | 每个 step 有超时+重试（最多 3 次），失败后标记错误并允许手动重试 |
| 人物一致性不理想 | 调优 consistency_tag 的详细程度；考虑引入 seed 值锁定（若 API 支持） |
