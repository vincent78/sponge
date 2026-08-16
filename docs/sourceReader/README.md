# Sponge 源码阅读与重实现文档库

> 状态：已完成（待复核生成稿）
> 生成日期：2026-08-16
> 基准提交：`23807238c62e0f3b3e2d9a341bbef50547d3f5ec`
> 工作区：dirty（存在与本任务无关的本地改动：`go.mod` / `go.sum` / sqlite 测试库等；本目录仅新增文档）
> 源码范围：整个仓库，排除 `third_party/` 第三方 protobuf、依赖缓存和构建产物
> 生成方式：源码、测试、配置与部署资产静态分析

## 快速摘要

### 架构总览（模块与依赖）

Sponge 不是运行时依赖注入大框架。它同时包含两套代码：

1. **生成期工具**：`cmd/sponge` 编译出的 CLI 与本地 UI。它读取数据库表或 Protobuf，把本仓库里的示例模板（以及 `sponge init` 后落到 `~/.sponge` 的模板副本）替换成用户项目。
2. **运行期模板与基础库**：`cmd/serverNameExample_*`、`internal/`、`api/` 是被生成项目的骨架；`pkg/` 是生成项目运行时真正依赖的公共库（Gin、gRPC、GORM、缓存、治理等）。

依赖方向固定为：CLI 命令 → `pkg/sql2code` / `pkg/replacer` → 模板文件；生成项目 → `pkg/*` → 数据库 / 缓存 / 注册中心。禁止反向依赖。

### 核心调用序列（逐步逻辑）

本库选定的代表性主流程是 **`sponge web http`：从一张 MySQL 表生成一套可运行的 HTTP CRUD 服务**。完整逐步拆解见 [02-简单例子-全路径走读.md](02-简单例子-全路径走读.md)，这里只保留骨架：

1. `cmd/sponge/main.go:main` 调用 `generate.Init()`，再执行 Cobra 根命令。
2. `commands.GenWebCommand` 挂上 `generate.HTTPCommand`。
3. `HTTPCommand.RunE` 调用 `sql2code.Generate`，从数据库取出 `SHOW CREATE TABLE`。
4. `parser.ParseSQL` 产出 model / dao / handler 等代码片段。
5. `httpGenerator.generateCode` 用 `pkg/replacer` 复制模板、替换占位符、写入输出目录。
6. 用户在生成项目里 `make run` 后，`GET /api/v1/user/{id}` 走 handler → dao → cache/db。

### 易错点与边界条件

- 未执行 `sponge init` 时，`generate.Init` 会因 `~/.sponge` 不存在而拒绝生成（`sponge`、`sponge -h`、`sponge init` 除外）。
- 模板不在当前 Git 工作区直接复制；默认源是用户家目录下的 `.sponge`。
- `serverNameExample`、`userExample` 是占位符，不是业务实体。
- `// todo generate ... here` 与 `// delete the templates code start/end` 是生成器锚点，不是普通注释。
- 表名或服务名以 `_test` / `-test` 结尾会被拒绝。
- 本仓库工作区当前 dirty；文档结论以基准提交为准，不以未提交改动为准。

---

## 这套文档要解决什么问题

读完后应能回答：

- Sponge 运行时到底有几块进程，各自干什么。
- 一次 `sponge web http` 从哪进、到哪出，中间经过哪些真实函数。
- 生成出的 HTTP 服务如何启动，一次 `GetByID` 如何落到数据库。
- 若要从零重实现一个精简版 Sponge，应按什么顺序搭建、每阶段如何验收。

阅读顺序强制为四层，不可跳级：**简单框架 → 全路径小例子 → 主链路逐步详解 → 按模块补齐全部逻辑**。

## 阅读顺序与所属层

