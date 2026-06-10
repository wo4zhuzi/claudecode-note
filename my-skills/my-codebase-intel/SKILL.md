---
name: my-codebase-intel
description: 个人代码库情报工作流，在 Claude Code 中用短指令完成项目看板、项目介绍、模块介绍、架构图/流程图生成、代码分析、bug 排查、重构评估和改动审查。组合使用 Understand-Anything 知识图谱、CodeGraph MCP 精确结构查询、必要源码读取和验证命令，基于可验证证据推进代码库理解与影响面分析。出图以 CodeGraph 原生导出（mermaid/dot）和自然语言绘图指令为主路径，draw.io MCP 仅作可选增强。触发词：项目看板、项目介绍、模块介绍、架构图、流程图、代码分析、bug 排查、重构评估、改动审查、代码库情报
---

# My Codebase Intel

使用这个 skill 减少大范围读文件，让项目理解、代码分析、架构出图和改动审查基于可验证证据推进。

## 默认原则

1. 先判断任务类型：项目看板、项目介绍、模块介绍、架构图/流程图生成、代码分析、bug 排查、重构评估或改动审查。
2. **开局做双前置检查**（决定后续走精确查询还是降级路径）：
   - `.understand-anything/knowledge-graph.json` 是否存在 → 决定能否用全局知识图谱。缺失且需要全局结构时，建议用户先运行 `/understand --language zh`。
   - `.codegraph/graph.db` 是否存在 → 决定 `mcpCodegraph*` 查询能否工作。缺失时，建议用户先运行 `codegraph build`。
   - 任一缺失时，对应能力**降级**为源码搜索 + 文件阅读 + 手动验证，并明确告知用户当前处于降级模式。
3. 优先使用 CodeGraph 查询符号定义、调用方、被调用方、依赖路径、复杂度、入口点、热点和 diff impact（具体工具见下方映射）。
4. 只读取和结论直接相关的源码、测试、配置或文档。
5. 输出时标注关键依据来自 Understand-Anything、CodeGraph、源码、测试、日志还是命令结果。
6. 工具不可用时要明确说明，并降级为源码搜索、文件阅读和手动验证路径。
7. 需要画图时，CodeGraph 原生导出、源码和验证命令负责事实来源；优先输出 mermaid/dot 与文字版规格；draw.io MCP 仅在检测到对应工具时作为可选增强。

事实优先级：

```text
源码 / 测试 / 日志 / 运行结果
> CodeGraph 精确结构查询
> Understand-Anything 全局知识图谱
> LLM 总结
```

## CodeGraph 工具映射

按任务类型选用工具，避免逐个试错、节省 token。所有工具均以 `.codegraph/graph.db` 已建好为前提。

| 任务类型 | 推荐 CodeGraph 工具 |
|---------|---------------------|
| 项目看板 / 项目介绍 | `mcpCodegraphStructure`（目录结构与内聚度）、`mcpCodegraphModuleMap`（最高连通文件）、`mcpCodegraphNodeRoles`（入口/核心/工具/死代码角色）、`mcpCodegraphExecutionFlow`（`list=true` 列出入口点）、`mcpCodegraphTriage`（风险排序队列）、`mcpCodegraphComplexity`（复杂度热点） |
| 模块介绍 / 代码分析 | `mcpCodegraphBrief`（token 高效文件摘要）、`mcpCodegraphContext`（函数完整上下文：源码+依赖+调用方）、`mcpCodegraphWhere`（定义与引用）、`mcpCodegraphQuery`（调用方/被调用方/最短路径）、`mcpCodegraphFnImpact`（函数级影响半径）、`mcpCodegraphAudit`（explain+impact+health 一次拿全） |
| bug 排查 | `mcpCodegraphExecutionFlow`（从入口或错误符号正向追踪）、`mcpCodegraphQuery`（`reverse=true` 反向找调用链）、`mcpCodegraphDataflow`（数据流/数据依赖影响）、`mcpCodegraphContext` |
| 重构评估 | `mcpCodegraphFnImpact`、`mcpCodegraphImpactAnalysis`（文件级影响面）、`mcpCodegraphComplexity`、`mcpCodegraphTriage`、`mcpCodegraphFindCycles`（循环依赖） |
| 改动审查 | `mcpCodegraphDiffImpact`（git diff 受影响函数与调用方）、`mcpCodegraphBranchCompare`（分支间结构差异）、`mcpCodegraphCheck`（CI 门禁/diff 谓词） |
| 架构图 / 流程图 | `mcpCodegraphExportGraph`（依赖图 dot/mermaid/json）、`mcpCodegraphSequence`（时序图 mermaid）、`mcpCodegraphCfg`（函数控制流图 mermaid/dot） |

**可选增强查询及其前置**（未跑前置时不要调用，改用关键词搜索或文件阅读）：

