# CLAUDE.md — Project Conventions for new-api

## MANDATORY: Read AGENTS.md with the Read tool

Do not treat `@AGENTS.md` as loaded. Claude Code does not reliably inline that import.

Before any planning, coding, reviewing, or answering a project question, you MUST call the Read tool on the repo-root file `AGENTS.md` and wait for the full contents. This is the first action of every session and every new task.

Rules:

- Do not start from memory, summaries, or this file alone.
- Do not skip the Read because a previous turn mentioned AGENTS.md.
- Do not replace the Read with a grep, glob, or partial skim.
- After reading, follow every rule in `AGENTS.md` for the rest of the work.
- If the task touches `web/`, also Read `web/AGENTS.md` before editing frontend files.
- If the task touches billing as defined under **Billing rules (mandatory read gate)** in `AGENTS.md`, also Read `.agents/rules/billing.md` in full before planning or editing. Tasks outside that definition may skip it.

## Onboarding Quick Reference (added by codebase-onboarding)

### Entry points

| Concern | Path |
|---------|------|
| Process start, background jobs, embedded `web/dist` | `main.go` |
| Route registration | `router/main.go` (api, relay, task, video, plugin, web) |
| Relay (`/v1/*`) | `router/relay-router.go` → `controller.Relay` |
| Model parsing, channel selection | `middleware/distributor.go`, `service/channel_select.go` |
| Provider adaptors + factory | `relay/channel/<provider>/`, `relay/relay_adaptor.go` |
| Per-format handlers | `relay/*_handler.go` (text, claude, gemini, image, audio, ...) |
| Billing / quota | `service/quota.go`, `service/text_quota.go`, `pkg/billingexpr/` |
| Console API | `router/api-router.go`, `controller/` |
| Models, migrations | `model/` (`AutoMigrate` in `model/main.go`) |
| JS task plugins | `plugins/tasks/<vendor>/`, runtime `pkg/jsplugin/` |
| Web console | `web/src/routes/` (pages), `web/src/features/` (modules) |

### Request lifecycle (`POST /v1/chat/completions`)

1. `middleware.TokenAuth` authenticates the API token.
2. `middleware.Distribute` parses the model, applies channel filters, selects a channel, and stores channel config in the gin context.
3. `controller.Relay` validates the body (`helper.GetAndValidateRequest`), builds `relaycommon.RelayInfo`, pre-consumes quota (`relay.PrepareRequestBilling`).
4. A format handler (`relay.TextHelper`, `relay.ClaudeHelper`, ...) picks the adaptor via `relay.GetAdaptor` and converts the request.
5. The adaptor calls upstream and streams the response; usage is settled and logged; failures go through `service.DecideRelayRetry`.

### Commands

- Backend: `go run main.go` (needs `web/dist`), build `go build ./...`
- Tests: `make test` (root + `relaykit` + `tokenkit`, `GOWORK=off`)
- Frontend: `cd web && bun install && bun run dev` / `bun run build` / `bun check -b` / `bun run test`
- Dev stack: `make dev` (Docker Postgres + Redis, web on 5173)
