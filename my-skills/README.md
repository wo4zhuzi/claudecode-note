# my-skills

我自己沉淀的 Claude Code standalone skill，随仓库版本化保存。换电脑或换工具时直接复制到 skill 目录即可复用，无需依赖任何插件市场。

## 目录结构

每个 skill 一个子目录，目录名即 skill 名，内含 `SKILL.md`：

```
my-skills/
└── <skill名>/
    └── SKILL.md
```

## 已收录

| Skill | 用途 |
|-------|------|
| [my-codebase-intel](./my-codebase-intel/SKILL.md) | 整合 CodeGraph MCP 与 Understand-Anything 的代码库情报工作流：项目看板、模块介绍、架构/流程图、代码分析、bug 排查、重构评估、改动审查 |

## 安装方式

Claude Code 原生识别 `~/.claude/skills/` 下的目录为 standalone skill。把需要的 skill 目录复制过去即可：

```bash
cp -r my-skills/my-codebase-intel ~/.claude/skills/
```

复制后执行 `/reload-plugins` 生效。换机器时把整个 `my-skills/` 都复制过去：

```bash
cp -r my-skills/* ~/.claude/skills/
```

机制细节参见仓库文章 [插件与技能的安装方式](../plugin-skill-use.md)。

## 维护约定

- 这里只放**我自己写的** skill；从第三方插件提取的 skill（如 `planning-with-files-zh`）不纳入，避免与上游版本脱节。
- 新增或修改 skill 后，同步更新本表与上方文章链接。
