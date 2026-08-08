# 18. `pkg` 其余包分类索引

> 本章保证 `pkg` 一级包不遗漏。每项给出职责、入口/数据流、使用者与生命周期、测试和重实现关注点。代码生成主链另见 [07-数据库到代码的核心算法.md](07-数据库到代码的核心算法.md)。

## 1. 网络与长连接

### `httpcli`

通用 HTTP client。`New` 创建可复用 Request，包级 `Get/Delete/Post/Put/Patch` 接收结果指针和 `WithParams/WithHeaders/WithTimeout`；Response 可读 body。请求对象负责序列化 query/body、发出请求、检查响应并反序列化。调用者是外部 API adapter；底层 transport 应长期复用，response body 必须关闭。测试覆盖各 method、reset、标准 envelope 和错误响应。重实现要明确非 2xx 策略、重试幂等性、超时由 context 主导、body 大小限制。

### `sse`

Server-Sent Events。Hub 管理按 UID 连接的 channel，支持广播/定向 push；HTTP handler 设置 event-stream header、监听断开、持续 flush；Client 支持连接/重连。SSE 是服务端单向文本流，连接生命周期等于请求 context。测试连接、广播、缺 UID、重连；重实现要处理慢消费者、channel 关闭、心跳和代理缓冲。

### `ws`

WebSocket client/server 封装。`NewClient(url, opts...)` 支持 dialer/header/ping/logger，后台读写与心跳维持连接。连接拥有 goroutine、ticker 和 socket，Close 必须让三者退出。测试默认/自定义 server、client ping；重实现要单 writer、读写 deadline、pong handler、背压、重连策略。

### `proxykit`

较底层的反向代理、backend/balancer/health check 组件，被 `gin/proxy` 组合。数据流是 route → balancer 选健康 backend → reverse proxy → 状态反馈。启动 health-check ticker，关闭时必须停止。测试轮询/随机等策略、上下线、转发失败；重实现要避免并发读写 backend 列表竞态。

## 2. 消息和后台任务

### `kafka`

生产者/消费者封装，围绕 broker/topic/group 配置发送和消费。消息生命周期是 produce ack → broker 保存 → group consumer 拉取 → handler 成功后提交 offset。client、consumer、producer 都要 Close。测试常需要真实 broker 或 mock；重实现必须明确 at-most/at-least-once、提交时机、重平衡、批量、重试和死信。

### `rabbitmq`

封装 connection/channel、exchange/queue/binding、发布与消费。AMQP channel 不是无限并发共享对象；连接断开要决定是否重连。消息在 ack/nack/requeue 间流转。重实现先做显式 ack，再加 publisher confirm、QoS、死信和幂等消费。

### `sasynq`

基于 Asynq 的 Redis 后台任务 client/server/scheduler/inspector 辅助。生产者 enqueue，server 按类型派发 handler，失败重试，scheduler 周期入队。各对象是长期资源；server shutdown 应等 handler。测试 payload 编解码、路由、重试和调度，业务 handler 必须幂等。

### `gocron`

定时任务包装，创建 scheduler、添加 cron/interval job、启动和停止。调用者是应用装配层；ticker/goroutine 必须关闭。重实现要定义错过执行、重叠执行、时区、panic recovery 和分布式多副本只执行一次；最后一点可配 `dlock`。

## 3. 分布式基础

### `dlock`

分布式锁抽象及 Redis/etcd 等实现（以目录文件为准）。典型数据流：按 key 获取带 TTL 的租约 → 执行业务 → 校验 owner token 后释放。锁对象和续租 goroutine按临界区存活。测试互斥、超时、租约到期、错误 owner；重实现禁止无 token 直接 DEL，业务时长可能超过 TTL 时必须续租或 fencing token。

### `etcdcli` / `consulcli` / `nacoscli`

仅负责构造官方 client、认证/TLS/日志和连接 option，为 servicerd/dlock 等上层提供依赖。它们是进程级长连接，创建者关闭。测试 option 映射和连接错误；不要在请求内初始化。

### `container/group`

按字符串 key 延迟创建并缓存对象，breaker middleware 用它为每个资源保存独立状态。公开入口围绕 Get/添加/删除（查看实际接口）。生命周期通常随进程；动态 key 必须清理或限制。测试同 key 单例、并发创建、不同 key 隔离。

## 4. 错误、身份、编码和安全

### `errcode`

结构化业务错误，维护 code/message/details 和全局注册表，供 Gin response、gRPC status、错误码列表使用。创建 → 包装底层 cause → 边界映射为协议响应。错误值不持有资源。测试重复 code、格式化、errors.Is/转换。重实现应把用户可见 message 与内部 cause 分开，并固定 code 兼容性。

### `jwt`

签发、解析、刷新 claims，`old_jwt` 保留兼容实现。调用者是 Gin/gRPC auth。token 是无服务端资源的凭证，但 key 是长期敏感配置。测试签名算法、过期、issuer、错误 key、刷新；重实现禁止接受 token 自带算法而不校验白名单，明确时钟偏差和 key rotation。

### `encoding`、`encoding/json`、`encoding/proto`

`Encoding` 抽象 Marshal/Unmarshal，按枚举/名称选择 JSON 或 protobuf；cache 使用它存对象。无资源生命周期。测试 nil、空值、错误目标和 round trip。重实现不要假设不同 protobuf schema 兼容，缓存值最好带版本。

