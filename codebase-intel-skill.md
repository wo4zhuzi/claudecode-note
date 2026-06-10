# 自建 my-codebase-intel：代码库情报工作流 skill

大型项目里，AI 理解代码最贵的开销不是推理，是**反复读文件**。一次「介绍一下这个模块」可能触发几十次 Read、Grep，把上下文窗口塞满源码，token 哗哗地烧。

CodeGraph 和 Understand-Anything 这两个工具正是冲着这个问题来的：把代码结构预先索引成图，AI 用「查图」代替「读文件」。我把它们整合成一个个人 skill `my-codebase-intel`，用短指令完成项目看板、项目介绍、模块介绍、架构/流程图、代码分析、bug 排查、重构评估、改动审查。这篇文章讲整合思路。

## 为什么这俩能省 token

两个工具职责不同，互补而非重叠：

| 工具 | 形态 | 擅长 | 数据来源 |
|------|------|------|---------|
| **CodeGraph** | MCP server（`mcpCodegraph*` 工具） | 精确结构查询：符号定义、调用方/被调用方、依赖路径、复杂度、影响半径、diff impact | `.codegraph/graph.db`（`codegraph build` 生成） |
| **Understand-Anything** | 插件（`understand-*` 技能） | 全局知识图谱：架构层级、业务域、关键文件、导览、可视化 dashboard | `.understand-anything/knowledge-graph.json`（`/understand` 生成） |

简单说：**CodeGraph 回答「这个函数被谁调用、改了影响多大」，Understand-Anything 回答「这个项目分几层、核心模块是哪些」**。前者是显微镜，后者是地图。官方数据 CodeGraph 能减少约 58% 的工具调用、降低约 16% 的费用——省的就是反复 Grep/Read 的那部分。

## 事实优先级：LLM 总结排最后

整合时最容易踩的坑，是把图谱查询结果或 LLM 总结**当成最终事实**直接下结论。图谱是索引的快照，可能过期；LLM 总结可能脑补。所以 skill 里钉死了一条优先级金字塔：

```text
源码 / 测试 / 日志 / 运行结果      ← 最高，唯一的「事实」
> CodeGraph 精确结构查询           ← 结构线索，需源码佐证
> Understand-Anything 全局知识图谱  ← 全局地图，可能滞后
> LLM 总结                         ← 最低，必须可追溯到上面三层
```

涉及行为判断（这个改动会不会破坏功能、这个 bug 根因在哪），永远要回到源码/测试/运行结果验证，不能停在「图谱说……」。

## 任务类型 → 工具映射

skill 的核心改进，是把 8 类任务**直接映射到具体工具名**，让模型少试错。抽象描述「查调用关系」会让模型挨个试工具，点名 `mcpCodegraphQuery` 则一步到位。

| 任务类型 | 推荐工具 |
|---------|---------|
| 项目看板 / 项目介绍 | `mcpCodegraphStructure`、`mcpCodegraphModuleMap`、`mcpCodegraphNodeRoles`、`mcpCodegraphExecutionFlow`(`list=true`)、`mcpCodegraphTriage`、`mcpCodegraphComplexity` + `understand` / `understand-dashboard` |
| 模块介绍 / 代码分析 | `mcpCodegraphBrief`、`mcpCodegraphContext`、`mcpCodegraphWhere`、`mcpCodegraphQuery`、`mcpCodegraphFnImpact`、`mcpCodegraphAudit` |
| bug 排查 | `mcpCodegraphExecutionFlow`(正向)、`mcpCodegraphQuery`(`reverse=true` 反向)、`mcpCodegraphDataflow`、`mcpCodegraphContext` |
| 重构评估 | `mcpCodegraphFnImpact`、`mcpCodegraphImpactAnalysis`、`mcpCodegraphComplexity`、`mcpCodegraphTriage`、`mcpCodegraphFindCycles` |
| 改动审查 | `mcpCodegraphDiffImpact`、`mcpCodegraphBranchCompare`、`mcpCodegraphCheck` + `understand-diff` |
| 架构图 / 流程图 | `mcpCodegraphExportGraph`、`mcpCodegraphSequence`、`mcpCodegraphCfg` |

几个高性价比的组合工具值得单独点名：

- **`mcpCodegraphAudit`**：一次返回 explain + impact + health，看单个文件/函数最省事。
- **`mcpCodegraphTriage`**：按风险（连通度 + 复杂度 + churn）排序的审计队列，做看板和重构评估时直接给出「该先看哪里」。
- **`mcpCodegraphBrief`**：token 高效的文件摘要，正好契合「省 token」的初衷。

### 有前置依赖的增强查询

