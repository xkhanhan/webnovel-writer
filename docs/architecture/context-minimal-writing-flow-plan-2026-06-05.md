# 写作流程上下文减负重构 Plan

> 日期：2026-06-05
> 状态：草案 v3
> 范围：重构 `skills/`、`agents/` 与 `references/` 的提示词与读取方式，减少主 agent 不必要上下文，降低 token 消耗
> 核心原则：先保住端到端流程，再压缩提示词；Skill 只写调度合同，Agent 自带专业流程，Runtime 负责硬校验
> 裁剪总纲：所有保留 / 下沉 / 删除决策由第 4 节四条裁剪判据（职责、token、噪音、读取方式）推导，不靠逐条拍清单
> v3 变更：① 新增第 4 节裁剪判据；② 第 5 节不可删清单收敛为跨层红线；③ 第 6 节纳入 references 与读取方式优化（以 reference-loading-map 为基线）；④ 第 12 节验收从文案级断言改为行为 / 契约级
> 工具能力基线：Claude Code 工具名、subagent / Skill 能力、shell 行为以官方最新文档和本机实际安装为准；不得靠记忆或经验猜测

---

## 0. 本版读法

这份 plan 不是为了把所有文件压成某个固定格式，也不是为了让现有测试继续变绿。

本轮真正要做的是：

1. 先确认 init / plan / write / review / query / learn / dashboard / doctor 的完整业务链路（第 3 节）。
2. 把跨层业务红线提成行为断言（第 5 节、第 12 节）。
3. 按第 4 节四条裁剪判据精简 Skill、Agent、references，并为每个读取动作定下读取方式（第 4 节、第 6 节）。
4. 用 runtime、prompt integrity、behavior eval 和 package validator 验收；验收对象是行为与契约，不是提示词文案（第 12 节）。

两条立场必须贯穿全程：

- **测试是探针，不是约束。** 瘦身会让一批文案级断言（`assert "字符串" in text`）变红，这些断言本身就是要清掉的噪音；它们保护的若是真红线，就改写成行为级断言并迁到生产方，而不是为了过测试保留废话。
- **不能删除第 3 节的端到端流程。** 任何格式化、瘦身、下沉 reference、压缩 schema、改读取方式的动作，都不得删掉第 3 节的业务步骤。第 3 节是红线来源，第 4 节是裁剪依据，二者不冲突。
- **工具能力不靠猜。** 任何涉及 `allowed-tools`、subagent 调用方式、Skill frontmatter、Skill / subagent 是否能加载彼此、Bash / PowerShell 行为、Read / Grep / Glob 参数能力的设计，都必须先核对 Claude Code 官方最新文档与本机当前版本；若二者有差异，以本机实际可用能力为落地准绳，并把差异记录到 Phase 0 清单。

---

## 1. 背景

当前 Webnovel Writer 已经不是单一提示词 Skill，而是一个有运行时主链的写作插件：

- `project-status` 判断项目短状态。
- `doctor` 做阶段感知体检。
- `placeholder-scan` 捕捉占位符与未补齐内容。
- `story-system` 生成 `.story-system/` 写前合同树。
- `write-gate` 在写前、提交前、提交后做批量校验。
- `context-agent` 负责写前上下文组装。
- `reviewer` 负责结构化审查。
- `review-pipeline` 生成报告、指标并落库。
- `data-agent` 负责提取 commit artifacts。
- `chapter-commit` 负责写后事实提交和 projection。
- `projections retry` 负责失败投影补跑。
- `backup` 负责按书项目根备份。

问题不在于缺少上下文，而是：

1. 主 agent 知道了太多 subagent 内部流程。
2. Skill 文本混有调度、教程、schema、示例和失败规则。
3. 有些信息应该由工具按需获取，却提前塞进主上下文。
4. 同一批上下文在 Skill、Agent、runtime 之间重复出现。
5. 长 schema 和长示例占用 token，但真正执行时只需要输入、输出、验收。

本轮重构要把写作流程从“主 agent 背完整教程”改成“主 agent 调度，subagent 专业执行，runtime 验收”。

---

## 2. 一句话目标

> 主 agent 不传教程，只传任务；subagent 自带教程；runtime 负责验收；流程完整性由断言表兜底。

---

## 3. 端到端流程基准

本节是重构的业务基准。后续任何压缩都必须先对照本节。

### 3.1 全局不可变式

所有 Skill / Agent 改写必须保留这些规则：

- 现有项目类 Skill 必须先解析真实书项目根，不能在插件目录写项目文件。
- `/webnovel-init` 在新项目尚未生成前不能用 `where` 把工作区解析成旧项目；必须用书名安全化得到目标目录。
- `.story-system/` 是写前合同与写后 commit 的主链事实源。
- `.webnovel/state.json` 是兼容投影 / read model，不重新变成写后事实真源。
- 调用 `story-system` 时，章级 query 必须来自详细大纲中的真实本章目标，禁止传 `{章纲目标}`、`第N章章纲目标` 等占位文本。
- 有具体章节写作 / 审查 / 合同刷新时，必须生成或确认 `.story-system/MASTER_SETTING.json`、`.story-system/volumes/`、`.story-system/chapters/`、`.story-system/reviews/`。
- 写章主链必须保留 `write-gate --stage prewrite`、`precommit`、`postcommit` 三道 gate。
- 必须用 `Agent` 工具显式调用 subagent，不得由主流程口头替代 `context-agent`、`reviewer`、`data-agent`、`deconstruction-agent` 的产物。
- 失败只补跑失败步骤，不全量回退。
- 能由 runtime 确定性校验的内容，提示词只保留最小说明和阻断边界。
- reference 只按需读取；不要为了“结构好看”新增没有真实复用价值的 reference。

### 3.2 `/webnovel-init` 完整流程

init 不是单纯采集器。精简时必须保留完整生成链：

