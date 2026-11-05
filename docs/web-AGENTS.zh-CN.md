# 前端开发规范（入口与摘要）

> **原文本身即为中文**，无需翻译。权威版本是 [`web/AGENTS.md`](../web/AGENTS.md)，包含完整条款、组件检索入口与更新日志；本文档只提供摘要与导航，两者不一致时以原文为准。

## 技术栈

| 类别 | 技术 |
|------|------|
| 包管理 | Bun |
| 框架 | React 19、TypeScript |
| 数据与请求 | @tanstack/react-query、axios、Zustand |
| 路由 | @tanstack/react-router（`src/routes/`） |
| 表格与列表 | @tanstack/react-table、@tanstack/react-virtual |
| 国际化 | i18next、react-i18next |
| UI 与样式 | Base UI、Hugeicons、Tailwind CSS、clsx / class-variance-authority |
| 表单 | React Hook Form、Zod |
| 图表 | @visactor/vchart |
| 工具 | oxfmt、oxlint、vitest |

## 必须遵守的要点

- **国际化**：所有用户可见文案通过 `useTranslation()` 的 `t('英文 key')` 使用；新增 key 需补齐 `src/i18n/locales/` 下的全部语言文件。常量中的消息只存 i18n key，展示时必须再经 `t()`。
- **数字与 locale**：普通数字用 `@/lib/format`，金额用 `@/lib/currency`；传给 `Intl.*`、`toLocaleString` 等之前必须经 `@/i18n/languages` 的 `toIntlLocale()`。`zhCN`/`zhTW` 不是合法 BCP 47 标签。
- **组件复用（强制）**：先检索 `src/components/` 与相关 `src/features/`，再评估组合或兼容扩展，最后才新增。已有业务封装能覆盖时必须使用，例如确认用 `@/components/confirm-dialog`、复制用 `@/components/copy-button`、列表用 `@/components/data-table`。
- **类型与检查**：避免 `any`，显式标注参数与返回值，仅类型导入用 `import type`。改动 TS/TSX 后必须执行 `bun check -b`；提交前对涉及文件执行 lint 并清零 error。
- **代码风格**：禁止两层及以上嵌套三元；对象（尤其 props）非必要不解构；单文件超过约 200 行考虑拆分。
- **数据请求**：`useQuery` / `useMutation` 配唯一 `queryKey`；错误统一走 `handleServerError`，业务失败保留原始响应契约。
- **路由**：TanStack Router 文件路由，`createFileRoute` 定义，搜索参数用 Zod `validateSearch` 校验，认证与重定向放在 `beforeLoad`。
- **测试**：Vitest + React Testing Library；测试放在模块专属 `__tests__/` 目录，按职责命名（`layout.test.ts`、`validation.test.ts` 等）；禁止 sleep、实现细节断言与仅为覆盖率的测试；UI 行为变更必须补回归测试。

## 目录约定

| 路径 | 用途 |
|------|------|
| `web/src/routes/` | 文件路由页面（`_authenticated/` 为登录后区域） |
| `web/src/features/<feature>/` | 业务模块（`components/`、`lib/`、`hooks/`、`api.ts`、`types.ts`、`constants.ts`） |
| `web/src/components/` | 通用组件（`ui/` 为基础控件） |
| `web/src/hooks/`、`web/src/lib/` | 通用 hooks 与工具函数 |
| `web/src/stores/` | Zustand store |
| `web/src/styles/` | 全局样式与主题 |

## 常用命令

```bash
cd web
bun install
bun run dev          # Rsbuild 开发服务器
bun run build        # 生产构建
bun check -b         # 类型检查
bun run lint         # oxlint
bun run test         # vitest
bun run i18n:sync    # 同步语言文件
```

完整规范（含 3.1–3.16 全部条款、组件检索表与更新日志）见 [`web/AGENTS.md`](../web/AGENTS.md)。
