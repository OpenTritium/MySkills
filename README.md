# MySkills

面向 Codex 和其他 Agent 的开发 skills，包含 Go、Rust、HTTP/API 和 Git/Jujutsu。

## 安装

当前系统中的 skills 目录：

- 通用全局目录：`~/.agents/skills`
- Codex 全局目录：`~/.codex/skills`

使用 Bun 安装到通用全局目录 `~/.agents/skills`：

> skills CLI 当前将 `cline` 的全局路径映射到 `~/.agents/skills`；这里使用 `cline` 只是为了写入通用目录。

```bash
bunx skills add OpenTritium/MySkills \
  --global \
  --agent cline \
  --skill '*' \
  --full-depth \
  --copy \
  --yes
```

`skills` CLI 的 `--agent codex` 在当前系统仍写入 `~/.agents/skills`，不会写入 Codex 专用目录。安装到 `~/.codex/skills` 请使用 Codex 自带安装器；在本仓库根目录执行以下命令可安装全部 Go skills：

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo OpenTritium/MySkills \
  --ref master \
  --path go/golang-* \
  --dest ~/.codex/skills
```

只安装单个 skill，例如使用 Bun 安装 `golang-graphql` 到通用全局目录：

```bash
bunx skills add OpenTritium/MySkills \
  --global \
  --agent cline \
  --skill golang-graphql \
  --full-depth \
  --copy \
  --yes
```