1. 确认 `CLAUDE_PLUGIN_ROOT` 与 `${CLAUDE_PLUGIN_ROOT}/scripts/webnovel.py` 可用。
2. 初始化前不使用 `where` 解析旧项目。
3. 加载最小参考：数据流规范、题材套路库、题材画像；其他 reference 按需加载。
4. 进入故事核采集前，询问灵感来源；参考书拆解是可选项，不默认执行。
5. 用户提供参考文本路径或摘录时，必须调用 `webnovel-writer:deconstruction-agent`，不得由 init 主流程口头替代。
6. `deconstruction-agent` 只返回 `init_reference_research`，不写任何文件，不创建 `.story-system`、`.webnovel`、`设定集`、`大纲`、`正文` 或 canon/read model。
7. 拆书结果只消费可迁移模式和差异化要求；`quality.passed=false`、`confidence < 0.85` 或有 warnings 时，不能折叠进创意约束包，只能展示风险并让用户确认。
8. Step 2-6 只能使用用户确认过、并已变形为本书差异化表达的模式。
9. 采集故事核、角色、金手指、世界观、力量规则、创意约束包。
10. 输出初始化摘要草案并等待用户明确确认。
11. 用书名安全化生成 `PROJECT_SLUG` 和 `PROJECT_ROOT`，展示 `WORKSPACE_ROOT`、`PROJECT_SLUG`、`PROJECT_ROOT`，确认后再写文件。
12. 运行 `webnovel.py init`。
13. 写入 `.webnovel/idea_bank.json`，只写最终确认的创意约束。
14. patch `大纲/总纲.md`，补齐故事一句话、核心主线 / 暗线、创意约束、反派分层、爽点里程碑。
15. init 完成后立即生成 MASTER 合同：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  story-system "${GENRE}" --genre "${GENRE}" --persist --format json
```

此处不传 `--chapter`，只生成 `MASTER_SETTING.json` 和 `anti_patterns.json`。

16. 验证 `state.json`、核心设定集、`大纲/总纲.md`、`idea_bank.json`、`.story-system/MASTER_SETTING.json`。
17. 失败恢复只补缺失字段、重跑最小步骤，不全量重问。

### 3.3 `/webnovel-plan` 完整流程

plan 不是只生成章纲。精简时必须保留规划到写作合同的桥：

1. 解析真实项目根，运行 `placeholder-scan`。
2. 读取 `.webnovel/state.json` 的初始化配置快照获取 genre；后续写作真源仍是 `.story-system/`。
3. 读取 `大纲/总纲.md`，确认卷名、章节范围、核心冲突、卷末高潮，不足则阻断。
4. 跨卷规划时读取最近摘要、核心角色状态、核心关系、活跃伏笔。
5. 补齐设定基线：世界观、力量体系、主角卡、反派设计；发现冲突先阻断。
6. 选择目标卷并确认范围。
7. 生成卷节拍表，必须有中段反转或明确无反转理由，危机链至少递增 3 次。
8. 生成卷时间线表，必须明确时间体系、时间跨度、倒计时事件。
9. 生成卷纲骨架，包含卷摘要、人物与反派层级、Strand、爽点、伏笔、约束触发。
10. 批量生成章纲，默认 `10章/批`，复杂题材可降到 `8章/批`，不建议超过 `12章/批`。
11. 每章必须包含目标、阻力、代价、时间锚点、章内跨度、与上章时间差、倒计时、爽点、Strand、反派层级、视角 / 主角、关键实体、本章变化、章末未闭合问题、钩子。
12. 结构化节点必须保留：`CBN`、`CPNs`、`CEN`、`必须覆盖节点`、`本章禁区`。
13. 新设定只增量写回现有设定集。
14. 验证节拍表、时间线、详细大纲、时间字段、倒计时、BLOCKER、节点承接。
15. 生成显式结构化写回文件 `大纲/第{volume_id}卷-总纲写回.json`。
16. 调用 `master-outline-sync`，只允许更新 V+1 卷锚点与显式伏笔 / open loop，不从自由文本推断。
17. 调用 `update-state -- --volume-planned ... --chapters-range ...`。
18. 当本次规划已落到具体章节后，必须用真实章纲目标刷新 Story System runtime 合同：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  story-system "${CHAPTER_GOAL}" --genre "${GENRE}" --chapter {chapter_num} \
  --persist --emit-runtime-contracts --format both
```

进入写章前不得保留当前章相关实体的 `[待...]`、`暂名`、`{占位}`。

### 3.4 `/webnovel-write` 完整流程

write 是本轮最重要的验收对象。精简时必须保留：

#### 准备

1. 设置 `WORKSPACE_ROOT`、`SCRIPTS_DIR`、`SKILL_ROOT`。
2. 运行 `preflight`。
3. 用 `where` 解析真实 `PROJECT_ROOT`。
4. 运行 `placeholder-scan`。
5. 从详细大纲解析真实 `CHAPTER_GOAL`。
6. 从 `.webnovel/state.json` 的初始化配置快照读取 genre。
7. 刷新章级 Story System runtime 合同：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  story-system "${CHAPTER_GOAL}" --genre "${GENRE}" --chapter {chapter_num} \
  --persist --emit-runtime-contracts --format both
```

8. 运行 prewrite gate：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage prewrite --format json
```

prewrite 必备：`MASTER_SETTING.json`、`volume_{NNN}.json`、`chapter_{NNN}.json`、`chapter_{NNN}.review.json`。

#### Step 1：写作任务书

必须调用 `webnovel-writer:context-agent`。

输入只给必要参数：章节号、项目根、脚本目录、存储路径 / state 兼容读取路径、输出要求。

输出必须是一份可独立支撑起草的五段写作任务书。上下文不足时返回 blocker，不让主流程自行补脑。

#### Step 2：起草正文

只根据任务书起草。不要重新加载长篇 core constraints 或 anti-AI guide。

有结构化节点时围绕 `CBN -> CPNs -> CEN` 展开。正文必须无占位符。

#### Step 3：审查

默认与 `--fast` 必须调用 `webnovel-writer:reviewer`，`--minimal` 可跳过。

