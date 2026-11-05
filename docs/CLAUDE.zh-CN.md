# CLAUDE.md — new-api 项目约定

> 本文档为 `CLAUDE.md` 的中文译本，用于阅读与上手。以仓库根目录 `CLAUDE.md` 原文为准。

## 强制要求：使用 Read 工具阅读 AGENTS.md

不要把 `@AGENTS.md` 当作已加载。Claude Code 不能可靠地内联该 import。

在任何规划、编码、评审或回答项目问题之前，你必须对仓库根目录的 `AGENTS.md` 调用 Read 工具，并等待完整内容返回。这是每个会话、每个新任务的第一个动作。

规则：

- 不要凭记忆、摘要或仅凭本文件开始工作。
- 不要因为上一轮提到过 AGENTS.md 就跳过 Read。
- 不要用 grep、glob 或部分略读替代 Read。
- 阅读后，在后续工作中遵守 `AGENTS.md` 中的每一条规则。
- 如果任务涉及 `web/`，在编辑前端文件前还要阅读 `web/AGENTS.md`。
- 如果任务涉及 `AGENTS.md` 中 **计费规则（强制阅读门槛）** 所定义的计费内容，在规划或编辑前还要完整阅读 `.agents/rules/billing.md`。不涉及该定义的任务可以跳过。

## 上手指南速查（由 codebase-onboarding 增补）

### 关键入口

| 关注点 | 路径 |
|--------|------|
| 进程启动、后台任务、内嵌 `web/dist` | `main.go` |
| 路由注册 | `router/main.go`（api、relay、task、video、plugin、web） |
| 中转（`/v1/*`） | `router/relay-router.go` → `controller.Relay` |
| 模型解析、渠道选择 | `middleware/distributor.go`、`service/channel_select.go` |
| 提供商适配器与工厂 | `relay/channel/<provider>/`、`relay/relay_adaptor.go` |
| 各协议处理器 | `relay/*_handler.go`（text、claude、gemini、image、audio 等） |
| 计费/额度 | `service/quota.go`、`service/text_quota.go`、`pkg/billingexpr/` |
| 控制台 API | `router/api-router.go`、`controller/` |
| 模型与迁移 | `model/`（`model/main.go` 中的 `AutoMigrate`） |
| JS 任务插件 | `plugins/tasks/<vendor>/`、运行时 `pkg/jsplugin/` |
| Web 控制台 | `web/src/routes/`（页面）、`web/src/features/`（模块） |

### 请求生命周期（`POST /v1/chat/completions`）

1. `middleware.TokenAuth` 认证 API 令牌。
2. `middleware.Distribute` 解析模型、应用渠道过滤、选择渠道，并将渠道配置写入 gin context。
3. `controller.Relay` 校验请求体（`helper.GetAndValidateRequest`），构建 `relaycommon.RelayInfo`，预扣额度（`relay.PrepareRequestBilling`）。
4. 协议处理器（`relay.TextHelper`、`relay.ClaudeHelper` 等）通过 `relay.GetAdaptor` 选择适配器并转换请求。
5. 适配器请求上游并流式回传；用量结算并记录日志；失败经 `service.DecideRelayRetry` 处理。

### 常用命令

- 后端：`go run main.go`（需要 `web/dist`），构建 `go build ./...`
- 测试：`make test`（根模块 + `relaykit` + `tokenkit`，`GOWORK=off`）
- 前端：`cd web && bun install && bun run dev` / `bun run build` / `bun check -b` / `bun run test`
- 开发栈：`make dev`（Docker Postgres + Redis，前端 5173）

> 更完整的中文上手指南见 [onboarding.zh-CN.md](./onboarding.zh-CN.md)。