| 顺序 | 文档 | 所属层 | 读完能回答 | 状态 |
|---|---|---|---|---|
| 0 | [README.md](README.md)（本文） | 地图 | 有哪些文档、先读哪篇、源码覆盖到哪 | 已完成 |
| 1 | [01-简单框架-系统骨架.md](01-简单框架-系统骨架.md) | 第一层 | 项目是干什么的、运行时有几块、请求大概怎么走 | 已完成 |
| 2 | [02-简单例子-全路径走读.md](02-简单例子-全路径走读.md) | 第二层 | 一次 `sponge web http` 如何变成目录，生成服务如何响应 `GetByID` | 已完成 |
| 3 | [03-详细逐步说明-主链路拆解.md](03-详细逐步说明-主链路拆解.md) | 第三层 | 主链每一跳的参数、返回值、错误传播、失败与边界 | 已完成 |
| 4 | [04-CLI入口命令树与生命周期.md](04-CLI入口命令树与生命周期.md) | 第四层 | 全部 Cobra 命令、init/upgrade/plugins、版本与模板目录 | 已完成 |
| 5 | [05-代码生成器与模板写入.md](05-代码生成器与模板写入.md) | 第四层 | 所有 `generate.*Command`、字段替换、文件变体、安全写入 | 已完成 |
| 6 | [06-SQL到代码片段引擎.md](06-SQL到代码片段引擎.md) | 第四层 | 四种数据库元数据、ParseSQL、类型映射、CRUD 片段 | 已完成 |
| 7 | [07-Protoc插件与增量合并.md](07-Protoc插件与增量合并.md) | 第四层 | 三个 protoc 插件、AST 合并、patch/merge | 已完成 |
| 8 | [08-UI-Assistant-Patch与性能测试.md](08-UI-Assistant-Patch与性能测试.md) | 第四层 | `sponge run` UI、AI assistant、template、perftest | 已完成 |
| 9 | [09-生成项目启动与HTTP请求链.md](09-生成项目启动与HTTP请求链.md) | 第四层 | 生成项目 main/initial/server/router/handler | 已完成 |
| 10 | [10-gRPC服务网关与RPC客户端.md](10-gRPC服务网关与RPC客户端.md) | 第四层 | gRPC server、interceptor、gateway、rpcclient | 已完成 |
| 11 | [11-数据层Model-DAO-Cache.md](11-数据层Model-DAO-Cache.md) | 第四层 | model/dao/cache/database 及事务、缓存击穿 | 已完成 |
| 12 | [12-pkg应用生命周期与Gin.md](12-pkg应用生命周期与Gin.md) | 第四层 | `pkg/app`、`pkg/gin`、`pkg/httpsrv` | 已完成 |
| 13 | [13-pkg-gRPC与服务注册发现.md](13-pkg-gRPC与服务注册发现.md) | 第四层 | `pkg/grpc`、`pkg/servicerd` | 已完成 |
| 14 | [14-pkg数据库缓存与查询.md](14-pkg数据库缓存与查询.md) | 第四层 | `pkg/sgorm`、`pkg/mgo`、`pkg/cache`、`pkg/goredis` | 已完成 |
| 15 | [15-pkg可观测性限流熔断与其余包.md](15-pkg可观测性限流熔断与其余包.md) | 第四层 | logger/tracer/shield 及其余一级包全量索引 | 已完成 |
| 16 | [16-配置错误码测试构建与部署.md](16-配置错误码测试构建与部署.md) | 第四层 | conf/errcode、Makefile、CI、K8s、自动测试 | 已完成 |
| 17 | [17-从零重新实现指南.md](17-从零重新实现指南.md) | 第四层 | 可运行里程碑、每阶段验收条件 | 已完成 |
| 18 | [18-源码索引与覆盖矩阵.md](18-源码索引与覆盖矩阵.md) | 第四层 | 文件→文档映射、未覆盖项、待确认项 | 已完成 |

## 目录树

```text
docs/sourceReader/
├── README.md                                  # 文档地图（本文件）
├── 01-简单框架-系统骨架.md                     # 第一层
├── 02-简单例子-全路径走读.md                   # 第二层
├── 03-详细逐步说明-主链路拆解.md               # 第三层
├── 04-CLI入口命令树与生命周期.md               # 第四层
├── 05-代码生成器与模板写入.md                  # 第四层
├── 06-SQL到代码片段引擎.md                     # 第四层
├── 07-Protoc插件与增量合并.md                  # 第四层
├── 08-UI-Assistant-Patch与性能测试.md          # 第四层
├── 09-生成项目启动与HTTP请求链.md              # 第四层
├── 10-gRPC服务网关与RPC客户端.md               # 第四层
├── 11-数据层Model-DAO-Cache.md                  # 第四层
├── 12-pkg应用生命周期与Gin.md                  # 第四层
├── 13-pkg-gRPC与服务注册发现.md                # 第四层
├── 14-pkg数据库缓存与查询.md                   # 第四层
├── 15-pkg可观测性限流熔断与其余包.md           # 第四层
├── 16-配置错误码测试构建与部署.md              # 第四层
├── 17-从零重新实现指南.md                      # 第四层
└── 18-源码索引与覆盖矩阵.md                    # 第四层
```

## 各文件大纲（标明所属层）

### README.md（地图）

- 仓库身份、分析基准、四层读法。
- 文档树、阅读顺序、覆盖矩阵、进度。

### 01-简单框架-系统骨架.md（第一层）

- Why：为什么必须先分清“生成器进程”和“被生成服务进程”。
- What：两类进程、主要目录、依赖方向、数据进出。
- How：一张骨架图 + 一条主路径一句话，不展开函数分支。

### 02-简单例子-全路径走读.md（第二层）

