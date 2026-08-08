# 02. CLI 与代码生成主流程

## 入口和命令树

`cmd/sponge/main.go` 只做两件事：先调用 `generate.Init()` 准备模板 replacer，再构造 `commands.NewRootCMD()` 并执行。

`commands/root.go` 用 Cobra 组装命令树。主要分组：

- `init`/`upgrade`：下载或更新 `~/.sponge` 模板，并检查 protoc 等插件。
- `web`：model、dao、cache、handler、完整 HTTP 服务等。
- `micro`：proto、service、gRPC、gateway、混合服务等。
- `run`：启动浏览器 UI 和它的本地后端。
- `config`、`merge`、`patch`、`template`：配置生成、代码合并、生成后修补、自定义模板。
- `assistant`：调用 LLM 辅助生成业务逻辑。
- `perftest`：HTTP/1.1、HTTP/2、HTTP/3、gRPC、WebSocket 压测。

`generate.Init()` 检查 `~/.sponge`，然后用 `replacer.New(SpongeDir)` 注册名为 `sponge` 的模板源。帮助和 init 命令被特意允许在尚未初始化时执行。

## 以 `sponge web http` 为例

入口是 `commands/generate/http.go` 的 `HTTPCommand()`。

### 1. 解析参数

Cobra 把 module、server、project、数据库 driver/DSN/table、extended API、mono-repo 等写入局部变量和 `sql2code.Args`。必填项由 `MarkFlagRequired` 声明。

多表时，第一张表用于生成完整项目；其余表只追加 handler/dao/cache/model 等业务模块。这能避免重复生成 main、配置和部署文件。

### 2. 数据库定义变成代码片段

`pkg/sql2code.Generate` 的工作顺序：

1. 校验参数。
2. `getSQL` 按来源拿建表语句：直接 SQL、文件或真实数据库；MongoDB 会先取字段再转换成中间 SQL 表达。
3. 把 Args 转成 parser options，例如 JSON 命名、空值风格、嵌入模型、扩展 API。
4. `parser.ParseSQL` 使用 SQL AST 遍历 `CREATE TABLE`。
5. 每张表一次性产生 model、JSON、DAO 更新字段、handler DTO、proto、service DTO、CRUD 元数据和表元数据。

返回值是 `map[string]string`，key 定义在 `pkg/sql2code/parser/parser.go`，包括 `model`、`dao`、`handler`、`proto`、`service`、`crud_info` 等。生成器不必重复解析同一张表。

`CrudInfo` 尤其重要：它描述主键和是否属于“常规 ID 风格”。非标准主键会让生成器改选 `.tpl` 通用模板，而不是硬套 uint64 ID 模板。

### 3. 选择模板文件

`httpGenerator.generateCode()` 明确列出：

- 要复制的目录，如启动入口、configs、deployments、scripts；
- 单文件，如 go.mod、Makefile、README；
- 每个 internal 目录选择哪些文件。

数据库类型、扩展 API、通用主键风格、mono-repo 会改变选择结果。例如 MongoDB 选择 `.mgo` 变体，通用主键选择 `.tpl`，mono-repo 不复制独立 go.mod/go.sum。

这种“白名单选模板”的方式是 Sponge 的关键设计：同一份模板仓库可以拼装出多种服务形态。

### 4. 替换与插入

`pkg/replacer` 负责递归复制模板，并应用字段替换。替换内容既包括简单占位符（module/server/project/table 名），也包括由 sql2code 产生的大块代码。

生成器还会用固定 mark 找插入点，例如 model、DAO update fields、proto、Dockerfile、配置。`startMark/endMark` 包围仅供模板展示的代码，最终输出时可以删除。

最后会调整 import、ID 类型、时间转换、部署名、镜像地址、数据库配置等，并运行格式化。理解某个输出异常时，建议依次查：

```text
SQL/parser 产物 → generator 选择了哪个模板 → fields/replacer 做了什么 → patch 是否再次修改
```

## gRPC 生成与 HTTP 的差异

`commands/generate/rpc.go` 的前半段与 HTTP 相同，仍通过 SQL 得到模型和 CRUD 元数据。后半段选择 proto、service、gRPC server、third_party 和 protoc 脚本。

基于已有 proto 的 `rpc-pb`、`grpc-gw-pb`、`grpc-http-pb` 路径不需要数据库定义，重点变为解析 API 契约并调用 protoc 插件。`cmd/protoc-gen-go-gin`、`cmd/protoc-gen-go-rpc-tmpl`、`cmd/protoc-gen-json-field` 是这条链的扩展点。

## UI 实际上仍调用 CLI

`sponge run` 在 `commands/run.go` 中启动 `cmd/sponge/server`。静态 Vue 资源通过 `embed.FS` 打进二进制；若用户指定非默认访问地址，会把资源复制到本地并替换 `appConfig.js`。

`POST /api/v1/generate` 接收前端拼出的命令参数。`server/handler.go` 的 `handleGenerateCode`：

1. 创建临时输出目录；
2. 使用 `gobash.Run(ctx, "sponge", args...)` 再调用 CLI；
3. 限时等待，收集 stdout/error；
4. 将输出目录压成 zip 返回；
5. 记录生成历史，并异步清理临时文件。

所以 UI 和 CLI 共用同一生成核心，而不是两套实现。排查 UI 生成失败时，先复制它的参数到终端复现。

## 其他维护命令

- `merge`：把新生成模块合并到已有项目，重点处理 router 和 error code 等公共注册点。
- `patch`：适配 mono-repo、复制 proto/go.mod、删除 `omitempty`、修正重复编号/错误码等机械改写。
- `template`：从 SQL/proto 字段构造自定义模板所需数据。
- `graph`：输出工程或依赖关系图。
