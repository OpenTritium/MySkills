# MySkills

面向 Codex 和其他 Agent 的开发 skills，包含：

- **Go**：代码质量、并发、数据库、依赖注入、GraphQL、gRPC、OpenAPI、测试、性能和可观测性
- **Rust**：架构、异步并发、错误处理、类型设计、安全、测试和性能审查
- **HTTP/API**：公共 HTTP/JSON API 设计
- **版本控制**：Git、Jujutsu 工作流

## 使用 Bun 安装

当前系统中的 skills 目录分布如下：

- 通用项目目录：`./.agents/skills`
- 通用全局目录：`~/.agents/skills`
- Codex 全局目录：`~/.codex/skills`

安装到当前项目的 `.agents/skills`：

```bash
bunx skills add OpenTritium/MySkills \
  --agent universal \
  --skill '*' \
  --full-depth \
  --copy \
  --yes
```

安装到通用全局目录 `~/.agents/skills`：

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

安装到 Codex 全局目录 `~/.codex/skills`：

```bash
bunx skills add OpenTritium/MySkills \
  --global \
  --agent codex \
  --skill '*' \
  --full-depth \
  --copy \
  --yes
```

只安装单个 skill，例如安装 `golang-graphql` 到 Codex：

```bash
bunx skills add OpenTritium/MySkills \
  --global \
  --agent codex \
  --skill golang-graphql \
  --full-depth \
  --copy \
  --yes
```

项目级安装时去掉 `--global`；项目级 `universal` 和 `codex` 都写入 `./.agents/skills`，全局安装则分别写入各自的全局目录。
