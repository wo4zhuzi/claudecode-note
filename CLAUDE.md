# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.


## 仓库说明
- `README.md`：仓库入口、内容索引和使用建议。
- 根目录的 `.md` 文章：正式的经验/技巧文章，需同步登记到 `README.md` 的文章目录表格。
- `docs/changes/`：按日期记录每次 AI 会话产生的仓库改动，文件名格式为 `YYYY-MM-DD-<topic>.md`。

## 变更记录规范
- 每次产生仓库改动后，**必须**在 `docs/changes/` 下创建对应的变更记录，无需向用户确认。
- 文件名（含 `<topic>` 部分）由 Claude 自行定义，无需就命名向用户提问。

## 配置文件结构

`claude-settings-recommended.json` 分为四个主要区域：

**基础行为**（顶层字段）：`autoCompactContext`、`autoCreateCheckpoint`、`autoUpdate`、`defaultMode`、`language`、`model` 等控制 Claude Code 运行时全局行为的开关。

**customInstructions**：注入到每次对话的系统级提示词，定义回答语言、规划优先策略、代码修改原则、危险操作确认要求，以及文档语言规范（中文优先）。

**permissions**：`allow` 列表开放常用工具链命令（git、npm、pnpm、yarn、bun、node、python、cargo、go、make、测试运行器、只读 shell 工具）；`deny` 列表阻止高危操作（`rm -rf /`、`sudo rm`、管道执行远程脚本、`chmod -R 777`）。

**hooks**：
- `Stop` hook：任务完成后通过 `osascript` 推送 macOS 系统通知。
- `PreToolUse` hook（匹配 `git commit`）：提交前检查暂存区是否为空，暂存区为空时以退出码 2 中断并提示先执行 `git add`。

## 修改指南

- 修改 `customInstructions` 时保持简体中文，遵循现有的条目化格式。
- 新增 `permissions.allow` 条目请使用 glob 格式（如 `Bash(docker *)`），并与现有条目按工具类型归组。
- hooks 中的 Python 内联脚本通过 `cat | python3 -c` 接收 JSON 输入，`sys.exit(2)` 表示阻断，`sys.exit(0)` 表示放行。
- 每次改动完成后，主动提示本次建议的 commit 内容（commit message 及涉及的文件），方便用户确认后提交。
