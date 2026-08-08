# 10 gRPC 服务、Gateway 与 RPC 客户端

## 1. Proto 是契约源头

`api/serverNameExample/v1/userExample.proto` 定义 package、Go package、service、RPC、message、校验和 HTTP 映射。生成物职责：

- `userExample.pb.go`：message/enum；
- `userExample_grpc.pb.go`：client、server 接口及注册函数；
- `userExample.pb.validate.go`：`Validate()`；
- `userExample_router.pb.go`：Gin/HTTP 路由胶水；
- gateway 相关代码把 HTTP/JSON 转发为 gRPC。

不要直接改 `.pb.go`；应修改 proto 后重新生成。

## 2. Proto 的 REST 映射

Create 的 `google.api.http` 指定 POST `/api/v1/userExample` 且 `body:"*"`；Delete/Get 的 `{id}` 来自路径；Update 同时有路径 id 和 body；List 是 POST body。字段校验如 `uint64.gte=1` 在生成的 `Validate` 中执行。

`api/types/types.proto` 复用分页查询：page、limit、sort、columns；Column 含 name、exp、value、logic。它很灵活，因此数据层必须用字段白名单和有限表达式防 SQL 注入。

## 3. NewGRPCServer 的组装

构造流程可以概括为：

```text
net.Listen
 -> grpc.NewServer(
      TLS credentials,
      ChainUnaryInterceptor(...),
      ChainStreamInterceptor(...))
 -> RegisterUserExampleServer(NewUserExampleServer())
 -> reflection/health 等附属注册
```

TLS type：空值为 insecure，`one-way` 为服务端证书，`two-way` 同时校验客户端 CA。配置错误会在启动期 panic，属于 fail-fast。

## 4. 拦截器链

Unary 默认 Recovery、RequestID、Log；按开关追加 Token、Metrics、RateLimit、CircuitBreaker、Tracing。Stream 默认 Recovery、Log，也可追加 Token/Metrics 等。拦截器等价于 HTTP middleware，但方法名、metadata 和 status code 都属于 gRPC 语义。

Token 示例比较固定 appID/appKey，只是教学占位。断路器通过 `ecode.StatusInternalServerError`、`StatusServiceUnavailable` 等 code 判定失败。metrics 开启时还会注册 `/metrics` 的 HTTP mux。

## 5. Service 的业务流程

`internal/service/userExample.go` 实现生成的 `UserExampleServer` 接口：

1. `req.Validate()`；
2. `interceptor.WrapServerCtx(ctx)` 将 gRPC metadata 中 request ID 等写入 context；
3. protobuf request 经 copier 转 model；
4. DAO 操作；
5. model 转 protobuf reply；
6. 返回 gRPC error code。

`internal/service/service.go` 用来集中登记服务，便于 server 构造器逐个注册。编译期断言 `var _ XxxServer = (*userExample)(nil)` 能在接口变化时立即报错。

## 6. gRPC Gateway 的两段链路

```text
HTTP JSON
 -> gateway HTTP server
 -> 根据 google.api.http 匹配
 -> JSON/proto 转换
 -> gRPC client call
 -> gRPC server interceptor
 -> service -> dao
 -> protobuf reply
 -> gateway 转 JSON
```

Gateway 与“Proto + Gin”不要混淆：前者是独立 HTTP-to-gRPC 代理链，业务只实现在 gRPC service；后者由 Gin 生成胶水直接调用 Logicer。Gateway 多一次序列化/网络边界，但能确保 HTTP 与 gRPC 共用同一业务入口。

## 7. rpcclient 的连接建立

`internal/rpcclient/serverNameExample.go` 从 `cfg.GrpcClient` 按 name（大小写不敏感）寻找目标，找不到就 panic。默认 endpoint 是 `host:port`。client option 依次开启 RequestID、日志、TLS、token，再按全局开关增加 trace、熔断、metrics，按 client 配置增加 timeout。

服务发现示例被注释：启用后 endpoint 变为 `discovery:///serviceName`，注入 Consul/Etcd/Nacos discovery，并开启负载均衡。连接由 `sync.Once` 惰性建立，`CloseServerNameExampleRPCConn` 在退出阶段释放。

## 8. 错误码的双世界

HTTP 使用 `ErrCode`/HTTP code，gRPC 使用 status code + 业务 code。`internal/ecode/systemCode_rpc.go` 提供 StatusInvalidParams、StatusNotFound、StatusInternalServerError；实体文件提供 Create/Update/List 等业务错误。Service 里校验错误调用 `.Err()`，内部错误常用 `.ToRPCErr()`，保证客户端收到标准 gRPC status。

不要返回原始 GORM error；否则客户端依赖实现细节并可能看到敏感 SQL。

## 9. 重实现与验证

1. 先只写一个 Ping proto，生成 message/grpc 文件。
2. 实现 server 接口并用 `bufconn` 做无端口单测。
3. 加 Validate，再覆盖非法参数测试。
4. 加 recovery/requestID/log 拦截器，验证 metadata 传播。
5. 加业务错误到 gRPC status 的映射测试。
6. 再加 gateway，使用 `httptest` 验证 JSON 名称、path id、HTTP status。
7. 最后加入 TLS、token、发现、熔断、metrics；每项分别测试，避免一次启用后无法定位握手失败。

重点测试：服务未注册、请求超时、客户端取消、not found、panic recovery、无效 token、TLS serverName 不匹配、优雅退出时存在流式 RPC。
