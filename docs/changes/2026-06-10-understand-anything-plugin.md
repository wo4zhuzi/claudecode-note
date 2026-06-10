# 2026-06-10 记录 Understand-Anything 插件安装

## 本次改动

### 1. `plugin-skill-use.md` 新增「案例三：understand-anything —— 第三方市场安装」

插在案例二之后、「关于 hook 路径变量」之前。前两个案例的插件都来自现成市场，本案例补上此前缺失的维度——**从一个尚未注册的第三方 GitHub 仓库安装插件**，需先添加市场再安装，共两步：

```
/plugin marketplace add Egonex-AI/Understand-Anything   # 添加第三方市场
/plugin install understand-anything@understand-anything # 从该市场安装
```

要点：点明安装标识 `<插件名>@<市场名>` 的构成；说明它提供 8 个 `understand-*` 技能 + agents + hooks、无冗余变体，故归入完整插件安装（与案例一同类，仅来源不同）；澄清它不附带 MCP server，会话里的 `mcpCodegraph*` 来自单独安装的 CodeGraph MCP，与本插件无关。

### 2. `README.md` 目录简介微调

[plugin-skill-use.md](../../plugin-skill-use.md) 的目录行简介补充「第三方市场安装」维度，表格行数不变。

### 3. 新增本变更记录

`docs/changes/2026-06-10-understand-anything-plugin.md`。

## 原因

- 补全「第三方 GitHub 市场安装」这一前两个案例都没演示的完整流程（先 `marketplace add` 再 `install`）。
- understand-anything 无冗余变体、各技能配合使用，故采用完整插件安装而非单技能提取。
- 澄清它不带 MCP server，避免读者把会话里的 `mcpCodegraph*` 工具误认为本插件提供，与 [MCP 应用安装指南](../../mcp-installation-guide.md) 中的 CodeGraph 区分开。
