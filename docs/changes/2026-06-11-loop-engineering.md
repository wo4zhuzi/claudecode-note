# 2026-06-11 新增专题文章：/loop 与循环工程实践

## 背景

用户问 `/loop` 指令怎么用、对「Loop Engineer」有什么落地方案。对话里已讲清三种模式、三种落地范式和工程纪律，用户要求把这部分内容整理成正式文章沉淀到仓库。本次产出是一篇方法论专题文章，不新建 skill、不改配置。

## 已核实的事实依据

1. `/loop` 是 Claude Code **内置 skill**（不在插件文件系统里，harness 直接提供），支持 interval / 动态 / 裸跑三种模式。动态模式下模型用 `ScheduleWakeup` 自选下次唤醒时间。
2. `ralph-loop` 是官方插件（`claude-plugins-official`），通过 **Stop hook** 在当前 session 内拦截退出、把同一 prompt 喂回；参数 `--max-iterations`、`--completion-promise`（精确字符串匹配，不能用于多条件退出）。
3. `/plan-loop` 来自已装的 `planning-with-files` 插件，是 plan 感知的 `/loop` 封装：默认 tick 重读 `task_plan.md`/`progress.md` → 跑 `check-complete.sh` → 没进展写 progress；配合 `/plan-goal` 设终止条件。

## 改动内容

### 新增文章：`loop-engineering.md`

讲 `/loop` 的用法与循环工程的三种落地范式：

1. 开篇：循环工程的本质 = 可重入循环 + 明确终止条件 + 持久化状态，三者缺一要么死循环要么丢上下文。
2. `/loop` 三种模式总表：interval（给间隔，cron 轮询外部状态）、动态（不给间隔，模型自选节奏）、裸跑（内置维护提示）。
3. 三种落地范式：方案 A Ralph Loop（Stop hook 喂回同一 prompt，completion-promise 防偷懒 + max-iterations 兜底，状态靠 git/文件传递）；方案 B Plan-Loop（plan 文件驱动，每 tick 强制落地，配 plan-goal 自动停）；方案 C interval 轮询（等 CI/部署/PR 队列，附「只等一次通知别用 loop」反模式）。
4. 工程纪律表：状态持久化 / 终止条件 / 节奏选择 / 幂等性 / 可观测 五条，各列做法 vs 反模式。
5. 选型一句话 + 可复制的带 completion-promise 提示词模板。

交叉引用 `refactor-workflow.md`、`planning-with-files-zh`、`plugin-skill-use.md`。

### 登记

- `README.md` 文章目录新增一行链接到 `loop-engineering.md`。
- 本变更记录。

## 验证

- 通读文章，确认命令/参数与已核实事实一致（尤其 ralph-loop 是 Stop hook、completion-promise 精确匹配、`/loop` 是内置 skill）。
- 三种范式各有适用场景与反模式，无 TBD/占位。
- README 新增行链接可达。

## 建议 commit

```
docs: 新增 /loop 与循环工程实践专题文章

- loop-engineering.md（/loop 三种模式 + 三种落地范式 + 工程纪律）
- README.md 文章目录登记
- docs/changes/2026-06-11-loop-engineering.md
```
