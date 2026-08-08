# 15. `pkg/grpc` 与服务注册发现

## 1. 四块拼图

gRPC 相关代码分为：

1. `grpc/server`：监听端口、安装 interceptor、注册生成的 service。
2. `grpc/client` 与 `grpc/grpccli`：建立连接；后者把常用选项组合成更高层入口。
3. `grpc/interceptor`、`metrics`、`gtls`、`keepalive`：横切能力。
4. `servicerd/registry` 和 `discovery`：把逻辑服务名变成随时更新的地址列表。

静态调用是 `dns:///host:port` 或直接 endpoint；动态调用通常是 `discovery:///service-name`，自定义 resolver 负责喂给 gRPC ClientConn 地址。

## 2. Server 的生成与启动

`server.Run(port, registerFn, options...)` 的执行顺序：

```text
解析 Option
 → net.Listen(:port)
 → 可选 CustomListener 统计连接
 → grpc.NewServer(TLS + interceptor chains)
 → registerFn(srv) 注册 generated service
 → 可选 serviceRegisterFn 把地址登记到注册中心
 → goroutine 中 srv.Serve(listener)
 → 返回 *grpc.Server
```

`WithSecure` 安装 transport credentials；`WithUnaryInterceptor` 和 `WithStreamInterceptor` 分别组成调用链；`WithServiceRegister` 注入外部注册动作；`WithStatConnections` 包装 listener。

资源生命周期注意：当前 `Run` 启 goroutine 后返回，`Serve` 错误会 panic；若 serviceRegisterFn 失败，已打开的 listener 需要关闭。重实现时建议让 server 对象持有 listener，提供阻塞 `Start`、`GracefulStop` 加超时退化为 `Stop`，并在关闭时先从注册中心摘除。

## 3. Client 与高层 `grpccli`

`client.NewClient(endpoint, opts...)` 依次添加 resolver、round_robin service config、TLS/明文 credentials、自定义 dial option、unary/stream interceptor，最后调用 `grpc.NewClient`。

解析优先级：显式 `WithServiceDiscoverBuilder` 高于 `WithServiceDiscover` 自动创建的 builder。没有 credentials 时默认 `insecure.NewCredentials()`，这表示“无 TLS”，不是“跳过证书校验的 TLS”。连接使用者负责 `ClientConn.Close()`。

`grpccli` 是便捷装配层，其 option 可开启 request ID、日志、trace、metrics、负载均衡、retry、circuit breaker、token、TLS 和 discovery。重实现时先手工组装 `client.NewClient`，理解 interceptor 顺序后再写这种聚合层，否则很难解释重复 interceptor 或优先级冲突。

## 4. Interceptor 请求链

Unary interceptor 类似 HTTP middleware；stream interceptor 还必须包装 `ClientStream/ServerStream`，处理 context 和每条消息。

| 文件 | 入口 | 作用与关键点 |
|---|---|---|
| `requstid.go` | `Unary/Stream*RequestID` | metadata 与 context 间传播 ID |
| `logging.go` | `Unary/Stream*Log` | 方法、耗时、状态、请求响应摘要；限制长度并脱敏 |
| `tracing.go` | tracing interceptors | 提取/注入 OpenTelemetry context |
| `metrics.go` | metrics interceptors | 统计调用量、状态和时延；方法名标签有界 |
| `jwtAuth.go` | server JWT auth | 从 metadata 验证 token，把 claims 放 context |
| `token.go` | client option/server token | appID/appKey 简单服务凭证 |
| `timeout.go` | client timeout | 派生带 deadline 的 context |
| `retry.go` | client retry | 只对配置 code 重试；必须考虑幂等性 |
| `recovery.go` | recovery | 捕获 panic 转成 gRPC status |
| `ratelimit.go` | server rate limit | BBR 自适应拒绝过载请求 |
| `breaker.go` | client/server breaker | SRE 熔断、分组、有效 code 和降级处理 |

