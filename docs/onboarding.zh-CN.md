# new-api 项目上手指南

> 本文档由代码库调研生成，用于快速建立对仓库的整体认识。
> 配套中文文档：
> - [AGENTS.zh-CN.md](./AGENTS.zh-CN.md)：项目规范（架构、后端/前端规则、治理约定）
> - [CLAUDE.zh-CN.md](./CLAUDE.zh-CN.md)：AI 助手会话的强制阅读约定
> - [web-AGENTS.zh-CN.md](./web-AGENTS.zh-CN.md)：前端开发规范入口（原文本身为中文）

## 项目概览

new-api 是一个用 Go 编写的 AI API 网关/代理。它把 40 多家上游 AI 提供商（OpenAI、Claude、Gemini、Azure、Bedrock 等）聚合到一套 OpenAI / Claude / Gemini 兼容接口之后，并提供用户体系、令牌、计费、限流、任务插件以及 React 管理后台。

## 技术栈

| 层次 | 技术 |
|------|------|
| 后端 | Go 1.26，Gin，GORM v2 |
| 独立模块 | `relaykit/`（Go 1.25.1，协议 DTO 与转换）、`tokenkit/`（文本/图片 token 计数） |
| 前端 | React 19、TypeScript、Rsbuild 2、TanStack Router/Query/Table、Zustand、Tailwind CSS 4、Bun |
| 数据库 | SQLite / MySQL ≥5.7.8 / PostgreSQL ≥9.6；日志库另支持 ClickHouse |
| 缓存 | Redis + 内存缓存（`common/redis.go`、`pkg/cachex/`） |
| 认证 | 会话、API 令牌、JWT、Passkey/WebAuthn、TOTP、OAuth/OIDC、Casbin（`service/authz/`） |
| 扩展 | JavaScript 任务插件（`plugins/tasks/`，运行时 `pkg/jsplugin/`）、Electron 封装 |
| CI | GitHub Actions：`go vet/build/test`、`bun check -b`、`vitest` |

## 架构

```
浏览器 ──▶ web/（React SPA，构建产物 web/dist 由 main.go 内嵌）
客户端 ──▶ Gin
            ├─ /api/*      → api-router    → controller/（控制台、认证、管理）
            ├─ /v1/*       → relay-router   → controller.Relay → relay/
            ├─ /v1/videos、/mj、任务、插件路由
            └─ 其余路径    → web-router（SPA）或按 FRONTEND_BASE_URL 重定向

relay/：按协议划分的处理器（text/claude/gemini/image/audio/...）
        └─ relay.GetAdaptor(api_type) → relay/channel/<provider>/（每家上游一个适配器）
        └─ relaykit/（DTO 与协议转换）  tokenkit/（token 计数）
service/：渠道选择、额度/计费、重试、任务
model/：GORM 模型、AutoMigrate、缓存
```

关键约束：`relaykit/` 与 `tokenkit/` 必须可独立构建（`GOWORK=off go build ./...`），不得反向依赖根模块。

## 关键入口

| 关注点 | 位置 |
|--------|------|
| 进程启动、后台任务、内嵌 SPA | `main.go` |
| 路由装配 | `router/main.go` |
| 中转路由（`/v1/*`、`/v1beta/*`、`/mj/*`） | `router/relay-router.go` |
| 控制台/管理路由 | `router/api-router.go` |
| 中转入口、重试循环、失败退款 | `controller/relay.go` |
| 模型解析与渠道分发 | `middleware/distributor.go` |
| 渠道选择算法 | `service/channel_select.go` |
| 适配器工厂 | `relay/relay_adaptor.go` |
| 文本中转主流程 | `relay/compatible_handler.go` |
| 计费/额度 | `service/quota.go`、`service/text_quota.go`、`pkg/billingexpr/` |
| 各上游适配器 | `relay/channel/`（约 40 个厂商目录） |
| 数据模型与迁移 | `model/main.go` |
| 后端/前端 i18n | `i18n/`、`web/src/i18n/locales/*.json` |

## 请求生命周期（`POST /v1/chat/completions`）

1. `middleware.TokenAuth` 解析 API 令牌，并加载其分组与模型限制。
2. `middleware.Distribute` 解析 `model`（JSON/表单/multipart），应用渠道过滤，调用 `service.SelectChannelForRequest`，随后把渠道 ID、类型、密钥、Base URL、覆盖配置写入 gin context。
3. `controller.Relay` 校验请求体，构建 `relaycommon.RelayInfo`，预扣额度，并进入重试循环（`common.RetryTimes`）。
4. 对应协议的处理器（`TextHelper`、`ClaudeHelper`、`GeminiHelper` 等）完成模型映射，通过 `relay.GetAdaptor(apiType)` 取得适配器并转换请求。
5. 适配器请求上游并回传（含流式）；用量结算写入消费日志；失败经 `service.DecideRelayRetry` 判断重试，并触发 `RefundFailedRequestBilling` 退款。

