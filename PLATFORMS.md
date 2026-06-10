# 跨平台兼容指南

`webnovel-full-write` 的核心（6 阶段 pipeline、8 维度审查、schema、目录容错）是**平台无关的 LLM 指令**。本指南说明如何在不同 AI 编程助手上使用。

## 通用前置条件

无论使用哪个平台，以下依赖必须满足：

| 依赖 | 说明 |
|------|------|
| [webnovel-writer](https://github.com/lingfengQAQ/webnovel-writer) | Python CLI，提供 preflight / story-system / write-gate / chapter-commit 等子命令 |
| Python 3.8+ | 运行 webnovel.py |
| 项目骨架 | `.story-system/`、`.webnovel/`、`正文/`、`大纲/`、`人物/`、`世界观/` 目录 |

安装 webnovel-writer：

```bash
git clone https://github.com/lingfengQAQ/webnovel-writer.git
# 记下克隆路径，后续配置需要
```

## 平台功能矩阵

| 功能 | Claude Code | Codex | OpenClaw | Hermes |
|------|------------|-------|----------|--------|
| 完整写作（内联） | ✅ | ✅ | ✅ | ✅ |
| --review-only | ✅ | ✅ | ✅ | ✅ |
| --fast 并行审查 | ✅ Agent | ⚠️ 看版本 | ⚠️ 看配置 | ⚠️ 看版本 |
| 目录容错 | ✅ | ✅ | ✅ | ✅ |
| 项目补全引导 | ✅ | ✅ | ✅ | ✅ |
| 自动修复 | ✅ | ✅ | ✅ | ✅ |

> **✅** = 原生支持 · **⚠️** = 取决于版本/后端，不支持时自动降级为内联模式

---

## 1. Claude Code（原生支持）

### 安装

```bash
cd /path/to/your/novel
git clone https://github.com/593pr/webnovel-full-write.git .claude/skills/webnovel-full-write
```

Claude Code 的插件系统会自动发现 `.claude/skills/*/SKILL.md`，无需额外配置。

### 环境变量

Claude Code 自动提供 `${CLAUDE_PROJECT_DIR}` 和 `${CLAUDE_PLUGIN_ROOT}`，skill 中的 Phase 0 脚本会自动使用这些变量。

### 使用

```bash
/webnovel-full-write 7 续写要求...
/webnovel-full-write 7 --fast           # 并行审查
/webnovel-full-write 7 --review-only    # 仅审查
```

---

## 2. Codex（OpenAI）

### 安装

```bash
cd /path/to/your/novel
git clone https://github.com/593pr/webnovel-full-write.git .codex/skills/webnovel-full-write
```

将 `SKILL.md` 注册为 Codex 的自定义指令。具体方式取决于 Codex 当前版本的指令管理机制：

**方式 A：作为项目指令**
将 `SKILL.md` 的内容复制到 Codex 的项目自定义指令中（通常在 `.codex/instructions.md` 或 Codex 设置面板中）。

**方式 B：作为斜杠命令**
若 Codex 支持自定义斜杠命令，将 SKILL.md 注册为命令，触发词设为 `webnovel-full-write`。

### 环境变量

Codex 不提供 `${CLAUDE_PLUGIN_ROOT}`。需手动配置 webnovel-writer 路径：

```bash
# 在 ~/.bashrc 或 ~/.zshrc 中设置
export WEBWRITER_DIR="/path/to/webnovel-writer"
```

或在项目根目录放置一个 `.env` 文件：

```
WEBWRITER_DIR=/path/to/webnovel-writer
```

Phase 0 的自动发现脚本会按以下顺序查找：
1. `${WEBWRITER_DIR}/scripts`
2. `${PWD}/webnovel-writer/scripts`
3. `${HOME}/.webnovel-writer/scripts`
4. 若均未找到 → 向用户提问

### 使用

在 Codex 对话中输入：

```
请按照 .codex/skills/webnovel-full-write/SKILL.md 中的流程，续写第7章：<续写要求>
```

或注册为自定义指令后直接：

```
/webnovel-full-write 7 续写要求...
```

### --fast 模式

取决于 Codex 版本的子代理支持。不支持时自动降级为内联模式（8 个维度依次审查）。

---

## 3. OpenClaw

### 安装

```bash
cd /path/to/your/novel
git clone https://github.com/593pr/webnovel-full-write.git .openclaw/skills/webnovel-full-write
```

OpenClaw 支持多种后端（Claude API、OpenAI API 等）。Skill 的加载方式取决于 OpenClaw 的配置：

**方式 A：作为 skill 文件**
将 `SKILL.md` 放入 OpenClaw 的 skills 目录（通常是 `.openclaw/skills/`）。

**方式 B：作为系统提示**
在 OpenClaw 的项目配置中将 `SKILL.md` 的内容作为系统提示注入。

### 环境变量

```bash
# 在 OpenClaw 项目设置或 shell 配置中
export WEBWRITER_DIR="/path/to/webnovel-writer"
```

### --fast 模式

取决于 OpenClaw 配置的 AI 后端是否支持并行子代理调用：
- **Claude API 后端**：支持 `tool_use` 并行调用，可实现 --fast
- **OpenAI API 后端**：支持 parallel function calling
- **不支持时**：自动降级为内联模式

---

## 4. Hermes

### 安装

```bash
cd /path/to/your/novel
git clone https://github.com/593pr/webnovel-full-write.git .hermes/skills/webnovel-full-write
```

Hermes 的技能/指令加载机制请参考 Hermes 文档。一般步骤：
1. 将 `SKILL.md` 放入 Hermes 的技能目录
2. 在 Hermes 配置中注册该技能
3. 必要时调整 frontmatter 格式以适配 Hermes 的元数据规范

### 环境变量

```bash
export WEBWRITER_DIR="/path/to/webnovel-writer"
```

### --fast 模式

取决于 Hermes 版本的并行任务能力。不支持时自动降级。

---

## 环境变量对照表

SKILL.md 中使用的环境变量及其在各平台的取值方式：

| 变量 | Claude Code | Codex | OpenClaw | Hermes | 说明 |
|------|------------|-------|----------|--------|------|
| `WORKSPACE_ROOT` | `${CLAUDE_PROJECT_DIR}` | `${CODEROOT}` 或 `$PWD` | `$PWD` | `$PWD` | 项目根目录 |
| `SCRIPTS_DIR` | `${CLAUDE_PLUGIN_ROOT}/scripts` | `${WEBWRITER_DIR}/scripts` | `${WEBWRITER_DIR}/scripts` | `${WEBWRITER_DIR}/scripts` | webnovel.py 所在目录 |
| `PROJECT_ROOT` | `webnovel.py where` 输出 | 同 | 同 | 同 | 由 CLI 自动确定 |
| `CHAPTER_GOAL` | `${USER_WRITING_REQUEST}` | 从用户输入提取 | 从用户输入提取 | 从用户输入提取 | 续写要求字符串 |

## 功能对应表

SKILL.md 中引用的能力在各平台的实现方式：

| 能力 | Claude Code | Codex | OpenClaw | Hermes |
|------|------------|-------|----------|--------|
| 读取文件 | `Read` 工具 | `read_file` | 取决于后端 | 取决于后端 |
| 写入文件 | `Write` 工具 | `write_file` | 取决于后端 | 取决于后端 |
| 编辑文件 | `Edit` 工具 | `replace_in_file` | 取决于后端 | 取决于后端 |
| 执行命令 | `Bash` 工具 | `execute_command` | 取决于后端 | 取决于后端 |
| 搜索内容 | `Grep` 工具 | `search_content` | 取决于后端 | 取决于后端 |
| 文件匹配 | `Glob` 工具 | `search_file` | 取决于后端 | 取决于后端 |
| 用户交互 | `AskUserQuestion` | 对话自然交互 | 对话自然交互 | 对话自然交互 |
| 子代理 | `Agent` 工具 | 取决于版本 | 取决于后端 | 取决于版本 |

> **注意**：上表中的工具名是各平台 SDK 层面的名称。在实际使用时，LLM 看到的是工具的函数描述（function description），会自动选择正确的工具。SKILL.md 中的 "向用户提问" 等描述性语言在任何平台上都能被 LLM 正确理解和执行。

## 常见问题

### Q: 我在 Codex/OpenClaw/Hermes 上运行，Phase 0 报 "未找到 webnovel-writer"

设置 `WEBWRITER_DIR` 环境变量指向 webnovel-writer 的克隆目录：

```bash
export WEBWRITER_DIR="/home/user/webnovel-writer"
```

或将 webnovel-writer 直接克隆到项目目录下：

```bash
cd /path/to/your/novel
git clone https://github.com/lingfengQAQ/webnovel-writer.git
```

### Q: --fast 模式不工作

确认当前平台是否支持并行子代理/子任务。若不支持，skill 会自动降级为内联模式（依次审查 8 个维度），功能不受影响，只是审查速度较慢。

### Q: 斜杠命令不识别

各平台的斜杠命令注册方式不同：
- **Claude Code**：自动识别 `.claude/skills/*/SKILL.md`
- **Codex**：需在 Codex 设置中注册
- **OpenClaw**：参考其 skill 管理文档
- **Hermes**：参考其插件/技能注册文档

如果平台不支持斜杠命令，可以直接在对话中说：

```
按照 SKILL.md 的流程，续写第7章：<续写要求>
```

LLM 会读取 SKILL.md 并按指令执行。

### Q: frontmatter 格式不兼容

SKILL.md 顶部的 YAML frontmatter（`---` 块）是 Claude Code 的元数据格式。如果其他平台报错，可以删除 frontmatter 块（只保留 `# 深度写作验证 Skill` 及其后的内容），核心指令不受影响。

---

*本指南基于各平台截至 2026 年中的公开信息编写。平台功能可能随版本更新变化，请以官方文档为准。*
