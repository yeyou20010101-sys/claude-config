# 流式 TTS 实时播报 — 实现计划

## Context

当前问题：chat 接口是非流式的（完整生成后一次返回 JSON），TTS 需要用户手动点击喇叭按钮并等待全部音频下载完才开始播放。用户体验是：文字全部显示完 → 点按钮 → 等下载 → 开始播，无法实现"边说边播"。

目标：LLM 流式生成文本的同时，遇到完整句子自动调 TTS，音频逐句推送到前端实时播放。

## 核心设计

```
后端 SSE 流:
  LLM stream=True → token → token → 遇句号 → TTS call → base64音频
  ─────────────────────────────────────────────────────────────→
  event: token    event: token    event: audio    event: done
  {text:"你好"}   {text:"。"}     {base64:"..."}  {metadata}

前端 ReadableStream:
  token → 实时显示文字
  audio → 解码base64 → Web Audio API 立即播放（队列顺序播放）
  done  → 保存历史、显示 TTS 重播按钮
```

**设计决策**：
- 断句在后端做（检测 `。！？\n`），因为后端有完整文本流
- TTS 用非流式 `call()` + `asyncio.to_thread()`，不用 `streaming_call()`，因为短句合成很快（<1s），避免 callback 跨线程复杂度
- 音频通过 SSE 传 base64（短句 MP3 ~10-50KB），避免额外 HTTP 请求
- 新增 `/api/chat/stream` 端点，保留原 `/api/chat` 不动（兼容性）

## 修改文件清单

### 1. `backend/film-agent/llm_client.py` — 新增流式 LLM 调用

新增 `try_assistant_reply_stream()` 生成器函数：
- 与 `try_assistant_reply()` 相同的 prompt 构造逻辑
- 调用 `client.chat.completions.create(stream=True, ...)`
- `yield` 每个 `chunk.choices[0].delta.content` 文本片段
- 结束后 `yield` 特殊 sentinel 携带 usage 信息

### 2. `backend/shared/tts.py` — 修复 + 新增异步包装

- `text_to_speech()` 增加 `voice`, `speech_rate`, `pitch_rate` 可选参数（修复 health/learn/finance 的 bug）
- 新增 `async def synthesize_async(text, **kwargs)` — 用 `asyncio.to_thread()` 包装同步调用

### 3. `backend/film-agent/main.py` — 新增 SSE 流式端点

新增 `POST /api/chat/stream`：
- 复用现有 skill dispatch 逻辑（`_center.dispatch()`）
- 调用 `try_assistant_reply_stream()` 获取文本流
- 断句缓冲区：累积字符，遇 `。！？\n` 或超 200 字切句
- 每切出一个完整句 → `asyncio.create_task()` 调 TTS → 结果放入 `asyncio.Queue`
- SSE generator 从两个来源取数据：LLM 文本 token + TTS 音频 queue
- 输出格式: `event: token\ndata: {text}\n\n` / `event: audio\ndata: {base64}\n\n` / `event: done\ndata: {metadata}\n\n`

### 4. `public/assets/js/tts-player.js` — 新增流式播放 API

新增方法：
- `TTSPlayer.createAutoPlayer()` — 返回 `{ enqueue(base64), stop(), onEnded(cb) }` 对象，管理音频队列自动顺序播放
- 复用现有 Web Audio API 播放逻辑（`_playBlob`）
- 支持并发：队列为空时收到 audio 立即播，播中收到则排队

### 5. `backend/film-agent/index.html` — 前端 SSE 消费

改造 `sendMessage()`：
- 将 `fetch().then(res.json())` 替换为 `fetch` + `ReadableStream` + SSE 解析
- 收到第一个 `token` 事件时创建气泡 DOM
- 每个 `token` 追加文字到气泡（不做 typewriter 动画，直接追加，因为已经是流式的）
- 收到 `audio` 事件 → `TTSPlayer.createAutoPlayer().enqueue(base64)`
- 收到 `done` 事件 → 保存 history、显示 TTS 重播按钮（复用现有 `TTSPlayer.createButton`）
- 新增"自动播报"toggle 按钮（默认开启），关闭时只显示文字不播语音

## 验证方法

1. 启动 film-agent: `cd backend/film-agent && uvicorn main:app --reload --port 8001`
2. 启动 Nginx 代理: `python scripts/dev-start.py`
3. 浏览器打开 film-agent 页面，输入文字发送
4. 验证: 文字边生成边显示，第一句话说完后 1-2 秒内开始语音播报
5. 验证: 切换"自动播报"toggle 可关闭自动语音
6. 验证: 消息气泡旁的 🔊 按钮仍可手动重播全文
7. 验证: 原有 `/api/chat` 非流式接口仍然正常工作（不破坏兼容性）
