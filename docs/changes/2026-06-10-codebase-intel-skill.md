# 2026-06-10 自建 my-codebase-intel skill 与专题文章

## 背景

已安装 CodeGraph MCP 与 Understand-Anything 插件，两者都能降低 AI 分析项目的 token 消耗。将其整合为一个个人代码库情报工作流 skill（`my-codebase-intel`），用短指令完成项目看板、项目介绍、模块介绍、架构/流程图、代码分析、bug 排查、重构评估、改动审查；并写一篇专题文章登记进笔记仓库。

用户提供了 skill 初稿，环境核实后基于初稿修订后落地。

## 环境核实结论（影响设计）

- 当前会话无任何 next-ai-draw-io / draw.io MCP 工具，初稿大段「用 next-ai-draw-io 出图」流程不可执行。
- `codegraph` MCP 只配在本项目作用域，本仓库无 `.codegraph/graph.db`、无 `.understand-anything/`，缺索引时 MCP 查询会失败。
- CodeGraph 自身可导出 mermaid/dot：`mcpCodegraphExportGraph`、`mcpCodegraphSequence`、`mcpCodegraphCfg`，出图不必依赖 draw.io MCP。

## 改动内容

### 新增 skill：`~/.claude/skills/my-codebase-intel/SKILL.md`

standalone skill，全局可用，与 `planning-with-files-zh` 同址同机制。相对初稿的关键修订：

- **绘图改为自然语言指令为主路径**：出图三层降级 —— CodeGraph 原生 mermaid/dot（主路径）→ 可复制给任意 draw.io AI 的自然语言指令 → 检测到 draw.io MCP 工具时才作可选增强。不假定 draw.io MCP 存在。
- **适用范围改为 Claude Code 专用**：description 去掉「帮助 Codex 用短指令」，改通用措辞。
- **补 CodeGraph 索引前置检查**：开局做双前置检查（`.understand-anything/knowledge-graph.json` 与 `.codegraph/graph.db`），任一缺失时对应能力降级为源码搜索 + 文件阅读，并告知用户。
- **CodeGraph 部分映射具体工具名**：新增「任务类型 → 推荐 `mcpCodegraph*` 工具」映射表；标注 `mcpCodegraphSemanticSearch`（需 `codegraph embed`）、`mcpCodegraphCoChanges`（需 `codegraph co-change --analyze`）的隐藏前置。
- **边界补两条**：不假定 draw.io MCP 存在、调用前先探测工具；不使用未跑前置的 CodeGraph 增强查询。

### 新增文章：`codebase-intel-skill.md`

讲整合动机、事实优先级金字塔、任务→工具映射表、出图三层降级、双前置检查与降级策略、安装方式（standalone skill）。

### 新增 `my-skills/` 目录（仓库内沉淀 skill 副本）

为便于换电脑/换工具时直接复用，在仓库内新建 `my-skills/` 存放自己沉淀的 skill 源文件，保持 `<skill名>/SKILL.md` 目录结构，与 `~/.claude/skills/` 一致：

- `my-skills/my-codebase-intel/SKILL.md` —— 从 `~/.claude/skills/` 复制的副本（与全局版一致）。
- `my-skills/README.md` —— 说明用途与换机器安装方式（`cp -r my-skills/<skill> ~/.claude/skills/` 后 `/reload-plugins`）。

注：`planning-with-files-zh` 是第三方提取的技能（来源见 `plugin-skill-use.md`），不属于自建，不放入 `my-skills/`。

### 登记

- `README.md` 文章目录新增 `codebase-intel-skill.md` 链接；「配置文件」区新增 `my-skills/` 目录说明。
- 本变更记录。

## 验证

- `/reload-plugins` 后 `my-codebase-intel` 已出现在技能列表，description 正确，无新增冗余项。
- skill 中引用的 `mcpCodegraph*` 工具名已逐一比对当前会话实际可用工具，无拼写错误或不存在的工具。
- 三种降级路径（缺索引 / 缺图谱 / 无 draw.io MCP）均有明确说明，无环境里不可执行的硬步骤。

## 建议 commit

```
feat: 新增 my-codebase-intel skill 与专题文章

- my-skills/my-codebase-intel/SKILL.md（仓库内沉淀副本，便于换机器复用）
- my-skills/README.md（沉淀目录说明）
- codebase-intel-skill.md（专题文章）
- README.md 登记文章与 my-skills 目录
- docs/changes/2026-06-10-codebase-intel-skill.md
```

注：实际生效的 skill 位于 `~/.claude/skills/my-codebase-intel/SKILL.md`（仓库外），仓库内 `my-skills/` 是其可移植副本；换机器时 `cp -r my-skills/my-codebase-intel ~/.claude/skills/` 后 `/reload-plugins` 即可。