- `mcpCodegraphSemanticSearch` 需先 `codegraph embed`（按语义检索符号）。
- `mcpCodegraphCoChanges` 需先 `codegraph co-change --analyze`（按 git 历史找共同变更文件）。

## 任务类型

### 项目看板

触发示例：

```text
使用 my-codebase-intel 帮我准备项目看板
```

执行流程：

1. 检查 `.understand-anything/knowledge-graph.json` 与 `.codegraph/graph.db`。
2. 如果图谱存在，按 `understand-dashboard` 的流程启动 dashboard，并给出带 token 的访问地址；如果无法启动，说明原因并继续生成 Markdown 看板。
3. 用 Understand-Anything 图谱整理项目层级、关键模块、导览、关键文件和业务域。
4. 用 CodeGraph 查询入口点（`mcpCodegraphExecutionFlow list=true`）、项目结构（`mcpCodegraphStructure`/`mcpCodegraphModuleMap`）、节点角色（`mcpCodegraphNodeRoles`）、风险队列（`mcpCodegraphTriage`）、复杂度热点（`mcpCodegraphComplexity`）。
5. 输出 Markdown 看板，包含：项目概览、模块分区、关键入口、主要依赖、复杂度热点、建议优先阅读文件、后续分析建议。

### 项目介绍

触发示例：

```text
使用 my-codebase-intel 生成项目介绍
```

执行流程：

1. 使用 Understand-Anything 获取项目全局结构、层级、tour、关键文件和文档节点。
2. 使用 CodeGraph 验证入口点（`mcpCodegraphExecutionFlow`）、核心模块（`mcpCodegraphModuleMap`/`mcpCodegraphNodeRoles`）、复杂度热点（`mcpCodegraphComplexity`）。
3. 必要时读取 README、manifest、入口文件和核心模块源码。
4. 输出：项目用途、技术栈、目录结构、核心模块、主要数据流、运行入口、测试入口、适合新手优先阅读的 5 个文件。

### 模块介绍

触发示例：

```text
使用 my-codebase-intel 介绍 <模块路径或模块名>
```

执行流程：

1. 用 CodeGraph 查询模块文件依赖（`mcpCodegraphFileDeps`）、导出符号（`mcpCodegraphFileExports`）、调用方与被调用方（`mcpCodegraphQuery`）、影响范围（`mcpCodegraphImpactAnalysis`）；用 `mcpCodegraphBrief` 拿 token 高效摘要。
2. 用 Understand-Anything 补充该模块在项目层级或业务域中的位置。
3. 读取直接相关源码，确认模块职责和关键路径。
4. 输出：模块职责、主要文件、核心函数或类型、输入输出、上游调用方、下游依赖、常见修改风险和建议验证方式。

### 架构图/流程图生成

触发示例：

```text
使用 my-codebase-intel 把这个项目整理成架构图
使用 my-codebase-intel 把这个项目整理成流程图
```

执行流程：

1. 检查 `.understand-anything/knowledge-graph.json` 与 `.codegraph/graph.db`。图谱存在时用 Understand-Anything 获取项目层级、组件、业务域、关键文件和导览；图谱缺失且需要全局结构时，建议先运行 `/understand --language zh`。
2. 用 CodeGraph 查询并**直接导出图源**，按用户目标选视角：
   - 项目级/模块级架构图：`mcpCodegraphExportGraph`（`format=mermaid` 或 `dot`，`file_level` 控制文件级/函数级）。
   - 跨模块调用时序：`mcpCodegraphSequence`（`format=mermaid`，可加 `dataflow=true` 标注参数与返回）。
   - 单函数控制流：`mcpCodegraphCfg`。
   - 入口点与调用链：`mcpCodegraphExecutionFlow`；复杂度热点：`mcpCodegraphComplexity`；改动视角叠加 `mcpCodegraphDiffImpact`。
3. 读取 README、manifest、入口文件、核心模块源码或测试，校验图中关键节点和边是否来自真实代码。
4. **出图三层降级**（主路径不依赖任何 draw.io MCP）：
   - **第一层（主路径）**：先输出文字版架构/流程摘要，再贴 CodeGraph 导出的 mermaid/dot —— 用户可直接在 Markdown 预览或任意 mermaid 渲染器查看。
   - **第二层**：在 mermaid/dot 之外，给出可复制给任意 draw.io AI 的自然语言绘图指令（描述节点、边、分组、层级、标签、颜色含义）。
   - **第三层（可选增强）**：先探测会话里是否存在 draw.io MCP 工具（如 `start_session`/`create_new_diagram`/`edit_diagram`/`get_diagram`/`export_diagram`）。存在才用：先 `start_session` 打开实时预览（确认 URL 带 `?mcp=`），再 `create_new_diagram` 或 `edit_diagram`，需保存时 `get_diagram` 或 `export_diagram`；遇 `No active session` 先重新 `start_session`。不存在则跳过本层并说明。
