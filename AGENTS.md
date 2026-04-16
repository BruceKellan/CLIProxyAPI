<!-- Generated: 2026-04-16 11:16:04 +0800 | Updated: 2026-04-16 11:25:00 +0800 -->

# CLIProxyAPI

## Purpose
CLIProxyAPI 是一个 Go 1.26+ 代理服务，将多个基于 CLI 的 AI 提供商能力封装为 OpenAI、Gemini、Claude、Codex 兼容 API。仓库包含服务入口、内部运行时与协议转换层、可嵌入 SDK、集成测试，以及围绕 OAuth、模型路由、热重载和多种持久化后端的运维与参考资料。

## Key Files

| File | Description |
|------|-------------|
| `CLAUDE.md` | 项目级 AI 上下文文档，包含架构总览、模块索引与仓库特定约束。 |
| `README.md` | 英文主说明文档，覆盖安装、配置与使用方式。 |
| `README_CN.md` | 中文使用说明。 |
| `README_JA.md` | 日文使用说明。 |
| `go.mod` | Go 模块定义与主要外部依赖。 |
| `go.sum` | Go 依赖校验和。 |
| `config.example.yaml` | 规范配置模板；修改配置结构时需要同步更新。 |
| `.env.example` | 环境变量示例。 |
| `Dockerfile` | 服务容器镜像构建入口。 |
| `docker-compose.yml` | 本地多服务运行示例。 |
| `.goreleaser.yml` | 发布与打包配置。 |

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| `.github/` | GitHub Actions 与仓库自动化配置。 |
| `ai_docs/` | AI 面向的架构与模块文档。 |
| `assets/` | 静态资源。 |
| `auths/` | 仓库内保留的认证相关目录；运行时默认认证状态目录以配置和 `cfg.AuthDir` 为准。 |
| `cmd/` | 可执行入口，主要包括 `cmd/server`。 |
| `docs/` | 额外项目文档与参考资料。 |
| `examples/` | 示例与集成样例。 |
| `internal/` | 内部实现：API、认证、执行器、翻译器、模型注册、热重载、存储、TUI 等。 |
| `sdk/` | 对外公开的可嵌入 SDK。 |
| `test/` | 跨模块集成测试。 |

## For AI Agents

### Working In This Directory
- 这是根目录导航文件；进入具体实现前，先读根目录 `CLAUDE.md`，再下钻到相关目录。
- 保持手术式修改；不要顺手做与任务无关的全仓整理。
- 修改配置、运行方式或仓库入口信息时，同步核对 `config.example.yaml`、`README*.md` 与 `CLAUDE.md` 是否仍一致。
- Go 代码遵循 `gofmt` / `goimports`；优先返回带上下文的错误，不要在请求路径使用 `log.Fatal` 或 `panic`。
- 使用 `logrus` 结构化日志，禁止记录 token、cookie 或其他敏感凭据。

### Testing Requirements
- 小范围变更运行受影响包测试；影响面不清晰时运行 `go test ./...`。
- 修改并发、热重载、共享状态相关逻辑时，补跑 `go test -race ./...`。
- 提交前对变更过的 Go 文件执行 `gofmt -w`。

### Common Patterns
- `cmd/` 提供进程入口；`internal/` 保存内部实现；`sdk/` 提供可复用公开接口。
- 新增功能常会联动配置、认证、执行器、协议转换、模型注册或管理 API 等多个层面。
- 配置以 YAML 为中心；持久化后端支持文件、PostgreSQL、Git 与对象存储。

## Dependencies

### Internal
- `cmd/server` 依赖 `internal/cmd`、`internal/api`、`sdk/cliproxy` 以及配置/认证装配逻辑。
- `sdk/` 基于 `internal/` 能力封装可复用的公开接口。
- `test/` 负责跨 `internal/` 与 `sdk/` 边界的集成验证。

### External
- `github.com/gin-gonic/gin` — HTTP 路由与中间件。
- `github.com/sirupsen/logrus` — 结构化日志。
- `github.com/fsnotify/fsnotify` — 配置与认证目录热重载监听。
- `github.com/go-git/go-git/v6` — Git 持久化后端支持。
- `github.com/jackc/pgx/v5` — PostgreSQL 后端支持。
- `github.com/minio/minio-go/v7` — 对象存储后端支持。
- `github.com/gorilla/websocket` — WebSocket 传输与中继。
- `github.com/charmbracelet/bubbletea` / `bubbles` / `lipgloss` — TUI 界面。

<!-- MANUAL: Add project-specific notes below this line if needed. -->
