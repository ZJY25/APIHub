# new-api 本地运行手册

> 适用环境：Windows + PowerShell。
> 本文命令均在本机实际验证通过（Go 1.27.3、Bun 1.4.4、默认 SQLite 数据库）。

## 一、前置条件

| 依赖 | 要求 | 用途 |
| --- | --- | --- |
| Go | 1.26 及以上（`go.mod` 声明 `go 1.26.10`） | 编译并运行后端 |
| Bun | 1.4 及以上 | 安装前端依赖、构建前端 |
| Git | 任意较新版本 | 拉取代码 |
| MySQL / PostgreSQL / Redis | 可选 | 仅在需要替代 SQLite 或启用共享缓存时使用 |

安装示例（Windows）：

```powershell
winget install --id GoLang.Go -e --accept-package-agreements --accept-source-agreements
winget install --id Oven-sh.Bun -e --accept-package-agreements --accept-source-agreements
```

若 Bun 的 winget 安装长时间卡在 GitHub 下载，可改用 npm 安装（需先装 Node.js）：

```powershell
npm install -g --allow-scripts=bun bun
```

安装后新开终端验证：

```powershell
go version
bun --version
```

## 二、首次启动

后端使用 `//go:embed web/dist` 内嵌前端产物，因此**必须先构建前端，再启动后端**。

```powershell
# 1. 安装前端依赖（仓库根目录执行）
cd web
bun install --frozen-lockfile
bun run build
cd ..

# 2. 编译并启动后端
go run .
```

启动后：

- 服务地址：<http://localhost:3000>
- 数据库：未配置 `SQL_DSN` 时自动使用 SQLite，文件为仓库根目录下的 `one-api.db`（含 `-wal`、`-shm` 附属文件）
- 日志目录：仓库根目录下的 `logs/`
- 首次访问会进入初始化向导，用于创建管理员账号；`/api/setup` 返回 `status: false` 表示尚未初始化

国内网络若拉取 Go 依赖超时（`proxy.golang.org` 不可达），先切换代理再重试：

```powershell
go env -w GOPROXY=https://goproxy.cn,direct
```

前端依赖下载慢时可临时指定镜像源（不改动仓库内的 `.npmrc`）：

```powershell
bun install --frozen-lockfile --registry https://registry.npmmirror.com
```

## 三、验证是否启动成功

```powershell
# 健康检查，返回 success: true
(Invoke-WebRequest http://localhost:3000/api/status -UseBasicParsing).Content

# 初始化状态，新库返回 {"status":false,"root_init":false,"database_type":"sqlite"}
(Invoke-WebRequest http://localhost:3000/api/setup -UseBasicParsing).Content

# 首页应返回内嵌的前端 HTML
(Invoke-WebRequest http://localhost:3000/ -UseBasicParsing).StatusCode
```

也可以在浏览器直接打开 <http://localhost:3000>，能正常看到控制台或初始化向导页面即表示前后端都已就绪。

## 四、前端开发模式（热更新）

前后端分开运行时，修改前端代码无需重新构建、也无需重启后端：

```powershell
# 终端 A：后端
go run .

# 终端 B：前端开发服务器
cd web
bun run dev -- --port 5173
```

打开 <http://localhost:5173>。该服务器会把 `/api`、`/v1`、`/mj`、`/pg` 请求代理到 `http://localhost:3000`（目标地址可通过 `VITE_REACT_APP_SERVER_URL` 修改）。

注意：修改 Go 代码后需要重启 `go run .`；修改前端代码只需保持 `bun run dev` 运行。

## 五、常用环境变量

可在仓库根目录新建 `.env` 文件（启动时会自动加载），或直接用 PowerShell 临时设置：`$env:SQL_DSN="..."`。

| 变量 | 说明 |
| --- | --- |
| `PORT` | 监听端口，默认 `3000`；也可用 `go run . --port 8080` |
| `SQL_DSN` | 主数据库连接串；留空使用 SQLite |
| `SQLITE_PATH` | SQLite 文件路径，默认 `one-api.db` |
| `LOG_SQL_DSN` | 独立日志库连接串，另支持 ClickHouse |
| `REDIS_CONN_STRING` | Redis 连接串；留空则不启用 Redis |
| `MEMORY_CACHE_ENABLED` | 设为 `true` 启用进程内内存缓存 |
| `SESSION_SECRET` | 认证密钥；多节点部署必须设置且一致 |
| `DEFAULT_LANGUAGE` | 日志与默认返回语言：`en`、`zh-CN`、`zh-TW` |
| `DEBUG` | 设为 `true` 输出调试日志 |
| `GIN_MODE` | 设为 `debug` 进入 Gin 调试模式 |

全部可配置项见仓库根目录的 `.env.example`。

## 六、改用 MySQL / PostgreSQL / Redis

`SQL_DSN` 按前缀决定数据库类型（见 `model/main.go` 的 `chooseDB`）：

```powershell
# PostgreSQL
$env:SQL_DSN = "postgresql://user:password@127.0.0.1:5432/new-api"

# MySQL（必须包含 parseTime=true）
$env:SQL_DSN = "user:password@tcp(127.0.0.1:3306)/new-api?parseTime=true"

go run .

# 可选：启用 Redis（本地单机通常不需要）
$env:REDIS_CONN_STRING = "redis://:password@127.0.0.1:6379/0"
```

切换数据库后首次启动会自动建表迁移。Redis 未配置时系统仍可正常运行，只是限流与缓存由单进程承担，多节点部署才需要共享 Redis。

## 七、常见问题

| 现象 | 处理方式 |
| --- | --- |
| `go` 或 `bun` 不是内部或外部命令 | 安装后重新打开终端，或手动把安装目录加入 `PATH` |
| `go mod download` 超时 | 执行 `go env -w GOPROXY=https://goproxy.cn,direct` |
| `bun install` 很慢 | 追加 `--registry https://registry.npmmirror.com` |
| 启动报 `pattern web/dist: no matching files found` | 忘记执行 `cd web && bun run build` |
| 端口被占用 | 用 `go run . --port 8080`，或 `netstat -ano \| findstr :3000` 查占用进程 |
| 想重新走初始化向导 | 停止服务，删除 `one-api.db`、`one-api.db-wal`、`one-api.db-shm`（PostgreSQL/MySQL 则清理 `setups` 表与 `role=100` 的用户），再启动 |

## 八、停止与清理

- 在运行 `go run .` 的终端按 `Ctrl+C`，进程会优雅退出并提示 `server exited`。
- 前端开发服务器同样用 `Ctrl+C` 结束。
- 需要彻底重置本地数据：删除 `one-api.db*` 与 `logs/` 后重新启动即可，属于本地调试数据，不涉及仓库文件。
