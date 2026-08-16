# pkg 可观测性、限流熔断与其余一级包

> 状态：待复核生成稿
> 生成日期：2026-08-16
> 基准提交：`23807238c62e0f3b3e2d9a341bbef50547d3f5ec`
> 工作区：dirty（存在与本任务无关的本地改动：`go.mod` / `go.sum` / sqlite 测试库等；本文件为新增文档）
> 源码范围：`pkg/logger`、`pkg/tracer`、`pkg/stat`、`pkg/prof`、`pkg/shield`、`pkg/errcode`、`pkg/aicli`、`pkg/copier`、`pkg/dlock`、`pkg/gobash`、`pkg/gofile`、`pkg/goast`、`pkg/jwt`、`pkg/gocrypto`、`pkg/kafka`、`pkg/rabbitmq`、`pkg/sasynq`、`pkg/gocron`、`pkg/sse`、`pkg/ws`、`pkg/proxykit`、`pkg/httpcli`、`pkg/conf`、`pkg/jy2struct`、`pkg/krand`、`pkg/process`、`pkg/container`、`pkg/utils`、`pkg/gotest`、`pkg/replacer`、`pkg/encoding`，以及已在 12/13/14 展开的一级包短索引
> 生成方式：源码、测试、配置与部署资产静态分析

相关文档：[09-生成项目启动与HTTP请求链](09-生成项目启动与HTTP请求链.md)（生成项目如何装配 logger/tracer/stat/shield）、[12-pkg应用生命周期与Gin](12-pkg应用生命周期与Gin.md)（`pkg/app`、`pkg/httpsrv`、`pkg/gin`）、[13-pkg-gRPC与服务注册发现](13-pkg-gRPC与服务注册发现.md)（`pkg/grpc`、`pkg/servicerd`）、[05-代码生成器与模板写入](05-代码生成器与模板写入.md)（`pkg/replacer` 主链）、[07-Protoc插件与增量合并](07-Protoc插件与增量合并.md)（`pkg/goast` 合并链）。

## 目录

