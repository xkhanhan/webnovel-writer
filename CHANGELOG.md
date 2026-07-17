# 更新日志

这里记录每个正式版本对作者和维护者的影响。发布说明优先面向中文网文作者：先说写作体验有什么变化，再补维护者关心的技术细节。

## v6.3.1 - writer agent 增加字数约束和输入日志

发版范围：`v6.3.0..v6.3.1`。

### 给作者看的变化

- writer 写作 agent 新增字数约束：默认 2500 字 ±10%，任务书指定字数时自动覆盖。
- writer 新增输入日志：每次写章时会把收到的任务书原文、文风文件加载结果、字数约束和黄金三章判断写入 `.webnovel/tmp/writer_input_ch{章号}.log`，方便排查写作质量问题。

### 是否需要改旧项目

不需要。

### 给维护者

- `agents/writer.md`：新增第 4 节字数约束、第一步吸收末尾增加输入日志 bash 写入、修正章节编号。

## v6.3.0 - 新增 writer 写作 agent，Step 2 从裸指令升级为专业写作 agent

发版范围：`v6.2.1..v6.3.0`。

### 给作者看的变化

- 写章的正文起草阶段（Step 2）从一句话裸指令升级为专业的 writer 写作 agent。writer 会先加载你的文风偏好和参考作家风格，再拆解章纲的节奏和情绪弧线，最后按人类作家的思维方式起草正文。
- 支持用户自定义文笔风格：在 `设定集/我的文风.md` 中写你的写作偏好，在 `设定集/参考作家.md`（或类似命名）中放参考范本，writer 会自动加载。
- anti-AI 写作规则从事后润色检查前置到写作阶段，writer 在起草时就内化这些规则，减少"先写 AI 味初稿再去洗"的问题。

### 是否需要改旧项目

不需要。已有书项目可以继续使用。如需自定义文风，在 `设定集/` 下新建 `我的文风.md` 即可。

### 给维护者

- 新增 `agents/writer.md`：writer 写作 agent 定义，含三阶段执行流程（拆解→起草→自检）、文风加载、反模式内化。
- `skills/webnovel-write/SKILL.md` Step 2 改为调用 `writer` agent，带 SubagentRun 汇总和错误处理。

## v6.2.1 - 修复 Windows 下写章提交偶发的「拒绝访问」

发版范围：`v6.2.0..v6.2.1`。

### 给作者看的变化

- 修复 Windows 上写章提交时偶发的 `WinError 5（拒绝访问）`：`.webnovel/` 下的故事资料文件被 VSCode、杀毒软件或同步盘短暂占用时，系统会自动等待并重试，不再直接失败（#125）。
- 建议 VSCode 用户把 `**/.webnovel/**` 加入 `files.watcherExclude`，项目尽量不放同步盘目录，可进一步降低占用冲突。

### 是否需要改旧项目

不需要。已有书项目继续使用，无需任何迁移。

### 给维护者

- `atomic_write_json` 的 `os.replace` 遇 `PermissionError` 改为指数退避重试（约 2.6 秒窗口），穷尽后如实抛错；全部 JSON 投影共用该写入函数，一并受益。
- 新增 4 个针对性测试，含 Windows 真实句柄占用复现。

### 验证

- 全量 pytest 通过（774 passed）。
- 版本同步、发布说明与插件包校验通过。

## v6.2.0 - 写章结果更清楚，失败后更好恢复

发版范围：`v6.1.0..v6.2.0`。

### 给作者看的变化

- 写章、审查、规划和初始化结束后，最终报告更像写作助手的汇报：会说明已完成、部分完成、需要你处理或未完成。
- `/webnovel-write` 中断后，重复执行同一章会优先检查可信断点，尽量从失败位置继续，减少重写和误覆盖。
- 写章过程减少技术细节打扰；只有创作方向、事实取舍、文件覆盖风险或阻断问题需要裁决时才询问。
- 写作流程的上下文读取更克制，初始化、规划、写章、审查、查询等命令更聚焦，减少无关资料塞满上下文。
- 章节提交前后的中间结果校验更稳，能更早发现缺失的审查、事实提取或故事资料同步结果。
- 文档补充了最终报告读法、恢复边界、日志用途和常见运维入口。

### 是否需要改旧项目

不需要。已有书项目可以继续使用，不需要迁移 `.story-system/` 或 `.webnovel/` 数据。

### 给维护者

- 新增作者术语表、异常目录、审查作者视图、最终报告 helper、写章 run ledger、脱敏 run log。
- 新增 `user-report`、`run-ledger`、`run-log` 统一 CLI 子命令。
- 收紧 commit artifacts、projection writers、write-gate 和 postcommit 的结构化校验。
- 轻量化多个 Skill / Agent 的提示词，补充 reference loading map 和 region-read 规则。
- 增加 prompt integrity、unit tests、behavior eval，覆盖 artifact ownership、最小写章模式、projection retry、blocking review、断点续跑和日志脱敏。
- `Plugin Release` 工作流改为推送到 `master` 后自动发版，并保留手动兜底入口。

### 验证

- 相关 pytest 通过。
- behavior eval 通过。
- `compileall` 通过。
- `git diff --check` 通过。
- 版本同步和插件包校验通过。

## v6.1.0 - 项目体检更稳，出问题更容易定位

- 增加 doctor、project-status、write-gate、projection 重放、hooks、行为评估和插件包校验。
- 强化 Story System 运行时健康检查和 Marketplace 发布校验。

## v6.0.0 - Story System 主链上线，长篇事实更不容易写乱

- 上线合同种子、运行时合同、章节提交、事件审计和投影链路。
- 补齐主链相关集成测试。
