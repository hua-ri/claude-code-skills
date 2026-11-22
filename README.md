# claude-code-skills

Claude Code Skills 集合 — 可复用的开发技能，让 AI 助手更高效地完成特定领域任务。

## 什么是 Claude Code Skills

Claude Code Skills 是一组结构化的指令文件，当你在 Claude Code 中开发特定类型的项目时，可以加载对应的 skill 来获得领域专家级别的辅助。

## 包含的 Skills

| Skill | 描述 |
|---|---|
| `dsh-plugin-dev` | DSH (DeepSeek Harness) 插件从零开发到发布的完整指南。涵盖插件骨架、Tool/Command/WebAPI 注册、Web UI 面板、CLI 封装、npm 发布、GitHub Actions CI/CD、插件市场上架。 |

## 使用方式

### 方式 1：克隆到项目中

```bash
# 克隆仓库
git clone https://github.com/hua-ri/claude-code-skills.git

# 复制 skills 到你的项目
cp -r claude-code-skills/.claude/skills/* your-project/.claude/skills/
```

### 方式 2：符号链接

```bash
git clone https://github.com/hua-ri/claude-code-skills.git ~/claude-code-skills

# 在你的项目中创建符号链接
mkdir -p your-project/.claude/skills
ln -s ~/claude-code-skills/.claude/skills/* your-project/.claude/skills/
```

### 方式 3：直接在此仓库中工作

如果你在此仓库目录下使用 Claude Code，所有 skills 自动可用。

## Skill 文件格式

每个 skill 是一个 Markdown 文件，包含 YAML frontmatter：

```markdown
---
name: skill-name
description: 简要描述，用于决定何时加载此 skill
---

# Skill 内容

详细的指令和参考信息...
```

## 贡献

欢迎提交 PR 添加新的 skills。请确保：

1. 每个 skill 基于真实项目的开发经验
2. 包含可复用的代码模板和踩坑记录
3. 遵循 frontmatter 格式规范

## License

MIT
