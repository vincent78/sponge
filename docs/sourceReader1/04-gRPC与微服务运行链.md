# 04. gRPC 与微服务运行链

## 契约优先

`api/serverNameExample/v1/userExample.proto` 同时描述：

- service 和 RPC 方法；
- request/reply message；
- validate 校验规则；
- `google.api.http` 到 REST 路由的映射；
- OpenAPI 文档信息；
- tagger 为 Gin 参数绑定补充的 tag。

运行时使用的 `*.pb.go`、`*_grpc.pb.go`、validate、gateway、swagger 文件都应从 proto 生成。修改接口时先改 proto，再运行项目 Makefile 中的 proto 目标。

## 启动和注册

gRPC main 与 HTTP main 同样经过 `initial.InitApp → CreateServices → app.Run`。区别是创建 `internal/server.NewGRPCServer`。

`internal/service/register.go`（及各实体 service 的 `init()`）维护注册函数切片。`userExample.go` 导入时追加：

```text
grpc.Server → RegisterUserExampleServer(server, NewUserExampleServer())
```

这与 HTTP 的 router 注册模式一致：每个业务文件自注册，中心 server 只遍历函数，不依赖具体实体。

## gRPC server 的组成

`internal/server/grpc.go` 的 `grpcServer` 同时持有 listener、`grpc.Server`、可选 metrics/pprof HTTP server 和 registry 信息。

启动时：

1. 可选注册服务实例；
2. 注册 metrics handler；
3. 若开启 metrics/pprof，另起一个 HTTP server；
4. 用带连接指标的 listener 执行 `grpc.Server.Serve`。

停止时先从 registry 注销，再 `GracefulStop`，最后关闭附属 HTTP server。

## 拦截器链

Unary 默认包含 recovery、request ID、logging；按配置追加 token/JWT、metrics、rate limit、circuit breaker、tracing。Stream 有对应的 stream interceptor 版本。

拦截器相当于 Gin middleware：处理横切能力，service 只保留业务编排。顺序同样重要，例如 recovery 应覆盖后续 handler，日志和 tracing 需要上下文中的 request ID。

TLS 支持 insecure、单向认证和双向认证，实际 credentials 由 `pkg/grpc/gtls` 构造。

## 一次 RPC 请求

以 `Create` 为例：

1. generated gRPC stub 解码请求并进入 interceptor 链。
2. `req.Validate()` 执行 proto 上声明的规则。
3. `interceptor.WrapServerCtx` 保留 request ID/trace 信息。
4. `copier.Copy` 把 protobuf request 转为 model。
5. service 调 DAO；DAO/cache/database 与 HTTP 共用。
6. 业务错误通过 `internal/ecode` 转成 gRPC status。
7. model 再转换为 protobuf reply 返回。

HTTP handler 与 gRPC service 是两种适配器，共享数据层而不互相调用。这是生成项目最重要的分层边界。

## Gateway 与混合模式

- `grpc-pb`：纯 gRPC。
- `grpc-gw-pb`：由 grpc-gateway 根据 `google.api.http` 暴露 HTTP/JSON。
- `grpc-http` / `grpc-http-pb`：在一个工程中组合 gRPC 与 HTTP 能力。
- `http-pb`：仍使用 Gin transport，但请求/响应类型来自 protobuf。

这些模式主要改变启动入口、server 组合、路由适配和生成文件选择；service/DAO 的核心思路保持一致。

## 服务注册与发现

`pkg/servicerd/registry` 抽象 Registry/Watcher，提供 etcd、consul、nacos 实现；server 在 Start/Stop 注册与注销。`pkg/servicerd/discovery` 把注册中心 watcher 接到 gRPC resolver/balancer，使客户端按服务名发现实例。

是否启用由 `App.RegistryDiscoveryType` 等配置控制。模板中 `.noregistry` 变体用于完全不引入这条链的简单服务。
