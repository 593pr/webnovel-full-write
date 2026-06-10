---
name: webnovel-full-write
description: 写小说章节并全量验证，也可单独审查已有章节。整合写作任务书→起草→深度审查→事实提取→提交→备份。当用户要续写/写新章节时触发，替代 webnovel-writer:webnovel-write，不依赖 Agent 子代理，所有审查和数据提取均在主流程内联执行。触发词：续写、写第X章、继续写、下一章。审查模式触发词：审查第X章、检查第X章、review、验证章节、只审查。
argument-hint: "<章号> [--review-only] [续写要求]"
allowed-tools: Read Write Edit Grep Bash Glob AskUserQuestion
---

# 深度写作验证 Skill

## 目标

产出可发布章节到 `正文/第{NNNN}章.md`（四位零填充），并对所有设定文件全量交叉验证。替代 webnovel-writer:webnovel-write 的 Agent 子代理依赖，全部在主流程内联执行。

默认 2000-2500 字，用户/大纲另有要求时从之。

## 模式

| 模式 | 触发方式 | 流程 |
|------|----------|------|
| 完整写作 | `/webnovel-full-write 6 续写要求...` | Phase 0→1→2→3→4→5→6 |
| 仅审查 | `/webnovel-full-write 6 --review-only` | Phase 0→3→4→5→6 |
| 仅审查（无提交） | 口头说"审查第X章但不提交" | Phase 3 独立执行，输出 review_results.json |

`--review-only` 模式下：
- 跳过 Phase 1（写作任务书）和 Phase 2（起草），因为正文已存在
- 仍执行 Phase 0 预检（确认章节文件存在、合同就绪）
- Phase 3 全量审查照常执行，加载所有设定文件
- 自动修复仍然生效

## 硬规则

- 严格按当前模式对应的 Phase 顺序执行，不跳步、不并步
- 审查不是走过场：每条 issue 必须引用设定原文作为 evidence
- 自动修复不改剧情方向、不删叙事节点、不改角色性格
- 发现无法自动修复的 blocking issue 时，用 AskUserQuestion 让用户裁决
- Phase 3/4 输出的 JSON 必须严格匹配 schema（见附录），否则 commit 会失败
- 本章文件命名必须零填充：`第0005章.md`，不是 `第5章.md`

## 依赖

本 skill 依赖以下外部组件：

| 组件 | 说明 | 获取方式 |
|------|------|----------|
| **webnovel-writer** | Python CLI（`webnovel.py`），提供 preflight / story-system / write-gate / review-pipeline / chapter-commit / projections / backup 等子命令 | `https://github.com/lingfengQAQ/webnovel-writer`（GPL-3.0） |
| Python 3.x | 运行 webnovel.py 及其脚本 | 系统自带或 `pip install` |
| ModelScope API（可选） | 向量嵌入（vector projection）。未配置时 vector 维度会失败但不阻塞 commit | 需在插件配置中填入有效 token |

webnovel-writer 安装后，`SCRIPTS_DIR` 指向其 `scripts/` 目录，`webnovel.py` 为入口脚本。所有 Phase 中的 bash 命令均通过该 CLI 执行。

## 优先级

用户要求 > 总纲 > 分卷大纲 > 情节构思 > 人物卡 > 修炼体系 > 伏笔记录 > MASTER_SETTING

## 模式识别

在开始执行前，先判断用户意图：

1. 用户提到了"审查""检查""review""验证"且没有提"写""续写" → `--review-only` 模式
2. 用户说"审查但不提交" → Phase 3 独立模式，审查完输出报告即停止
3. 用户提到了"续写""写第X章""继续写""下一章" → 完整写作模式
4. 不确定时 → 用 AskUserQuestion 确认

---

## Phase 0：预检

```bash
export WORKSPACE_ROOT="${CLAUDE_PROJECT_DIR:-$PWD}"
export SCRIPTS_DIR="${CLAUDE_PLUGIN_ROOT:?}/scripts"
export SKILL_ROOT="${CLAUDE_PLUGIN_ROOT:?}/skills/webnovel-write"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" preflight
export PROJECT_ROOT="$(python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" where)"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" placeholder-scan --format text
```

刷新合同树（从 `.webnovel/state.json` 读 genre，从详细大纲解析本章目标）：

```bash
GENRE="$(python -X utf8 -c "import json,sys; s=json.load(open('${PROJECT_ROOT}/.webnovel/state.json',encoding='utf-8')); pi=s.get('project_info',{}); print(pi.get('genre') or s.get('project',{}).get('genre',''))")"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" \
  story-system "${CHAPTER_GOAL}" --genre "${GENRE}" --chapter {chapter_num} --persist --emit-runtime-contracts --format both

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage prewrite --format json
```