两个工具默认不可用，要先建额外索引，**没建就别调用**（否则静默失败）：

- `mcpCodegraphSemanticSearch` —— 按语义检索符号，需先 `codegraph embed`。
- `mcpCodegraphCoChanges` —— 按 git 历史找共同变更文件，需先 `codegraph co-change --analyze`。

未跑前置时，skill 降级为关键词搜索或文件阅读。

## 出图的三层降级

最初的草稿把出图完全押在 `next-ai-draw-io` MCP 上。但实测环境里**这个 MCP 未必装**——会话工具列表里只有 `mcpCodegraph*`，没有任何 `start_session`/`create_new_diagram` 之类的 draw.io 工具。如果 skill 把它写成硬步骤，一调用就报错。

更关键的发现是：**CodeGraph 自己就能出图**，直接导出可渲染的 mermaid/dot。所以出图改成三层降级，主路径完全不依赖任何 draw.io MCP：

```text
第一层（主路径）：CodeGraph 原生导出 mermaid/dot + 文字版规格
  ├─ mcpCodegraphExportGraph  → 依赖图（dot/mermaid/json）
  ├─ mcpCodegraphSequence     → 时序图（mermaid）
  └─ mcpCodegraphCfg          → 函数控制流图（mermaid/dot）
        ↓ 用户可直接在 Markdown 预览 / 任意 mermaid 渲染器查看

第二层：可复制给任意 draw.io AI 的自然语言绘图指令
        （描述节点、边、分组、层级、标签、颜色含义）

第三层（可选增强）：先探测会话里是否存在 draw.io MCP 工具，
        存在才调用 start_session → create/edit_diagram → export
        不存在则跳过并说明
```

这样设计的好处：**即使一个 draw.io MCP 都没装，也能立刻产出能渲染的图**，完全契合「省 token、可验证」的初衷。draw.io MCP 从「主路径」降为「锦上添花」。

## 双前置检查与降级

skill 开局做两个检查，决定走精确查询还是降级路径：

| 检查项 | 缺失时的建议 | 降级方式 |
|--------|------------|---------|
| `.understand-anything/knowledge-graph.json` | 提示运行 `/understand --language zh` | 全局结构改用源码搜索 + 文件阅读 |
| `.codegraph/graph.db` | 提示运行 `codegraph build` | 结构查询改用 Grep + Read + 手动验证 |

原始草稿只检查了 understand 图谱，漏了 codegraph 索引——这是个不对称。实际上 CodeGraph 是项目级作用域，每个项目都要单独 `codegraph build`，缺索引时所有 `mcpCodegraph*` 查询都会失败。补上这个检查后，skill 在任一工具缺席时都能优雅降级，不会写出环境里不可执行的硬步骤。

## 安装方式

`my-codebase-intel` 是个人定制的工作流，没有多语言/多平台变体，所以直接做成 **standalone skill**，放进 `~/.claude/skills/`，全局所有项目可用——和 [`planning-with-files-zh` 的安装机制](./plugin-skill-use.md)完全一致：

```bash
# 目录即技能，无需任何配置文件
~/.claude/skills/my-codebase-intel/SKILL.md
```

写好后执行 `/reload-plugins` 生效，技能列表里就能看到 `my-codebase-intel`。它依赖的两个底层工具按各自方式安装：

- **CodeGraph**：MCP，安装见 [MCP 应用安装指南](./mcp-installation-guide.md) 的 CodeGraph 章节，每个项目跑一次 `codegraph build`。
- **Understand-Anything**：第三方市场插件，安装见 [插件与技能的安装方式](./plugin-skill-use.md) 案例三。

## 用法

短指令触发，skill 自动判断任务类型、走对应工具链：

```text
使用 my-codebase-intel 帮我准备项目看板
使用 my-codebase-intel 介绍 src/auth 模块
使用 my-codebase-intel 把这个项目整理成架构图
使用 my-codebase-intel 分析 handleLogin 函数
使用 my-codebase-intel 排查 登录后 token 不刷新
使用 my-codebase-intel 评估重构 PaymentService
使用 my-codebase-intel 审查当前改动
```

## 设计原则小结

- **职责分工**：CodeGraph 管精确结构，Understand-Anything 管全局地图，源码管事实。
- **事实优先级**：LLM 总结排最后，行为判断必须回到源码/测试/运行结果。
- **映射具体工具名**：减少模型试错，省 token。
- **三层出图降级**：主路径 = CodeGraph 原生 mermaid/dot，不假定任何 draw.io MCP 存在。
- **双前置检查**：图谱和索引都查，任一缺失都能优雅降级。
