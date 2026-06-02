# Codex 与 Claude Code 协作实践

> 适用场景：团队中同时使用 OpenAI Codex 和 Anthropic Claude Code，或个人在两个工具之间切换。

---

## 协作模式

### 单人切换模式

同一人根据任务性质选择工具：

- **Codex**：全文件范围批量重构、搜索替换、独立且描述清晰的单次任务
- **Claude Code**：需要来回问答的复杂逻辑、多步骤有状态的任务、需要解释和推理的场景

两个工具可以操作同一个仓库，切换时无需额外设置，但要确保上一个工具的变更已提交，避免工作区脏状态干扰下一个工具的上下文。

### 双人分工模式

两个人协作同一仓库，各自使用不同工具。核心挑战不是工具兼容性，而是**约定同步**——两个工具各自读取自己的配置文件，如果配置文件里写的是不同的测试命令、不同的分支规范，协作就会产生摩擦。

解决方式见下一节。

### 流水线模式

Codex `plan-with-file` 生成任务清单 → Claude Code 逐项执行，或反向：Claude Code 设计方案 → Codex 批量实现细节。

这是最有价值也最容易出问题的模式，见第三节。

### 互相审查模式

一个工具生成代码，另一个工具做 code review。两个模型有不同的训练偏好和盲点，互相审查能发现单个工具容易忽略的问题。实际使用时直接把文件路径或 diff 交给另一个工具，要求它找问题即可。

---

## AGENTS.md 与 CLAUDE.md 的融合

### 文件职责

- `AGENTS.md`：Codex 读取，说明仓库结构、可用命令、约定规范
- `CLAUDE.md`：Claude Code 读取，说明相同内容，但可包含 Claude 特有的行为指令（如 hooks 说明、回答语言要求）

两个文件的**公共内容**：目录结构、分支规范、包管理器、测试命令、提交格式。  
**工具特有内容**：Codex 的 sandbox 权限声明、Claude Code 的 hooks 配置说明、各自的行为约定。

### 重复内容：提取到 CONVENTIONS.md

把公共内容单独放到一个文件，两个工具文件只做引用：

```markdown
<!-- AGENTS.md -->
# 仓库约定
见 CONVENTIONS.md。

## Codex 专用
- 验证命令：`pnpm test && pnpm lint`
- 每次任务结束后运行验证，不通过不提交
```

```markdown
<!-- CLAUDE.md -->
# 仓库约定
见 CONVENTIONS.md。

## Claude Code 专用
- 回答语言：中文
- 默认使用 plan 模式，执行前先输出步骤
```

```markdown
<!-- CONVENTIONS.md -->
## 包管理器
使用 pnpm。禁止混用 npm / yarn。

## 分支规范
- 功能分支：feat/<name>
- 修复分支：fix/<name>

## 提交格式
<type>(<scope>): <summary>
```

这样修改约定只需改 `CONVENTIONS.md` 一处，两个工具文件不会分叉。

### 冲突内容：显式声明工具范围

当两个工具确实需要不同行为时，在各自文件顶部声明"此文件仅供 [工具名] 读取"，并保留差异：

```markdown
<!-- AGENTS.md 顶部 -->
> 此文件供 OpenAI Codex 读取。Claude Code 请查看 CLAUDE.md。
```

典型冲突场景及处理方式：

| 冲突类型 | 错误做法 | 正确做法 |
|----------|----------|----------|
| 两个文件写了不同的测试命令 | 各自维护，逐渐偏移 | 统一写入 CONVENTIONS.md，各文件引用 |
| Codex 要求英文注释，Claude 要求中文 | 沉默冲突 | 在各自文件显式声明，PR review 时人工对齐 |
| 两个文件描述的目录结构不一致 | 哪个对就看哪个 | 目录结构只在 CONVENTIONS.md 维护 |

### 双人团队的同步工作流

PR 模板中加一条检查项：

```markdown
- [ ] 若修改了仓库约定（命令、目录结构、规范），已同步更新 CONVENTIONS.md
```

不需要每次都手动对比三个文件，只要约定改动走 CONVENTIONS.md，两个工具文件自然保持一致。

---

## plan-with-file 跨工具执行

### Codex 生成的任务格式

`plan-with-file` 把规划结果写入一个 Markdown 文件（通常是 `plan.md`），格式是 checkbox 列表：

```markdown
- [x] 安装依赖并初始化项目结构
- [ ] 在 src/api/user.ts 中实现 getUser 函数
- [ ] 为 getUser 编写单元测试
- [ ] 运行 pnpm test 确认通过
```

Codex 执行时会勾选已完成项，未完成项保持 `- [ ]`。

### Claude Code 能否接着执行？

**可以，但有条件。**

Claude Code 读取 Markdown 文件并按顺序执行任务没有技术障碍——直接告诉它"读取 plan.md，从第一个未勾选项开始执行"即可。真正的风险是**隐含上下文**：Codex 在执行已勾选任务时积累了上下文（知道某个函数叫什么、某个变量在哪里），这些上下文不在文件里，Claude Code 从零开始读不到。

两种情况：

1. **已勾选任务做了代码变更** → Claude Code 可以通过读文件获取结果，风险低
2. **已勾选任务只做了决策（如"确定用 JWT 方案"）** → 决策结果没有写入文件，Claude Code 可能做出不一致的选择

### 让任务文件对两个工具都可执行

写任务描述时遵循以下规范，两个工具都能无歧义地执行：

**包含文件路径，不用指代词：**

```markdown
# 差
- [ ] 给那个 API 函数加类型

# 好
- [ ] 为 src/api/user.ts 中的 getUser 函数添加返回类型 Promise<User>
```

**写明验收条件，不用模糊表述：**

```markdown
# 差
- [ ] 确保测试正常

# 好
- [ ] 运行 `pnpm test src/api/user.test.ts`，所有用例通过
```

**决策结果写进文件，不留在工具上下文里：**

```markdown
# 差
- [x] 确定认证方案

# 好
- [x] 确定认证方案：使用 JWT，token 存储在 httpOnly cookie，有效期 7 天
      实现文件：src/middleware/auth.ts
```

**命令用完整形式：**

```markdown
# 差
- [ ] 跑一下测试

# 好
- [ ] 运行 `pnpm test`，确认无新增失败
```

### 实际操作流程

```
1. Codex：接收需求，运行 plan-with-file，生成 plan.md
2. 人工检查：确认每条任务描述是否自包含，补全模糊项
3. git commit plan.md
4. Claude Code：读取 plan.md，从第一个 - [ ] 开始执行
5. 每完成一项，勾选对应 checkbox 并提交
```

第 2 步的人工检查是关键——不需要很长时间，扫一遍任务描述，遇到"那个"、"之前的"、"上一步的"这类指代词就补全成具体路径或决策结果。

---

## 总结

| 问题 | 建议 |
|------|------|
| 两个工具文件的重复内容 | 提取到 CONVENTIONS.md，两个文件只引用 |
| 两个工具文件的冲突内容 | 显式声明各自文件的读取工具，保留合理差异 |
| plan-with-file 能否跨工具执行 | 可以，前提是任务描述自包含：有路径、有命令、无指代词 |
| 双人协作的同步成本 | PR 模板加检查项，约定改动统一走 CONVENTIONS.md |