- 选定例子：`sponge web http` 从 MySQL `user` 表生成 HTTP CRUD 服务。
- 入口：CLI（UI 最终也是调用同一条 CLI）。
- 走读：`main` → `HTTPCommand.RunE` → `sql2code.Generate` → `httpGenerator.generateCode` → `replacer.SaveFiles`。
- 用脱敏输入/输出把链路串起来；只走主成功路径。
- 附带生成项目启动后 `GET /api/v1/user/1` 的成功路径速览，细节放到第三层。

### 03-详细逐步说明-主链路拆解.md（第三层）

- 按跳拆开生成链：参数校验、DSN 适配、SHOW CREATE TABLE、ParseSQL、模板选择、字段替换、落盘、configmap。
- 按跳拆开运行链：InitApp、CreateServices、NewRouter、GetByID、DAO 缓存、响应。
- 每跳补失败、重试、边界；仍按主链路组织，不按模块倾倒全部源码。

### 04-CLI入口命令树与生命周期.md（第四层）

- `cmd/sponge/main.go`、`commands.NewRootCMD` 全命令树。
- `Init` / `Upgrade` / `Plugins`、版本文件、`SpongeDir`。
- `generate.Init` / `InitFS`、允许无模板运行的命令。

### 05-代码生成器与模板写入.md（第四层）

- `web` / `micro` 下全部 Generator：http、http-pb、grpc、grpc-http、handler、dao、model、cache、service、protobuf、rpc-conn、config、swagger、configmap。
- `replacer.Replacer` 接口、Field、忽略规则、已存在文件取消生成。
- 文件变体：`.tpl` / `.exp` / `.mgo` / `.noregistry`。

### 06-SQL到代码片段引擎.md（第四层）

- `sql2code.Args`、`getSQL`、四种驱动的表信息抽取。
- `parser.ParseSQL`、`makeCode`、类型映射、CrudInfo、模板渲染。
- 与 `sql2code_test.go`、`parser_test.go` 的交叉验证。

### 07-Protoc插件与增量合并.md（第四层）

- `cmd/protoc-gen-go-gin`、`cmd/protoc-gen-go-rpc-tmpl`、`cmd/protoc-gen-json-field`。
- `pkg/goast` 合并、`commands/merge`、`commands/patch`。

### 08-UI-Assistant-Patch与性能测试.md（第四层）

- `sponge run`、`cmd/sponge/server` 路由与 zip 下载。
- `assistant`、`template`、`graph`、`perftest`、`cmd/perftest`。

### 09-生成项目启动与HTTP请求链.md（第四层）

- 全部 `cmd/serverNameExample_*` 的 main/initial。
- `internal/server/http.go`、`internal/routers`、`internal/handler` 全 API。
- `pkg/gin` 中间件在生成项目中的装配。

### 10-gRPC服务网关与RPC客户端.md（第四层）

- `internal/server/grpc.go`、`internal/service`、`internal/rpcclient`。
- 网关示例 `grpcGwPbExample`、双协议 `grpcHttpPbExample`。

### 11-数据层Model-DAO-Cache.md（第四层）

- `internal/model`、`internal/dao`、`internal/cache`、`internal/database`。
- 事务方法、singleflight、占位缓存、Mongo 变体。

### 12-pkg应用生命周期与Gin.md（第四层）

- `pkg/app`、`pkg/httpsrv`、`pkg/gin/*` 全子包。

### 13-pkg-gRPC与服务注册发现.md（第四层）

- `pkg/grpc/*`、`pkg/servicerd`、etcd/consul/nacos 客户端。

### 14-pkg数据库缓存与查询.md（第四层）

- `pkg/sgorm`、`pkg/mgo`、`pkg/goredis`、`pkg/cache`、查询构造。

### 15-pkg可观测性限流熔断与其余包.md（第四层）

- logger、tracer、stat、prof、shield。
- 其余一级包：aicli、copier、dlock、encoding、errcode、gobash、gofile、jwt、kafka、rabbitmq、sasynq、sse、ws、proxykit、utils 等，逐包给出公开接口与调用关系，不用“略”。

### 16-配置错误码测试构建与部署.md（第四层）

- `pkg/conf`、`internal/config`、`internal/ecode`、`pkg/errcode`。
- `test/auto-test`、`test/server`、Makefile、scripts、deployments、GitHub Actions。

### 17-从零重新实现指南.md（第四层）

- 按可运行闭环拆里程碑：先 CLI 骨架，再 sql2code，再 replacer，再 HTTP 模板，再治理组件。
- 每阶段客观验收条件。

### 18-源码索引与覆盖矩阵.md（第四层）

- 目录/文件 → 文档锚点。
- 测试覆盖缺口、推断项、待确认项。

## 源码覆盖矩阵

