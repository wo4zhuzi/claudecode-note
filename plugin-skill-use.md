# 插件与技能的安装方式

Claude Code 支持两种扩展机制：**插件（Plugin）** 和 **独立技能（Standalone Skill）**。理解两者的区别，才能在「整包安装」和「按需提取」之间做出合适选择，避免技能列表被冗余内容污染。

## 插件 vs 独立技能

| 特性 | 插件（Plugin） | 独立技能（Skill） |
|------|--------------|----------------|
| 安装方式 | `/plugin install <name>@<source>` | 将技能目录放入 `~/.claude/skills/` |
| 管理方式 | `installed_plugins.json` 统一管理 | 目录即技能，无需配置文件 |
| 粒度 | 整包安装，包含所有技能 / hooks / scripts | 可按需选取单个技能 |
| 更新 | `/plugin update` 自动更新 | 手动覆盖目录 |
| 适用场景 | 需要完整插件功能时 | 只需插件中部分技能时 |

核心差异是**粒度**：插件是整包，技能是单元。当一个插件里大部分内容你都用得上时，直接装插件最省心；当插件塞了一堆你用不到的变体时，从中提取单个技能更干净。

## 案例一：superpowers —— 完整插件安装

superpowers 直接从插件市场安装：

```
/plugin install superpowers@claude-plugins-official
```

它提供 brainstorming、systematic-debugging、test-driven-development 等 13 个核心开发工作流技能，彼此配合作为整体使用，没有冗余变体。这类插件适合完整安装。

## 案例二：planning-with-files-zh —— 单技能提取

`planning-with-files` 是一个多语言规划插件，完整版包含 6 个语言变体：

- `planning-with-files`（英文主版本）
- `planning-with-files-ar`（阿拉伯文）
- `planning-with-files-de`（德文）
- `planning-with-files-es`（西班牙文）
- `planning-with-files-zh`（简体中文）
- `planning-with-files-zht`（繁体中文）

我只需要简体中文版。**安装完整插件会引入 5 个永远用不到的语言变体**，污染技能列表，也增加每次 `/reload-plugins` 的扫描负担。

### 解决方案：从缓存提取单个技能

插件完整包在安装后会缓存于 `~/.claude/plugins/cache/`，即使后续 `/plugin uninstall` 卸载了插件，缓存目录仍然保留。从缓存中提取单个技能目录，复制到 `~/.claude/skills/`，Claude Code 会将其识别为独立技能：

```bash
mkdir -p ~/.claude/skills

cp -r ~/.claude/plugins/cache/planning-with-files/planning-with-files/2.43.0/skills/planning-with-files-zh \
      ~/.claude/skills/planning-with-files-zh
```

执行 `/reload-plugins` 后，只有 `planning-with-files-zh` 出现在技能列表中，其余 5 个语言变体不会被注册。

### 操作流程小结

完整流程是「先装插件 → 缓存留存 → 卸载插件 → 从缓存提取单技能」：

```
/plugin install planning-with-files@planning-with-files   # 安装，缓存写入
/plugin uninstall planning-with-files@planning-with-files  # 卸载，缓存保留
cp -r .../cache/.../skills/planning-with-files-zh ~/.claude/skills/  # 提取单技能
/reload-plugins                                            # 生效
```

## 关于 hook 路径变量

独立技能模式下，SKILL.md 里 hook 命令使用的 `${CLAUDE_SKILL_DIR}` 会被正确注入为 `~/.claude/skills/<技能名>`，hooks 正常工作。

需要注意：部分 SKILL.md 正文里还有 `${CLAUDE_PLUGIN_ROOT}` 这类**插件专属变量**，它们在独立技能模式下可能为空。但这些通常只出现在「给 AI 看的示例文档」中，不是自动执行的 hook，不影响技能运行。提取技能后值得快速扫一眼，确认没有依赖插件专属变量的关键逻辑。

## 未来更新方式

独立技能不会随插件系统自动更新，升级需手动操作：

```bash
# 方式一：缓存还在时，直接覆盖
cp -r ~/.claude/plugins/cache/planning-with-files/planning-with-files/<新版本>/skills/planning-with-files-zh \
      ~/.claude/skills/planning-with-files-zh

# 方式二：缓存已清理，从源仓库重新获取
git clone --depth=1 https://github.com/OthmanAdi/planning-with-files /tmp/pwf-repo
cp -r /tmp/pwf-repo/skills/planning-with-files-zh ~/.claude/skills/planning-with-files-zh
rm -rf /tmp/pwf-repo
```

更新后执行 `/reload-plugins` 生效。

## 选择原则

- 插件提供整包能力且没有冗余变体时，直接 `/plugin install`。
- 插件包含大量用不到的变体（多语言、多平台配置等）时，从缓存提取单个技能目录放入 `~/.claude/skills/`。
- `~/.claude/skills/` 是 Claude Code 原生支持的独立技能路径，目录即技能，无需额外配置文件。
- 提取单技能前确认它不依赖 `${CLAUDE_PLUGIN_ROOT}` 等插件专属变量的关键逻辑。