reviewer 输出 `.webnovel/tmp/review_results.json` 后，必须调用：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" review-pipeline \
  --chapter {chapter_num} \
  --review-results "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --metrics-out "${PROJECT_ROOT}/.webnovel/tmp/review_metrics.json" \
  --report-file "审查报告/第{chapter_num}章审查报告.md" \
  --save-metrics
```

写章主链中 reviewer 只调用一轮。`blocking=true` 的问题必须定点修复，或经用户裁决后才进入润色 / 提交。非 blocking issue 交给润色。

#### Step 4：润色

只改表达，不改事实。

可保留现有 reference 加载，但不要让主 Skill 携带长教程：

- `references/polish-guide.md`
- `references/writing/typesetting.md`
- `references/style-adapter.md`

顺序：修复非 blocking issue -> 风格适配 -> 排版 -> Anti-AI 终检。

`anti_ai_force_check=fail` 时不进入提交。`--minimal` 仅排版。

#### Step 5：提交

必须调用 `webnovel-writer:data-agent` 生成三份 artifacts：

- `.webnovel/tmp/fulfillment_result.json`
- `.webnovel/tmp/disambiguation_result.json`
- `.webnovel/tmp/extraction_result.json`

data-agent 不直接写 state / index / summaries / memory / vectors，也不直接写 projection。

随后运行 precommit gate：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage precommit --format json
```

再运行 `chapter-commit`：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" chapter-commit \
  --chapter {chapter_num} \
  --review-result "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --fulfillment-result "${PROJECT_ROOT}/.webnovel/tmp/fulfillment_result.json" \
  --disambiguation-result "${PROJECT_ROOT}/.webnovel/tmp/disambiguation_result.json" \
  --extraction-result "${PROJECT_ROOT}/.webnovel/tmp/extraction_result.json"
```

自动判定：`blocking_count > 0`、`missed_nodes` 非空或 `pending` 非空 -> rejected，否则 accepted。

#### Step 6：提交后验证与备份

必须运行 postcommit gate：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  write-gate --chapter {chapter_num} --stage postcommit --format json
```

projection_status 五项 `state/index/summary/memory/vector` 必须全部 `done` 或 `skipped`。

projection 失败只补跑：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" \
  projections retry --chapter {chapter_num} --format json
```

最后备份必须以解析后的 `PROJECT_ROOT` 为准：

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" backup \
  --chapter {chapter_num} \
  --chapter-title "{title}"
```

禁止从工作区父目录执行裸 `git add .`。

### 3.5 `/webnovel-review` 完整流程

独立 review Skill 可以比写章 Step 3 更完整，但不能伪造 reviewer 结果：

1. 解析真实项目根。
2. 如目标章缺少 runtime 合同，先用真实 `CHAPTER_GOAL` 刷新 `story-system --emit-runtime-contracts`。
3. 读取必要 reference：core constraints、review schema；其他按 issue 需要读取。
4. 加载 `.webnovel/state.json` 兼容投影和待审正文。
5. 调用 `webnovel-writer:reviewer` 输出 `.webnovel/tmp/review_results.json`。
6. 调用 `review-pipeline --save-metrics` 生成报告与 metrics 并写入 `review_metrics`。
7. 调用 `update-state -- --add-review ...` 写兼容审查记录。
8. 存在 `blocking=true` 时，询问用户立即修复或稍后处理。

### 3.6 查询、学习、面板、体检完整流程

`webnovel-query`：

- 只读。
- 先解析项目根。
- 先识别查询类型，再用最窄工具。
- 数据源优先级固定：`.story-system` 写前合同 -> latest accepted `CHAPTER_COMMIT` -> `memory-contract` -> `.webnovel/state.json` / `index.db` fallback。
- 时序查询优先用 `knowledge query-entity-state` / `knowledge query-relationships`。
- 降级到 legacy fallback 时必须明说。

`webnovel-learn`：

- 解析项目根。
- 读取 state 获取当前章节号，失败可用 `source_chapter: null`。
- 必须调用 `project-memory add-pattern`，不得手写 JSON。

`webnovel-dashboard`：

- 只读。
- 解析 dashboard 模块与项目根。
- 前端 dist 缺失时提示，不写项目文件。
- 面板必须暴露 Story Runtime 主链状态，例如 `/api/story-runtime/health`。

`webnovel-doctor`：

- 只读诊断，不修复、不安装依赖、不启动 dashboard。
- 先 `project-status --format summary`，再 `doctor --format text`。
- 缺失项按 runtime 推导的阶段解释，不把 init 项目误判成已写多章项目。

---

## 4. 裁剪判据与职责边界

本节是后续所有 Phase 的总纲。每一处“保留 / 下沉 / 删除 / 怎么读”的决策都由下面四条判据推导，不靠在各 Phase 里逐条枚举“必须保留”。

### 4.1 四条裁剪判据

**判据一·职责（谁生产、谁消费，信息就归谁）**

- 主 agent 是调度者，只保留**契约的形状**：调哪个 subagent、得到哪几份产物、产物流向哪个 runtime 命令、什么情况算 blocker。
- 字段级细则归生产方：artifact 字段名由 data-agent 生产、由 runtime validator 校验，归 `data-agent.md` 与 runtime，不进主 Skill。
- 一段信息只该有一个真源；出现在第二处即为重复，删到只剩生产方。

**判据二·token（测主 agent 的常驻输入，不测文件行数）**

- 优化对象是“主 agent 写一章实际加载的上下文 token”，不是文件有多长。
- 只在某 subagent / 某 step 执行时才需要的内容，绝不进主 agent 常驻上下文。
- 短文件全文读没问题，别为瘦而瘦；靶心是“`always` 全文读的大文件”（见第 6 节）。

**判据三·噪音（只留指令与红线，删元叙述与叮嘱）**

