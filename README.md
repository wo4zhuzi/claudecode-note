# Claude Code 使用技巧

> 收录 Claude Code 的配置实践与使用经验，持续更新。

## 文章目录

| 文章 | 简介 |
|------|------|
| [settings.json 配置详解](./claude-settings-explained.md) | 推荐配置每一项的设计原因与决策依据 |
| [Codex 与 Claude Code 协作实践](./codex-claude-collaboration.md) | 协作模式、配置文件融合策略、plan-with-file 跨工具执行 |
| [插件与技能的安装方式](./plugin-skill-use.md) | Plugin 与 Standalone Skill 的区别、第三方市场安装、按需提取单个技能的方法 |
| [MCP 应用安装指南](./mcp-installation-guide.md) | MCP 安装方式、实用 MCP 推荐、CodeGraph 完整安装步骤 |
| [自建代码库情报 skill](./codebase-intel-skill.md) | 整合 CodeGraph 与 Understand-Anything 的个人分析工作流、任务→工具映射、出图三层降级 |
| [AI 辅助模块重构工作流](./refactor-workflow.md) | 理解契约化、specs 验收标准、验证门禁与原子 commit 的六阶段重构闭环 |
| [/loop 与循环工程实践](./loop-engineering.md) | /loop 三种模式、Ralph Loop / Plan-Loop / interval 轮询三种落地范式与五条工程纪律 |

## 自建 Skill

- [my-skills/](./my-skills/) — 个人沉淀的 standalone skill 集合，随仓库版本化，换机器时直接复制到 `~/.claude/skills/`

## 配置文件

- [setting/claude-settings-recommended.json](./setting/claude-settings-recommended.json) — 可直接复制到 `~/.claude/settings.json` 的推荐配置
