# 13. `pkg` 总览与应用生命周期

> 本章回答两个问题：`pkg` 为什么存在，以及一个 Sponge 服务从启动到退出时这些包怎样协作。阅读对象是假设只学过 Go 基础、还没有独立写过后端服务的新手。

## 1. 先建立正确认识

`pkg` 不是一个可以单独启动的应用，而是一组可被生成项目复用的运行时积木。生成器把业务骨架写到项目里，业务代码再调用这里的数据库、缓存、HTTP、gRPC、日志、追踪等能力。

可以把它分为六层：

| 层次 | 主要包 | 解决的问题 |
|---|---|---|
| 生命周期 | `app`、`process`、`prof` | 同时启动多个服务器，监听退出信号，释放资源 |
| 接入与通信 | `gin/*`、`httpsrv`、`grpc/*`、`httpcli`、`sse`、`ws`、`proxykit` | 收发 HTTP、gRPC、SSE、WebSocket 请求 |
| 数据 | `sgorm/*`、`mgo/*`、`cache`、`goredis` | SQL/MongoDB/Redis/内存缓存访问 |
| 分布式组件 | `servicerd/*`、`etcdcli`、`consulcli`、`nacoscli`、`dlock`、`kafka`、`rabbitmq`、`sasynq` | 注册发现、锁、消息和异步任务 |
| 横切能力 | `logger`、`tracer`、`shield/*`、`stat/*`、`errcode`、`jwt` | 日志、链路、保护、指标、错误和身份 |
| 开发工具 | `conf`、`encoding`、`copier`、`goast`、`gofile`、`gotest`、`replacer`、`sql2code` 等 | 配置、序列化、代码生成与测试辅助 |

依赖方向应尽量是“业务层 → `pkg` 抽象 → 第三方库”。不要让 `pkg` 反向依赖某个具体业务模块。

## 2. 一次完整启动过程

典型的组合顺序如下：

```text
读取配置 conf
  ├─ 初始化 logger（拿到 *zap.Logger）
  ├─ 初始化 tracer（安装全局 TracerProvider）
  ├─ 初始化 sgorm / mgo / goredis
  ├─ 初始化 registry/discovery
  ├─ 创建 HTTP Server（Gin 路由 + 中间件）
  ├─ 创建 gRPC Server（服务注册 + interceptor）
  └─ app.New(servers, closes).Run()
        ├─ errgroup 并发调用每个 server.Start
        ├─ watch 等待系统信号或任意服务报错
        └─ 顺序执行 closes，释放数据库、缓存、追踪等资源
```

这里要分清两种“注册”：

1. `registerFn(*grpc.Server)` 把生成的 gRPC 实现绑定到进程内的 gRPC Server。
2. `serviceRegisterFn()` 把本进程的地址登记到 etcd/Consul/Nacos，使别的进程能找到它。

## 3. `pkg/app`：进程总管

### 3.1 公开契约

`IServer` 只有三个方法：

```go
type IServer interface {
    Start() error
    Stop() error
    String() string
}
```

`App` 保存 `servers []IServer` 和 `closes []Close`。`New` 只是装配，真正工作发生在 `Run`。

### 3.2 `Run` 的数据流

`Run` 创建 `errgroup.WithContext`，为每个 server 启一个 goroutine，打印 `String()` 后调用 `Start()`；另一个 goroutine 执行 `watch(ctx)`。任何 server 返回错误，errgroup 的 context 会取消，`watch` 进入清理路径。最后 `eg.Wait()` 返回错误时直接 panic。

新手容易误解的地方：

- `Start` 通常应阻塞到服务器停止；如果它启动 goroutine 后立刻返回 `nil`，errgroup 会把它当作任务已结束。
- `IServer.Stop` 在 `app.go` 中并未直接遍历调用，停止动作通常作为 `Close` 传入。因此装配时不能漏掉 server 的关闭函数。
- `closes` 按传入顺序执行，并非逆序；任意一个返回错误后，后续清理不会继续。
- `SIGTRAP` 不退出，而是调用 `prof.Profile.StartOrStop()` 切换性能采样。

### 3.3 退出触发源

`watch` 同时监听：

- errgroup context：某个服务失败；
- `SIGINT`：终端 Ctrl+C；
- `SIGTERM`：容器编排常用的优雅终止；
- `SIGHUP`：也被当作退出；
- `SIGTRAP`：切换 profile。

`signal.Notify` 的 channel 容量为 1，可避免信号到来时发送方被阻塞。清理后 `watch` 返回，进而让 `Run` 结束。

### 3.4 谁使用它、资源归谁

它通常由生成项目的 `cmd/<server>/main.go` 或启动装配层使用。`app` 不创建数据库和服务器，因此不拥有具体资源；创建者必须把正确的 `Close` 闭包交给它。一个稳妥的关闭顺序是：先停止入口流量，再注销服务，再等在途请求，再关闭数据库/Redis/trace exporter，最后同步日志。

### 3.5 从零重实现

先实现 `IServer` 与单服务启动，再加入多服务 errgroup，再加入信号监听，最后加入幂等清理。建议改进点是：`sync.Once` 保证只清理一次、聚合所有 close 错误、为优雅退出设置超时、明确反向关闭顺序，并避免在库内部 panic。

### 3.6 测试

`app_test.go` 用假的 server/close 验证正常和错误路径。重实现至少测试：所有 Start 被调用、一个 Start 失败会触发清理、close 错误可观察、重复信号不会重复释放。

## 4. 生命周期辅助包

### `process`

跨平台进程终止工具。公开入口 `Kill(pid)`，Unix 和 Windows 分文件实现“是否存活、尝试优雅退出、必要时强杀”。调用者是 CLI/开发工具，不应拿它代替服务自身的优雅退出。测试覆盖非法 PID、不存在进程、优雅退出成功和强杀兜底；重实现时必须保持平台 build tags，避免误杀 PID 复用后的新进程。

### `prof` 与 `gin/prof`

`prof` 负责 CPU/heap 等采样文件的开启/停止，`app` 用 `SIGTRAP` 控制它；`gin/prof.Register` 把 pprof 和可选 IO wait 端点挂到 Gin。生产环境必须加鉴权或仅监听管理网，采样文件句柄必须关闭。

### `conf`

配置读取与热更新辅助。它通常最早初始化，输出会被数据库、日志、服务端使用。重实现应区分：默认值、文件值、环境覆盖、解析校验、动态更新；配置对象发布给并发读者时要不可变或原子替换。测试应覆盖缺失文件、非法格式和覆盖优先级。

## 5. 推荐阅读和重实现顺序

1. 用假 `IServer` 重写 `app`，理解资源所有权。
2. 只启动一个无中间件的 HTTP 服务。
3. 加 logger、request ID 和统一 response。
4. 加 GORM，并实现一个 CRUD 请求。
5. 加 cache-aside，区分未命中与空值占位符。
6. 加 gRPC client/server 和 interceptor。
7. 最后加注册发现、tracing、限流和熔断。

不要一开始复制所有 option。每加一个 option，都应回答：默认值是什么、谁消费它、错误如何返回、资源由谁关闭。

## 6. 本章源码检查点

- `pkg/app/app.go`：启动、信号和清理的完整实现。
- `pkg/app/app_test.go`：最小可测试替身。
- `pkg/process/kill*.go`：跨平台差异。
- `pkg/prof/*`、`pkg/gin/prof/prof.go`：进程信号和 HTTP 管理端点两种控制面。
