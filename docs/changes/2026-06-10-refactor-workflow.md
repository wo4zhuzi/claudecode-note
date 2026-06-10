# 2026-06-10 新增专题文章：AI 辅助模块重构工作流

## 背景

用户想沉淀一套「和 AI 协作重构项目模块」的工程实践方法论。痛点是 AI 重构有两大风险：AI 对模块的理解还没达标就动手；改着改着跑偏，等回头 review 才发现、回滚代价已经很大。

用户已装齐所需工具（`my-codebase-intel` skill、`planning-with-files-zh`、CodeGraph MCP、Understand-Anything、superpowers），缺的只是把它们串成一条固定流程并写下来。本次产出是一篇方法论专题文章，不新建 skill、不改代码。

## 已确认的关键决策

1. **沉淀形式：专题文章**（不新建 skill）。
2. **回滚机制抽象成「验证门禁」**：文章不绑定"必须写测试"，而是立原则——每个重构步骤必须可验证。有测试框架就写 characterization tests 跑测试；没有就降级到脚本复现 / 手动复现 / 输入输出对比。git 原子 commit 是粗网（能撤销），验证门禁是细网（早发现跑偏）。

## 改动内容

### 新增文章：`refactor-workflow.md`

讲 AI 辅助模块重构的六阶段工作流闭环：

1. 问题：重构 = 行为不变地改结构，"行为基线"是锚点；两大风险。
2. 核心思路：理解契约化 + 步骤可验证 + 改动原子化。
3. 阶段①理解模块 → 产出"行为契约"（用 `my-codebase-intel` + CodeGraph 钉公开接口/输入输出/副作用/不变量/调用方，存进 `findings.md`）。
4. 阶段②理解达标 gate（显式批准后才进入写计划）。
5. 阶段③写细计划（plan-with-file + specs，specs 钉"目标行为 + 验收标准 + 怎么验证"而非逐行代码；按影响半径先改叶子）。
6. 阶段④验证门禁（本文核心：每步必须可验证，有测试写 characterization tests、无测试降级到脚本/手动/输出对比）。
7. 阶段⑤逐步执行 + 原子化改动（git worktree 隔离，每个 spec 步骤一个原子 commit，`progress.md` 记审计轨迹）。
8. 阶段⑥跑偏回滚（验证门禁早发现 + git revert/reset，用 progress.md 定位发散起点）。
9. 工具串联总表。

交叉引用 `codebase-intel-skill.md`、`my-skills/my-codebase-intel/`、`planning-with-files-zh` 及 superpowers 的 brainstorming / writing-plans / TDD / using-git-worktrees。

### 登记

- `README.md` 文章目录新增一行链接到 `refactor-workflow.md`。
- 本变更记录。

## 验证

- 通读文章，六阶段闭环完整、无 TBD/占位、术语与已有文章一致。
- 验证门禁一节"无测试框架"分支有明确可执行的降级手段，未写"必须有测试"的硬前提。
- README 新增行链接可达。

## 建议 commit

```
docs: 新增 AI 辅助模块重构工作流专题文章

- refactor-workflow.md（六阶段重构闭环方法论）
- README.md 文章目录登记
- docs/changes/2026-06-10-refactor-workflow.md
```
