# webnovel-full-write

深度写作验证 Skill —— Claude Code 专用，为网文写作提供**起草→审查→事实提取→提交→备份**的全流程自动化。

## 功能

- **全量验证**：8 维度审查（连续性、设定、角色、时间线、AI文风、逻辑、节奏、伏笔）
- **自动修复**：非破坏性自动修复（不改剧情/不删节点/不改角色性格）
- **事实提取**：从正文抽取结构化事件、状态变更、伏笔，存入项目数据库
- **合同驱动**：通过 Story System 合同约束每章的叙事边界和审查标准
- **四种模式**：完整写作 / 快速写作（并行审查）/ 仅审查+提交 / 仅审查不提交
- **目录容错**：缺失设定文件不阻断——自动降级对应审查维度，并引导作者补全
- **项目补全引导**：检测到缺失项时，给出"现在就能做"的下一步操作，而非直接报错退出

## 前置依赖

| 依赖 | 说明 |
|------|------|
| [Claude Code](https://claude.ai/code) | 运行 skill 的宿主环境 |
| [webnovel-writer](https://github.com/lingfengQAQ/webnovel-writer) | Python CLI，提供 preflight / story-system / write-gate / review-pipeline / chapter-commit / projections / backup（GPL-3.0） |
| Python 3.8+ | 运行 webnovel.py |

webnovel-writer 通过 Claude Code 插件市场安装，或直接从 GitHub 克隆到插件目录。

### 可选依赖

- **ModelScope API token**：用于向量嵌入（vector projection）。在 webnovel-writer 插件的设置中填入 token（通常在 Claude Code 设置 → 插件 → webnovel-writer → ModelScope Token）。未配置时 vector 维度会失败，但不阻塞章节 commit。

## 项目结构

本 skill 假设你的小说项目遵循以下目录约定：

```
你的小说项目/
├── .story-system/           # 合同系统（框架生成）
│   ├── MASTER_SETTING.json  # 项目总设定
│   ├── anti_patterns.json   # 写作禁忌模式
│   ├── chapters/            # 章节合同（chapter_001.json ...）
│   ├── reviews/             # 审查合同（chapter_001.review.json ...）
│   ├── volumes/             # 卷合同
│   ├── commits/             # 提交记录（自动生成）
│   └── events/              # 事件日志（自动生成）
├── .webnovel/               # 项目运行时数据
│   ├── state.json           # 项目元信息（genre、当前进度等）
│   ├── tmp/                 # 临时文件（review/extraction/fulfillment JSON）
│   ├── summaries/           # 章节摘要
│   └── backups/             # 归档备份
├── 正文/                    # 章节正文（第0001章.md ... 四位零填充）
├── 大纲/                    # 大纲文件（总纲、分卷大纲、伏笔记录等）
├── 人物/                    # 人物卡（主角、配角、关系图等）
├── 世界观/                  # 世界观设定（修炼体系、势力宗门、世界地图等）
└── 审查报告/                # 审查报告（自动生成）
```

> **注意**：`大纲/`、`人物/`、`世界观/` 下的具体文件名可自由命名——skill 通过 glob 自动发现所有 `.md` 文件。

## 安装

1. 确保 Claude Code 已安装
2. 安装 webnovel-writer 插件（通过 Claude Code 插件市场或手动克隆）
3. 初始化项目：
   ```bash
   python webnovel.py --project-root /path/to/your/novel init --genre 玄幻
   ```
   > `--genre` 支持任意类型（玄幻、都市、科幻 等），仅作示例。
4. 将本 skill 安装到你的项目中：

   **方式 A：git clone**
   ```bash
   cd /path/to/your/novel
   git clone https://github.com/593pr/webnovel-full-write.git .claude/skills/webnovel-full-write
   ```

   **方式 B：手动复制**
   将 `SKILL.md` 放入 `.claude/skills/webnovel-full-write/` 目录

## 使用

### 完整写作模式

```bash
/webnovel-full-write 7 续写要求...
```

执行全部 6 个 Phase：预检→加载设定→起草→审查→事实提取→提交→备份。

续写要求可选。若不提供，从章节合同的 `chapter_directive` 提取。

### 快速写作模式

```bash
/webnovel-full-write 7 --fast 续写要求...
```

Phase 3 使用 8 个并行 Agent 同时审查 8 个维度，审查速度更快但 token 消耗约 3-4 倍。适合关键章节的深度审查。

### 仅审查模式

```bash
/webnovel-full-write 7 --review-only
```

跳过写作任务书和起草（Phase 1/2），对已有正文执行审查→事实提取→提交→备份。

### 仅审查不提交

直接说：**"审查第 7 章但不提交"**

只执行 Phase 3（审查），输出 `review_results.json` 后停止。

## Pipeline 全貌

```
Phase 0: 预检 ─── placeholder 扫描、项目骨架检查、缺失项引导
Phase 1: 加载 ─── 分级加载设定（Tier 1/2/3），缺项降级不阻断
Phase 2: 起草 ─── 续写正文，保存到 正文/第000N章.md
Phase 3: 审查 ─── 8 维度审查（默认内联 / --fast 并行 Agent）
Phase 4: 提取 ─── 3 个 JSON（事件/覆盖度/歧义）
Phase 5: 提交 ─── precommit gate → commit → projection → postcommit gate
Phase 6: 备份 ─── git commit + tag
```

## 配置

### MASTER_SETTING.json

项目核心配置，skill 读取以下字段：

- `core_tone`：叙事调性（如 "先压后爆"）
- `code_metaphor_ratio`：代码隐喻目标比例
- `viewpoint`：叙事视角
- 所有写作规则

### anti_patterns.json

定义 AI 文风检测模式、禁止句式、过度用词等。

### 章节合同（chapter_00N.json）

每章的叙事边界：
- `chapter_directive`：本章目标、时间锚点、结尾状态
- `must_cover_nodes`：必须覆盖的叙事节点
- `forbidden_zones`：本章禁用内容
- `dynamic_context`：可选的场景/节奏建议

## 目录容错

本 skill 不会因为缺了几个设定文件就拒绝工作。设定文件分为三级：

| 级别 | 含义 | 缺失时行为 |
|------|------|------------|
| 🔴 Tier 1 | 阻断级（MASTER_SETTING.json, state.json） | 报错，提示初始化项目 |
| 🟡 Tier 2 | 降级级（总纲、伏笔、人物卡、修炼体系 等） | 警告 + 对应审查维度自动降级 + 引导补全 |
| 🟢 Tier 3 | 建议级（世界地图、分卷大纲 等） | 静默跳过 |

项目设定不全时，skill 不会直接退出——而是输出一份清晰的"项目设定完整性报告"，列出缺失项，并给出"现在就能做"的补全引导。

## 已知问题

| 问题 | 影响 | 解决方案 |
|------|------|----------|
| ModelScope API token 未配置导致 vector projection 401 | vector 维度始终失败 | 不阻塞 commit；需在 webnovel-writer 插件配置中填入有效 token |
| story-system --persist 可能被 guard hook 拦截 | Phase 0 合同生成失败 | 手动参考上一章合同模板创建 chapter_00N.json 和 review.json |
| 首次写入时 chapter_00N.json 不存在 | prewrite gate 返回 `phase_not_ready_for_prewrite` | 按上一章合同结构手动创建后重跑 |

## 许可

本 skill 文件（SKILL.md / README.md）采用 MIT License。

依赖的 webnovel-writer 插件为 GPL-3.0（参见 https://github.com/lingfengQAQ/webnovel-writer）。

---

*为 Claude Code 构建 —— 让 AI 写出结构严谨的网文章节。*