| 噪音类型 | 例子 | 处理 |
|---|---|---|
| 重复 | schema 字段在 Skill / Agent 各写一遍 | 删一处，留生产方 |
| 元叙述（教它怎么想） | reviewer 的“思维链（ReAct）：先读→对比→判断” | 删，不改变输出 |
| 过度否定堆叠 | 一连串“不要…禁止…” | 区分红线与叮嘱：红线留，叮嘱删 |
| 给错对象 | 主 agent 拿到只有 subagent 用的细则 | 按判据一迁走 |
| 凑结构 | 为段落模板硬撑的空段 | 删，并松绑对应的结构型测试 |

**判据四·读取方式（该全文读的全文读，该部分读的按需读）**

每个“读文件”动作必须标注读取方式，不默认全文 `cat`：

- **全文读**：短文件，且必须整体理解（schema、铁律、方法论总纲）。
- **区段读**：只需某节时，使用 Claude Code 当前可用的内置读取 / 搜索工具或宿主 shell 等价能力定位稳定标题锚点后读取片段；不要在 plan 中假定某个具体命令或参数一定是内置能力。
- **检索读**：结构化数据（CSV / JSON）优先用本项目 runtime 工具，如 `reference_search.py`、`knowledge query-*`、专用 schema/validator；只有确认宿主可用时才使用 shell 工具按字段取，绝不默认 `cat` 整表。
- **不读**：内容已迁走或不再被消费的文件，清理（见第 6 节）。

### 4.2 职责边界

| 层 | 保留什么 | 不做什么 |
|---|---|---|
| Skill | 项目根保护、调度顺序、runtime 命令、Agent 输入契约形状、成功标准、失败恢复 | 不讲 subagent 内部教程，不复制长 schema，不吞掉 runtime gate |
| Agent | 专业流程、最小必要规则、输出合同、边界 | 不依赖 `agents/references/*`，不写主链投影，不替 runtime commit |
| Runtime | schema、gate、commit、projection、backup、状态推进 | 不承载 LLM 写作判断 |
| Skill references | 长示例、详细采集字段、章节节点细则、润色清单 | 不作为 subagent 的隐藏说明书；不被无差别全文加载 |

### 4.3 Agent 单文件约束

Agent 当前只能加载自己的单个 Agent 文件，因此不新增 `agents/references/*`。Agent 需要的专业规则（含其产物的完整 schema）必须压缩后保留在单文件内——这正是判据一“字段细则归生产方”的落点。

---

## 5. 不可删红线与可下沉内容

本节不再逐条枚举字段级“必须保留”。字段、示例、采集细则的去留交给第 4 节判据；本节只钉两件事：可下沉对象的范围，和**绝不可删的跨层红线**。

### 5.1 可下沉 / 压缩 / 改按需读（示例，非穷举，按第 4 节判据裁）

- subagent 内部查询流程与推断说明。
- reviewer 维度解释、元叙述（ReAct）与长示例。
- data artifact 的完整 payload 细则（迁到 data-agent 单文件，不在主 Skill 重复）。
- init 详细采集字段、题材列表、反套路库。
- plan 的 CBN / CPN / CEN 详细规则与示例。
- polish 长 checklist。
- 第 6 节列出的 `always` 全文读大 reference（改区段读 / 检索读 / 条目化）。

### 5.2 不可删的跨层红线（穷举，由第 12 节行为测试守护）

这些是跨 Skill / Agent / Runtime 的业务红线，任何瘦身都不得删，且必须有对应的行为 / 契约级断言（第 12 节）：

- 项目根保护、init 目录安全化、用户确认前不写 canon。
- `placeholder-scan` 出现在 plan / write 关键节点。
- 真实 `CHAPTER_GOAL` 解析，禁止占位 query。
- `story-system --persist --emit-runtime-contracts` 的章级刷新。
- `write-gate` prewrite / precommit / postcommit 三道 gate，顺序不可乱。
- 必须用 `Agent` 工具显式调用 subagent，不得主流程口头替代。
- reviewer 原始 JSON 经 `review-pipeline --save-metrics` 落库。
- data-agent 产三份 artifacts；artifact 字段由 runtime validator 守，不靠主 Skill 文案。
- `chapter-commit` 是唯一事实提交入口，驱动 projection。
- postcommit projection 五项验证；失败只 `projections retry`，不回退写作步骤。
- `backup --project-root "${PROJECT_ROOT}"`，禁止裸 `git add .`。
- plan 的节拍表、时间线、章纲节点、设定写回、结构化总纲写回、状态更新。

> 字段级条目（如 `planned_nodes` 等具体字段名）**不在本清单**——它们由判据一归到生产方 agent 与 runtime schema，由第 12 节契约级测试守护，不再作为主 Skill 的文案红线。

---

## 6. 修改范围

### 6.1 重点文件

| 类型 | 文件 |
|---|---|
| 写章 Skill | `webnovel-writer/skills/webnovel-write/SKILL.md` |
| 写前 Agent | `webnovel-writer/agents/context-agent.md` |
| 数据 Agent | `webnovel-writer/agents/data-agent.md` |
| 审查 Agent | `webnovel-writer/agents/reviewer.md` |
| 拆书 Agent | `webnovel-writer/agents/deconstruction-agent.md` |
| 初始化 Skill | `webnovel-writer/skills/webnovel-init/SKILL.md` |
| 规划 Skill | `webnovel-writer/skills/webnovel-plan/SKILL.md` |
| 审查 Skill | `webnovel-writer/skills/webnovel-review/SKILL.md` |
| 查询 Skill | `webnovel-writer/skills/webnovel-query/SKILL.md` |
| 轻量 Skill | `webnovel-learn`、`webnovel-dashboard`、`webnovel-doctor` |

### 6.2 references 与读取方式优化

references 是本轮被低估的 token 面：顶层 `references/` 加各 Skill 的 `references/` 合计 60+ 个文件。优化不靠新增，而靠三件事——以现有加载映射为基线、给每个读取动作定读取方式、清掉已迁走的死文件。

#### 6.2.1 基线：reference-loading-map

`references/index/reference-loading-map.md` 已登记每个 Skill 每个 step 的实际 reference 消费，并已区分三类：