5. 架构图规格至少包含：节点、边、分组、层级、标签和颜色含义。流程图规格至少包含：开始节点、结束节点、步骤、判断分支、异常路径和跨模块调用。

默认图谱分层：

```text
入口 / 用户接口
> 核心模块 / 业务域
> 数据 / 外部服务 / 工具链
```

常用标记：

- 高复杂度节点：标注复杂度或维护风险（来自 `mcpCodegraphComplexity`/`mcpCodegraphTriage`）。
- 强依赖边：标注主要调用、导入或数据流关系。
- 变更影响面：标注受 `mcpCodegraphDiffImpact` 或调用方分析影响的区域。

### 代码分析与修改方案

触发示例：

```text
使用 my-codebase-intel 分析 <目标函数或模块>
```

执行流程：

1. 用 CodeGraph 查询目标定义位置（`mcpCodegraphWhere`）、完整上下文（`mcpCodegraphContext`）、调用方/被调用方（`mcpCodegraphQuery`）、复杂度（`mcpCodegraphComplexity`）和影响半径（`mcpCodegraphFnImpact`）；需要一次拿全结构+影响+健康度时用 `mcpCodegraphAudit`。
2. 用 Understand-Anything 判断目标属于哪个层级、模块或业务域。
3. 读取直接相关源码和测试。
4. 在提出修改前先说明根因或当前判断。
5. 输出最小修改方案、涉及文件、验证命令和剩余风险；用户未要求实现时不修改代码。

### Bug 排查

触发示例：

```text
使用 my-codebase-intel 排查 <现象或错误>
```

执行流程：

1. 先复现或收集现象、失败命令、日志和错误信息。
2. 用 CodeGraph 从相关入口或错误符号追踪：`mcpCodegraphExecutionFlow` 正向追调用链，`mcpCodegraphQuery reverse=true` 反向找调用方，`mcpCodegraphDataflow` 看数据依赖。
3. 读取最小相关源码集合定位根因。
4. 输出根因、证据、最小修复方案、验证命令和风险边界。

### 重构评估

触发示例：

```text
使用 my-codebase-intel 评估重构 <函数、类型或模块>
```

执行流程：

1. 用 CodeGraph 查询调用方与跨模块依赖（`mcpCodegraphQuery`/`mcpCodegraphImpactAnalysis`）、复杂度（`mcpCodegraphComplexity`）、影响半径（`mcpCodegraphFnImpact`）、循环依赖（`mcpCodegraphFindCycles`）、风险排序（`mcpCodegraphTriage`）。
2. 用 Understand-Anything 判断涉及层级和业务域。
3. 输出重构风险、分阶段方案、兼容边界、测试范围和回滚建议。
4. 用户未确认前不执行重构。

### 改动审查

触发示例：

```text
使用 my-codebase-intel 审查当前改动
```

执行流程：

1. 查看当前 git diff 或 staged diff 的变更文件。
2. 如果 Understand-Anything 图谱存在，分析变更文件所在层级、组件和相关节点。
3. 使用 `mcpCodegraphDiffImpact` 查询受影响函数、调用方和跨模块依赖；需要跨分支对比用 `mcpCodegraphBranchCompare`，需要门禁校验用 `mcpCodegraphCheck`。
4. 按严重程度输出 findings，重点看行为回归、遗漏测试、边界条件、文档同步和验证缺口。

## 输出要求

- 结论优先，先说当前判断和最重要风险。
- 引用具体文件、函数、模块或命令结果作为依据。
- 区分“已验证事实”和“基于图谱/调用关系的推断”。
- 需要用户决策时，给出推荐方案和取舍依据。
- 不输出大段工具原始结果，保留高信号摘要。
- 架构图任务先给文字版架构摘要，再给图谱规格；流程图任务先给流程摘要，再给流程规格。说明用了哪层出图路径（mermaid/dot、自然语言指令、还是 draw.io MCP），以及降级原因（如未检测到 draw.io MCP）。

## 边界

- 不直接读取或修改 `.codegraph/graph.db`。
- 不把 Understand-Anything、CodeGraph 或 LLM 总结当作最终事实。
- 不假定 draw.io MCP 存在；调用前先探测对应工具，缺失时走 mermaid/dot + 自然语言指令路径。
- 不使用未跑前置（`codegraph embed` / `codegraph co-change --analyze`）的 CodeGraph 增强查询。
- 涉及行为判断时，不跳过源码、测试、日志或运行结果验证。
- 声称完成前，不跳过测试或等价手动验证。
- 不为了理解背景而一次性读取大量无关文件。
