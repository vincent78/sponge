# 03. HTTP 服务运行链

## 启动阶段

以 `cmd/serverNameExample_httpExample/main.go` 为入口：

```text
initial.InitApp()
  → 解析 -c/-version
  → config.Init(YAML)
  → logger.Init
  → 可选 tracer/stat
  → database.InitDB / InitCache

initial.CreateServices()
  → server.NewHTTPServer(...options)
  → routers.NewRouter()

app.New(services, closes).Run()
  → errgroup 并发 Start
  → 监听 OS signal/服务错误
  → 顺序执行 Stop、关闭 DB/Redis/tracer/logger
```

`pkg/app` 定义很小的 `IServer`：`Start`、`Stop`、`String`。因此 HTTP、gRPC 或组合服务都能用相同生命周期管理。任一服务返回错误会取消 errgroup context，watch 随即关闭全部资源。

## HTTP server

`internal/server/http.go` 的 `httpServer` 实现 `app.IServer`：

- `Start`：若配置 registry，先注册实例，再阻塞运行 `httpsrv.Server`。
- `Stop`：注销实例，然后带超时执行 HTTP graceful shutdown。
- `newServer`：按 TLS 配置选择普通 HTTP、自签名、自动证书或外部证书。

`HTTPOption` 隔离生产模式、TLS、注册发现等可选依赖。构造函数内部设置 Gin mode、创建 router，再交给标准库 `http.Server`。

## 路由和中间件顺序

`internal/routers/routers.go` 使用 `gin.New()`，中间件按注册顺序包裹请求：

1. Recovery、CORS；
2. 可选全局 timeout；
3. RequestID；
4. Logging；
5. 可选 metrics、rate limit、circuit breaker、tracing、pprof；
6. `/health`、`/ping`、`/codes`；
7. 非生产环境的 `/config`、Swagger；
8. `/api/v1` 业务路由。

中间件顺序会影响日志是否拿到 request ID、panic 是否被捕获以及 tracing 覆盖范围，修改时不能随意重排。

业务路由使用自动注册：`internal/routers/userExample.go` 的 `init()` 把闭包追加到 `apiV1RouterFns`。`NewRouter()` 最后统一遍历。增加生成模块时只要文件被编译进包，路由就会注册，不需要手改中心大 switch。

## 一次 Create 请求

以 `POST /api/v1/userExample` 为例：

1. Gin 匹配到 `handler.UserExampleHandler.Create`。
2. `ShouldBindJSON` 把 body 绑定到 `types.CreateUserExampleRequest` 并执行 tag 校验。
3. `copier.Copy` 转为 `model.UserExample`，传输层与持久化层解耦。
4. `middleware.WrapCtx(c)` 把 Gin 上下文信息放入标准 `context.Context`。
5. handler 调用 `iDao.Create`。
6. DAO 操作 GORM/MongoDB，并按策略删除或回填缓存。
7. `response.Success` 输出统一成功结构；失败走 `ecode`/`errcode` 到统一 HTTP code 和业务 code。

构造 handler 时，`NewUserExampleHandler()` 创建 DAO，并把 `database.GetDB()` 与业务 cache 注入。接口字段 `iDao` 让测试可以替换实现。

## DAO、cache 与 database

- `internal/database`：进程级连接初始化和 getter，知道具体 driver/连接池。
- `internal/dao`：实体级查询语义，如 Create、UpdateByID、GetByColumns；不解析 HTTP。
- `internal/cache`：实体缓存 key、序列化和失效策略；可选 memory/redis。
- `internal/model`：数据库结构和 GORM/BSON tag。

读操作常见模式是 cache-aside：先读缓存，未命中查 DB 后写缓存。写操作要更新或删除关联缓存。具体模板可能因 driver 和扩展 API 不同，因此排查脏数据时同时阅读 DAO 和 cache，而不是只看 handler。

## 错误处理边界

参数错误属于调用方问题，直接映射 `InvalidParams`；未找到记录需要识别 `database.ErrRecordNotFound`；其余 DB 错误记录详细日志，但响应对外隐藏内部细节。请求 ID 被加入日志字段，便于跨层关联。

`pkg/errcode` 提供通用错误模型与响应转换，`internal/ecode` 保存项目业务错误。前者可复用，后者由生成器按实体扩展。