- **直接 Read 的 md**（整文件加载）——问题集中在这里。
- `reference_search.py` 检索 CSV（按 `--table --query --genre` 返回条目）——已是“按字段读”的范本。
- `story-system` 间接消费 CSV——已是按需。

CSV 那条线已经做对了，本轮**不重做检索层**；只治“直接 Read 的 md 全文加载”，并把读取方式登记进 loading-map，使其从“读哪些文件”升级为“怎么读这些文件”。

#### 6.2.2 token 靶心：`always` 全文读的大 md

以下是“直接 Read 且 always / 高频触发”的大文件，是 init / plan / write 每跑必吞的常驻成本，按判据四改读取方式：

| 文件 | 行数 | 谁全文读 | 目标读取方式 |
|---|---|---|---|
| `references/genre-profiles.md` | 552 | init + plan 双重 always | 区段读：按目标 genre 定位该题材段 |
| `creativity/selling-points.md` | 513 | init Step5 always | 区段读 / 条目化 |
| `reading-power-taxonomy.md` | 268 | plan Step7 | 区段读 |
| `outlining/chapter-planning.md` | 260 | plan Step7 | 区段读「结构化节点规范」一节 |
| `creativity/creativity-constraints.md` | 248 | init Step5 always | 区段读 `8.1 五维评分` 等命中节 |
| `webnovel-write/references/polish-guide.md` | 246 | write Step4 always | 区段读 / 条目化 checklist |
| `shared/cool-points-guide.md` | 242 | plan / review 触发 | 区段读 |

短文件（如 `strand-weave-pattern.md` 90 行）维持全文读，不动。

#### 6.2.3 清理死 reference（处置前先核验 CSV 覆盖）

`reference-gap-register.md` 记录的 `writing/*.md → CSV` 迁移已部分完成：loading-map 的「当前非直接调用项」确认 `combat-scenes`、`dialogue-writing`、`emotion-psychology`、`scene-description`、`desire-description`、`genre-hook-payoff-library` 等已不再被直接 Read（由 CSV 承担触发），但文件仍在，合计约 1400 行死内容。处置步骤：

1. 对每个候选 md，先核验 `场景写法.csv` / `写作技法.csv` 是否真覆盖其内容——不盲删。
2. 已覆盖：删除，或保留为指向 CSV 的空壳。
3. 未覆盖：先把缺口条目人工补进 CSV（遵循 `csv/README.md` 手动迁移规则），再处置 md。

#### 6.2.4 修正过时的“新增候选”

v2 第 6.2 列的候选已与现状脱节，按现状重判：

| v2 候选 | 现状 | 处置 |
|---|---|---|
| `blocking-override-guidelines.md` | 已存在并落位（gap-register 2026-04-16） | 删候选，改为“沿用现有” |
| `chapter-node-rules.md` | 与现有 `outlining/chapter-planning.md`「结构化节点规范」重复 | 不新建，对该节做区段读 |
| `init-flow.md` | 与现有 `init-collection-schema.md` 重复 | 不新建，沿用并改区段读 |
| `subagent-contracts.md` | 与判据一冲突（契约形状在主 Skill，schema 在生产方 agent） | 不新建 |
| `polish-checklist.md` | 可作为 `polish-guide.md` 条目化的产物 | 仅在 6.2.2 条目化时顺带产出 |

原则不变：不为“三段式结构”强行新增 reference；优先改读取方式与清死文件，而非加文件。

---

## 7. Phase 0：基线统计、读取审计与红线测试

### 7.1 目标

先确认哪些文本该保留、下沉、删除、改读取方式，并把跨层红线先补成行为测试，形成瘦身前的绿色基线。

### 7.2 要做

