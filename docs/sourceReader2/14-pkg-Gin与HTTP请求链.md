# 14. `pkg/gin` 与 HTTP 请求链

## 1. 边界：`gin` 不是 HTTP Server

`pkg/gin` 目录自身没有根包入口，它按子包提供中间件、响应、校验、静态文件、代理等积木；真正监听端口、实现 `app.IServer` 的工作主要在 `pkg/httpsrv`。业务项目创建 `*gin.Engine`，注册中间件和路由，再交给 HTTP Server。

```text
TCP listener / http.Server
  → Gin Engine
    → RequestID
    → Logging / Metrics / Tracing
    → CORS / Auth / RateLimit / CircuitBreaker
    → generated Handler
    → response.Success/Error
    → HTTP response
```

中间件顺序不是装饰：进入请求时从左到右，`c.Next()` 返回后从右到左。request ID 必须早于依赖它的日志；recovery 应包住可能 panic 的代码；鉴权应早于昂贵业务逻辑。

## 2. 上下文与请求 ID

`middleware.RequestID` 从请求头读取或生成 ID，同时写入 Gin context、Go `context.Context` 和响应头。`GCtxRequestID`、`HeaderRequestID`、`CtxRequestID` 及对应 `Field` 帮助各层统一取值。

`WrapCtx(c)` 把 Gin 请求信息桥接为标准 context，便于 DAO/gRPC client 不依赖 Gin。`AdaptCtx` 做反向适配。重实现时应只把 request-scoped 的小值放入 context，不能把数据库连接、配置单例塞进去；context key 要用私有类型避免冲突。

## 3. 日志、指标与追踪

### `middleware.Logging`

记录方法、路径、状态码、耗时、客户端等信息，可配置 logger、最大 body 长度、忽略路由、哪些错误码重点输出，以及 request ID 来源。`SimpleLog` 是精简版。body 读取后若不恢复，请求处理器会读不到，因此重实现要特别测试 body 可再次读取，并对密码/token 做脱敏。

### `middleware/metrics`

`Metrics(r, opts...)` 暴露 Prometheus 指标并返回请求中间件，支持修改 metrics 路径和忽略状态码、路径、方法。标签不能直接使用无限多的原始 URL，否则动态 ID 会造成高基数；应使用 Gin 的路由模板。

### `middleware.Tracing`

用 OpenTelemetry 从入站 header 提取父 span，上报 HTTP span，再把新 context 传给下游。可注入 propagator 和 provider。服务名决定 trace 归属；关闭阶段必须 flush tracer provider。

## 4. 鉴权和会话

仓库存在两套 JWT 包装：`middleware/jwtAuth.go` 直接基于 `pkg/jwt`，以及 `middleware/auth/jwt.go` 提供初始化、签发、解析、刷新和 Gin 中间件的较完整组合。公开能力包括 `InitAuth`、`GenerateToken`、`ParseToken`、`RefreshToken`、`Auth`、`GetClaims`。

请求链是：读取 `Authorization` → 检查 scheme → 验签及过期时间 → 可选额外校验 → claims 写入 Gin context → 业务读取。`WithAuthIgnoreMethods` 的 gRPC 对应物按完整方法名忽略；HTTP 版应谨慎开放路由。

`auth/session.go` 解码 Rails signed cookie 并提取 user ID；`middleware/session.go` 包装成中间件。会话密钥是安全资源，不可记录日志，轮换时应考虑旧 key 的过渡期。

## 5. 保护中间件

- `Cors`：包装 `gin-contrib/cors`，可配置 origin/method/header/credential/max-age。允许 cookie 时不能随意使用 `*` origin。
- `RateLimit`：基于 `shield/ratelimit` 的自适应 BBR 限流；成功进入后要调用返回的 `DoneFunc` 反馈耗时和结果。
- `Timeout`：限制 handler 执行时间。超时响应不等于业务 goroutine 已停止，下游必须尊重 context cancellation。
- `CircuitBreaker`：按分组获取 SRE breaker，根据响应状态反馈成功/失败；熔断时执行降级 handler 或返回错误。分组 key 应稳定且有界。

## 6. 输出与输入校验

### `response`

`Result` 是统一 envelope。`Output` 写指定业务码，`Out` 接收 `*errcode.Error`，`Success`/`Error` 是常用途径。HTTP 状态码和业务错误码是两个维度；重实现时先规定映射，不要所有失败都返回 HTTP 200，也不要把内部 error 原文泄露给客户端。

### `validator`

`Init`/`NewCustomValidator` 基于 go-playground validator，为 Gin binding 提供自定义校验和错误翻译。校验属于边界层：格式不合法直接拒绝；跨表存在性等业务规则仍应在 service/DAO 中验证。

### `handlerfunc`

提供健康检查、ping、错误码列表、SPA 浏览器刷新回退。健康检查应区分 liveness 与 readiness：前者只判断进程活着，后者检查是否能接流量。

## 7. 静态内容、Swagger 与代理

- `frontend.FrontEnd`：可从磁盘或 embed FS 提供前端资源，支持指定文件内容改写和 404 回首页；适合 SPA。
- `staticfs.StaticFS`：目录静态服务，支持 index、缓存时长/容量和中间件。
- `staticfs.ListDir`：文件列表和下载，可设置路径前缀、文件/目录过滤。必须防目录穿越。
- `swagger.DefaultRouter/CustomRouter`：从字节或文件暴露 Swagger UI/JSON。生产环境可关闭或鉴权。
- `proxy.Proxy`：在 Gin 上建立反向代理，支持管理端点、日志、负载策略、健康检查和每条 pass 的中间件。代理要正确处理转发头、hop-by-hop header、超时和后端健康状态。

## 8. `httpsrv` 与资源生命周期

`pkg/httpsrv` 把 `http.Server` 适配成可启动/停止的服务，并提供 option 设置地址、handler、读写超时、TLS 等。创建者拥有 listener/server；`Start` 阻塞服务，`Stop` 应带超时 context 调 `Shutdown`，让在途请求完成。它通常被放入 `app.New` 的 servers，关闭函数放入 closes。

## 9. 新手重实现路线

1. `gin.New()` + recovery，注册 `/health`。
2. 写最小 `Result` 和 `Success/Error`。
3. 加 request ID，并在 handler、service、DAO 中传同一个 context。
4. 加结构化日志，验证一次请求只有明确的入口/错误日志。
5. 加 validator 和 JWT。
6. 加 metrics/tracing，确认 label 和 span 名称有界。
7. 最后实现限流、熔断、静态资源和代理。

每一步至少用 `httptest` 验证：正常、非法输入、未授权、handler panic、超时、并发。仓库各子包的 `*_test.go` 基本对应公开入口，是理解预期行为的最佳示例。

## 10. 常见失败

- Logging 在 RequestID 前注册，日志缺关联 ID。
- 中间件 `Abort` 后仍继续写响应。
- 读取 body 后没有放回 `Request.Body`。
- 把 Gin context 保存到 goroutine，在请求结束后继续使用。
- trace 初始化了但退出时没有 `Close`，尾部 span 丢失。
- `/metrics`、pprof、Swagger、目录列表暴露到公网。