## 目录地图

| 路径 | 用途 |
|------|------|
| `common/` | 环境变量、DB/Redis 客户端、JSON 封装、额度计算、缓存、加密 |
| `constant/` | 渠道类型、API 类型、context key |
| `middleware/` | 认证、分发、限流、CORS、请求体限制、审计 |
| `controller/` | HTTP 处理器（中转 + 控制台） |
| `relay/` | 协议处理器、helper、各上游适配器 |
| `service/` | 渠道选择、计费、任务、授权、HTTP 传输 |
| `model/` | GORM 模型、迁移、渠道/用户缓存 |
| `setting/` | 基于 option 的运行时配置（倍率、计费、运营） |
| `plugins/tasks/<vendor>/` | JS 任务插件（视频/音乐/图片），随二进制内嵌 |
| `dto/`、`relaykit/types` | 请求/响应 DTO |
| `web/` | React 控制台（`routes/` 页面、`features/` 模块、`components/`） |
| `docs/openapi/`、`docs/plugin-api/` | HTTP 与插件 API 契约 |

## 编码约定（摘要）

- **文件命名**：Go 用 `snake_case.go`，测试与被测文件同目录、以 `_test.go` 结尾；React 组件用 `PascalCase.tsx`，工具函数用 `kebab-case.ts`。
- **JSON**：禁止直接使用 `encoding/json`，统一走 `common.Marshal/Unmarshal/DecodeJson`；`relaykit/` 内使用 `kitutil.*`。
- **数据库**：所有代码必须同时兼容 SQLite/MySQL/PostgreSQL；行锁用 `lockForUpdate(tx)`，保留字列用 `commonGroupCol`/`commonKeyCol`，禁止无兼容回退的方言专有 SQL。
- **中转 DTO**：可选标量字段用指针 + `omitempty`，确保客户端显式传入的 `0`/`false` 不会被丢弃。
- **测试**：使用 `testify require/assert`；小改动优先扩展已有测试文件；禁止只提覆盖率或只证明“能跑”的测试。
- **前端**：先检索并复用 `web/src/components/` 中的组件；所有用户可见文案通过 `t('英文 key')`，并补齐 7 个语言文件；任何 `Intl.*` 调用前先经 `toIntlLocale()`。
- **文档**：除非明确要求，不得在 `docs/` 或插件目录下新增文档文件。
- **Git**：提交信息采用约定式前缀（`feat(plugins):`、`fix(xai):`、`chore(deps):`），开发分支为 `main`。

## 常用命令

```bash
go run main.go                 # 启动后端（需要 web/dist；CI 会创建占位文件）
go build ./... ; go vet ./...
make test                      # 根模块 + relaykit + tokenkit，GOWORK=off
make dev                       # Docker 启动 Postgres/Redis + 前端开发服务器 :5173
cd web && bun install && bun run dev|build
cd web && bun check -b && bun run lint && bun run test
docker compose up -d           # 接近生产形态的栈，应用端口 3000
```

## 按任务查找

| 我想…… | 去哪里看 |
|---------|----------|
| 新增一个上游提供商 | `relay/channel/<provider>/`，在 `relay/relay_adaptor.go` 注册，并补充 `constant/api_type.go` |
| 新增/修改中转接口 | `router/relay-router.go` → `controller.Relay`，以及相应的 `relay/*_handler.go` |
| 调整渠道选择逻辑 | `middleware/distributor.go`、`service/channel_select.go`、`dto/channel_constraints.go` |
| 修改价格/计费 | `pkg/billingexpr/`、`service/quota.go`（先完整阅读 `.agents/rules/billing.md`） |
| 新增数据库字段/表 | `model/*.go` + `model/main.go` 的 AutoMigrate（需验证三种数据库） |
| 新增 JS 任务插件 | `plugins/tasks/<vendor>/`，并阅读 `docs/plugin-api/v1.md` |
| 新增控制台页面 | `web/src/routes/_authenticated/...` + `web/src/features/<feature>/` |
| 新增管理端 API | `router/api-router.go`、`controller/`、`web/src/features/system-settings/` |

## 备注

- 项目规范的权威来源仍是根目录 `AGENTS.md`、`web/AGENTS.md`，计费相关任务还需阅读 `.agents/rules/billing.md`；本文只做导航，不替代上述规则。
- `relaykit/`、`tokenkit/` 与根模块是三个独立 Go 模块，改动后需分别执行 `GOWORK=off go build ./...`。
