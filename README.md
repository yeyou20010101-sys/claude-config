# Claude Code 多设备配置同步

本仓库用于跨设备同步 Claude Code 配置（settings、hooks、CLAUDE.md、skills 等）。

## 新设备 Setup

```bash
# 1. 克隆到 ~/.claude（覆盖已有的默认 .claude 目录）
mv ~/.claude ~/.claude.bak
git clone git@github.com:yeyou20010101-sys/claude-config.git ~/.claude

# 2. 创建设备专属配置
cp settings.example.json settings.local.json
# 编辑 settings.local.json，填入本机的 API token、模型映射等
```

## 文件分工

| 文件 | 同步 | 说明 |
|---|---|---|
| `settings.json` | ✅ | 共享 hooks、权限、偏好 |
| `settings.local.json` | ❌ gitignored | 每台设备私有的 env（token、模型） |
| `CLAUDE.md` | ✅ | 全局编码规范 |
| `.claude.json` | ✅ | 跨设备状态指针 |
| `history.jsonl` | ✅ | 命令历史 |
| `plans/` | ✅ | Plan mode 方案文档 |

## 自动同步机制

SessionStart → `git pull --rebase`（开始前拉最新）
SessionEnd   → `git add -A && git commit && git push`（结束后推送）

网络故障不阻塞，10 秒超时即跳过。

## 注意事项

- **永远不要提交 `settings.json`**（含 API token）
- `.gitignore` 已排除敏感文件和运行时数据
- PostToolUse hook 会在项目目录执行 `git add -A`，如需限定范围自行调整
- `.claude.json` 两台设备同时修改可能冲突，内容简单手动解决即可
