# 双实例并行运行方案

## 背景

用户需要：
- **dev 分支**：日常开发，当前端口配置（8000-8004 + 9000）
- **main 分支**：稳定生产实例，不同端口，持续运行供他人使用

## 总体方案

使用 **Git Worktree** 创建 main 分支的独立工作副本，通过环境变量切换端口，实现两个实例同时运行互不干扰。

### 端口分配

| 服务 | dev (当前) | main (稳定) |
|------|-----------|-------------|
| health-agent | 8000 | 8010 |
| film-agent | 8001 | 8011 |
| learn-agent | 8002 | 8012 |
| finance-agent | 8003 | 8013 |
| auth-service | 8004 | 8014 |
| Nginx/Proxy | 9000 | 9010 |

### 关于合并冲突

所有改造都是**向后兼容**的：端口默认值保持不变，仅新增从环境变量读取的能力。改造后两个分支代码完全相同，仅靠各自 `.env` 文件中不同的环境变量值来区分端口。dev → main 合并不会产生冲突。

---

## 前置条件

- [ ] 用户先将 dev 分支的改动合并到 main，确保 main 包含最新稳定代码

## 实施步骤

### 步骤 1：改造 `scripts/dev-start.py` — 端口可配置化

**现状问题**：第 281-287 行硬编码了端口 8000-8004。

**改动**：服务端口列表改为从环境变量读取（与各 agent 的 `config.py` 一致）：
- `HEALTH_AGENT_PORT`（默认 8000）
- `FILM_AGENT_PORT`（默认 8001）
- `LEARN_AGENT_PORT`（默认 8002）
- `FINANCE_AGENT_PORT`（默认 8003）
- `AUTH_PORT`（默认 8004）

代理端口已支持 `XJY_PORT` 环境变量（默认 9000），无需改动。

`_generate_nginx_conf()` 新增端口占位符替换逻辑。
- **进程管理改进**：当前第 332 行退出时 `taskkill /F /IM python.exe` 无差别杀光所有 Python 进程。改为：
  - 启动每个 `subprocess.Popen` 时记录 `proc.pid`
  - `Ctrl+C` 退出时逐个 `taskkill /PID` 只杀自己启动的进程
  - Nginx 已用 `_stop_nginx(nginx_bin)` 停自己的实例，无需改动

### 步骤 2：改造 `deploy/nginx/nginx.conf.template` — 端口占位符

将硬编码的端口替换为占位符：
- `{{HEALTH_PORT}}`, `{{FILM_PORT}}`, `{{LEARN_PORT}}`, `{{FINANCE_PORT}}`, `{{AUTH_PORT}}` — upstream 定义
- `{{LISTEN_PORT}}` — server listen 指令

### 步骤 3：改造 `deploy/proxy_server.py` — 端口可配置化

`ROUTES` 字典（第 15-21 行）硬编码了端口，改为从环境变量读取，与 agent config 保持一致。

### 步骤 4：创建 main 分支 worktree 并配置

```bash
# 在项目父目录创建 main 的 worktree
git worktree add ../xiaojingyu-main main

# 在 main worktree 的 backend/.env 中设置偏移端口
HEALTH_AGENT_PORT=8010
FILM_AGENT_PORT=8011
LEARN_AGENT_PORT=8012
FINANCE_AGENT_PORT=8013
AUTH_PORT=8014
XJY_PORT=9010
```

---

## 涉及文件

| 文件 | 操作 | 说明 |
|------|------|------|
| `scripts/dev-start.py` | 修改 | 端口从环境变量读取；nginx 配置生成传递端口 |
| `deploy/nginx/nginx.conf.template` | 修改 | 端口替换为占位符 |
| `deploy/proxy_server.py` | 修改 | ROUTES 端口从环境变量读取 |
| `backend/.env`（main worktree） | 修改 | 设置 main 实例的偏移端口 |

## 验证方式

1. dev 分支默认启动 `python scripts/dev-start.py`，确认端口行为与改造前一致
2. dev 分支设置环境变量 `XJY_PORT=9010` 等后启动，确认使用新端口
3. main worktree 配置 `.env` 偏移端口后启动，与 dev 实例同时运行，互不干扰
4. 浏览器分别访问 `https://localhost:9000` 和 `https://localhost:9010`，确认均可正常使用