1. 统计 8 个 Skill、4 个 Agent 与 references 的体量；references 直接用 reference-loading-map + 行数表（见第 6 节），不另起炉灶。
2. 对每个文件按第 4 节判据标出归属：主 agent（契约形状）/ subagent / runtime / 下沉 references / 改读取方式 / 第 3 节红线。
3. **token 基线**：测“主 agent 写一章实际加载的上下文”，而非文件行数——记录 webnovel-write 主链一次 default 写章，主 Skill + 内联内容 + 全文读 reference 的近似 token，作为降幅度量基准。
4. **读取审计**：对照 loading-map「直接 Read 的 md」，逐条标 全文 / 区段 / 检索 / 不读（第 6.2.2 靶心优先）。
5. **先补红线测试再瘦身**：把第 5.2 跨层红线中当前只有文案级断言或无断言的，补成行为 / 契约级断言（第 12 节），使其在改动前先变绿。
6. **工具能力复核**：核对 Claude Code 官方最新 tools / subagents / skills 文档、本机 plugin-dev 指导和当前安装版本；确认 `Agent`、`Skill`、`AskUserQuestion`、`Read`、`Grep`、`Glob`、`Bash`、`PowerShell` 等是否存在、如何授权、在 subagent 中如何表现。复核结果落盘，后续提示词不得写入未核实的工具能力。
7. 跑基线验证：

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_prompt_integrity.py -q --no-cov
python -X utf8 webnovel-writer/scripts/run_behavior_evals.py --format json
python -X utf8 webnovel-writer/scripts/validate_plugin_package.py --format json
```

### 7.3 验收

- 得到每个 Skill / Agent / reference 的“保留 / 下沉 / 删除 / 读取方式”清单，**落盘为可追踪文件**，不只是口头结论。
- token 基线数值已记录，作为后续降幅度量基准。
- 第 5.2 全部跨层红线都有行为 / 契约级断言且通过。
- 当前验证命令通过。

---

## 8. Phase 1：精简 `webnovel-write`

### 8.1 目标

让写章 Skill 从“详细教程”变成“调度合同”，但保留完整写章主链。

### 8.2 必须保留

- 模式：默认 / `--fast` / `--minimal`。
- 准备：`preflight`、`where`、`placeholder-scan`、真实 `CHAPTER_GOAL`、`story-system` 合同刷新、`write-gate prewrite`。
- 三个 Agent 调用：`context-agent`、`reviewer`、`data-agent`。
- 起草只吃五段写作任务书。
- reviewer 原始 JSON + `review-pipeline --save-metrics`。
- blocking issue 只定点修复或用户裁决，不伪造通过。
- 润色顺序和 anti-AI 终检。
- data-agent 三份 artifacts。
- `write-gate precommit`。
- `chapter-commit`。
- `write-gate postcommit`。
- `projections retry`。
- `backup --project-root "${PROJECT_ROOT}"`。
- 成功标准与失败恢复。

### 8.3 可压缩

- context-agent 怎么查上下文。
- reviewer 怎么逐维度审查。
- data-agent 完整 payload schema。
- 长润色教程和大段示例。

### 8.4 Agent 调用目标形态

context-agent：

```text
chapter={chapter_num}
project_root=${PROJECT_ROOT}
scripts_dir=${SCRIPTS_DIR}
storage_path=${PROJECT_ROOT}/.webnovel
state_file=${PROJECT_ROOT}/.webnovel/state.json（projection/read-model，仅兼容读取）
要求：先 research，再按 本章硬性约束 -> CBN/CPNs/CEN -> 本章禁区 -> 风格指引 -> dynamic_context 补充参考 的顺序输出五段写作任务书；上下文不足时返回 blocker。
```

reviewer：

```text
chapter={chapter_num}
chapter_file=${CHAPTER_FILE}
project_root=${PROJECT_ROOT}
scripts_dir=${SCRIPTS_DIR}
output=${PROJECT_ROOT}/.webnovel/tmp/review_results.json
要求：只输出结构化审查结果，不评分，不口头总结。
```

data-agent：

```text
chapter={chapter_num}
chapter_file=${CHAPTER_FILE}
project_root=${PROJECT_ROOT}
scripts_dir=${SCRIPTS_DIR}
output_dir=${PROJECT_ROOT}/.webnovel/tmp
要求：生成 fulfillment_result.json、disambiguation_result.json、extraction_result.json；不直接写 projection。
```

### 8.5 风险控制

- `write-gate precommit` 与 `artifact_validator` 兜底 schema。
- behavior eval 检查三道 gate、三类 artifacts、chapter-commit、postcommit、backup。
- prompt integrity 检查禁止裸 `git add .`、禁止主流程口头替代 subagent。

---

## 9. Phase 2：精简 4 个 Agent

### 9.1 `context-agent`

目标：成为上下文压缩器，输出稳定 `chapter_task_brief`。

必须保留：

- `memory-contract load-context`。
- `query-entity`、`query-rules`、`get-timeline` 按需查询。
- load-context 已包含内容不重复查。
- `.story-system/` 合同优先，`state.json` 仅兼容读取。
- `chapter_directive.goal` / 章纲真实目标优先，`dynamic_context` 只作写法参考。
- 五段任务书：开篇委托、这章的故事、这章的人物、怎么写更顺、收在哪里。
- 红线校验和上下文不足 blocker。

可以删除或压缩：

- 长示例。
- 过细推断说明。
- 不必要术语解释。

### 9.2 `data-agent`

目标：只做事实提取和 artifacts 生成。

必须保留：

- 读取正文、实体索引和别名。
- 三份 artifact 文件名。
- `fulfillment_result.json` 顶层 `planned_nodes`、`covered_nodes`、`missed_nodes`、`extra_nodes`。
- `disambiguation_result.json` 顶层 `pending`。
- `extraction_result.json` 顶层 `accepted_events`、`state_deltas`、`entity_deltas`、`entities_appeared`、`scenes`、`summary_text`。
- `accepted_events` 子项最小字段：`event_id`、`chapter`、`event_type`、`subject`、`payload`。
- `state_deltas` 字段命名：`field`、`old`、`new`。
- `entity_deltas` 字段命名：`entity_type`。
- 禁止直接写 state / index / summaries / memory / vectors / projection。

可以删除或压缩：

- 各 event_type 完整 payload 长说明。
- 长 JSON 示例。
- 兼容旧字段名的详细解释。

### 9.3 `reviewer`

目标：只做可验证事实审查。

必须保留：

- 五个维度：setting、timeline、continuity、character、logic。
- 每个维度都给 `dimension_results`，无问题也写 `pass`。
- 每个 issue 有 evidence 和 fix_hint。
- 不评分、不评价文笔、不建议新增剧情、不暴露未发生大纲。
- 输出严格 JSON。

必须删除或改写：

- “思维链 / ReAct”类表述。
- 过长审查教程。

### 9.4 `deconstruction-agent`

目标：拆参考书的可迁移模式，不污染新书 canon。

必须保留：

- quick / deep / auto 路由。
- 只有书名/平台无文本时，不得凭记忆编造黄金三章、角色、设定、剧情。
- 不写任何文件。
- 不生成新书 canon。
- 输出 `init_reference_research` JSON。
- `quality`、`resume_state`、`do_not_copy`、`canon_contamination_warnings`。
- 快速模式、深度模式、情节点、质量门控、抽象转化规则。

可以压缩：

- 长质量门控表。
- 超长 schema 细节。
- 深度拆解分阶段长说明。

---

## 10. Phase 3：精简 init / plan / review Skills

### 10.1 `webnovel-init`

Skill 可以更短，但必须保留第 3.2 节完整链。

压缩方向：

- 详细采集字段保留在现有 `init-collection-schema.md`，对其做区段读；不新建 init-flow.md（见 6.2.4）。
- 题材列表只保留 canonical 集合和少量示例。
- CLI 参数长表可收缩为“参数来自采集对象”，但要保留执行 init 的事实。
- 创意约束细则、反套路库、世界观设计指南按需读取。

不可删：

- Step 1.5 灵感来源询问。
- deconstruction-agent 调用边界。
- 用户确认前不写 canon。
- project root 安全化和确认。
- `idea_bank.json`。
- patch 总纲。
- init 后 MASTER_SETTING 生成。
- 验证与最小回滚。

### 10.2 `webnovel-plan`

Skill 可以更短，但必须保留第 3.3 节完整链。

压缩方向：

- CBN / CPN / CEN 细则保留在现有 `outlining/chapter-planning.md`「结构化节点规范」，对该节做区段读；不新建 chapter-node-rules.md（见 6.2.4）。
- 长 reference 表可改为“按阶段触发读取”。
- 结构化节点示例下沉。

不可删：

- placeholder-scan。
- 跨卷状态读取。
- 设定基线补齐。
- 卷节拍表。
- 卷时间线。
- 卷纲。
- 批量章纲。
- 设定写回。
- 显式 `大纲/第{volume_id}卷-总纲写回.json`。
- `master-outline-sync`。
- `update-state`。
- 真实 `CHAPTER_GOAL` 刷新 Story System 合同。

### 10.3 `webnovel-review`

Skill 可以更短，但必须保留第 3.5 节完整链。

压缩方向：

- reviewer 审查方法留给 reviewer。
- 证据查询过程不在 Skill 展开。

不可删：

- 合同缺失时补 `story-system`。
- reviewer Agent 调用。
- `review-pipeline --save-metrics`。
- `update-state --add-review`。
- blocking 用户裁决。

---

## 11. Phase 4：精简轻量 Skills

### 11.1 `webnovel-query`

目标：查询先分类，再用最窄工具。

保留：

- 只读。
- 项目根保护。
- `.story-system` -> latest accepted commit -> memory-contract -> projection fallback 的优先级。
- 降级说明。

优化：

- 不默认全量 `memory-contract load-context`；按查询类型调用最窄工具。
- 角色状态用 `knowledge query-entity-state`。
- 关系用 `knowledge query-relationships`。
- 规则用 `memory-contract query-rules`。
- 伏笔用 open-loop 查询。

### 11.2 `webnovel-learn`

目标：保持极简。

保留：

- 项目根保护。
- 读当前章节号。
- `project-memory add-pattern`。
- 不手写 JSON。

### 11.3 `webnovel-dashboard`

目标：保持只读面板。

保留：

- 只读边界。
- `story-runtime/health`。
- 项目根解析。
- 前端 dist 校验。

可调整：

- 不默认安装依赖；缺依赖时提示命令。
- 启动前可运行轻量检查。

### 11.4 `webnovel-doctor`

目标：保持只读诊断。

保留：

- `project-status` 先行。
- `doctor` 阶段感知检查。
- 不修复、不安装、不启动 dashboard。

可调整：

- frontmatter description 改成简洁中文触发型描述。

---

## 12. Phase 5：测试与行为验证

### 12.1 修改文件

- `webnovel-writer/scripts/data_modules/tests/test_prompt_integrity.py`
- `webnovel-writer/evals/fixtures/behavior/fast.json`
- 新增行为 / 契约级断言；删除或迁移过时的文案级断言（见 12.2）。

### 12.2 验收原则：行为 / 契约级，不锚文案

总原则：断言“做没做对”，不断言“文案里有没有某串字”。`fast.json` 的 `commit_projection_runtime`（真跑 commit → projection 并断言状态）是样板；`assert "字符串" in text` 是要逐步退役的形式。

#### 必保行为（用行为 / 契约断言守，不锚具体措辞）

- 写章主链：三道 gate（prewrite / precommit / postcommit）按序落地；reviewer 一轮；blocking 只定点修复或用户裁决；data-agent 三份 artifacts；`chapter-commit` 提交并驱动 projection；postcommit 五项 projection 全 done/skipped；失败只 `projections retry`；`backup --project-root`。
- artifact 契约：三份 artifact 字段由 runtime schema（`chapter_commit_schema` / `story_event_schema` / `schemas`）校验；**新增 precommit 负向用例**——缺 `missed_nodes` / `pending` / 关键字段时 precommit 必须拦截。这取代“主 Skill 文案里必须出现字段名”的检查。
- 8 个 Skill 可发现、frontmatter 合法、description 可触发、中文优先。
- 4 个 Agent 单文件、frontmatter 含 `name`/`description`/`model`/`color`、`tools` 最小集、不依赖 `agents/references/*`。
- Agent 边界：data-agent 不直接写 projection；reviewer 只输出结构化 JSON 且覆盖 5 维；context-agent 输出五段任务书、用 `load-context`；deconstruction-agent 不写文件、产 `init_reference_research`、防 canon 污染、保留 Step 1.5 与确认门。
- plan 保留 `.story-system/` 主链与节拍表/时间线/章纲节点/总纲写回/状态更新；query / dashboard / doctor 只读不写项目文件。

#### 该删 / 该迁 / 该松绑的现有断言

| 现有断言 | 问题 | 处置 |
|---|---|---|
| `test_webnovel_write_data_agent_prompt_requires_extraction_schema` | 逐字要求主 Skill 写出 schema 字段名，与判据一冲突 | **删**；字段保障迁到 data-agent 单文件 + precommit 负向用例 |
| `test_data_agent_is_described_as_extraction_only...` 的字段名清单 | 检查 data-agent.md 含字段名（文案级） | 保留并加强为 data-agent 单文件 schema 的契约校验 |
| `test_agent_template_structure`（要求 1–8 连续编号段） | 强迫凑结构；删 reviewer 的 ReAct 节会误伤 | **松绑**：删 ReAct 后重排编号或下调段数要求，别为过测试留空段 |
| 各 `assert "字符串" in SKILL.md` 措辞锚定项 | 锚文案、阻碍瘦身 | 改为行为 / 契约断言，或迁到生产方 |

### 12.3 验证命令

文档与提示词层：

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_prompt_integrity.py -q --no-cov
python -X utf8 webnovel-writer/scripts/run_behavior_evals.py --format json
python -X utf8 webnovel-writer/scripts/validate_plugin_package.py --format json
```

稳妥回归：

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests webnovel-writer/scripts/tests -q --no-cov
```

### 12.4 验收标准

- 8 个 Skill 仍可发现；4 个 Agent frontmatter 合法。
- 插件结构符合官方 `plugin-structure`；Skill / Agent 修改符合 `plugin-dev`。
- behavior eval、package validator、prompt integrity 全过（在退役文案断言、补齐行为 / 契约断言之后）。
- 第 5.2 跨层红线全部有行为 / 契约级断言守护。
- **token：写章主链“主 agent 写一章加载的上下文”相对 Phase 0 基线下降，并给出数值**，不是“明显下降”的主观判断。
- 主 Skill 不再携带 subagent 长流程与 data schema；schema 真源唯一在生产方 agent + runtime。
- references：第 6.2.2 靶心大文件已改按需读；死 reference 已核验并处置。
- 第 3 节端到端流程未被瘦身删掉。

---

## 13. 官方 plugin-dev 约束

后续所有修改必须遵循本机官方插件指导：

```text
C:\Users\lcy\.claude\plugins\marketplaces\claude-plugins-official\plugins\plugin-dev
```

落地规则：

- 插件结构遵循 `plugin-structure`：`.claude-plugin/plugin.json` 位于插件根的 `.claude-plugin/`；`skills/`、`agents/`、`hooks/`、`scripts/` 保持插件根层级。
- Skill 遵循 `skill-development`：每个 Skill 一个目录，必须有 `SKILL.md`；frontmatter 至少包含 `name` 和具体触发型 `description`；详细资料可以放入该 Skill 自己的 `references/`，按需读取。
- Agent 遵循 `agent-development`：每个 Agent 是 `agents/*.md` 单文件；frontmatter 包含 `name`、`description`、`model`、`color`，`tools` 限定到最小必要集合。
- 本项目的 4 个 Agent 由 Skill 显式调用，不依赖自动触发；Agent `description` 只需说明调用方、职责和交付产物。
- Agent 不使用外部 reference 作为隐藏说明书。
- Hook 相关修改继续遵循 `hook-development`：插件级 `hooks/hooks.json` 使用 wrapper 格式，命令路径使用 `${CLAUDE_PLUGIN_ROOT}`。
- 每轮改完插件组件后，按 `plugin-validator` 思路校验 manifest、skills、agents、hooks、README、LICENSE、敏感信息和路径可移植性。

---

## 14. 风险与控制

| 风险 | 影响 | 控制 |
|---|---|---|
| 为了格式精简删掉真实流程 | 写作链断裂 | 第 3 节作为流程断言，prompt integrity / behavior eval 覆盖 |
| 主 Skill 过度精简 | Agent 输入不足 | Skill 保留 Agent 最小输入合同 |
| schema 压缩后 agent 漏字段 | commit 前失败 | `artifact_validator` 与 `write-gate precommit` 阻断 |
| Agent 单文件过短 | 专业执行质量下降 | 单文件保留最小必要流程、边界和输出合同 |
| reference 下沉过多 | 执行时忘记读取 | reference 只给长细则，主流程保留触发条件 |
| 区段读锚点漂移 | 读到错段或空内容 | reference 用稳定标题锚点；区段读失败时回退全文读并告警 |
| 文案级断言阻碍瘦身 | 误把测试当约束 | 退役文案断言、改行为 / 契约级（第 12 节）；删 / 迁前确认非红线 |
| query 默认少查导致答案不完整 | 查询质量下降 | 分类后按需补查，回答中说明降级 |
| state.json 被误当事实源 | 写后事实漂移 | 明确 `.story-system` 与 accepted commit 优先级 |
| init 拆书污染 canon | 新书设定侵权或撞梗 | deconstruction-agent 不写文件，init 用户确认后才写入差异化模式 |

---

## 15. 推荐施工顺序

1. Phase 0：基线统计、读取审计、token 基线，**先把第 5.2 红线补成行为测试形成绿色基线**。
2. `webnovel-write`：先保全写章主链，再瘦主 Skill；同步退役锚文案的 schema 断言。
3. `context-agent`：稳定五段任务书和上下文压缩。
4. `data-agent`：schema 作为单文件唯一真源，删长 payload 教程；加 precommit 负向用例。
5. `reviewer` + `webnovel-review`：删 ReAct 元叙述并重排段号，统一结构化审查链。
6. `webnovel-init` + `deconstruction-agent`：保留确认门和防 canon 污染。
7. `webnovel-plan`：保留规划到合同的桥。
8. references 与读取方式：按第 6.2 改靶心大文件读取方式、清死文件、登记进 loading-map。
9. `webnovel-query` / `webnovel-learn` / `webnovel-dashboard` / `webnovel-doctor`。
10. 测试与 eval 收口。

原因：

- 写章链路最吃 token，也最容易受上下文污染影响；`webnovel-write` 承载最多 subagent 内部细节，先改收益最大。
- 先补红线行为测试、固定绿色基线，后续每个 Skill 才能放心瘦身且不误删红线。
- reference 读取优化独立成步，因为它跨多个 Skill 且以 loading-map 为统一基线。

---

## 16. 最终效果

目标完成后，写章链路应该是：

```text
preflight / where / placeholder-scan
  ↓
解析真实 CHAPTER_GOAL
  ↓
story-system 刷新 runtime contracts
  ↓
write-gate prewrite
  ↓
context-agent 生成五段写作任务书
  ↓
主 agent 根据任务书起草
  ↓
reviewer 审查
  ↓
review-pipeline 落库
  ↓
主 agent 定点修复和润色
  ↓
data-agent 生成 artifacts
  ↓
write-gate precommit
  ↓
chapter-commit
  ↓
write-gate postcommit
  ↓
projection retry（仅失败时）
  ↓
backup
```

每一层只知道自己需要知道的东西：

- 主 agent 知道调度和不可跳过的 runtime 命令。
- context-agent 知道上下文压缩。
- reviewer 知道审查。
- data-agent 知道事实提取。
- runtime 知道校验、提交、投影和状态推进。

这才是本轮上下文减负的核心：不是把文档变短，而是让每一段上下文只在正确的时刻出现。
