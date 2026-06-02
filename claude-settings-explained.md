# settings.json 配置详解

> 对应配置文件：[setting/claude-settings-recommended.json](./setting/claude-settings-recommended.json)

本文逐项说明每个配置的**设计原因**，而非重复字段定义。

---

## 基础开关

```json
"autoCompactContext": true
```
长对话中 token 会持续积累，溢出后任务强制中断。开启自动压缩，Claude 会在接近上限前主动摘要历史，保证长任务连续运行。

```json
"autoCreateCheckpoint": true
```
每次高风险操作（文件批量修改、重构）前自动保存检查点，可一键回滚，等同于给每次操作加了一个安全网。

```json
"autoUpdate": false
```
禁止自动更新。版本升级可能带来行为变更（新默认值、废弃参数），手动控制升级时机可避免在关键任务中途出现非预期变化。

```json
"defaultMode": "plan"
```
进入任何任务前强制先输出计划供确认，再执行。对文件系统、git、部署等不可逆操作，"先想清楚再动手"是最低成本的防错机制。

```json
"disableExtensionInstallation": true
```
阻止 Claude 自主安装 IDE 扩展，保持开发环境可控，避免引入未审查的第三方插件。

```json
"language": "zh"
```
固定默认响应语言为中文，无需在每次对话 prompt 中重申语言要求。

```json
"model": "claude-sonnet-4-6"
```
Sonnet 是能力与成本的平衡点，覆盖日常编码任务。遇到需要更深推理的复杂任务，用 `/model` 临时切换到 Opus。

---

## customInstructions

`customInstructions` 会注入到每次对话的系统提示词，相当于为所有对话预设了一套行为规范。

**"先规划后执行"与"危险操作确认"**

`defaultMode: "plan"` 控制的是 Claude 的运行模式，但模式可以被用户绕过。`customInstructions` 里再次强调这两点，是双重保障——即使模式被切换，Claude 也会保持这个习惯。危险操作（删除、push、部署）的 blast radius 大、难恢复，多一次确认的成本极低。

**"代码简洁，注释仅写 WHY"**

阻止 Claude 生成过度注释和解释性 docstring。注释应该说明代码**为什么**这么写（隐含约束、特殊 workaround），而不是重复代码**做了什么**。

**"修改已有文件优先于新建文件"**

防止 Claude 习惯性地新建文件而不是修改现有文件，减少无谓的文件数量膨胀。

**回答结构：结论优先**

工程场景下最常见的需求是"这个问题怎么解"，不是"给我讲一遍背景"。结论放第一行，关键依据放第二段，细节放最后，方便快速扫读。

**"先定位根因，再给解决方案"**

阻止 Claude 在不理解问题的情况下直接猜测方案。错误的方案不只是没帮助，还会占用用户的排查时间。

**文档默认简体中文**

`/init`、`/review` 等命令默认生成英文文档。这条指令覆盖这个默认行为，确保生成的 CLAUDE.md、README 等文档使用简体中文。

---

## permissions

### allow 列表

```json
"Bash(git *)", "Bash(npm *)", "Bash(pnpm *)", ...
```

开放所有主流工具链命令：构建工具（npm/pnpm/yarn/bun/cargo/go/make）、运行时（node/python）、测试运行器（pytest/jest/vitest）、类型检查与构建（tsc/vite），以及只读 shell 工具（ls/find/grep/rg/cat/echo/pwd/which/env）。

不在列表内的命令会弹出权限确认。这个列表的目标是：**日常开发零打断，非常规操作需确认**。

`WebSearch` 和 `WebFetch` 也在允许列表，Claude 可以在需要时主动查询文档，无需每次手动批准。

### deny 列表

```json
"Bash(rm -rf /)",        // 递归删除根目录
"Bash(sudo rm *)",        // 以 root 权限删除
"Bash(curl * | bash *)",  // 下载并直接执行远程脚本
"Bash(wget * | sh *)",    // 同上
"Bash(chmod -R 777 *)"   // 递归开放所有权限
```

deny 的优先级高于 allow，这五类是最典型的高危操作：不可逆、影响范围大、或存在供应链攻击风险。无论对话上下文如何，这些命令都不会被执行。

---

## hooks

### Stop hook：任务完成通知

```json
"command": "osascript -e 'display notification \"Claude 已完成\" with title \"Claude Code\"' 2>/dev/null || true"
```

Claude 执行完毕时触发 macOS 系统通知。适用场景：跑测试、做代码审查、大规模重构等耗时任务，不用盯着终端等待，做其他事情时也不会错过完成信号。

`2>/dev/null || true` 确保非 macOS 环境下静默忽略，不影响其他平台使用。

### PreToolUse hook：git commit 前检查暂存区

```json
"matcher": "Bash(git commit*)"
```

每次 Claude 准备执行 `git commit` 时触发，用内联 Python 脚本检查暂存区：

```python
r = subprocess.run(['git', 'diff', '--cached', '--stat'], capture_output=True, text=True)
if not r.stdout.strip():
    print('暂存区为空，请先 git add 相关文件')
    sys.exit(2)
```

- `sys.exit(2)`：退出码 2 表示**阻断**，Claude 不会继续执行 commit 命令
- `sys.exit(0)`：退出码 0 表示**放行**

防止 Claude 在未 `git add` 的情况下提交空 commit，避免产生无意义的 git 历史记录。
