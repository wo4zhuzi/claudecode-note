# 2026-06-09 插件与技能安装实践

## 本次改动

### 1. 安装 superpowers 插件（完整安装）
通过插件市场安装 `superpowers@claude-plugins-official`，提供 brainstorming、systematic-debugging、test-driven-development 等 13 个核心开发工作流技能。整包无冗余变体，适合完整安装。

### 2. 提取 planning-with-files-zh 单技能（避免冗余）
`planning-with-files` 完整版含 6 个语言变体（en/ar/de/es/zh/zht）。为避免引入 5 个用不到的语言变体污染技能列表，采用 Standalone Skill 方式，仅从插件缓存提取中文版：

```bash
cp -r ~/.claude/plugins/cache/planning-with-files/planning-with-files/2.43.0/skills/planning-with-files-zh \
      ~/.claude/skills/planning-with-files-zh
```

执行 `/reload-plugins` 后只有 `planning-with-files-zh` 被注册。

### 3. 新增正式文章
- 根目录新增 [plugin-skill-use.md](../../plugin-skill-use.md)：系统讲解插件 vs 独立技能的区别、安装方式、单技能提取与未来更新。
- `README.md` 文章目录新增该条目。

### 4. CLAUDE.md 新增变更记录规范
明确每次仓库改动必须在 `docs/changes/` 留记录、无需向用户确认，文件命名由 Claude 自行决定。

## 原因

- 插件提供整包能力且无冗余时，直接 `/plugin install`。
- 插件含大量用不到的变体（多语言、多平台）时，从缓存提取单个技能目录放入 `~/.claude/skills/`，这是 Claude Code 原生支持的独立技能路径。
