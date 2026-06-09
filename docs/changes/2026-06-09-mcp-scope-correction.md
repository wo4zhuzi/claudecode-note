# 修正 MCP 作用域说明

修正 `mcp-installation-guide.md` 中关于 `claude mcp add` 作用域的多处错误描述。错误的事实通过 `claude mcp add -h` 与实际配置文件 `~/.claude.json` 核实后更正。

## 核实到的实际行为

- 配置文件是 `~/.claude.json`，**不是** `~/.claude/settings.json`，也不是 `~/.claude/mcp.json`。
- CLI **没有** `-g` / `--global` 选项；作用域通过 `-s/--scope` 设置，可选值 `local`（默认）、`user`、`project`。
- `local`（默认）：存入 `~/.claude.json` 的 `projects["<项目路径>"].mcpServers`，仅当前项目、仅自己可见。
- `user`：存入 `~/.claude.json` 顶层 `mcpServers`，对自己的所有项目生效（真正的「全局」）。
- `project`：存入项目根目录 `.mcp.json`，可提交，团队共享。

## 修改点

1. **方式一作用域说明**：用核实后的 `-s/--scope` 三作用域表格替换，纠正默认作用域（local，非 project）、存储路径（`~/.claude.json`），并注明无 `-g` 选项。
2. **方式二**：全局配置文件由错误的 `~/.claude/mcp.json` 更正为 `~/.claude.json`。
3. **CodeGraph 第 2 步**：保留默认命令并解释其 local 作用域行为，全局选项由错误的 `-g` 更正为 `--scope user`。
4. **选择原则**：建索引命令由不存在的 `codegraph init -i` 统一为文档中实际使用的 `codegraph build`。
