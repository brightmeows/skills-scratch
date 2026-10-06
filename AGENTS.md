# skills-scratch — Agent Guide

## 仓库定位

临时技能仓库:实验性、未定型的技能先进这里,成熟后移入正式仓库(workflow-skills、rust-meta-skills 等)并从本仓库删除。技能本体即 `skills/<name>/SKILL.md` 单文件。

| 技能 | 用途 |
|------|------|
| `codeberg-github-migration` | Codeberg 仓库迁移为 GitHub 主仓 + Codeberg 镜像 |
| `git-hash-repo-conversion` | Git 仓库 sha256/sha1 对象格式转换 |

### 本仓库特有

- 技能移入正式仓库时:在正式仓库补齐技能目录与 index.json 条目,从本仓库删除技能目录并同步 index.json 与 marketplace.json

## 边界

### Always

- 修改 `.md` 后通过 `pre-commit run markdownlint` 验证格式
- 修改任何 `SKILL.md` 后,重算 sha256 并同步 `.well-known/agent-skills/index.json` 的 `digest` 与 `description`(三处必须逐字一致:frontmatter、index.json、实际文件)
- 新增/移除技能目录时,同步更新 `.well-known/agent-skills/index.json` 与 `.claude-plugin/marketplace.json` 的 `skills`
- 提交信息使用 Conventional Commits;每完成一个逻辑单元立即原子提交
- 提交前执行 `git status` + `git diff` 确认只包含预期变更

### Ask

- 需修改 `.markdownlint.toml` 基线或 `.pre-commit-config.yaml` 钩子逻辑时先确认
- 需要新增、移除或拆分技能时先确认

### Never

- 勿启用 `.markdownlint.toml` 中禁用的规则(注释已说明原因)
- 勿在未更新 digest 的情况下提交 `SKILL.md`(`check-well-known-digest` 会拦截)
- 勿在 SKILL.md 中写入未经验证的事实;页面结构、命令用法一类事实改动后必须实机复核

## 命令

```bash
# 全量校验(提交前必跑)
pre-commit run --all-files

# 单独任务
pre-commit run markdownlint
pre-commit run check-well-known-list        # skills/ 目录与 index.json 双向一致
pre-commit run check-well-known-digest      # index.json digest 与 SKILL.md 匹配
pre-commit run check-skill-name-consistency # frontmatter name/description 与 index.json 一致
pre-commit run check-plugin-skills-list     # marketplace.json skills 与技能目录一致
pre-commit run check-skill-md-format        # frontmatter 字段格式与 body 行数合规

# 修改 SKILL.md 后更新 digest
sha256sum skills/<name>/SKILL.md
```

## SKILL.md 编写约定

- frontmatter `name` 与父目录名一致,小写连字符
- `description` 第一句为中文功能描述,第二句为触发条件;总长不超过 1024 字符;关键词格式 `Keywords:`,中文在前英文在后
- 正文少于 500 行;事实类内容须在改动时重新验证,不依赖记忆
- 中文文本使用全角引号,技能名用反引号包裹

## 维护指南

按「起步、观察、补充、精简、重复」的增量迭代维护本文件:

- 代理反复忽略某条规则 → 补充到对应边界节
- 代理已能稳定遵循某条规则 → 从边界节移除(已内化)
- 规则可被 hook 或 lint 强制 → 迁移到 `.pre-commit-config.yaml`,指向工具配置
- 边界节膨胀超过 10 条 → 审计精简,工具可强制的移出