`write-gate` 必须返回 `"ok": true`。否则报告缺失项并阻断。

### --review-only 入口

当识别为 `--review-only` 模式时，Phase 0 完成后直接跳到这里：

1. 确认章节文件存在：`正文/第{NNNN}章.md`
2. 读取章节全文
3. 跳至 Phase 3（全量审查），审查完成后继续 Phase 4→5→6
4. 若用户说"审查但不提交"，Phase 4 生成 JSON 后即停止，不执行 Phase 5/6

---

## Phase 1：全量设定加载 + 写作任务书

> **仅完整写作模式执行此 Phase。`--review-only` 模式跳过。**

### 1.1 加载所有设定文件

按以下顺序自动发现并读取（用 Glob 发现，全部读取不省略）：

**第一组：核心合同（路径固定，框架约定）**
```
${PROJECT_ROOT}/.story-system/MASTER_SETTING.json
${PROJECT_ROOT}/.story-system/anti_patterns.json
${PROJECT_ROOT}/.story-system/chapters/chapter_00{N}.json        # 本章合同
${PROJECT_ROOT}/.story-system/reviews/chapter_00{N}.review.json  # 本章审查合同
${PROJECT_ROOT}/.story-system/volumes/volume_001.json            # 卷合同
```

若上一章合同存在则读取：`${PROJECT_ROOT}/.story-system/chapters/chapter_00{N-1}.json`

**第二组：大纲（glob `大纲/*.md`，全部读取）**

**第三组：人物（glob `人物/*.md`，全部读取）**

**第四组：世界观（glob `世界观/*.md`，全部读取）**

**第五组：上一章正文**
```
${PROJECT_ROOT}/正文/第{N-1:04d}章.md                           # 若存在则读取末尾 30 行
```

### 1.2 生成五段式写作任务书

按固定顺序输出，不得调换：

**一、本章硬性约束**
- 从 `chapter_00{N}.json` 的 `chapter_directive` 提取。若为空，用用户提供的续写要求替代。
- 包含：本章目标、时间锚点、章跨度、本章结尾状态
- 若 `chapter_directive` 中没有 `time_anchor`，从上一章末尾推导

**二、CBN/CPNs/CEN**
- CBN（核心叙事节点）：从 `must_cover_nodes`、情节构思、用户要求中提取
- CPNs：按优先级排序的叙事情节节点
- CEN：本章结尾可自然延伸到的悬念/钩子

**三、本章禁区**
- 从 `review.json` 的 `forbidden_zones` 提取
- 从 `anti_patterns.json` 提取通用禁忌
- 结合情节构思判断本章不应出现的内容（如蛰伏章不能有战斗爆点）

**四、风格指引**
- 从 MASTER_SETTING 提取调性（如"先压后爆"等该项目约定的核心调性）
- 主角 OOC 警戒线（参考人物卡——从 `人物/` 目录下主角文件中提取性格边界）
- 代码隐喻密度控制（若项目使用代码隐喻风格，从 MASTER_SETTING 读取目标比例）
- 禁止的 AI 文风模式（从 anti_patterns.json 提取）

**五、dynamic_context 补充参考**
- 从 `chapter_00{N}.json` 的 `dynamic_context` 提取场景写法/节奏建议
- 仅作风格参考，不能覆盖章纲约束

---

## Phase 2：起草正文

### 2.1 衔接处理

- 读取上一章最后 30 行，确认结尾情境
- 本章开头必须与上一章结尾无缝衔接（时间/地点/状态一致）

### 2.2 写作约束

- 只根据任务书起草，不自行发挥未授权的剧情线
- 围绕 CBN→CPNs→CEN 展开叙事
- 纯正文输出，无占位符（如 `[待补充]`、`{XX}` 等）
- 中文思维写作
- 若项目使用代码隐喻/跨界比喻风格，自然嵌入不强行解释（"懂的人自然懂"原则）
- 主角内心独白风格参考人物卡中的性格设定，不过量
- 遵守 MASTER_SETTING 中定义的所有写作规则

### 2.3 保存

写入 `正文/第{NNNN}章.md`（4 位零填充）。若文件已存在且有内容，追加到已有内容之后；若需替换，用 AskUserQuestion 确认。

---

## Phase 3：全量审查 + 自动修复

### 3.1 审查维度

逐维度检查，每个 issue 必须附带 **evidence**（引用设定文件原文或指出逻辑矛盾）。

