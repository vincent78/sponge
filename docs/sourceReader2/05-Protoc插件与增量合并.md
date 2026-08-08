# 05 Protoc 插件与增量合并

## 1. protoc 插件如何被调用

当命令中出现 `--go-gin_out=.`，protoc 会在 PATH 查找 `protoc-gen-go-gin`，通过标准输入发送 `CodeGeneratorRequest`，插件通过标准输出返回响应。项目使用 `protogen.Options.Run` 封装协议。`ParamFunc: flags.Set` 把 `--go-gin_opt=k=v` 送入插件自己的 FlagSet。

仓库有三个插件：

| 插件 | 输入 | 主要输出 |
|---|---|---|
| `protoc-gen-go-gin` | service、message、google.api.http 注解 | Gin 绑定路由、handler/service 逻辑模板、router 注册、ecode |
| `protoc-gen-go-rpc-tmpl` | gRPC service/message | service 实现模板、测试/客户端模板、ecode |
| `protoc-gen-json-field` | Proto 描述 | 描述 service/method/message 字段的 JSON/Go 数据文件，供模板系统使用 |

## 2. 共同解析层

插件内部 `parse` 将 protogen 对象转成模板友好的结构：

- `PbService`：服务名、包、方法列表；
- `ServiceMethod`/`RPCMethod`：方法名、请求/响应类型、流式类型、HTTP method/path；
- `Field`：字段名、Go 类型、零值、注释；
- import package map：处理跨 proto message 引用。

HTTP 方法来自 `google.api.http` 注解。`GetMethods` 还处理 additional bindings；path 中的变量会形成 path params；自定义 method/selector 会影响 Gin 绑定策略。四种 RPC 调用类型由 client/server streaming 两个布尔值组合。

解析和渲染分开很值得复刻：不要在模板里直接遍历 `protogen.Method`。先构建可单测 IR，再由 handler/router/service generator 渲染。

## 3. protoc-gen-go-gin 调用链

```text
main
  ├─ 解析 plugin/moduleName/serverName/*Out/suitedMonoRepo
  └─ protogen.Options.Run
      └─ 对每个 f.Generate 文件
          ├─ 拒绝 *_test.proto
          ├─ saveGinRouterFiles
          │   └─ router.GenerateFiles → *_router.pb.go
          ├─ plugin=handler
          │   └─ handler.GenerateFiles → logic/router/*_http.go
          ├─ plugin=service
          │   └─ service.GenerateFiles → logic/router/*_rpc.go
          └─ plugin=mix
              └─ handler 分支，但省去独立 HTTP ecode
```

router pb 是 proto 到 Gin 的协议适配层：定义 `XxxLogicer`、注册函数、method/path 与 middleware 组合、HTTP/RPC response 选项。handler/service 模板则属于项目业务层。

插件若需要写业务文件，会检查 moduleName/serverName；缺失时当前实现会 panic。输出目录默认 `internal/handler|service`、`internal/routers`、`internal/ecode`，mono-repo 则前置 serverName。

## 4. 为什么产生 `.go.gen20...`

业务逻辑文件不能直接覆盖，因为用户可能已经实现旧方法。`saveFile` 的策略是：

- 目标不存在：写正式 `.go`；
- 目标已存在且不允许覆盖：写一个带生成时间的临时文件；
- 随后由 `sponge merge` 进行 AST 级合并。

这解释了 merge 默认模糊匹配 `*.go.gen20*`。同一源文件存在多个生成文件时，`runMerge` 选择最新名字对应的生成物。正式文件先备份到 `/tmp/sponge_merge_backup_code/<timestamp>`。

## 5. protoc-gen-go-rpc-tmpl

该插件聚焦 gRPC service 业务模板。参数包括 moduleName、serverName、tmplOut、ecodeOut、suitedMonoRepo。`service.GenerateFiles` 产生 service template、service test/client template 和 error code。解析层还计算 request/reply import、proto 文件名、四类流式调用签名及字段零值。

它与 go-gin 的职责不同：标准 `protoc-gen-go-grpc` 已负责 gRPC server interface；本插件生成“如何实现这些方法”的 Sponge 项目代码。

## 6. protoc-gen-json-field

该插件把 Proto 服务结构转成后续自定义模板可消费的数据。`parser.GetServices` 收集 method、注释、字段、import；`generate.GenerateFiles` 建 `ProtoInfo → Service → RPCMethod → Field` 并渲染。它也解析 HTTP rule、path params、selector 与流式类型。

注意名字容易误导：它不是简单地修改 protobuf struct 的 json tag，而是输出字段/服务描述，服务于 `sponge template protobuf` 的数据驱动生成。

## 7. merge 命令

三类用户入口映射到多轮合并：

- `merge http-pb`：handler、router、ecode；
- `merge rpc-gw-pb`：service/HTTP gateway 相关；
- `merge rpc-pb`：gRPC service、ecode、客户端测试。

`mergeParams.runMerge` 先按目录寻找临时文件并组成 `srcFile → genFile`，再按类型调用：

- `module.ParseErrorCode`：合并错误码；
- `module.ParseRouterCode`：合并路由注册；
- `module.ParseHandlerAndServiceCode`：合并 import、struct/method；
- `module.ParseGRPCMethodsTestAndBenchmarkCode`：合并测试和 benchmark。

解析层基于 `pkg/goast`，比较已有方法，新增缺失方法，并保留已存在业务实现。成功后删除临时生成文件；失败时不应删除。备份是最后的恢复线。

## 8. 边界与复刻要点

- 一个 proto 多 service、跨包 message、additional_bindings、四种 streaming 都要有测试。
- proto 文件后缀 `_test.proto` 被显式禁止，因会生成 Go test 命名冲突。
- 模板根据实际输出字符串删除未使用 import，这是脆弱点；复刻可生成 AST 后由 goimports 整理。
- 生成文件时间戳若仅靠文件名排序，应保证固定可排序格式并避免同刻碰撞。
- merge 必须识别“同名方法是保留、覆盖还是冲突”。当前 assistant merge 可选覆盖同名函数，而 protoc merge 主旨是保留用户逻辑，两者语义不同。
- 写入前必须 gofmt；合并失败要指出源文件、生成文件和具体 AST 节点。

## 9. 从零实现建议

1. 先写只输出 `_router.pb.go` 的 protoc 插件，验证 stdin/stdout 协议。
2. 为 HTTP annotation 建 IR，并用 golden files 覆盖 GET/POST/path/body/additional binding。
3. 写正式文件/临时文件两路保存策略。
4. 用 Go AST 实现“只添加不存在的方法和 import”。
5. 加备份、dry-run、diff，再自动删除成功合并的临时文件。
6. 最后扩展 gRPC streaming、mono-repo 和 ecode 合并。
