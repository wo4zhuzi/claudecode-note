# 修正 CodeGraph 安装文档中的错误命令

## 变更内容

- 修正 `mcp-installation-guide.md` 中 CodeGraph 安装步骤的错误命令：
  - `codegraph install`（不存在）→ `claude mcp add codegraph -- codegraph mcp`
  - `codegraph init -i`（不存在）→ `codegraph build`
  - 升级/卸载改为 `npm update -g` 和 `claude mcp remove codegraph`

## 涉及文件

- `mcp-installation-guide.md`（修正）