| 维度 | 检查内容 | 数据源 |
|------|----------|--------|
| **continuity** | 情节与前章衔接有无断裂；已有伏笔是否遗漏 | 上一章正文 + 章节概要 + 伏笔记录 |
| **setting** | 地名、功法名、境界名、宗门结构是否与设定一致 | 世界观/*.md + MASTER_SETTING.json |
| **character** | 主角言行是否符合人物卡；配角是否 OOC；对话是否符合角色性格 | 人物/*.md |
| **timeline** | 时间流逝是否合理；事件因果链是否成立；不能出现"同一天做了两周的事" | 分卷大纲 + 上一章结尾时间 |
| **ai_flavor** | 是否有 AI 文风痕迹（排比句泛滥、每段结尾升华、过度解释情绪、成语堆砌） | anti_patterns.json |
| **logic** | 战力体系是否混乱；角色能力是否与境界匹配；物品/灵石数量是否前后矛盾 | 修炼体系.md + 总纲.md |
| **pacing** | 节奏是否符合当前阶段（压→日常密度高但信息量低；爆→冲突集中节奏快） | 分卷大纲 + MASTER_SETTING |
| **foreshadow** | 本章是否埋了新伏笔（若有则记录）；是否有旧伏笔到了回收节点 | 伏笔记录.md |

### 3.2 Severity 判定

- **critical**: 与总纲/核心设定直接矛盾，会导致后续章节连锁崩塌 → blocking=true
- **high**: 与人物卡/修炼体系显著矛盾 → blocking=true
- **medium**: 措辞不当、风格偏差、小逻辑瑕疵 → blocking=false
- **low**: 建议性优化，不改也不影响阅读 → blocking=false

### 3.3 自动修复规则

- blocking=true 且可修复 → 立即用 Edit 修复正文，标注修复位置
- blocking=true 且不可修复（如涉及剧情方向） → 用 AskUserQuestion 让用户裁决
- blocking=false → 记录在 issue 列表中，可在 Phase 4 润色时修复
- 修复后不重新跑全量审查（只确认修复正确）

### 3.4 输出 review_results.json

保存到 `${PROJECT_ROOT}/.webnovel/tmp/review_results.json`。

严格匹配 `review_schema.py` 的 `ReviewResult.to_dict()` 格式：

```json
{
  "chapter": 5,
  "issues": [
    {
      "severity": "medium",
      "category": "setting",
      "location": "line X",
      "description": "问题描述",
      "evidence": "设定文件原文引用",
      "fix_hint": "修复建议",
      "blocking": false
    }
  ],
  "issues_count": 1,
  "blocking_count": 0,
  "has_blocking": false,
  "summary": "审查总结"
}
```

运行 review-pipeline：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" review-pipeline \
  --chapter {chapter_num} \
  --review-results "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --metrics-out "${PROJECT_ROOT}/.webnovel/tmp/review_metrics.json" \
  --report-file "审查报告/第{chapter_num}章审查报告.md" \
  --save-metrics
```

---

## Phase 4：事实提取

从正文提取结构化事实，生成三个 JSON 文件到 `${PROJECT_ROOT}/.webnovel/tmp/`。

### 4.1 extraction_result.json

> **关键**：所有字段必须在顶层，不能包在 `extraction`/`chapter` 等外层对象里。

```json
{
  "accepted_events": [
    {
      "event_id": "EVT-00N-001",
      "chapter": 5,
      "event_type": "state_changed",
      "subject": "奕辰",
      "payload": { "action": "具体行为", "result": "结果" }
    }
  ],
  "state_deltas": [
    {
      "entity": "奕辰",
      "field": "location.current",
      "old_value": "旧值",
      "new_value": "新值"
    }
  ],
  "entity_deltas": [],
  "entities_appeared": [
    { "name": "奕辰", "type": "protagonist" }
  ],
  "scenes": [
    {
      "location": "地点",
      "time": "时间",
      "summary": "场景摘要"
    }
  ],
  "chapter_meta": {},
  "dominant_strand": "",
  "summary_text": "本章摘要字符串"
}
```

**event_type 必须使用别名**（脚本会自动映射到标准类型）：

| 你写的值 | 自动映射为 |
|----------|-----------|
| `state_changed` | `character_state_changed` |
| `world_rule` 或 `rule_revealed` | `world_rule_revealed` |
| `breakthrough` 或 `power_up` | `power_breakthrough` |
| `artifact` 或 `item_obtained` | `artifact_obtained` |
| `promise` | `promise_created` |
| `promise_resolved` | `promise_paid_off` |
| `mystery_introduction` 或 `open_loop` | `open_loop_created` |
| `loop_closed` | `open_loop_closed` |
| `relationship_change` | `relationship_changed` |
| `rule_broken` | `world_rule_broken` |

**accepted_events 每项必须包含**: `event_id`(str), `chapter`(int), `event_type`(str), `subject`(str), `payload`(dict)

**entities_appeared 每项必须是对象**: `{"name": "str", "type": "str"}`，不能是裸字符串。

**scenes 每项必须是对象**: `{"location": "str", "time": "str", "summary": "str"}`。

### 4.2 fulfillment_result.json

```json
{
  "planned_nodes": ["CPN-1", "CPN-2", "..."],
  "covered_nodes": ["CPN-1", "CPN-2", "..."],
  "missed_nodes": [],
  "extra_nodes": []
}
```

四个字段均为顶层必填。`planned_nodes` 来自 Phase 1 任务书中的 CBN/CPNs/CEN。

### 4.3 disambiguation_result.json

```json
{
  "pending": []
}
```

`pending` 为顶层必填列表。本章如无歧义项则为空数组。

---

## Phase 5：提交

### 5.1 Precommit Gate

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage precommit --format json
```

若返回 `"ok": false`，根据 `errors[].repair` 修复对应 JSON 文件后重跑。

### 5.2 Chapter Commit

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" chapter-commit \
  --chapter {chapter_num} \
  --review-result "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --fulfillment-result "${PROJECT_ROOT}/.webnovel/tmp/fulfillment_result.json" \
  --disambiguation-result "${PROJECT_ROOT}/.webnovel/tmp/disambiguation_result.json" \
  --extraction-result "${PROJECT_ROOT}/.webnovel/tmp/extraction_result.json"
```

自动判定：
- `blocking_count > 0` 或 `missed_nodes` 非空 或 `pending` 非空 → **rejected**
- 否则 → **accepted**

### 5.3 验证投影

五项（state/index/summary/memory/vector）中 state/index/summary 必须 done。

若 projection 失败，运行：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  projections retry --chapter {chapter_num} --format json
```

### 5.4 Postcommit Gate

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage postcommit --format json
```

---

## Phase 6：备份

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" backup \
  --chapter {chapter_num} \
  --chapter-title "{章节标题}"
```

备份以 `PROJECT_ROOT` 为准，禁止从工作区父目录裸 `git add`。

---

## 错误恢复

| 失败点 | 恢复方式 |
|--------|----------|
| Phase 0 预检失败 | 报告缺失项，不继续 |
| Phase 0 合同缺失（chapter_00N.json / review.json 不存在） | 手动创建合同文件（参考上一章模板），然后重跑 write-gate |
| Phase 0 story-system 被 guard hook 拦截 | 绕过：手动创建 contracts 后直接调用 write-gate |
| Phase 3 blocking issue 无法自动修复 | AskUserQuestion 让用户裁决 |
| Phase 5 precommit schema 报错 | 根据 repair 提示修正 JSON，重跑 precommit |
| Phase 5 commit 报错 | 修正 JSON 后重跑 chapter-commit |
| Phase 5 vector projection 失败（ModelScope API 401 等） | 外部基础设施问题，不阻塞。state/index/summary/memory 四维 done 即可 commit |
| Phase 5 postcommit schema_error（如 "missing extraction_result"） | 工具层面问题，commit 已 accepted 则忽略；重跑 postcommit gate 验证 |
| Phase 6 backup 失败 | 报告错误但不回退（commit 已完成） |
| 用户中途打断 | 从断点继续，不回退已完成阶段 |

---

## 附录：核心 Schema 速查

### review_results.json 必填字段

```
chapter (int)
issues (array of objects)
  ├── severity: "critical" | "high" | "medium" | "low"
  ├── category: "continuity" | "setting" | "character" | "timeline" | "ai_flavor" | "logic" | "pacing" | "other"
  ├── location: string
  ├── description: string
  ├── evidence: string
  ├── fix_hint: string
  └── blocking: bool
issues_count (int)
blocking_count (int)          ← 必须在顶层！不能只放在 summary 里
has_blocking (bool)
summary (string)
```

### extraction_result.json 必填字段

```
accepted_events (array, 顶层)
state_deltas (array, 顶层)
entity_deltas (array, 顶层)
entities_appeared (array of objects, 非字符串数组)
scenes (array of objects)
summary_text (string)
```

### fulfillment_result.json 必填字段

```
planned_nodes (array)
covered_nodes (array)
missed_nodes (array)
extra_nodes (array)
```

### disambiguation_result.json 必填字段

```
pending (array)
```
