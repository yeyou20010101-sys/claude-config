---
name: new-project
description: Used when user wants to create a new project. Follow workflow: ask for project name (Chinese), discuss and confirm development process and tech stack, then create README.md with the plan. Activates when user says 'create project', 'new project', 'start a project', '新建项目', '创建项目'.
version: 1.0.0
---

# New Project Skill

Guides the creation of a new project with structured discussion and documentation.

## Workflow

### Step 1: 确定项目名称

**规则：**
- 项目名称必须使用**英文**
- 如果用户提供中文，自动翻译为合适的英文项目名称
- Mac 本地路径格式：`~/01_Projects/YY.MM.DD_ProjectName/`
- Windows 本地路径格式：`D:\Projects\YY.MM.DD_ProjectName\`

**翻译原则：**
- 中文 → 英文 单词翻译（如：博客系统 → blog-system）
- 使用中划线分隔单词，全小写
- 保持简洁，2-4 个单词为佳

**示例：**
| 中文名称 | 英文名称 |
|----------|----------|
| 博客系统 | blog-system |
| 智能家居控制 | smart-home-control |
| 学习笔记管理 | notes-management |

**询问用户：**
```
请提供项目名称：___
示例：blog-system、smart-home-control
（或输入中文，我会自动翻译）
```

### Step 2: 探讨项目信息

与用户讨论并确认以下内容：

1. **项目背景**
   - 项目用途是什么？
   - 目标用户是谁？
   - 解决什么问题？

2. **技术路线**
   - 使用什么编程语言/框架？
   - 技术栈选择理由？
   - 是否有偏好技术？

3. **开发流程**
   - 开发阶段划分？
   - 里程碑设置？
   - 预期时间安排？

4. **项目结构**
   - 目录结构规划？
   - 主要模块划分？
   - 代码规范要求？

### Step 3: 确认并记录

将讨论结果整理成 README.md：

```markdown
# 项目名称

## 项目概述
[背景、目标、解决的问题]

## 技术栈
| 类别 | 选择 | 理由 |
|------|------|------|
| 语言 | xxx | xxx |
| 框架 | xxx | xxx |
| 数据库 | xxx | xxx |
| 其他 | xxx | xxx |

## 开发流程
### Phase 1: [名称]
- [ ] 任务1
- [ ] 任务2

### Phase 2: [名称]
- [ ] 任务1
- [ ] 任务2

## 项目结构
```
project-name/
├── src/
├── docs/
├── tests/
└── README.md
```

## 技术路线图
[可视化的时间线或流程图]
```

---

## 示例对话

**用户**: 创建一个新项目

**我**: 请提供项目名称（中文）：___

**用户**: 智能家居控制系统

**我**: 好的，让我们讨论一下项目细节：

1. **项目背景**：这个系统用于什么场景？家庭、办公室？
2. **技术路线**：你有偏好的技术栈吗？Python/Node.js/其他？
3. **功能范围**：主要控制哪些设备？灯光、空调、窗帘？

请告诉我这些信息，我会整理成完整的技术方案。

---

## 平台检测规则

```bash
# macOS 检测
if [[ "$(uname)" == "Darwin" ]]; then
    BASE_PATH="~/01_Projects"
fi

# Windows 检测
if [[ "$(uname)" == "MINGW"* ]]; then
    BASE_PATH="D:\Projects"
fi
```

---

## 创建命令

```bash
# 创建项目目录
mkdir -p ~/01_Projects/$(date +%y.%m.%d)_项目名称

# 初始化 README.md
touch ~/01_Projects/$(date +%y.%m.%d)_项目名称/README.md
```