推荐客户端顺序是 request ID/trace → metrics/log → timeout → retry → breaker（实际需按“每次逻辑调用”还是“每次重试尝试”决定指标位置）；服务端通常 recovery 最外层，随后 request ID/trace/log/metrics/auth/protection。必须用测试明确顺序，而不是相信名称。

## 5. TLS、保活、指标、压测

- `gtls/server.go`：构造服务端单向/双向 TLS credentials。
- `gtls/client.go`：按 CA、client cert/key 构造客户端 credentials。
- `gtls/certfile.Path`：定位证书测试资源。
- `keepalive`：提供 server/client keepalive 参数；太激进会增加网络和 CPU，太宽松不能及时发现死连接。
- `metrics`：client/server Prometheus 指标以及连接 listener；可注册到独立 HTTP mux。
- `benchmark.New`：包装 ghz runner，根据 proto、方法和请求执行压测；它是开发工具，不进入业务请求链。

证书文件、metrics HTTP server、ClientConn 都有关闭责任；credentials 本身通常不需要关闭。

## 6. 注册中心的统一抽象

`registry.Registry` 只有 `Register/Deregister`；`Discovery` 有 `GetService/Watch`；`Watcher.Next` 阻塞到首次非空或实例变化，`Stop` 终止观察。统一对象 `ServiceInstance` 包含 ID、Name、Version、Metadata 和 Endpoints。

Endpoint 带 scheme 和参数，例如：

```text
grpc://127.0.0.1:9000?isSecure=false
http://127.0.0.1:8000?isSecure=false
```

ID 标识实例，Name 标识服务集合。多个实例应同名不同 ID。Metadata 用于版本、地域等有限标签，不能塞大对象。

## 7. etcd / Consul / Nacos 适配器

### etcd

`registry/etcd.NewRegistry` 同时返回 Registry 和 ServiceInstance；`New` 接收现有 client。注册通常创建 lease，把序列化实例写入 namespace 下的 key，并用 keepalive/重试续租；watcher 观察前缀变化，维护内存实例列表。生命周期是 client → registry → lease/watch；关闭时先 deregister/stop watcher，再 close client。

### Consul

将实例转换为 Consul Agent service registration，支持健康检查。`Client` 包装查询/watch；watcher 基于阻塞查询索引获得变化。若开启健康检查却没有对应端点，实例会一直不健康。

### Nacos

将 endpoint 拆为 IP/port/kind，支持 prefix、weight、cluster、group、default kind；watcher基于 Nacos naming subscription。Group/cluster 必须在注册和发现两边完全一致。

三个实现应满足同一契约，但一致性和通知语义不同。业务只能假设“最终获得可用地址集合”，不能假设每个瞬时变化都被逐条看到。

## 8. discovery Builder/Resolver

`discovery.NewBuilder` 实现 gRPC `resolver.Builder`，scheme 固定为 `discovery`。`Build`：

1. 从 target path 去掉前导 `/` 得到 service name。
2. 在 goroutine 中调用 `Discovery.Watch`。
3. 等 watcher 建立，默认最多 10 秒。
4. 创建 resolver 并启动 `watch()`。
5. watcher 每次返回实例集合，resolver 过滤 endpoint 的 secure 属性，转换为 `resolver.Address`，调用 `ClientConn.UpdateState`。

`Close` 应 cancel context 并 Stop watcher。这里的生命周期非常关键：ClientConn 活着，resolver/watcher 才应活着；关闭连接后必须退出 goroutine。

## 9. 从零重实现实验

1. 写内存 Registry/Discovery，map 保存实例，channel 通知 watcher。
2. 启两个相同 service name、不同 ID 的 gRPC server。
3. 实现 resolver，把两个地址交给 ClientConn。
4. 开 round_robin，连续请求观察实例轮转。
5. 删除一个实例，验证 watcher 更新后不再调用它。
6. 再替换为 etcd/Consul/Nacos 实现。

测试必须覆盖：首次列表、空列表、增删实例、watch context 超时、Stop 幂等、非法 endpoint、TLS 属性过滤、注册租约失效、ClientConn 关闭无 goroutine 泄漏。
