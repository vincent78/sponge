# 17. `pkg/logger`、`tracer`、`shield`：横切能力

## 1. Logger

`logger.Init(opts...)` 构建 zap logger，并设置包级默认 logger。Option 支持 level、console/json format、是否写滚动文件、文件名/大小/备份数/保留天数/压缩/本地时间，以及 zap hooks。`Get`/`GetWithSkip` 取 logger；`Debug/Info/...` 和 `Debugf/Infof/...` 是包级便捷函数；`Sync` 刷缓冲。

`Field` 是 `zapcore.Field` 别名，`String/Int/Duration/Err/Any` 等函数避免业务直接依赖 zap 构造器。`WithFields` 派生带固定字段的 logger。`ReplaceGRPCLoggerV2` 把 gRPC 内部日志接入 zap。

生命周期：启动最早初始化；所有组件共享 `*zap.Logger` 或显式派生 logger；退出最后 `Sync`。不要每个请求 Init。生产 JSON 便于收集，开发 console 易读。敏感字段必须在进入 logger 前脱敏；高频路径避免 `Any` 的反射成本。

重实现最小版：先创建一个 `zap.Logger` 并显式注入，再加全局便捷 API，最后加 lumberjack 滚动和 hooks。测试 level 过滤、格式、文件轮转 option、字段类型、caller skip 和 Sync。

## 2. Tracer

`tracer.NewResource` 用 service name/version/environment/attributes 构造 OTel resource。Exporter 可以是 console、文件或 Jaeger；`Init(exporter,res,fraction)` 安装 provider，`InitWithConfig` 是组合入口，`GetProvider` 取 provider，`Close(ctx)` shutdown/flush。

`SetTraceName` 设置 instrumentation name；`NewSpan(ctx, spanName, tags)` 从父 context 创建 span 并写 attributes。调用模式必须是：

```go
ctx, span := tracer.NewSpan(ctx, "dao.get_user", tags)
defer span.End()
```

新 context 必须传给下游，否则 trace 断链。错误不仅要写 tag，还应 `RecordError` 并设置 span status。采样 fraction 影响成本但不能用作安全过滤；敏感参数仍不能写 attribute。

HTTP middleware 和 gRPC interceptor 负责跨进程传播 trace context，GORM plugin 负责数据库子 span。退出时 `Close` 要在进程硬退出前完成，否则批处理 exporter 中的数据会丢。

## 3. `shield/window`：所有自适应算法的地基

Window 由固定数量 Bucket 构成；Bucket 保存多种 Metric 的 float64 数组。RollingPolicy 根据时间推进 bucket，过期桶 reset；Iterator 遍历当前窗口；`Sum/Avg/Min/Max/Count` 做聚合。`RollingCounter` 在其上提供 Add/Value/Reduce。

核心思想：不保存每个请求，只保存时间桶聚合值，空间固定。窗口长度 `Window` 和桶数 `Bucket` 决定时间分辨率；桶太少反应粗糙，太多锁竞争和计算增加。时钟跳跃、并发 rotate、边界时刻是最重要测试。

## 4. SRE 熔断器

`circuitbreaker.CircuitBreaker` 对外提供 Allow/MarkSuccess/MarkFailed（以源码接口为准）；`NewBreaker` 返回基于 Google SRE 思路的 `Breaker`。Options：success 系数、最小请求量、统计窗口、桶数。

它不是连续失败 N 次就永久打开，而是根据窗口内 requests 与 accepts 计算拒绝概率；错误增多时逐渐丢弃请求，恢复后随统计窗口滑动自动收敛。`ErrNotAllowed` 表示本次调用在业务执行前被拒绝。

正确调用协议：

```text
Allow
 ├─ rejected → fallback / fast fail
 └─ allowed → 调用下游
      ├─ 被定义为成功 → MarkSuccess
      └─ 被定义为失败 → MarkFailed
```

“哪些错误算失败”必须由 HTTP/gRPC adapter 配置。例如参数错误不是下游故障，通常不应触发熔断。`container/group` 按 key 缓存 breaker，key 必须有界，否则内存持续增长。

## 5. BBR 自适应限流

`ratelimit.Limiter.Allow()` 返回 `DoneFunc` 或 `ErrLimitExceed`；调用完成必须执行 done，并传 `DoneInfo`。`NewLimiter` 返回 BBR 实现，option 控制窗口、桶、CPU threshold、CPU quota。

BBR 使用滑动窗口估计最大吞吐和最小延迟，再结合当前在途请求与 CPU 使用率判断系统是否过载。`shield/cpu` 从 cgroup 或 psutil 读取配额和利用率，兼容容器/主机；`stat/cpu`、`stat/mem` 提供更通用采样。

限流和熔断区别：限流保护“自己”不被过载，熔断保护“自己调用的依赖”并阻止故障扩散。常见顺序是入口先限流，出站依赖再熔断；两者都要反馈真实执行结果。

## 6. Gin/gRPC 适配

算法位于 `shield`，协议适配位于 `gin/middleware` 和 `grpc/interceptor`。这样算法不依赖网络框架。适配层负责：确定分组 key、从响应判断成功、把拒绝映射为 HTTP/gRPC 错误、执行降级 handler、记录指标。

重实现时也应保持三层：

1. window/counter 纯数据结构；
2. breaker/limiter 纯算法；
3. Gin/gRPC adapter 做协议映射。

## 7. 可观测性组合

一次请求至少关联：request ID（单请求日志检索）、trace ID（跨服务调用树）、metrics label（总体趋势）。三者不是替代关系。日志字段应包含 request/trace ID；span 记录关键事件；metrics 只使用低基数标签。

建议失败排查顺序：指标发现异常 → trace 定位慢/错的服务和调用 → request ID 搜索详细日志。保护组件的拒绝次数、breaker 状态、CPU、inflight 必须有指标，否则只能看到“请求失败”而不知道为何主动拒绝。

## 8. 重实现测试矩阵

- logger：并发写、level、caller、滚动、敏感字段策略。
- tracer：父子 span、跨 HTTP/gRPC 传播、采样 0/1、shutdown flush。
- window：时间推进、多桶跨越、并发 Add、空窗口 reduce。
- breaker：低请求量不误判、故障上升时拒绝、恢复后放行、错误分类。
- limiter：CPU 阈值、inflight、done 必调、突发流量和稳定流量。
- adapter：拒绝码/降级内容正确，业务未执行，日志和指标可观察。
