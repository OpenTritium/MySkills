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

## 技能索引

本项目按领域维护 skills；目录名是安装路径，frontmatter 中的 `name` 是
触发名称。`.NET 10` 子集精选自 [dotnet/skills](https://github.com/dotnet/skills)，
仅保留可独立使用且与本项目现有职责边界清晰的能力。

### .NET

- `dotnet-webapi`：ASP.NET Core 10 endpoint、DTO、OpenAPI 和错误处理
- `run-tests`：构造正确的 .NET 10 TUnit 运行和过滤命令
- `writing-tunit-tests`：编写可靠的 .NET 10 异步 TUnit 测试与断言
- `dotnet-aot-compat`：处理 .NET 10 trimming 和 Native AOT 分析警告
- `msbuild-antipatterns`：审查 .NET 10 MSBuild 项目文件反模式
- `analyzing-dotnet-performance`：基于证据分析 .NET 10 性能风险

### Go

`golang-benchmark`、`golang-cli`、`golang-code-style`、`golang-concurrency`、
`golang-context`、`golang-continuous-integration`、`golang-data-structures`、
`golang-database`、`golang-dependency-injection`、`golang-dependency-management`、
`golang-design-patterns`、`golang-documentation`、`golang-error-handling`、
`golang-gopls`、`golang-graphql`、`golang-grpc`、`golang-lint`、`golang-modernize`、
`golang-naming`、`golang-observability`、`golang-openapi`、`golang-performance`、
`golang-project-layout`、`golang-refactoring`、`golang-safety`、`golang-security`、
`golang-structs-interfaces`、`golang-testing`、`golang-troubleshooting`

### Rust

`rust-api-consolidation`、`rust-architecture-entropy-review`、`rust-async-concurrency`、
`rust-big-o-optimizer`、`rust-concurrency-testing`、`rust-ecosystem`、
`rust-encode-invariant`、`rust-error-silence`、`rust-func-smell`、`rust-guard-clauses`、
`rust-high-snr-comment`、`rust-import-hygiene`、`rust-logging-review`、
`rust-method-placement`、`rust-naming-smell`、`rust-resource-lifecycle`、`rust-snafu`、
`rust-state-machine`、`rust-structure-refactor`、`rust-testing-strategy`、
`rust-tracing-context`、`rust-unsafe-checker`、`rust-zero-alloc`

### HTTP、Jujutsu 和通用

- `stripe-api-design`：公共 HTTP/JSON API 和 webhook wire contract
- `vcs-naming`：Git commit、branch、jj bookmark 和 workspace 的命名规范
- `jujutsu`、`jujutsu-parallel`、`vcs-router`：版本控制后端与 Jujutsu 工作流