口径：仓库内约 607 个自有 `.go` 文件（不含 `third_party/` 与 `docs/`）。“覆盖”指有对应文档追踪到真实符号与调用关系，不是“提到目录名”。细表见 [18-源码索引与覆盖矩阵.md](18-源码索引与覆盖矩阵.md)。

| 源码范围 | 计划文档 | 覆盖状态 |
|---|---|---|
| `cmd/sponge/main.go`、`commands/root.go`、`init.go`、`upgrade.go`、`plugins.go` | 01、04 | 已覆盖 |
| `cmd/sponge/commands/web.go`、`generate/http.go`、`generate/common.go`、`generate/init.go` | 01、02、05 | 已覆盖 |
| `cmd/sponge/commands/generate/*.go` 其余生成器 | 05 | 已覆盖 |
| `cmd/sponge/commands/micro.go` 及 grpc/pb 生成器 | 05、10 | 已覆盖 |
| `cmd/sponge/commands/run.go`、`cmd/sponge/server/` | 02、08 | 已覆盖 |
| `cmd/sponge/commands/assistant*`、`template*`、`merge*`、`patch*`、`perftest*`、`graph.go` | 07、08 | 已覆盖 |
| `cmd/protoc-gen-go-gin/`、`cmd/protoc-gen-go-rpc-tmpl/`、`cmd/protoc-gen-json-field/` | 07 | 已覆盖 |
| `cmd/perftest/` | 08 | 已覆盖 |
| `cmd/serverNameExample_httpExample/`、`internal/handler`、`internal/routers`、`internal/server/http.go` | 02、09 | 已覆盖 |
| `cmd/serverNameExample_grpc*`、`internal/service`、`internal/rpcclient`、`internal/server/grpc.go` | 10 | 已覆盖 |
| `internal/model`、`internal/dao`、`internal/cache`、`internal/database` | 02、11 | 已覆盖 |
| `internal/config`、`internal/ecode`、`internal/types`、`api/` | 09、16 | 已覆盖 |
| `pkg/sql2code/` | 02、06 | 已覆盖 |
| `pkg/replacer/` | 02、05 | 已覆盖 |
| `pkg/app/`、`pkg/gin/`、`pkg/httpsrv/` | 09、12 | 已覆盖 |
| `pkg/grpc/`、`pkg/servicerd/` | 10、13 | 已覆盖 |
| `pkg/sgorm/`、`pkg/mgo/`、`pkg/cache/`、`pkg/goredis/` | 11、14 | 已覆盖 |
| `pkg/logger/`、`pkg/tracer/`、`pkg/shield/`、`pkg/stat/`、`pkg/prof/` | 15 | 已覆盖 |
| 其余 `pkg/*` 一级包 | 15 | 已覆盖 |
| `configs/`、`deployments/`、`scripts/`、`Makefile*`、`.github/` | 16 | 已覆盖 |
| `test/auto-test/`、`test/server/`、各 `*_test.go` | 02、16 | 已覆盖（阅读测试；未执行套件） |
| `docs/`（本目录以外的已有笔记）、`examples/`、`assets/` | 18 | 仅作交叉引用，不作为行为证据源 |

## 术语

| 术语 | 含义 |
|---|---|
| Sponge 工具 | `cmd/sponge` 编译出的 CLI 和 `sponge run` UI |
| 模板仓库 | `sponge init` 后位于 `~/.sponge` 的模板副本 |
| 示例模板代码 | 本仓库 `internal/`、`api/`、`cmd/serverNameExample_*` |
| 生成项目 | `sponge web http` 等命令写出的独立工程 |
| 字段代码片段 | sql2code 根据表结构动态生成的 model/dao/handler/proto 文本 |
| 工程骨架模板 | main、server、脚本、部署文件等相对稳定的素材 |

## 分析约定

- 只有源码、测试、配置或 Schema 支持时才陈述为事实。
- 架构解释或可能行为标为“推断”。
- 未在本机跑通的生产参数标为“待确认”。
- 文档使用仓库相对路径，不写本机绝对路径。
- DSN、密码一律脱敏。
- 本次任务未运行测试套件；测试结论来自阅读测试代码，不宣称测试通过。

## 当前进度

1. 已完成：仓库盘点、入口定位、代表性功能选定、文档库规划。
2. 已完成：第一层系统骨架。
3. 已完成：第二层 `sponge web http` 全路径走读。
4. 已完成：第三层主链路逐步拆解。
5. 已完成：第四层 04–18 按模块补齐（CLI、生成器、sql2code、protoc/merge、UI/assistant/perftest、HTTP/gRPC 运行时、数据层、pkg、配置测试部署、重实现路线、覆盖矩阵）。
6. 未执行：单元测试套件与 `test/auto-test` 脚本；结论来自静态阅读。
7. 待确认：生产参数、以及当前基准提交下自动测试是否仍通过。