- [快速摘要](#快速摘要)
- [为什么这样设计（Why）](#为什么这样设计why)
- [它是什么（What）](#它是什么what)
- [代码如何实现（How）](#代码如何实现how)
  - [可观测性装配图](#可观测性装配图)
  - [限流熔断图](#限流熔断图)
  - [已在 12/13/14 展开的一级包](#已在-121314-展开的一级包)
  - [pkg/app（链 12）](#pkgapp链-12)
  - [pkg/httpsrv（链 12）](#pkghttpsrv链-12)
  - [pkg/gin（链 12）](#pkggin链-12)
  - [pkg/grpc（链 13）](#pkggrpc链-13)
  - [pkg/servicerd（链 13）](#pkgservicerd链-13)
  - [pkg/consulcli（链 13）](#pkgconsulcli链-13)
  - [pkg/etcdcli（链 13）](#pkgetcdcli链-13)
  - [pkg/nacoscli（链 13）](#pkgnacoscli链-13)
  - [pkg/sgorm（链 14）](#pkgsgorm链-14)
  - [pkg/mgo（链 14）](#pkgmgo链-14)
  - [pkg/cache（链 14）](#pkgcache链-14)
  - [pkg/goredis（链 14）](#pkggoredis链-14)
  - [pkg/sql2code（链 06）](#pkgsql2code链-06)
  - [pkg/logger](#pkglogger)
  - [pkg/tracer](#pkgtracer)
  - [pkg/stat](#pkgstat)
  - [pkg/prof](#pkgprof)
  - [pkg/shield](#pkgshield)
  - [pkg/errcode](#pkgerrcode)
  - [pkg/aicli](#pkgaicli)
  - [pkg/copier](#pkgcopier)
  - [pkg/dlock](#pkgdlock)
  - [pkg/gobash](#pkggobash)
  - [pkg/gofile](#pkggofile)
  - [pkg/goast](#pkggoast)
  - [pkg/jwt](#pkgjwt)
  - [pkg/gocrypto](#pkggocrypto)
  - [pkg/kafka](#pkgkafka)
  - [pkg/rabbitmq](#pkgrabbitmq)
  - [pkg/sasynq](#pkgsasynq)
  - [pkg/gocron](#pkggocron)
  - [pkg/sse](#pkgsse)
  - [pkg/ws](#pkgws)
  - [pkg/proxykit](#pkgproxykit)
  - [pkg/httpcli](#pkghttpcli)
  - [pkg/conf](#pkgconf)
  - [pkg/jy2struct](#pkgjy2struct)
  - [pkg/krand](#pkgkrand)
  - [pkg/process](#pkgprocess)
  - [pkg/container](#pkgcontainer)
  - [pkg/utils](#pkgutils)
  - [pkg/gotest](#pkggotest)
  - [pkg/replacer](#pkgreplacer)
  - [pkg/encoding](#pkgencoding)
- [调用关系表](#调用关系表)
- [阅读源码建议顺序](#阅读源码建议顺序)
- [重新实现检查清单](#重新实现检查清单)

---

## 快速摘要

### 架构总览（模块与依赖）

本篇是 `pkg/` 里**不负责 HTTP/gRPC 传输、不负责数据库/缓存**的一级包全量索引。依赖方向固定：

```text
生成项目 main / CLI
  → logger.Init / tracer.InitWithConfig / stat.Init
  → gin/grpc 中间件
      → shield.ratelimit.BBR / shield.circuitbreaker.Breaker
      → errcode（HTTP 200+body.code 或 gRPC status）
  → 可选：jwt / copier / httpcli / kafka / rabbitmq / sasynq / sse / ws
CLI 生成器
  → replacer / goast / gobash / gofile / jy2struct / aicli / krand / process
```

四类职责：

1. **可观测性**：`logger`（zap 全局实例）、`tracer`（OTel + Jaeger）、`stat`（CPU/内存打印与告警）、`prof`（SIGTRAP 采样与 pprof 路由）。
2. **治理**：`shield` 提供 BBR 自适应限流和 SRE 自适应熔断；Gin/gRPC 中间件只做适配，算法在本包。
3. **错误契约**：`errcode` 把业务码、HTTP 状态、gRPC `codes.Code` 三套数字空间分开，并提供 `Responser`。
4. **工具与可选集成**：AI、加解密、JWT、消息队列、SSE/WS、反向代理、配置解析、随机 ID、进程杀掉、测试替身、模板替换。

已在 [12](12-pkg应用生命周期与Gin.md) / [13](13-pkg-gRPC与服务注册发现.md) / [14](14-pkg数据库缓存与查询.md) 展开的包（`app`、`gin`、`httpsrv`、`grpc`、`servicerd`、`consulcli`、`etcdcli`、`nacoscli`、`sgorm`、`mgo`、`cache`、`goredis`）以及 [06](06-SQL到代码片段引擎.md) 的 `sql2code`，本篇只保留短节和公开入口，细节不重复。

### 核心调用序列（逐步逻辑）

以生成 HTTP 服务启动并处理一次被限流的请求为例（装配点见 [09](09-生成项目启动与HTTP请求链.md)）：

1. `cmd/serverNameExample_httpExample/initial/initApp.go:InitApp` 调用 `logger.Init(WithLevel, WithFormat, WithSave)`；失败则 `panic`。
2. 若 `cfg.App.EnableTrace`，调用 `tracer.InitWithConfig(appName, env, version, jaegerHost, jaegerPort, samplingRate)`；内部 `NewJaegerAgentExporter` 失败会 `panic`。
3. 若 `cfg.App.EnableStat`，调用 `stat.Init(WithLog(logger.Get()), WithAlarm(), WithPrintField(...))`，后台 ticker 每分钟打印 CPU/内存。
4. `pkg/app.New(...).Run` 监听 `SIGTRAP`，交给 `pkg/prof.NewProfile().StartOrStop()`。
5. `internal/routers.NewRouter`：`EnableLimit` 时挂 `middleware.RateLimit()` → `ratelimit.NewLimiter`（窗口 10s、100 bucket、CPU 阈值 800）；`EnableCircuitBreaker` 时挂 `middleware.CircuitBreaker` → `group.Group` 按路由懒创建 `circuitbreaker.NewBreaker`；`EnableTrace` 时挂 `middleware.Tracing`。
6. 请求进入 `RateLimit`：`BBR.Allow()` 若返回 `ErrLimitExceed`，`response.Output(c, 429)` 并 `Abort`，**不**调用后续熔断中间件。
7. 通过限流后 `CircuitBreaker`：`Allow()` 失败则 `MarkFailed` + HTTP 503；成功则 `c.Next()`，再按 `Writer.Status()` 是否在 `{500,503}` 里决定 `MarkFailed` / `MarkSuccess`。
8. handler 业务失败走 `pkg/gin/response.Error` → HTTP **200** + `body.code≠0`（见 [09](09-生成项目启动与HTTP请求链.md)）；熔断中间件看到的是 HTTP 200，默认**不会**把业务码当失败。
9. 进程退出时 `initial.Close` 调用 `tracer.Close`、`logger.Sync`。

### 易错点与边界条件

- `logger.Init` 未调用时，第一次 `Info/Error` 会 `checkNil` → 默认 `Init()`（终端、debug、console）。`Sync` 忽略含 `/dev/stdout` 的错误。
- CPU 用量在 shield 里是 **千分制**：800 表示 80%。`stat` 告警阈值是 **0~1 的小数**（默认 0.8）。两套刻度不要混用。
- BBR 只在 CPU ≥ 阈值 **且** `inFlight > maxInFlight` 时丢弃；CPU 降回阈值后还有 1 秒冷却窗口。
- SRE 熔断 `WithSuccess` 注释写默认 0.5，**代码默认是 0.6**（`k = 1/0.6 ≈ 1.667`）。本地拒绝会 `MarkFailed`（往窗口加 0），故意抬高后续丢弃率。
- gRPC 熔断拦截器在 `err == nil` 时**不**调用 `MarkSuccess`（只在有 error 的分支记账）。HTTP 中间件在 `c.Next()` 之后总会 Mark。这是源码事实，重实现时不要“想当然对齐”。
- `errcode.NewError` / `NewRPCStatus` 对重复 code **panic**。HTTP 系统码 10000x，业务码 `HCode(n)+k` ∈ 200000~300000；RPC 系统码 30000x，业务码 `RCode(n)+k` ∈ 400000~500000。
- JWT / AES / DES 都带**硬编码默认密钥**，生产必须覆盖。
- `kafka` / `rabbitmq` / `sasynq` / `sse` / `ws` / `dlock` / `gocron` / `gocrypto` 在本仓库生成模板里**没有业务调用方**，只提供库与自测。
- 本次未运行测试套件；测试结论来自阅读 `*_test.go`，不宣称测试通过。

---

## 为什么这样设计（Why）

生成项目需要一套可开关的运行时基础设施：日志必须全局可取、链路必须能关、限流熔断必须自适应而不是固定 QPS，错误码必须能同时服务 HTTP JSON 和 gRPC status。这些能力如果散落在模板里，每个生成项目都会分叉。Sponge 把它们收成无业务依赖的一级包，模板只负责 `Init` 和中间件装配。其余包（JWT、消息队列、SSE）是“生成后可选能力”，本仓库自身的 CLI 只用其中一小部分（`replacer`、`goast`、`aicli`、`gofile`）。

## 它是什么（What）

| 类别 | 包 | 一句话 |
|---|---|---|
| 可观测 | `logger` `tracer` `stat` `prof` | 全局日志、Jaeger 采样、资源打印、pprof/信号采样 |
| 治理 | `shield` | 滑动窗口 + BBR 限流 + SRE 熔断 + cgroup CPU |
| 错误 | `errcode` | HTTP/RPC 两套码表 + Responser |
| AI | `aicli` | ChatGPT / Gemini / DeepSeek 统一 `Assistanter` |
| 生成期 | `replacer` `goast` `gofile` `gobash` `jy2struct` | 模板替换、AST 合并、文件遍历、命令执行、YAML/JSON→struct |
| 安全 | `jwt` `gocrypto` | HMAC JWT、AES/DES/RSA、bcrypt |
| 消息与任务 | `kafka` `rabbitmq` `sasynq` `gocron` | 生产消费、Asynq 队列、cron |
| 实时与代理 | `sse` `ws` `proxykit` `httpcli` | SSE Hub、WebSocket、反向代理、JSON HTTP 客户端 |
| 杂项 | `conf` `krand` `process` `container` `utils` `gotest` `copier` `dlock` `encoding` | 配置、随机、杀进程、懒加载 Group、测试替身、拷贝、分布式锁、编解码 |

## 代码如何实现（How）

### 可观测性装配图

生成 HTTP 服务启动时，可观测组件由 `InitApp` 和 `app.Run` 两处挂上。节点对应真实符号。

```mermaid
flowchart TD
  MAIN["cmd/serverNameExample_httpExample/main.go:main"] --> INIT["initial.InitApp"]
  INIT --> LOG["logger.Init WithLevel WithFormat WithSave"]
  INIT --> TRACE{"cfg.App.EnableTrace?"}
  TRACE -->|是| TP["tracer.InitWithConfig → NewJaegerAgentExporter → otel.SetTracerProvider"]
  TRACE -->|否| STATQ
  TP --> STATQ{"cfg.App.EnableStat?"}
  STATQ -->|是| ST["stat.Init WithLog WithAlarm WithPrintField"]
  STATQ -->|否| APP["app.New.Run"]
  ST --> APP
  APP --> WATCH["app.watch 监听 SIGINT SIGTERM SIGHUP SIGTRAP"]
  WATCH -->|SIGTRAP| PROF["prof.NewProfile.StartOrStop 写 /tmp 下 cpu/mem/goroutine 等"]
  INIT --> ROUTER["routers.NewRouter"]
  ROUTER --> RID["middleware.RequestID"]
  ROUTER --> GLOG["middleware.Logging WithLog logger.Get"]
  ROUTER --> GTR{"EnableTrace?"}
  GTR -->|是| MTR["middleware.Tracing app.Name"]
  ROUTER --> HP{"EnableHTTPProfile?"}
  HP -->|是| GPROF["pkg/gin/prof.Register /debug/pprof"]
  CLOSE["initial.Close"] --> TC["tracer.Close"]
  CLOSE --> LS["logger.Sync"]
```

### 限流熔断图

HTTP 与 gRPC 共用 `pkg/shield` 算法，适配层分别在 `pkg/gin/middleware` 与 `pkg/grpc/interceptor`（适配细节见 [12](12-pkg应用生命周期与Gin.md) / [13](13-pkg-gRPC与服务注册发现.md)）。

```mermaid
flowchart TD
  REQ["入站请求"] --> RL["ratelimit.BBR.Allow"]
  RL -->|ErrLimitExceed| H429["HTTP: response.Output 429 Abort\n gRPC: StatusLimitExceed.ToRPCErr"]
  RL -->|DoneFunc| CB["circuitbreaker.Breaker.Allow"]
  CB -->|ErrNotAllowed| H503["HTTP: Output 503 或 degradeHandler\n gRPC: StatusServiceUnavailable.ToRPCErr\n 并 MarkFailed"]
  CB -->|nil| NEXT["handler / invoker"]
  NEXT --> DONE["HTTP: 按 Writer.Status 是否 500/503\n gRPC: 仅当 err!=nil 时按 codes 记账"]
  DONE -->|命中失败码| MF["MarkFailed → window.Add 0"]
  DONE -->|未命中| MS["MarkSuccess → window.Add 1"]
  H429 --> END["结束"]
  H503 --> END
  MS --> END
  MF --> END
```

BBR 丢弃判定（`shouldDrop`）：

```mermaid
flowchart TD
  CPU["cpu EMA 千分制"] --> CMP{"cpu < CPUThreshold 800?"}
  CMP -->|是| PREV{"prevDropTime == 0?"}
  PREV -->|是| ACCEPT["放行"]
  PREV -->|否| COOL{"距上次开始丢弃 ≤ 1s?"}
  COOL -->|是| INF1{"inFlight > 1 且 > maxInFlight?"}
  COOL -->|否| CLEAR["清空 prevDropTime 放行"]
  CMP -->|否| INF2{"inFlight > 1 且 > maxInFlight?"}
  INF2 -->|是| DROP["记录 prevDropTime 返回 ErrLimitExceed"]
  INF2 -->|否| ACCEPT
  INF1 -->|是| DROP2["继续丢弃"]
  INF1 -->|否| ACCEPT
```

---

### 已在 12/13/14 展开的一级包

这些包仍必须在本索引出现，细节以对应文档为准。

#### pkg/app（链 12）

- **Why**：用 errgroup 同时拉起多个 `IServer`，并统一处理关闭与 SIGTRAP 采样。
- **公开入口**：`IServer`、`Close`、`App`、`New`、`(*App).Run`。
- **谁调用**：全部 `cmd/serverNameExample_*/main.go`。
- **关键行为**：`watch` 监听 `SIGINT/SIGTERM/SIGHUP/SIGTRAP`；`SIGTRAP` → `prof.NewProfile().StartOrStop()`；其它信号 `stop()` 后返回。
- **测试**：`pkg/app/app_test.go`。详见 [12](12-pkg应用生命周期与Gin.md)。

#### pkg/httpsrv（链 12）

- **Why**：把 `http.Server` 和多种 TLS 策略（外部证书、Let’s Encrypt、自签、远程拉证书）收成 `TLSer`。
- **公开入口**：`TLSer`、`Server`、`New`、`Mode`、`NewTLSExternalConfig`、`NewTLSEAutoEncryptConfig`、`NewTLSSelfSignedConfig`、`NewTLSRemoteAPIConfig`。
- **谁调用**：`internal/server/http.go`。
- **测试**：`http_test.go`、`tls_*_test.go`。详见 [12](12-pkg应用生命周期与Gin.md)。

#### pkg/gin（链 12）

- **Why**：生成项目 HTTP 面的中间件、响应、pprof、反向代理封装。
- **公开入口（按子包）**：
  - `middleware`：`RateLimit`、`CircuitBreaker`、`Timeout`、`Tracing`、`RequestID`、`Logging`、`Cors`、`Auth`、`metrics.Metrics`
  - `response`：`Success`、`Error`、`Output`、`Out`
  - `handlerfunc`：`CheckHealth`、`Ping`、`ListCodes`
  - `prof.Register`（Gin Engine 版 pprof，与 `pkg/prof.Register` 的 `http.ServeMux` 版不同）
  - `proxy.New`（内部用 `pkg/proxykit`）
  - `swagger`、`frontend`、`staticfs`、`validator`
- **谁调用**：`internal/routers`、生成 handler。详见 [12](12-pkg应用生命周期与Gin.md) 与 [09](09-生成项目启动与HTTP请求链.md)。

#### pkg/grpc（链 13）

- **Why**：生成项目的 gRPC server/client、拦截器、metrics、TLS、resolver。
- **公开入口（与本篇交叉）**：`interceptor.UnaryServerRateLimit`、`StreamServerRateLimit`、`UnaryServerCircuitBreaker`、`StreamServerCircuitBreaker`、`UnaryClientCircuitBreaker`、`StreamClientCircuitBreaker`；限流失败 → `errcode.StatusLimitExceed.ToRPCErr`；熔断失败 → `StatusServiceUnavailable.ToRPCErr`。另有 `server.New`、`grpccli`、`metrics`、`gtls`、`keepalive`、`benchmark`。
- **谁调用**：`internal/server/grpc.go`、`internal/rpcclient`。
- **测试**：`pkg/grpc/interceptor/breaker_test.go`、`ratelimit` 相关测试及 `server_test.go`。详见 [13](13-pkg-gRPC与服务注册发现.md)。

#### pkg/servicerd（链 13）

- **Why**：把实例注册到 Consul/Etcd/Nacos，并提供 gRPC resolver 做发现。
- **公开入口**：`registry` 包下 Consul/Etcd/Nacos 的 `NewRegistry`/`Register`/`Deregister`；`discovery` 的 resolver/builder。
- **谁调用**：带 registry 的 `internal/server` 变体。详见 [13](13-pkg-gRPC与服务注册发现.md)。

#### pkg/consulcli（链 13）

- **Why**：给 servicerd 一个带 Option 的 Consul API 客户端。
- **公开入口**：`Init(addr, opts...) *api.Client`；`WithWaitTime` `WithScheme` `WithDatacenter` `WithToken` `WithConfig`。
- **测试**：`consulcli_test.go`。详见 [13](13-pkg-gRPC与服务注册发现.md)。

#### pkg/etcdcli（链 13）

- **Why**：给 servicerd/dlock 一个 etcd v3 客户端。
- **公开入口**：`Init(endpoints, opts...) *clientv3.Client`；`WithDialTimeout` `WithAuth` `WithSecure` `WithAutoSyncInterval` `WithLog` `WithConfig`。
- **测试**：`etcdcli_test.go`。详见 [13](13-pkg-gRPC与服务注册发现.md)。

#### pkg/nacoscli（链 13）

- **Why**：拉 Nacos 配置并创建 naming 客户端。
- **公开入口**：`GetConfig(*Params)`、`NewNamingClient`、`Init`（空实现占位）、`WithAuth` `WithClientConfig` `WithServerConfigs`。
- **测试**：`nacos_test.go`。详见 [13](13-pkg-gRPC与服务注册发现.md)。

#### pkg/sgorm（链 14）

- **Why**：GORM 连接 MySQL/Postgres/SQLite 及查询构造。
- **公开入口**：各驱动 `Init`、`query` 分页/条件、`glog`、`dbclose`。详见 [14](14-pkg数据库缓存与查询.md)。

#### pkg/mgo（链 14）

- **Why**：Mongo 官方驱动封装与 query。
- **公开入口**：`Init`、`query` 分页。详见 [14](14-pkg数据库缓存与查询.md)。

#### pkg/cache（链 14）

- **Why**：带 singleflight 与占位的 Redis/内存缓存。
- **公开入口**：`NewRedisCache` `NewRedisClusterCache` `NewMemoryCache`；构造需要 `encoding.Encoding`（常用 `encoding.JSONEncoding{}`）。详见 [14](14-pkg数据库缓存与查询.md)。

#### pkg/goredis（链 14）

- **Why**：go-redis 客户端初始化（单机/集群）。
- **公开入口**：`Init` 及 Option。详见 [14](14-pkg数据库缓存与查询.md)。

#### pkg/sql2code（链 06）

- **公开入口**：`sql2code.Generate`、`GenerateOne`、`parser.ParseSQL`、`GetMysqlTableInfo`、`GetPostgresqlTableInfo`、`GetSqliteTableInfo`、`GetMongodbTableInfo`。
- 详见 [06](06-SQL到代码片段引擎.md)。

---

### pkg/logger

**路径**：`pkg/logger/`

**Why**：给生成项目和 CLI 一个可切终端/文件、可切 console/json 的全局 zap，避免每个服务自己拼 `zap.Config`。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `Init(opts ...Option) (*zap.Logger, error)` | 初始化全局 `defaultLogger` / `defaultSugaredLogger` |
| `WithLevel` `WithFormat` `WithSave` `WithHooks` | 级别、json/console、落盘、hook |
| `WithFileName` `WithFileMaxSize` `WithFileMaxBackups` `WithFileMaxAge` `WithFileIsCompression` `WithLocalTime` | lumberjack 切分 |
| `Debug/Info/Warn/Error/Panic/Fatal` 及 `*f` | 包级日志 |
| `Get` `GetWithSkip` `WithFields` `Sync` | 取实例、跳过 caller、刷盘 |
| `Field` 及 `Int`/`String`/`Err`/`Any` 等 | zap Field 别名 |
| `ReplaceGRPCLoggerV2` | 把 gRPC 内部日志接到 zap |

**谁调用**：`initial.InitApp`（所有 `serverNameExample_*`）、`internal/handler`、`internal/dao`、`internal/service`、`internal/database/*`、`internal/routers`、`pkg/grpc/interceptor/logging.go`、`pkg/grpc/grpccli`、`cmd/sponge/server`、`cmd/sponge/commands/assistant`。

**关键行为**：

1. 默认：`level=debug`、`encoding=console`、`isSave=false`（打到 stdout）。非法 level 回退 debug。`WithFormat` 只有 `"json"` 会改编码，其它值保持 console。
2. 终端路径 `log2Terminal`：用 JSON 拼出 `zap.Config`，`EncodeTime` 为 `2006-01-02 15:04:05.000`；console 用彩色 Level。
3. 文件路径 `log2File`：`lumberjack.Logger` 默认 `out.log`、10MB、100 备份、30 天、不压缩。时间用 ISO8601。**`WithLocalTime` 写入了 `fileOptions.isLocalTime`，但构造 lumberjack 时未赋值 `LocalTime` 字段，该选项当前无效。**
4. `WithFileMaxBackups` 判断的是 `f.maxBackups > 0`（已有默认 100），不是入参；传入 0 仍会被写进去。
5. `getLogger`/`getSugaredLogger` 每次 `AddCallerSkip(1)`，使包级函数显示真正调用行。
6. `checkNil`：未 Init 则默认 Init；Init 失败 panic。
7. `Init` 里 `log2Terminal` 失败会 **panic**，返回的 `error` 在终端路径上实际为 nil（`err` 未再赋值）。文件路径 `err` 一直是 nil。
8. `ReplaceGRPCLoggerV2`：`AddCallerSkip(5)` + field `grpc_system=true`；`V(level)` 用内部 `verbosity=0`，即 `0<=level` 恒真。
9. 失败语义：日志函数本身不返回 error；`Fatal`/`Panic` 走 zap 语义（进程退出 / panic）。`Sync` 对 stdout 的 sync 错误吞掉。

**测试文件**：`logger_test.go`（终端/json/文件/hook）、`type_test.go`、`grpcLogger_test.go`。

---

### pkg/tracer

**路径**：`pkg/tracer/`

**Why**：把 OpenTelemetry TracerProvider、Jaeger exporter、Resource、Span 辅助函数收口，生成项目只调一次 `InitWithConfig`。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `Init(exporter, res, fractions ...float64)` | 建全局 `tp`，`ParentBased(TraceIDRatioBased)`，设置 W3C TraceContext+Baggage |
| `InitWithConfig(appName, appEnv, appVersion, jaegerAgentHost, jaegerAgentPort, jaegerSamplingRate)` | Resource + Jaeger Agent + Init + `SetTraceName` |
| `Close(ctx)` | `tp.Shutdown`；`tp==nil` 返回 nil |
| `GetProvider()` | `tp==nil` 则 panic |
| `NewJaegerExporter(url, opts...)` | Collector HTTP，可选用户名密码 |
| `NewJaegerAgentExporter(host, port)` | Agent UDP |
| `NewConsoleExporter` `NewFileExporter` | stdout / 文件（测试用） |
| `NewResource` + `WithServiceName/Version/Environment/Attributes` | Resource，默认 `demo-service` / `v0.0.0` / `dev` |
| `NewSpan(ctx, spanName, tags)` `SetTraceName` | 按类型把 tags 转 attribute |

**谁调用**：`initial.InitApp`（`EnableTrace`）、`initial.Close` → `tracer.Close`、`internal/database/redis.go`（把 tracer 接到 Redis）、Gin `middleware.Tracing` 使用全局 Provider（不直接 import tracer 包）。

**关键行为**：

1. 采样：`fractions` 空或 `>=1` → 1.0 全采；`<=0` → 0；`(0,1)` 按比例。`InitWithConfig` 把配置里的 `TracingSamplingRate` 原样传入。
2. `InitWithConfig` 里 Jaeger Agent 失败 **panic** `"init trace error:" + err`。
3. `NewFileExporter` 创建文件失败也 panic。
4. `NewSpan` 不识别的类型用 `fmt.Sprintf("%+v")` 当 string；`nil` 值跳过。必须 `span.End()`。
5. 失败语义：初始化失败即进程起不来；运行期 span 不返回 error。

**测试文件**：`tracerProvider_test.go`、`jaeger_test.go`、`resource_test.go`、`span_test.go`、`console_test.go`。

---

### pkg/stat

**路径**：`pkg/stat/`（子包 `cpu/`、`mem/`）

**Why**：周期性打印进程/系统资源，并在非 Windows 上用 SIGTRAP 触发 `pkg/prof` 采样（与 `app.watch` 的 SIGTRAP 是同一信号）。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `Init(opts ...Option)` | 启动永不退出的 ticker goroutine |
| `WithPrintInterval` | 最小 1s，默认 1 分钟 |
| `WithLog` `WithPrintField` | 替换 zap、附加字段 |
| `WithAlarm(AlarmOption...)` | 启用告警；Windows 直接 return 不启用 |
| `WithCPUThreshold` `WithMemoryThreshold` | 包级阈值，范围 `[0,1)`，默认 0.8 |
| `WithCustomHandler` | 替换默认打印 |
| `StatData` `System` `Process` | 快照结构 |

**谁调用**：`initial.InitApp` 在 `EnableStat` 时。`notify.go`（linux/darwin）`init` 里消费 `notifyCh`，`syscall.Kill(pid, SIGTRAP)`。

**关键行为**：

1. `getStatData`：`mem.GetSystemMemory/GetProcessMemory`、`cpu.GetSystemCPU/GetProcess`，内存单位已 `>>20` 成 MB；`Sys.MemUsage` 四舍五入为整数百分比。panic 被 recover 吞掉。
2. 告警 `statGroup.check`：必须攒满 **3 个采样点** 才判断。CPU：进程 CPU 三次平均 `>= threshold*100`（注意 `Proc.CPUUsage` 已是百分数）。内存：进程 RSS 三次平均 / `Sys.MemTotal` `>= threshold`。触发后 `triggerInterval=900` 秒内不重复。
3. `WithAlarm` 的 `AlarmOption` 作用在**包级变量**上，空的 `alarmOptions` 结构体只是载体。
4. 告警触发 `sendSystemSignForLinux`：非阻塞写入 `notifyCh`；`notify.go` 再 SIGTRAP。与 `app.watch` 共用 SIGTRAP → 可能启动 profile 采样。
5. `WithCustomHandler` 用 `printInfoInterval` 做 context timeout；handler error 只 `zapLog.Warn`。
6. 失败语义：Init 不返回 error；采集失败打印后返回零值结构体。

**测试文件**：`stat_test.go`、`alarm_test.go`、`cpu/cpu_test.go`、`mem/memory_test.go`。

---

### pkg/prof

**路径**：`pkg/prof/`

**Why**：两套入口——信号触发写文件采样，以及给 `http.ServeMux` 挂标准 pprof（Gin 版在 `pkg/gin/prof`）。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `NewProfile` `(*Profile).StartOrStop` | 第一次开始采样，第二次停止；默认最长 60s |
| `SetDurationSecond` `EnableTrace` | 时长、是否采 `runtime/trace` |
| `Register(mux *http.ServeMux, opts...)` | `/debug/pprof/*`，可选 `fgprof` 的 `/profile-io` |
| `WithPrefix` `WithIOWaitTime` | 路由前缀、IO wait |

**谁调用**：`pkg/app.watch`（SIGTRAP）；`pkg/stat` 告警间接 SIGTRAP。Gin 路由用的是 `pkg/gin/prof.Register`，不是本包。`internal/server/grpc.go` import 了 `pkg/prof`（gRPC 侧 HTTP mux 挂 pprof，细节见 [13](13-pkg-gRPC与服务注册发现.md)）。

**关键行为**：

1. 状态机：`status` 原子 0/1，`CompareAndSwap`。`startProfile` 打开 cpu/mem/goroutine/block/mutex/threadcreate（可选 trace），文件目录 `os.TempDir()/<serverName>_profile/`，文件名 `{time}_{pid}_{server}_{kind}.out`。
2. `stopProfile` 执行全部 `closeFns`，然后 `p = NewProfile()` **只改了局部变量，没有写回调用方**，第二次 Stop 依赖包级 `status` 而不是实例字段清空。`closeFns` 仍挂在原实例上；`app.watch` 复用同一个 `profile` 指针，Stop 时 `closeFns` 还在。
3. `checkTimeout` 到时若仍是 start 状态则 stop。手动 stop 通过 `stopCh` 唤醒。
4. `Register` 失败语义：路由注册无 error。采样创建文件失败只 `fmt.Println`。
5. 目录权限 `0766`。

**测试文件**：`profile_test.go`、`http_test.go`。

---

### pkg/shield

**路径**：`pkg/shield/{ratelimit,circuitbreaker,window,cpu}`

**Why**：在进程过载时用自适应算法丢请求，而不是固定 QPS。算法来自 Sentinel BBR 思路和 Google SRE 自适应节流，窗口统计自研。

#### window：滑动计数窗口

**公开入口**：`Window`/`NewWindow`、`RollingPolicy`/`NewRollingPolicy`、`RollingCounter`/`NewRollingCounter`、`Iterator`、聚合函数 `Sum/Avg/Min/Max/Count`。

**关键行为**：

- `Window` 是环形 `[]Bucket`，每个 Bucket 有 `Points []float64` 和 `Count`。
- `RollingPolicy.timespan` = `time.Since(lastAppendTime)/bucketDuration`；时间回拨时返回 `size`（整窗失效）。
- `apply`：跨越的 bucket `ResetBuckets`，再在当前 offset 上 `Append` 或 `Add`。`Add` 在 Count==0 时退化为 Append，否则把值加到 `Points[0]`。
- `Reduce` 只迭代**尚未过期**的 bucket：`count = size - timespan`。
- `RollingCounter.Add` 对负数 **panic** `"cannot decrease in value"`。

**测试**：`window_test.go`、`policy_test.go`、`counter_test.go`、`reduce_test.go`。

#### cpu：进程 CPU 千分比

**公开入口**：`ReadStat(*Stat)`、`GetInfo() Info`、`ErrNoCFSLimit`。

**关键行为**：

1. `init`：先 `newCgroupCPU()`，失败再 `newPsutilCPU(500ms)`；都失败则打印错误，提示把配置里 `enableStat/enableLimit/enableCircuitBreaker` 设为 false，**不 panic**。此后 `ReadStat` 得到 Usage=0，BBR 几乎不会因 CPU 丢包。
2. 后台 500ms ticker 调用 `stats.Usage()`，非 0 才 `atomic.StoreUint64`。
3. cgroup 用量公式（千分制）：
   `u = (total-preTotal) * cores * 1000 / ((system-preSystem) * quota)`
   `total` 来自 `cpuacct.usage`，`system` 来自 `/proc/stat` 的 cpu 行前 7 字段，换算成纳秒（`clockTicks=100`）。
   `quota = min(cpuset 个数, cfs_quota/cfs_period)`；无 quota 时用 cpuset 长度。
4. psutil 回退：`cpu.Percent(interval, false)[0] * 10`，把 0~100 的百分数变成 0~1000。
5. BBR 的 `cpuproc` 再对 `ReadStat` 做 EMA：`gCPU = prev*0.95 + usage*0.05`，且 `usage` cap 在 1000。ticker 同样 500ms；panic 后重启 goroutine。

**测试**：`stat_test.go`、`cgroup_test.go`、`utils_test.go`。

#### ratelimit BBR

**公开入口**：`Limiter` 接口、`BBR`、`NewLimiter`、`ErrLimitExceed`、`DoneFunc`/`DoneInfo`、`WithWindow`/`WithBucket`/`WithCPUThreshold`/`WithCPUQuota`、`(*BBR).Allow`/`Stat`。

**默认参数**（`NewLimiter`）：

| 参数 | 默认 | 含义 |
|---|---|---|
| `Window` | 10s | 统计窗口 |
| `Bucket` | 100 | bucket 数 → 每个 bucket 100ms |
| `CPUThreshold` | 800 | CPU 千分制阈值（80%） |
| `CPUQuota` | 0 | 0 表示不按配额缩放 |

Gin/gRPC 适配层默认值与上表相同。

**算法要点**（必须按此重实现）：

1. `maxPASS`：对 passStat 窗口（**跳过当前 bucket**，`i` 从 1 到 Bucket-1）取各 bucket 点数之和的最大值，下限 1。结果缓存一个 bucketDuration。
2. `minRT`：对各历史 bucket 的 `sum(Points)/Count` 取最小，`Ceil` 后若 `<=0` 则改为 1。同样缓存。
3. `maxInFlight = Floor(maxPASS * minRT * bucketPerSecond / 1000) + 0.5` 再转 int64。`bucketPerSecond = 1s / bucketDuration`。物理含义：用“单桶最大通过量 × 最小 RT(ms) × 每秒桶数 / 1000”估计并发上限。
4. `shouldDrop`：见上文流程图。`inFlight > 1` 是为了至少放行 1 个探测请求。
5. `Allow`：drop 则 `(nil, ErrLimitExceed)`；否则 `inFlight++`，返回 `DoneFunc`：记录 `max(rt_ms, 1)` 到 rtStat，`inFlight--`，`passStat.Add(1)`。`DoneInfo.Err` **未被使用**。
6. `WithCPUQuota(q)`：`cpu = gCPU * NumCPU / q`，用于容器 quota 与宿主机核数不一致时校正。

**谁调用**：`pkg/gin/middleware.RateLimit`、`pkg/grpc/interceptor.UnaryServerRateLimit` / `StreamServerRateLimit`。生成项目由 `EnableLimit` 打开。

**失败语义**：只返回 `ErrLimitExceed`。HTTP 变 429 文本即 `err.Error()`；gRPC 变 `StatusLimitExceed`（码 300020）的 `ToRPCErr`。

**测试**：`bbr_test.go`（并发 Allow、warmup、forceAllow）。

#### circuitbreaker SRE

**公开入口**：`CircuitBreaker` 接口、`Breaker`、`NewBreaker`、`ErrNotAllowed`、`StateOpen=0`、`StateClosed=1`、`WithSuccess`/`WithRequest`/`WithWindow`/`WithBucket`。

**默认参数**（`NewBreaker`）：

| 参数 | 默认 | 含义 |
|---|---|---|
| `success` | **0.6**（注释曾写 0.5，以代码为准） | `k = 1/success ≈ 1.667` |
| `request` | 100 | 窗口内总请求低于此值时强制关闭并放行 |
| `bucket` | 10 | |
| `window` | 3s | 每桶 300ms |

**算法要点**（Google SRE adaptive throttling）：

1. `summary()`：窗口内 `total = Σ bucket.Count`，`accepts = Σ Points`（成功记 1，失败记 0 仍增加 Count）。
2. `requests = k * accepts`，表示“按当前成功率反推允许的尝试次数”。
3. 若 `total < request` **或** `float64(total) < requests`：CAS Open→Closed，**允许**。
4. 否则 CAS Closed→Open，丢弃概率 `dr = max(0, (total - requests) / (total+1))`，`trueOnProba(dr)` 为真则 `ErrNotAllowed`。
5. `k` 越小越激进（更容易丢）。`success=0.6` 比 0.5 更宽松。
6. `MarkSuccess` → `Add(1)`；`MarkFailed` → `Add(0)`。本地拒绝也 MarkFailed，让 `total` 涨而 `accepts` 不涨，从而 `dr` 升高。

**谁调用**：`pkg/gin/middleware.CircuitBreaker`（按 `c.FullPath()` 从 `container/group.Group` 取 Breaker）、`pkg/grpc/interceptor` 的 client/server unary/stream（按 method / FullMethod）。

**HTTP 适配关键点**（源码事实）：

- 默认失败 HTTP 码：500、503。`internal/routers.NewRouter` 额外 `WithValidCode(errcode.InternalServerError.Code(), ServiceUnavailable.Code())` 即 **100003 和 100013**。熔断判断用的是 `c.Writer.Status()`（标准 HTTP 状态），这两个业务码**不会**被命中；真正触发失败记账的仍是 HTTP 500/503。
- handler 走 `response.Error` 时 HTTP 状态是 **200**，熔断视为成功。
- 限流中间件在熔断之前，429 不会进入熔断。

**gRPC 适配关键点**：默认失败码 `codes.Internal`、`codes.Unavailable`。**成功路径（err==nil）不 MarkSuccess**，窗口只在出错或本地拒绝时更新。这会让成功流量几乎不增加 `accepts`，熔断更容易在错误出现后打开。重实现若要对齐“成功也记账”，这是行为分叉点。

**测试**：`sre_test.go`、`circuitbreaker_test.go`。

---

### pkg/errcode

**路径**：`pkg/errcode/`

**Why**：HTTP JSON `{code,msg,data}` 与 gRPC `status.Status` 共用一套“系统码 + 业务码”分配规则，并在网关把 RPC 错误转成 HTTP。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `Error` `NewError` `(*Error).Err` `ErrToHTTP` `Code/Msg/Details` `WithDetails` `RewriteMsg` `WithOutMsgI18n` `ToHTTPCode` | HTTP 错误对象 |
| `ParseError` `GetErrorCode` `Is` `ListHTTPErrCodes` | 从 `error.Error()` 文本反解析 |
| `RPCStatus` `NewRPCStatus` `(*RPCStatus).Err` `ErrToHTTP` `ToRPCErr` `ToRPCCode` | gRPC 错误 |
| `GetStatusCode` `IsStatus` `ListGRPCErrCodes` `ToHTTPErr` | RPC → HTTP |
| `HCode(num)` `RCode(num)` | 业务码段生成器 |
| `Responser` `NewResponser` `SkipResponse` `ShowConfig` | Gin 响应策略 |
| 系统码变量 | 见下表 |

**码段（代码事实，注释“10000~20000”与实现不符）**：

| 空间 | 范围 | 生成方式 |
|---|---|---|
| HTTP 成功 | 0 | `Success` |
| HTTP 系统 | 100001~100425 | `http_system_code.go` 字面量 |
| HTTP 业务 | 200000 + num*100 + k | `HCode(1)+1 = 200101`；`num` 必须 1~999，否则 panic |
| RPC 成功 | 0 | `StatusSuccess` |
| RPC 系统 | 300001~300023 | `rpc_system_code.go` |
| RPC 业务 | 400000 + num*100 + k | `RCode` 返回 `codes.Code` |

HTTP 系统码与 `ToHTTPCode` 映射（节选）：`0→200`，`100001→400`，`100002/100016→401`，`100003→500`，`100004→404`，`100006/100010→408`，`100007/100009→429`，`100008/100011→403`，`100012→405`，`100013→503`，`100021→501`，`100023→502`，`100409→409`，`100425→425`，未列出 → 500。

RPC `ToRPCCode`：把 30000x 自定义码映射到标准 `codes.*`（如 `StatusInvalidParams→InvalidArgument`，`StatusUnauthorized→Unauthenticated`）。

**谁调用**：`internal/ecode/*` 再导出；`pkg/gin/response`；`pkg/gin/handlerfunc.ListCodes`；`pkg/grpc/interceptor` 限流/熔断/logging；`protoc-gen-go-gin` 生成的 router；`errcode.ShowConfig` 挂在非 prod 的 `/config`。

**Responser 关键行为**（`NewResponser(isFromRPC, httpErrors, rpcStatus)`）：

1. `Success`：HTTP 200，`code=0,msg=ok`。
2. `ParamError`：HTTP 200，`InvalidParams`（100001），**不回传原始校验 error**。
3. `isFromRPC=false`（纯 HTTP）：`ParseError` → 100003/500 或 100013/503 用真实 HTTP 状态；`ErrToHTTP` 带 `[standard http code]` 则 `ToHTTPCode()`；用户定义码若在 map 里则转标准 HTTP 文案；否则 HTTP 200 + 业务 code。
4. `isFromRPC=true`：`codes.Unknown` 时从 `st.String()` 解析 `code = , msg = `；标准 Internal/Unavailable 用 HTTP 500/503；消息含 `ToHTTPCodeLabel` 则用 `convertToHTTPCode`。
5. `SkipResponse` 供生成 router 跳过自动写响应（本包只定义变量）。

**失败语义**：重复注册 code → panic。`ParseError` 解析失败返回 `code=-1,msg=unknown error`。`HCode/RCode` 越界 panic（英文文案分别写 “0 to 1000” / “NO range”，实际是 1~999）。

**测试文件**：`http_error_test.go`、`rpc_error_test.go`、`response_test.go`。

---

### pkg/aicli

**路径**：`pkg/aicli/`（`chatgpt/`、`gemini/`、`deepseek/`）

**Why**：CLI `sponge assistant` 要对接多家模型，统一 `Send` / `SendStream` / 上下文刷新。

**公开入口**：

| 符号 | 包 | 职责 |
|---|---|---|
| `Assistanter` `StreamReply` | `aicli` | 接口；`Content chan string` + `Err` |
| `GenericRoleDesc*` `GopherRoleDesc*` | `aicli` | 角色提示词常量 |
| `chatgpt.NewClient` `Client.Send/SendStream/ListModelNames/RefreshContext/ModifyInitialRole` | chatgpt | OpenAI SDK |
| `WithMaxTokens` `WithModel` `WithTemperature` `WithInitialRole` `WithEnableContext` `WithInitialContextMessages` | chatgpt | 默认模型 `gpt-4o`，maxTokens 8192，temperature 0 |
| `gemini.NewClient` `Close` `Send/SendStream/...` | gemini | 默认 `gemini-2.5-flash` |
| `deepseek.NewClient` | deepseek | 复用 chatgpt.Client，改 `BaseURL=https://api.deepseek.com/`，默认模型 `deepseek-chat` |

**谁调用**：`cmd/sponge/commands/assistant/common.go`、`generate.go`。

**关键行为**：

1. API key 或 prompt 为空 → `errors.New(...)`。
2. chatgpt `Send`：拼 system/history/fileIDs/user → `CreateChatCompletion`；空 choices → `"reply content is empty"`。`enableContext` 时追加 user+assistant。文件通过 `CreateFile(Purpose: assistants)` 上传，只把 ID 写进 system 消息，**不是** vision 多模态。
3. `SendStream` 在独立 goroutine 里 `Recv`，`ctx.Done` 或 `io.EOF` 结束；`Err` 写在 `StreamReply` 上，调用方必须先读完 channel 再看 `Err`（文档契约，测试里也是如此）。
4. `WithMaxTokens`：若 `<1000` 先赋默认 8192，**随后无条件再赋入参**，因此 100 仍会变成 100。
5. gemini 把历史做成 `genai.Text("role: content")`，文件按扩展名 MIME 读成 `Blob`。`Close` 必须调。
6. deepseek 若模型仍是 chatgpt 默认 `gpt-4o`，改成 `deepseek-chat`。

**失败语义**：网络/SDK error 原样返回；流式错误放在 `StreamReply.Err`，不 panic。

**测试文件**：`chatgpt/client_test.go`、`gemini/client_test.go`、`deepseek/client_test.go`。

---

### pkg/copier

**路径**：`pkg/copier/copy.go`

**Why**：handler/service 在 DTO 与 model 之间拷贝时自动处理 `time.Time` ↔ RFC3339 字符串。

**公开入口**：`Copy(dst, src)`（DeepCopy + 时间转换器）、`CopyDefault`（jinzhu 默认）、`CopyWithOption`、`Converter`。

**谁调用**：`internal/handler/userExample.go`、`userExample_logic.go`、`internal/service/userExample.go`、`mixExample/initial/initApp.go`。

**关键行为**：空字符串 → 零值 `time.Time` 或 `nil *time.Time`；`*time.Time==nil` → `""`。转换类型不匹配返回 `fmt.Errorf("expected ... got %T")`。

**测试文件**：`copy_test.go`。

---

### pkg/dlock

**路径**：`pkg/dlock/`

**Why**：可选的 Redis/Etcd 分布式锁，生成模板未强制依赖。

**公开入口**：`Locker`（`Lock/Unlock/TryLock/Close`）、`NewRedisLock`、`NewRedisClusterLock`、`NewEtcd`。

**谁调用**：仓库内无业务调用方，仅 `redis_test.go`、`etcd_test.go`。

**关键行为**：

- Redis：`go-redsync`；client/key 空返回 error；`TryLock` 失败返回 `(false, err)`（含锁被占）；`Unlock` 忽略 bool；`Close` 空操作。
- Etcd：`concurrency.NewSession(WithTTL)`，ttl≤0 用 15s；构造时用 ttl 秒的 context（cancel 泄漏，nolint）。`TryLock` 对 `ErrLocked` 返回 `(false, nil)`。`Close` 关 session。

**失败语义**：构造失败返回 error；Etcd session 失败不创建 mutex。

**测试文件**：`redis_test.go`、`etcd_test.go`。

---

### pkg/gobash

**路径**：`pkg/gobash/gobash.go`

**Why**：CLI 要跑 `protoc`、插件安装、git 等命令，并实时读 stdout。

**公开入口**：`Exec(name, args...) ([]byte, error)`、`Run(ctx, name, args...) *Result`、`Result{StdOut, Err, Pid}`、`ParsePid`。

**谁调用**：`commands/generate/common.go`、`upgrade.go`、`plugins.go`、`merge/common.go`、`patch/copy-proto.go`、`assistant/common.go`、`graph.go`、`server/handler.go`、`perftest/grpc`、`template/protobuf.go`。

**关键行为**：`LookPath` 找不到则 error。`Exec` 阻塞读完 stdout，stderr 在 `Wait` 失败时作为 error 文本。`Run` 异步：先推一行 `"cmd [pid]=N"`，再按行推 stdout；`ctx.Done` 设 `Err`。注释写明**永久阻塞命令会泄漏 goroutine**。`ParsePid` 解析 `[pid]=` 后的整数。

**测试文件**：`gobash_test.go`。

---

### pkg/gofile

**路径**：`pkg/gofile/`

**Why**：生成器大量列目录、滤后缀、拼跨平台路径、从字节里抠锚点。

**公开入口**：`IsExists`、`GetRunPath`、`GetFilename/Suffix/Dir/...`、`CreateDir`、`Join`、`IsWindows`、`GetPathDelimiter`、`ListFiles`（`WithSuffix/Prefix/Contain/NoAbsolutePath`）、`ListDirsAndFiles`、`FuzzyMatchFiles`、`ListDirs`、`FilterDirs`、`ListSubDirs`、`FindSubBytes`/`FindAllSubBytes`/`FindSubBytesNotIn`。

**谁调用**：几乎所有 `commands/generate/*`、`patch/*`、`merge/*`、`assistant/*`、`pkg/replacer`、`pkg/sql2code`。

**关键行为**：`ListFiles` 默认转绝对路径后递归；过滤器按**文件名**而非全路径（prefix/contain 用 `GetFilename`）。`FuzzyMatchFiles` 只支持 `*` 的 `path.Match`。`FindSubBytes` 用第一次出现的 start/end，若 `startIndex>=endIndex` 返回空（**不保证** end 在 start 之后的正确配对）。

**测试文件**：`filePath_test.go`、`fileContent_test.go`。

---

### pkg/goast

**路径**：`pkg/goast/`（细节链 [07](07-Protoc插件与增量合并.md)）

**Why**：protoc 插件和 `sponge merge` 要把“新生成的代码”合并进用户改过的文件，必须按 AST 块而不是文本覆盖。

**公开入口**：

| 符号 | 职责 |
|---|---|
| `ParseFile` `ParseGoCode` `AstInfo` | 切出 package/import/const/var/type/func 块 |
| `ParseImportGroup` `ParseConstGroup` `ParseVarGroup` `ParseTypeGroup` `ParseInterface` `ParseStruct` `ParseStructMethods` | 细解析 |
| `FilterFuncCodeByFile` `FilterFuncCode` `FuncInfo` | 按自定义标记过滤函数 |
| `CodeAst` `NewCodeAst` `NewCodeAstFromData` | 可合并的代码对象 |
| `WithCoverSameFunc` `WithIgnoreMergeFunc` | 同名函数覆盖 / 忽略 |
| `MergeGoFile` `MergeGoCode` | src（用户）+ gen（生成）合并 |

**谁调用**：`commands/merge/module/*`、`commands/assistant/merge.go` `generate.go`、`patch/modify-dup-num.go` `modify-dup-err-code.go`。

**关键行为（入口级）**：`MergeGoFile(srcFile, genFile)` 先解析用户文件（带 opts），再解析生成文件（**不传 opts**），再 `mergeCode`。重复锚点 `errDuplication`。完整合并规则、ignore 列表、cover 语义见 [07](07-Protoc插件与增量合并.md)。

**测试文件**：`ast_test.go`、`merge_test.go`、`filter_test.go`。

---

### pkg/jwt

**路径**：`pkg/jwt/`，旧包 `pkg/jwt/old_jwt/`（已 Deprecated）

**Why**：HTTP/gRPC 鉴权中间件需要签发/校验 HMAC JWT，并支持 access+refresh 双 token。

**公开入口（新包）**：

| 符号 | 默认 |
|---|---|
| `Claims{UID, Fields, RegisteredClaims}` 及 `Get/GetString/GetInt/GetInt64/GetBool/GetFloat64` | JSON 反序列化数字为 float64，GetInt 兼容 |
| `GenerateToken(uid, opts...) (jwtID, token, err)` | HS256，过期 24h，key 见源码硬编码，jti=`krand.NewStringID()` |
| `ValidateToken` `RefreshToken` | 只接受 HMAC |
| `GenerateTwoTokens` `RefreshTwoTokens` `Tokens` | access 30min，refresh 30d，强制两端 jti 相同 |
| `GetClaimsUnverified` | 不验签 |
| `HS256/384/512` `ErrTokenExpired` | |

选项：`WithIssuer/Subject/Audience/Expires/Deadline/NotBefore/IssuedAt/JwtID`；`WithGenerateTokenSignKey/Method/Fields/Claims`；校验/刷新对应 `WithValidateTokenSignKey`、`WithRefreshTokenExpire` 等。

**谁调用**：`pkg/gin/middleware/auth/jwt.go`、`pkg/gin/middleware/jwtAuth.go`、`pkg/grpc/interceptor/jwtAuth.go`。

**关键行为**：

1. `RefreshTwoTokens`：先验 refresh；access 用 **未校验签名** 的 `GetClaimsUnverified` 只比对 `ID` 与 `UID`。不匹配 → `errNotMatch`。refresh 剩余寿命 `<3h` 才续 refresh，access 每次都续。
2. `Claims.NewToken`：若原来设了 NotBefore，刷新时改成 `now`（不是保留原 nbf）。
3. 旧包 `old_jwt`：必须先 `Init`，否则 `errInit`。`Claims` 含 `Name`；另有 `CustomClaims`。默认 key 与新包**不是同一串**。`ParseToken` 验签失败 → `errSignature`。

**失败语义**：签名方法非 HMAC → `unexpected signing method`；过期为 `jwt.ErrTokenExpired`。

**测试文件**：`jwt_test.go`；`old_jwt/jwt_test.go`、`option_test.go`。

---

### pkg/gocrypto

**路径**：`pkg/gocrypto/`（`wcipher/` 为底层模式）

**Why**：提供 AES/DES/RSA/哈希/密码哈希的默认参数封装，避免业务直接拼 cipher。

**公开入口**：`AesEncrypt/Decrypt` 及 Hex 变体；`Des*`；`RsaEncrypt/Decrypt/Sign/Verify` 及 Hex/Base64；`Md5/Sha1/Sha256/Sha512/Hash`；`HashAndSaltPassword`/`VerifyPassword`（bcrypt DefaultCost）；`wcipher.NewAES/NewDES`、`NewCBCMode/ECB/CFB/OFB/CTR`。

**默认**：AES key 16 字节、DES key 8 字节、模式 **ECB**、RSA PKCS#1、哈希 SHA1。全部可 Option 覆盖。

**谁调用**：仓库内无业务调用方（仅 `aes.go`/`des.go` 引用 `wcipher`）。供生成项目选用。

**关键行为**：ECB 不安全，默认却是 ECB。Hex API 对密文做 hex 编解码。RSA 签名哈希默认 SHA1。`VerifyPassword` 只返回 bool，不暴露 bcrypt error。

**测试文件**：`aes_test.go`、`des_test.go`、`rsa_test.go`、`hash_test.go`、`password_test.go`、`wcipher/cipher_test.go`。

---

### pkg/kafka

**路径**：`pkg/kafka/`

**Why**：对 IBM sarama 做同步/异步生产者、消费组/分区消费者、积压查询。

**公开入口**：`InitSyncProducer`/`SyncProducer.SendMessage/SendData/Close`；`InitAsyncProducer`/`AsyncProducer`；`InitConsumerGroup`/`Consume`/`ConsumeLoop`/`ConsumeCustomLoop`；`InitConsumer`/`ConsumePartition`/`ConsumeAllPartition`；`InitClientManager`/`GetBacklog`；大量 `*WithTLS`/`*WithVersion` 等 Option。

**默认同步生产者**：Kafka `V2_1_0_0`、`WaitForAll`、`HashPartitioner`、`Return.Successes=true`、`ClientID=sarama`。

**谁调用**：仓库内无业务 import，仅包内 `*_test.go`。

**关键行为**：

1. `SendData`：支持 `ProducerMessage`/`[]byte`/`string`/`*Message`，其它类型 sonic JSON。
2. `ConsumeLoop`：失败指数退避 1s→30s；`Consume` 返回 nil 视为 rebalance，重置 backoff 再加入。
3. 消费组 handler 在 `autoCommitEnable=false` 时手动 `sess.Commit`。
4. 失败：sarama 构造/发送错误原样返回；loop 里打 zap 后重试，直到 `ctx.Done`。

**测试文件**：`producer_test.go`、`consumer_test.go`、`backlog_test.go`。

---

### pkg/rabbitmq

**路径**：`pkg/rabbitmq/`

**Why**：封装 direct/topic/fanout/headers/x-delayed-message 以及 pub/sub（fanout）和连接自动重连。

**公开入口**：`NewConnection`（默认 URL `amqp://guest:guest@localhost:5672/`，重连间隔 3s）；`NewDirect/Topic/Fanout/Headers/DelayedMessageExchange`；`NewProducer` + `PublishDirect/Fanout/Topic/Headers/DelayedMessage`；`NewConsumer` + `Handler`；`NewPublisher`/`NewSubscriber`；声明/绑定/QoS/死信 Option。`ErrClosed = amqp.ErrClosed`。

**谁调用**：仅包内测试。

**关键行为**：

1. `NewConnection` URL 空 → error；连上后 `monitor` 监听 block/close，断开按 `reconnectTime` 重连。
2. Producer 默认 persistent=true、mandatory=true、publisher confirm 关。
3. Consumer 默认 autoAck=true、persistent=true。手动 ack 时 Handler 返回 error 的处理见 `consumer.go` 消费循环（Nack/Ack）。
4. 延迟消息依赖 `x-delayed-message` 插件。TLS URL（amqps）必须 `WithTLSConfig`。

**失败语义**：声明/发布错误返回；连接断开由 monitor 重连，业务侧需能处理 `ErrClosed`。

**测试文件**：`connection_test.go`、`producer_test.go`、`consumer_test.go`、`publisher_test.go`、`subscriber_test.go`、`common_test.go`。

---

### pkg/sasynq

**路径**：`pkg/sasynq/`

**Why**：对 hibiken/asynq 做 Redis 单机/哨兵/集群切换、类型安全 Task、Zap 日志中间件。

**公开入口**：`RedisConfig`/`RedisMode*`/`GetAsynqRedisConnOpt`；`NewClient`/`Enqueue`/`EnqueueNow`；`NewTask[T]` `RegisterTaskHandler[T]` `HandleFunc`；`NewServer`/`Register`/`Run`/`WaitShutdown`；`DefaultServerConfig`（Concurrency=10，队列 critical:6/default:3/low:1）；`NewScheduler`；`NewInspector`；`LoggingMiddleware`；`WithMaxRetry/Timeout/Deadline/TaskID/Queue`。

**谁调用**：仅包内测试。

**关键行为**：`NewServer` 若 `serverCfg.Config==nil` 则改用 Default。`Run` 失败 **panic**。`WaitShutdown` 听 SIGINT/SIGTERM。payload JSON 反序列化失败作为 handler error（asynq 会重试）。

**测试文件**：`client_test.go`、`server_test.go`、`scheduler_test.go`。

---

### pkg/gocron

**路径**：`pkg/gocron/`

**Why**：对 robfig/cron 做命名任务表，支持只跑一次和秒级粒度。

**公开入口**：`Init`、`Run(tasks ...*Task)`、`IsRunningTask`、`GetRunningTasks`、`DeleteTask`、`Stop`、`EverySecond/Minute/Hour`、`Everyday`、`WithGranularity`、`WithLog`。`Task{TimeSpec, Name, Fn, IsRunOnce}`。

**默认**：`granularity=SecondType`（`cron.WithSeconds()`），zap production logger。

**谁调用**：仅 `cron_test.go`。

**关键行为**：未 Init 就 `Run` → `"cron is not initialized"`。同名任务拒绝。`IsRunOnce` 包装 Fn，执行完 `DeleteTask`。`Everyday(n)` 实际是 `@every {n*24}h`。多任务错误用 ` || ` 拼接。`Stop` 停 cron，不清空 name 映射。

**测试文件**：`cron_test.go`。

---

### pkg/sse

**路径**：`pkg/sse/`

**Why**：给 Gin 提供按 uid 推送的 SSE Hub，可选事件存储与断线重放。

**公开入口**：`Event`/`CheckValid`/`CloseEvent`/`DefaultEventType="message"`；`Store` 接口；`NewHub`（默认 pushBuffer 1000、worker 10、maxRetry 3）；`Hub.Serve`/`Push`/`PushHeartBeat`/`OnlineClientsNum`/`Close`/`PrintPushStats`；`NewClient`；`NewAsyncTaskPool`；`NewSafeMap`；`PushRequest`（HTTP 推送体）。

**谁调用**：仅包内测试（`httpcli` 用于 server_test）。

**关键行为**：

1. `Serve`：uid 空 → 400。设置 `text/event-stream` 头。从 query/header 读 `event_type`/`Last-Event-ID` 做重放（需 Store + `WithEnableResendEvents`）。
2. `Push`：事件无 ID 则生成；写入 `broadcast`；worker 从 pool 推到 `UserClient.Send`；缓冲满走 `pushFailedHandleFn` 或打日志。
3. 注册/注销走 channel，`run` 单 goroutine 维护 SafeMap。
4. Client 断线按 `WithClientReconnectTimeInterval` 重连。

**失败语义**：uid 空拒绝连接；Push 在 hub 已 Close 时返回 error。

**测试文件**：`hub_test.go`、`server_test.go`、`client_test.go`、`event_test.go`、`task_test.go`、`safemap_test.go`、`benchmark_test.go`。

---

### pkg/ws

**路径**：`pkg/ws/`

**Why**：薄封装 gorilla/websocket 的升级与客户端拨号。

**公开入口**：`NewServer(w, r, loopFn, opts...)`、`LoopFn`、`IsClientClose`；`NewClient(url)`；`WithUpgrader`（默认 `CheckOrigin: true` 允许所有来源）、`WithNoClientPingTimeout`、`WithPing`、`WithServerLogger`/`WithClientLogger`。

**谁调用**：仅 `server_test.go`。

**关键行为**：`NewServer` 立刻 Upgrade；`loopFn` 由调用方读消息。`noClientPingTimeout>0` 时设 ReadDeadline。默认允许任意 Origin——生产必须换 Upgrader。`IsClientClose` 判断 close 错误文本。

**测试文件**：`server_test.go`。

---

### pkg/proxykit

**路径**：`pkg/proxykit/`

**Why**：无 Gin 依赖的反向代理内核：健康检查 + 三种负载均衡 + 动态路由管理。`pkg/gin/proxy` 是它的 Gin 外壳（见 [12](12-pkg应用生命周期与Gin.md)）。

**公开入口**：`Backend`/`ParseBackends`/`NewBackend`/`DefaultTransport`；`Balancer`、`NewRoundRobin`、`NewLeastConnections`、`NewIPHash`；`NewProxy`/`ServeHTTP`；`StartHealthChecks`（默认 interval 5s、timeout 2s）；`NewRouteManager`/`Route`/`ManagementRequest`；`Chain`/`SetLogger`；`ErrNoHealthyBackends`。

**谁调用**：`pkg/gin/proxy/proxy.go`。

**关键行为**：

1. `Proxy.ServeHTTP`：`balancer.Next` 失败 → HTTP 503 `"service not available"`；成功则 `IncrementActiveConns`，defer Decrement，再 `httputil.ReverseProxy`。
2. RoundRobin：只在 healthy 列表上原子递增取模。LeastConnections：选 `activeConns` 最小的健康节点。IPHash：`crc32(clientIP)`，IP 从 `X-Forwarded-For` / `X-Real-IP` / `RemoteAddr` 取。
3. 健康检查：对 backend URL 做 TCP 或 HTTP，失败标记 unhealthy。
4. RouteManager 提供管理 HTTP API 动态加路由（`gin/proxy` 挂到 prefix 上）。

**测试文件**：`proxy_test.go`、`balancer_test.go`、`backend_test.go`、`health_test.go`、`router_test.go`、`middleware_test.go`。

---

### pkg/httpcli

**路径**：`pkg/httpcli/httpcli.go`

**Why**：测试和少量内部调用需要“发 JSON、解 JSON”的短客户端，两种风格：链式 `New().SetURL().GET()` 与函数式 `Get/Post/...`。

**公开入口**：`New` `Request` `Response`；`Get/Delete/Post/Put/Patch`；`WithParams/Headers/Timeout`；`StdResult{Code,Msg,Data}`；`KV`。默认 timeout 30s。仅支持 JSON body。

**谁调用**：大量 `pkg/gin/*_test.go`、`internal/handler/*_test.go`、`pkg/sse/server_test.go`。

**关键行为**：函数式 API 要求 HTTP **恰好 200** 否则 `notOKErr`；再 `BindJSON`。非 200 不尝试解析业务码。链式 API 把状态码留在 `Response` 上由调用方处理。`SetBody` 对非 string/[]byte 做 sonic marshal。

**测试文件**：`httpcli_test.go`。

---

### pkg/conf

**路径**：`pkg/conf/parse.go`

**Why**：生成项目 `internal/config.Init` 用 viper 读 yaml/json/toml，并可选热更新。

**公开入口**：`Parse(file, obj, reloads...)`、`ParseConfigData`、`Show`（脱敏）。

**谁调用**：`internal/config/serverNameExample.go`、`serverNameExample_cc.go`、`mixExample/initial/initApp.go`、`commands/generate/template.go`、`perftest/http/agent.go`。

**关键行为**：

1. `obj` 必须非 nil 指针。扩展名决定 `SetConfigType`；**toml 文件名不去掉后缀**，其它格式会去掉后缀当 ConfigName。
2. 有 `reloads` 则 `WatchConfig`：变更时先把 obj 反射重置为零值再 Unmarshal，成功才调 reload。Windows 上可能触发两次。
3. `Show` 默认额外隐藏字段名含 `"dsn"`/`"password"`/`"pwd"` 的行，以及 `user:pass@host` 形式的 DSN 密码。

**失败语义**：读文件/Unmarshal 失败返回 error，不 panic。热更新 Unmarshal 失败只打印，不调 reload。

**测试文件**：`parse_test.go`。

---

### pkg/jy2struct

**路径**：`pkg/jy2struct/`

**Why**：`sponge` 从 yaml/json 配置生成 Go struct 源码（`commands/generate/config.go`）。

**公开入口**：`Convert(*Args)`、`Args{Format,Data,InputFile,Name,SubStruct,Tags}`、`ParseJSON`/`ParseYaml`、`FmtFieldName`、`ForceFloats`。

**谁调用**：`cmd/sponge/commands/generate/config.go`。

**关键行为**：Format 必须 json 或 yaml。json 会 `disambiguateFloatInt`（整数形态的 float64 输出 `int64`）。字段名走 initialism 表（ID/HTTP/URL...）。`SubStruct=true` 时嵌套 map 抽成独立 type，名称用字段名。`readFile` 在 Copy 失败时返回 `[]byte{}, nil`（**吞掉 error**）。输出经 `go/format.Source`。

**测试文件**：`jy2struct_test.go`、`covert_test.go`。

---

### pkg/krand

**路径**：`pkg/krand/krand.go`

**Why**：JWT jti、请求 ID、模板占位需要短随机串和可排序 ID。

**公开入口**：`R_NUM=1` `R_UPPER=2` `R_LOWER=4` `R_All=7`；`String`/`Bytes`；`Int`/`Float64`；`NewID`/`NewStringID`/`NewSeriesID`。

**谁调用**：`pkg/jwt/option.go`、`pkg/gin/middleware/requstid.go`、`pkg/grpc/interceptor/requstid.go`、`cmd/sponge/server/handler.go`、`perftest/*`、`commands/template/protobuf.go`。

**关键行为**：`kind` 用位组合当 `kinds[kind-1]` 下标，非法 kind 当 `R_All`。默认长度 6。`Int()` 无参 0~100。`NewID` = `UnixMilli()*1e6 + rand.Int63n(1e6)`。`NewStringID` 为其 hex。`NewSeriesID` 微秒时间 20 位 + 6 位随机（去掉格式化里的点）。`init` 里 `rand.Seed(Now)`（全局 math/rand，非 crypto）。

**测试文件**：`krand_test.go`。

---

### pkg/process

**路径**：`pkg/process/`

**Why**：UI/`sponge run` 要停掉子进程：先 SIGTERM 等最多 5s，再 SIGKILL。

**公开入口**：`Kill(pid int) error`。Unix：`kill_unix.go`（SIGTERM → SIGKILL，`Kill(pid,0)` 探测，EPERM 视为仍在运行）。Windows：`kill_windows.go`。

**谁调用**：`cmd/sponge/server/handler.go`。

**关键行为**：`pid<1` 立即 error。优雅退出轮询 20×250ms=5s（注释写 10 次，代码是 20）。`forceKill` 失败且进程仍在 → 权限/不存在类 error。

**测试文件**：`kill_test.go`、`kill_unix_test.go`、`kill_windows_test.go`。

---

### pkg/container

**路径**：`pkg/container/group/group.go`

**Why**：熔断器要按路由/方法懒创建、缓存 `CircuitBreaker` 实例。

**公开入口**：`NewGroup(fn)`、`(*Group).Get(key)`、`Reset`、`Clear`。

**谁调用**：`pkg/gin/middleware/breaker.go`、`pkg/grpc/interceptor/breaker.go`。

**关键行为**：`fn==nil` panic。`Get` 双检锁。`Reset` 换 new 函数并 `Clear` 掉已有实例。值类型是 `interface{}`，调用方必须断言。

**测试文件**：`group_test.go`、`example_test.go`。

---

### pkg/utils

**路径**：`pkg/utils/`

**Why**：DSN 适配、端口探测、类型转换、防崩 goroutine，被生成项目和 CLI 同时使用。

**公开入口**：

| 文件 | 符号 |
|---|---|
| `dsn.go` | `AdaptiveMysqlDsn`（去掉 `mysql://`）、`AdaptivePostgresqlDsn`（URL→kv，默认 sslmode=disable，Parse 失败 **panic**）、`AdaptiveSqlite`、`AdaptiveMongodbDsn`、`DeleteBrackets` |
| `host.go` | `GetHostname`、`GetAvailablePort`、`GetLocalHTTPAddrPairs` |
| `type_convert.go` | `StrToInt/Int64/Uint*` 及 `*E` 变体（非 E 失败返回 0）、`IntToStr`、`ProtoInt32ToInt` 等 |
| `time.go` | 若干 Format/Parse（含 RFC3339、带 ms/TZ） |
| `saferun.go` | `SafeRun`（recover 后继续）、`SafeRunWithTimeout` |
| `wait_print.go` | `NewWaitPrinter` 周期打印等待 |
| `browser.go` | `AutoOpenBrowser` |

**谁调用**：`internal/database/*`、`internal/dao`、`internal/handler`（ID 转换）、`sql2code`、`commands/generate/common.go`、大量测试里 `GetLocalHTTPAddrPairs`。

**失败语义**：`AdaptivePostgresqlDsn` 非法 URL panic；非 E 转换静默 0；`SafeRun` 吞 panic。

**测试文件**：`dsn_test.go`、`host_test.go`、`type_convert_test.go`、`time_test.go`、`saferun_test.go`、`wait_print_test.go`。

---

### pkg/gotest

**路径**：`pkg/gotest/`

**Why**：DAO/Handler/Service 单测需要 sqlmock + miniredis + 临时 HTTP/gRPC 端口，避免每个测试文件重复搭建。

**公开入口**：`NewDao(cache, testData)`（sqlmock + gorm mysql dummy + 可选 Cache）、`Dao.Close` `GetAnyArgs` `AnyTime`；`NewCache`/`NewRCCache`（miniredis）；`NewHandler`（Gin Engine + 动态端口）；`NewService`（grpc.Server）。

**谁调用**：`internal/dao/userExample_test.go`、`internal/handler/*_test.go`、`internal/service/*_test.go`、`internal/cache/*_test.go`、`pkg/cache/*_test.go`。

**关键行为**：`NewDao` 失败 panic。`GetAnyArgs` 按结构体字段生成 sqlmock 参数，时间字段用 `AnyTime` matcher。`Handler` 用 `utils.GetLocalHTTPAddrPairs`。`Service` 起 grpc 并返回 ClientConn。

**测试文件**：`dao_test.go`、`cache_test.go`、`handler_test.go`、`service_test.go`。

---

### pkg/replacer

**路径**：`pkg/replacer/`（主链 [05](05-代码生成器与模板写入.md)）

**Why**：代码生成的唯一写盘引擎：列模板文件、替换 Field、尊重 ignore、处理 `.tpl/.exp/.mgo` 变体。

**公开入口**：`Replacer` 接口；`New(path)`（os 文件系统）；`NewFS(path, embed.FS)`；`Field{Old,New,IsCaseSensitive}`；`SetReplacementFields`、`SetSubDirsAndFiles`、`SetIgnoreSubDirs/Files`、`SetOutputDir`、`SaveFiles`、`ReadFile`、`GetFiles`、`SaveTemplateFiles`。

**谁调用**：全部 `commands/generate/*.go`。

**关键行为（入口级）**：`IsCaseSensitive` 且首字母为字母时拆成大小写两对 Field。`SaveFiles` 按 ignore 跳过、已存在文件可取消生成（见 05）。本篇不重复模板变体规则。

**测试文件**：`replacer_test.go`（夹具 `testDir/`）。

---

### pkg/encoding

**路径**：`pkg/encoding/`（README 将其列为其余一级包，12/13/14 未展开）

**Why**：cache 层要把任意对象变成 `[]byte`；同时给 gRPC 提供 json codec 注册。

**公开入口**：`Codec`/`RegisterCodec`/`GetCodec`；`Encoding`/`Marshal`/`Unmarshal`（要求指针，否则 `ErrNotAPointer`；失败时回退 `BinaryMarshaler`）；`JSONEncoding`/`JSONGzipEncoding`/`JSONSnappyEncoding`/`GobEncoding`/`MsgPackEncoding`；`encoding/json` 子包 `init` 注册 name=`json` 的 protojson codec（`EmitUnpopulated`、`DiscardUnknown`）；`encoding/proto`。

**谁调用**：`pkg/cache/redis.go`、`memory.go`；json/proto 子包自注册。

**关键行为**：`Marshal` 若 `e==nil` 且实现了 `BinaryMarshaler` 则走标准库。gzip/snappy 包一层压缩。cache 的 `newObject` 用于 Unmarshal 到具体类型。

**测试文件**：`encoding_test.go`、`json/json_test.go`、`proto/proto_test.go`。

---

## 调用关系表

下表覆盖 logger / tracer / shield / errcode 主路径。更多 Gin/gRPC 装配见 [09](09-生成项目启动与HTTP请求链.md)、[12](12-pkg应用生命周期与Gin.md)、[13](13-pkg-gRPC与服务注册发现.md)。

| 调用方文件与符号 | 关系 | 被调用方文件与符号 | 触发与输入 | 返回与后续处理 | 错误、状态与副作用 |
|---|---|---|---|---|---|
| `cmd/serverNameExample_httpExample/initial/initApp.go:InitApp` | 调用 | `pkg/logger.Init` | 配置 `Logger.Level/Format/IsSave` | 全局 zap；`logger.Info` 打印完成 | Init 失败 panic；未调用时后续日志会默认 Init |
| `InitApp` | 条件调用 | `pkg/tracer.InitWithConfig` | `EnableTrace` 及 Jaeger host/port、采样率 | 全局 TracerProvider + propagator | Agent exporter 失败 panic |
| `InitApp` | 条件调用 | `pkg/stat.Init` | `EnableStat`；`WithLog(logger.Get())` `WithAlarm()` | 后台 ticker 打 statistics | Windows 上 Alarm 不启用；过载 SIGTRAP |
| `pkg/app/app.go:watch` | 调用 | `pkg/prof.(*Profile).StartOrStop` | 收到 SIGTRAP | 开始或停止写 profile 文件 | 采样错误只打印；默认 60s 超时停止 |
| `internal/routers/routers.go:NewRouter` | 条件调用 | `pkg/gin/middleware.RateLimit` | `EnableLimit` | 包级单一 `BBR` | 超限 HTTP 429 Abort |
| `middleware.RateLimit` | 调用 | `pkg/shield/ratelimit.NewLimiter` / `(*BBR).Allow` | 默认 10s/100/800 | `DoneFunc` 或 `ErrLimitExceed` | 放行后 `inFlight++`，Next 后记 RT 与 pass |
| `NewRouter` | 条件调用 | `middleware.CircuitBreaker` | `EnableCircuitBreaker` | 按 `FullPath` 懒创建 Breaker | 拒绝 HTTP 503；业务 `response.Error` 的 HTTP 200 记成功 |
| `middleware.CircuitBreaker` | 调用 | `pkg/container/group.(*Group).Get` → `circuitbreaker.NewBreaker` | key=`c.FullPath()` | 每路由一个 SRE Breaker | 本地拒绝 `MarkFailed`（Add 0） |
| `pkg/grpc/interceptor.UnaryServerRateLimit` | 调用 | `ratelimit.(*BBR).Allow` | 与 HTTP 相同默认参数 | handler 前后 Done | 失败 `errcode.StatusLimitExceed.ToRPCErr` |
| `interceptor.UnaryServerCircuitBreaker` | 调用 | `Breaker.Allow/Mark*` | key=`info.FullMethod` | 可 `unaryServerDegradeHandler` | 失败 `StatusServiceUnavailable.ToRPCErr`；**成功不 MarkSuccess** |
| `pkg/gin/response.Error` | 调用 | `errcode` 字段 `Code/Msg` | handler 传入 `*Error` | JSON `{code,msg,data}` HTTP 200 | 不改 HTTP 状态；与熔断 Writer.Status 脱节 |
| `response.Output` | 调用 | 调用方传入 HTTP 码 | 限流 429、熔断 503、`ToHTTPCode()` | 直接写状态码 | 熔断能看到 500/503 |
| `internal/ecode/systemCode_http.go` | 再导出 | `errcode` 系统码 | 生成时复制 | handler 只 import ecode | 重复 NewError 会在 init panic |
| `errcode.NewResponser` | 实现 | `defaultResponse.Success/Error` | pb 路由 `isFromRPC` | 网关把 gRPC status 转 HTTP | Unknown 解析 `code=, msg=` 文本 |
| `initial.Close` | 调用 | `tracer.Close` `logger.Sync` | 进程退出 | Shutdown 导出剩余 span | tracer 未 Init 则 Close 返回 nil |

## 阅读源码建议顺序

1. `initial/initApp.go` + `close.go`：看 logger/tracer/stat 如何开关。
2. `pkg/logger/logger.go` → `option.go` → `method.go`。
3. `pkg/tracer/tracerProvider.go` → `jaeger.go` → `span.go`。
4. `pkg/shield/window` → `cpu/stat.go` → `ratelimit/bbr.go` → `circuitbreaker/sre.go`。
5. `pkg/gin/middleware/ratelimit.go` 与 `breaker.go`，对照 `internal/routers/routers.go` 的配置开关。
6. `pkg/errcode/http_error.go` + `http_system_code.go` + `response.go`，对照 [09](09-生成项目启动与HTTP请求链.md) 的 handler 出口。
7. 按需：`jwt`（鉴权中间件）、`aicli`（assistant CLI）、`replacer`/`goast`（生成链 05/07）、消息与 SSE 包（本仓库无运行时强制依赖）。

## 重新实现检查清单

- [ ] 全局 logger：未 Init 能默认打终端；文件用 lumberjack；`Sync` 忽略 stdout 错误。
- [ ] Tracer：采样率三分法；Jaeger Agent 失败即无法启动；`Close` 可空操作。
- [ ] stat：默认 1 分钟打印；告警三次均值、900s 冷却、非 Windows SIGTRAP。
- [ ] prof：SIGTRAP 切换采样；文件落在 TempDir；HTTP mux 与 Gin 两套 Register 不要混。
- [ ] BBR：CPU EMA decay=0.95；阈值默认 800（千分制）；窗口 10s/100 桶；`maxInFlight` 公式与 1s 冷却；`DoneInfo.Err` 可忽略。
- [ ] SRE：默认 success=0.6、request=100、window=3s、bucket=10；`k=1/success`；失败 Add(0)；HTTP 按 **Writer.Status** 记账。
- [ ] errcode：四段码空间；重复 code panic；`HCode/RCode` 仅 1~999；Responser 区分 isFromRPC。
- [ ] JWT：覆盖默认密钥；双 token 校验 jti+uid；refresh 剩余 <3h 才续 refresh。
- [ ] 生成器仍用 `replacer.Replacer` + `goast.MergeGo*`，不要在本层改算法。
- [ ] 用各包 `*_test.go` 的用例名做验收（BBR 并发、SRE markSuccess/Failed、errcode ParseError、jwt 双 token），不要宣称未运行的测试已通过。