### `gocrypto` 与 `gocrypto/wcipher`

AES/DES/RSA、hash、密码 bcrypt 和加密 option。对称加解密处理 mode/padding/key/IV；RSA 包含加密、签名、验证；hash 仅做摘要，密码必须用慢哈希。无长期资源，但密钥要从安全配置读取并清零/限制日志。测试 round trip、非法 key/ciphertext、签名篡改。新实现不要把 MD5/SHA 当密码哈希，DES 仅兼容旧系统，优先 AEAD（AES-GCM）。

### `krand`

生成字符串、字节、整数、小数、ID/序列 ID。先确认源码使用的是密码学还是伪随机源：会话 token 必须 `crypto/rand`，测试数据可用 `math/rand`。测试长度、字符集、范围、并发唯一性；“看起来随机”不等于安全唯一。

## 5. 数据转换与系统小工具

### `copier`

结构体复制/字段映射，通常用于 model ↔ DTO。反射转换不拥有资源。测试指针、嵌套、零值、不同类型；重实现要防止把 password/internal 字段意外复制到响应。

### `utils`

包含 DSN 自适应、类型转换、主机/端口、时间、浏览器打开、`SafeRun`、等待打印等。`SafeRun` 捕获 goroutine panic，`SafeRunWithTimeout` 管理 cancel；`GetAvailablePort` 只能表示检查瞬间可用，随后仍有 TOCTOU 竞争。逐文件测试已有较好覆盖。重实现时避免“失败返回零值”的转换掩盖错误，核心路径使用带 `E` 的版本。

### `gobash`

`Exec`/`Run(ctx, name,args...)` 执行外部命令，Result 保存 stdout/stderr/error，`ParsePid` 辅助解析。子进程生命周期受 context 控制。测试成功、失败、取消；重实现不要把用户字符串拼 shell，始终参数数组传递，并限制输出大小。

### `gofile`

文件/目录复制、查找、写入等辅助，被生成器使用。文件句柄应函数内及时关闭；批量操作可能覆盖用户文件，调用者需确定目标。测试临时目录、权限、符号链接和错误路径。重实现优先原子写：临时文件 + rename。

### `stat/cpu`、`stat/mem`

读取 CPU/内存统计，为监控和 shield 提供基础数据。采样器可能有后台 ticker，全局启动一次并关闭。测试解析固定样本比依赖机器数值更稳定；容器内应读取 cgroup 限额而非只看宿主机。

## 6. AI 与格式转换

### `aicli`

`Assistanter` 统一 Send/stream 等能力，`chatgpt`、`deepseek`、`gemini` 适配各 SDK；option 设置 model、temperature、max tokens、initial role 和上下文。Client 长期复用 HTTP transport，stream 必须消费/关闭。测试大多需要 API key，单元测试应 mock transport。重实现要处理限流、超时、流结束、上下文长度、供应商差异和密钥保密。

### `jy2struct`

读取 JSON/YAML，递归合并对象/数组元素类型，格式化字段名，输出 Go struct 文本。`Convert(Args)` 是高层入口。无长期资源。测试混合数组、null、数字、非法输入、字段命名；重实现要规定冲突类型降级为 `interface{}`/`any` 的规则和 tag 输出。

## 7. 代码生成与测试辅助

### `goast`

解析/修改 Go AST，供生成后代码插入、格式化和检查。数据流是 source → parser AST → visitor/修改 → printer/format。无长期资源。测试应比较格式化后的语义，不依赖无意义空格；重实现避免字符串替换 Go 语法。

### `replacer`

模板工程的复制和占位符替换，生成器最终落盘的重要环节。输入模板目录、替换规则和目标目录，遍历文本/路径并写出。测试 `testDir`。重实现要区分二进制文件、文件名替换、覆盖策略、执行失败后的半成品清理。

### `sql2code` / `sql2code/parser`

将数据库 DDL/元数据解析为模板数据，再生成 model、DAO、request/proto 等片段。公开入口 `GenerateOne/Generate`。这是生成链核心，详细过程应结合数据库到 Struct 专章阅读。测试包含多表和错误 DDL。重实现要先定义中间模型，再分别写 DB introspector、parser、renderer，避免三者耦合。

### `gotest`

为生成的 DAO/service/handler 测试提供 Cache、Dao、Service、Handler 假对象及测试数据查询，减少模板测试重复。对象只在测试期间存活。测试验证 sqlmock 参数、HTTP/gRPC server 启动和数据抽取。重实现时辅助器失败应调用 `t.Helper` 并给清晰错误，不能吞掉断言。

## 8. 全量重实现验收表

重写每个包时逐项回答：

1. 对外最小接口是什么，是否泄露第三方具体类型？
2. 谁创建、谁共享、谁关闭？是否有 goroutine/ticker/socket/file？
3. context 从哪里来，取消是否传到底层？
4. 默认 option 是什么，零值是否合法？
5. 错误是包装、映射还是原样返回？调用者如何分类？
6. 并发安全吗？全局变量会不会污染测试？
7. 单元测试、集成测试、竞态测试分别覆盖什么？
8. 是否有密钥、token、个人数据进入日志/trace？

推荐最后运行：目标包单测、`go test -race`（并发包）、真实中间件集成测试，以及 `git diff --check`。外部依赖测试应通过 build tag 或环境变量显式区分，不能让普通单测偶然访问生产服务。
