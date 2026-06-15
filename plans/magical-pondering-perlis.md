# 前后端分离启动方案

## Context

当前项目只有 `python scripts/dev-start.py` 一键启动全部服务（后端+代理+前端），但开发时需要前后端分离启动以便独立调试。用户希望：

```bash
# 终端 1：启动所有后端
cd backend && python start.py

# 终端 2：启动前端 dev server
cd frontend && npm run dev
```

其中前端 Vite dev server 自带 API 代理到各后端端口（已在 `vite.config.ts` 中配置），因此不再需要 Nginx/Python 代理层。

## 实施方案

### 1. 新建 `backend/start.py` — 后端统一启动脚本

复用 `scripts/dev-start.py` 中已有的逻辑：
- **Python 检测**：优先 `XJY_PYTHON` 环境变量 → Anaconda `smdm` 环境 → 当前 `sys.executable`
- **环境变量加载**：从 `backend/.env` 加载端口等配置
- **启动 5 个 FastAPI 服务**：health(8000), film(8001), learn(8002), finance(8003), auth(8004)
- **Ctrl+C 优雅关闭**：终止所有子进程

与 `dev-start.py` 的区别：
- **不启动** Nginx / Python 代理（不需要反向代理）
- **不处理** SSL 证书、Junction、前端构建
- 仅聚焦后端服务启动
- 各服务端口可通过环境变量覆盖（`HEALTH_AGENT_PORT` 等）

### 2. 新建 `frontend/package.json` — 前端统一入口

在 `frontend/` 根目录创建最小 `package.json`，脚本委托到 `agent-workbench/`：

```json
{
  "name": "xiaojingyu-frontend",
  "private": true,
  "scripts": {
    "dev": "cd agent-workbench && npm run dev",
    "build": "cd agent-workbench && npm run build"
  }
}
```

不创建 workspace 模式（`pnpm-workspace.yaml`），因为 `agent-shells/` 不是 npm 包。保持简单。

## 涉及文件

| 操作 | 文件 | 说明 |
|------|------|------|
| **新建** | `backend/start.py` | 后端统一启动脚本，约 70 行 |
| **新建** | `frontend/package.json` | 前端统一 npm 入口，约 10 行 |

## 启动效果

```bash
# 终端 1
cd backend
python start.py
# 输出：
# ========================================
#   小鲸鱼后端 — 5 个服务
# ========================================
# [health-agent]  启动 :8000
# [film-agent]    启动 :8001
# [learn-agent]   启动 :8002
# [finance-agent] 启动 :8003
# [auth-service]  启动 :8004
# 全部就绪，Ctrl+C 停止

# 终端 2
cd frontend
npm run dev
# → Vite dev server :5173，API 自动代理到 8000-8004
```

浏览器访问 `http://localhost:5173` 即可开发调试。

## 验证方式

1. 终端 1 运行 `cd backend && python start.py`，确认 5 个服务打印就绪日志
2. 终端 2 运行 `cd frontend && npm run dev`，确认 Vite 启动
3. 浏览器访问 `http://localhost:5173`，测试各 Agent 对话功能正常
4. Ctrl+C 分别停止前后端，确认进程正确退出
