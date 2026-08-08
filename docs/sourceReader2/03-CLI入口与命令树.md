# 03 CLI 入口与命令树

> 阅读范围：`cmd/sponge`。目标不是背命令，而是理解“参数如何变成一次生成任务”。

## 1. 先建立整体模型

`sponge` 是一个 Cobra CLI。入口只做三件事：初始化模板、构造命令树、执行命令。

```text
cmd/sponge/main.go
  ├─ generate.Init()        检查 ~/.sponge，并建立模板 Replacer
  ├─ commands.NewRootCMD()  组装 Cobra 命令树
  └─ rootCMD.Execute()      解析 os.Args，进入某个 RunE
```

`main` 先调用 `generate.Init` 很关键：除 `sponge`、帮助和 `init` 外，大多数命令都要求 `~/.sponge` 已存在。它不是配置目录，而是生成时读取的完整模板源码树。未初始化时直接返回提示，命令树甚至不会执行。

## 2. 根命令

`commands/root.go::NewRootCMD` 创建 `Use: "sponge"` 的根节点，开启 `SilenceErrors` 和 `SilenceUsage`，所以错误最终由 `main` 统一打印。版本优先读 `~/.sponge/.github/version`；读不到时提示执行 `sponge init`。

命令树如下：

```text
sponge
├─ init                         下载模板并安装缺失插件
├─ upgrade                      更新 CLI、模板、内置 protoc 插件
├─ plugins                      检查/安装外部工具
├─ web
│  ├─ model / dao / cache
│  ├─ handler / handler-pb
│  ├─ http / http-pb
│  └─ swagger
├─ micro
│  ├─ protobuf / model / dao / cache / service
│  ├─ rpc / rpc-pb / rpc-gw-pb
│  ├─ grpc-conn
│  ├─ grpc-http / grpc-http-pb
│  └─ service-handler
├─ config                       生成配置代码
├─ run                          启动 Web UI
├─ merge                        合并 protoc 产生的增量文件
├─ patch                        对生成项目做专项修补
├─ graph                        调用 spograph 绘图
├─ template                     用自定义模板生成
├─ assistant                    AI 生成、合并、清理、聊天
└─ perftest                     HTTP/2/3、WebSocket、gRPC 压测
```

`web.go` 和 `micro.go` 本身不生成文件，只是命令分组器。真正逻辑在 `commands/generate/*.go` 的各个 `RunE`。

## 3. 初始化、升级与插件

`init` 的调用链：

```text
InitCommand.RunE
  ├─ runUpgrade("latest")
  │  ├─ go install .../cmd/sponge@latest
  │  ├─ copyToTempDir → 从 GOPATH/pkg/mod 复制源码到 ~/.sponge
  │  └─ updateSpongeInternalPlugin → 安装三个自带 protoc 插件
  ├─ checkInstallPlugins
  └─ installPlugins
```

`copyToTempDir` 会替换整个 `~/.sponge`，设置权限，并删除 CLI、本仓库 `pkg`、测试和资产等生成时不需要的部分。版本选择由 `getLatestVersion` 完成。复刻时应避免依赖 shell 的 `ls/cp/rm/chmod`，优先用 Go 文件 API，并使用“复制到临时目录后原子替换”，否则中途失败可能留下半套模板。

`plugins` 用 `which` 判断命令是否存在；Go 插件并发执行 `go install`，每项三分钟超时；Go 和 protoc 只打印人工安装提示。三个仓库内插件会跟随 Sponge 版本，而不是永远安装 latest，这是保证模板和插件协议一致的关键。

## 4. 一条生成命令的共同骨架

以 `sponge web http` 为例：

1. Cobra 将 flags 写入局部变量和 `sql2code.Args`。
2. 校验必填参数，规范化 project/server 名称；禁止 server 名以 `-test`、`_test` 结尾。
3. 将逗号分隔表名拆开；第一张表生成完整项目，其余表只增量生成 handler/service 等业务层。
4. `sql2code.Generate` 从数据库得到按用途分类的代码片段。
5. `httpGenerator.generateCode` 选模板文件、设置替换字段、保存工程。
6. 后续表交给 `handlerGenerator`；最后补 configmap 并打印使用说明。

RPC 同构：第一张表由 `rpcGenerator` 建工程，后续表由 `serviceGenerator` 增量写入。MongoDB 会强制关闭 GORM model 嵌入。mono-repo 模式改变输出路径、import 和是否复制 `go.mod/go.sum`。

## 5. 其他顶层命令

- `run` 校验 `--addr` 的 scheme/host；IP 地址还要求 URL 端口等于 `--port`。随后尝试打开浏览器并启动 Gin。
- `graph` 不内置绘图算法，而是检查 `spograph` 后透传 project/server 参数。
- `patch` 聚合一组窄而明确的迁移命令，不应理解成统一补丁引擎。
- `merge` 处理插件生成的带时间戳临时 Go 文件，先备份再 AST 合并。
- `perftest` 的独立 `cmd/perftest/main.go` 实际复用 `commands.PerftestCommand()`，相当于裁剪版二进制入口。

## 6. 错误和边界

- 模板未初始化：非展示类命令在解析参数前失败。
- 空表名片段被忽略，但完全为空依赖 Cobra required flag，只能保证字符串非空，不能保证每段有效。
- 输出目录已有最新文件时，底层 Replacer 可能拒绝覆盖；这是保护手写代码的策略。
- 外部命令依赖 PATH，且升级过程强依赖 GOPATH module cache。
- `context.WithTimeout` 多处未显式 cancel，是可改进点。
- UI 将参数字符串按空格切分，不支持 shell 引号语义；路径含空格是边界问题。

## 7. 从零重实现顺序

1. 先用 Cobra 实现根命令和空的 web/micro 分组。
2. 定义统一 `Generator` 接口：`Validate → BuildIR → SelectTemplate → Render → Save`。
3. 做模板仓库初始化和明确的模板版本清单。
4. 只实现单表 model，再实现完整 HTTP，再扩展多表。
5. 将 SQL、Proto 两种输入都转换成稳定中间表示，避免 generator 直接依赖数据库结果 map。
6. 最后加入 upgrade/plugins/UI/assistant/perftest，它们是外围能力。

新手调试建议在 `main`、目标命令 `RunE`、`sql2code.Generate`、具体 generator 的 `generateCode`、`replacer.SaveFiles` 依次打断点。
