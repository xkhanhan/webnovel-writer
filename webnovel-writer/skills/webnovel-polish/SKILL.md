---
name: webnovel-polish
description: 独立润色章节正文，去除 AI 味，按文风参考让句子更像人类作者写的 {题材} 小说。
allowed-tools: Read Write Edit Grep Bash Agent AskUserQuestion
argument-hint: "<章号或文件路径> [--dry-run]"
---

# 章节润色

## 目标

对已有的章节正文做纯文风润色：去除 AI 痕迹，按用户文风或参考作家风格调整句子，让正文更像人类作者写的 {题材} 小说。

不改故事。不改设定。不改情节走向。只改"怎么写"。

## 硬规则

- 必须通过 `Agent` 工具调用 `polisher`，禁止主流程自行润色
- 润色只改表达不改事实
- 不传入审查报告——这是纯文风润色，不是审查修复
- `--dry-run` 模式只输出润色建议，不写文件

## 执行流程

### Step 1：解析项目和章节

```bash
export WORKSPACE_ROOT="${CLAUDE_PROJECT_DIR:-$PWD}"
export SCRIPTS_DIR="${CLAUDE_PLUGIN_ROOT:?}/scripts"
export SKILL_ROOT="${CLAUDE_PLUGIN_ROOT:?}/skills/webnovel-write"

python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" preflight
export PROJECT_ROOT="$(python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${WORKSPACE_ROOT}" where)"
```

确定目标文件：
- 如果参数是章号（如 `1` 或 `0001`），查找 `正文/第{NNNN}章-*.md`
- 如果参数是文件路径，直接使用

文件不存在或为空 → 阻断，提示用户。

### Step 2：获取题材

```bash
GENRE="$(python -X utf8 -c "import json,sys; s=json.load(open('${PROJECT_ROOT}/.webnovel/state.json',encoding='utf-8')); pi=s.get('project_info',{}); print(pi.get('genre') or s.get('project',{}).get('genre','未知'))")"
```

### Step 3：调用 polisher

Use the Agent tool to run `webnovel-writer:polisher`.

Task:
- chapter_file={目标文件路径}
- genre={GENRE}
- project_root=${PROJECT_ROOT}
- scripts_dir=${SCRIPTS_DIR}
- 只输出润色后的完整正文 + 润色摘要，不改故事结构。

### Step 4：输出结果

**非 dry-run 模式**：
- 用 `Write` 把润色后的正文覆盖写入原文件
- 向用户展示润色摘要（删/换/调/增各多少处）
- 如果 polisher 判定"原文已达标"，不覆盖，告知用户

**dry-run 模式**：
- 不写文件
- 向用户展示润色摘要和主要改动的对比

### 最终报告

```text
总状态：已完成 / 原文已达标 / 未完成

一、润色结果
- 目标文件：{路径}
- 润色摘要：删除 N 处 / 替换 N 处 / 调整 N 处 / 增加 N 处

二、文风参考
- anti-ai-guide：已加载
- 我的文风.md：{已加载 / 未找到}
- 参考作家：{已加载文件名 / 未找到}

三、下一步建议
- 可以继续写下一章：/webnovel-write {next_chapter}
- 可以审查本章：/webnovel-review {chapter}
```
