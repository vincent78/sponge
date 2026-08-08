# Sponge 源码阅读笔记（新手版）

这套笔记以当前仓库源码为准，目标不是复述 README，而是回答三个问题：代码从哪里开始、一次请求/一次生成经过哪里、修改某类功能应该找哪些文件。

## 先建立正确认识

Sponge 同时包含三类代码：

1. `cmd/sponge`：真正的 Sponge 命令行程序和可视化代码生成服务。
2. `internal`、`cmd/serverNameExample_*`、`api`：作为生成素材存在的示例业务服务骨架。名字中的 `serverNameExample`、`userExample` 会在生成时被替换。
3. `pkg`：生成出的服务会复用的公共组件，也是 Sponge 的运行时基础库。

因此，不要从 `internal/handler` 一路读完后就认为已经读完 Sponge；那只是“被生成项目”的一半。最核心的生成逻辑在 `cmd/sponge/commands/generate`、`pkg/sql2code` 和 `pkg/replacer`。

## 推荐阅读顺序

| 顺序 | 文档 | 读完能回答 |
|---|---|---|
| 1 | [01-仓库地图与阅读方法.md](01-仓库地图与阅读方法.md) | 每个目录负责什么，哪些是生成文件/模板 |
| 2 | [02-CLI与代码生成主流程.md](02-CLI与代码生成主流程.md) | `sponge web http` 如何变成一套项目 |
| 3 | [03-HTTP服务运行链.md](03-HTTP服务运行链.md) | HTTP 服务怎样启动，一次 CRUD 请求怎样流动 |
| 4 | [04-gRPC与微服务运行链.md](04-gRPC与微服务运行链.md) | proto、拦截器、service、注册发现怎样串起来 |
| 5 | [05-核心基础库导读.md](05-核心基础库导读.md) | `pkg` 中常用组件的边界和入口 |
| 6 | [06-配置测试部署与实战路线.md](06-配置测试部署与实战路线.md) | 怎样本地调试、测试，以及从哪里开始改代码 |
| 7 | [07-数据库到Struct的完整生成链.md](07-数据库到Struct的完整生成链.md) | 数据库元数据怎样变成 Go struct，四种数据库有何差异 |
| 8 | [08-Struct之外的CRUD代码如何生成.md](08-Struct之外的CRUD代码如何生成.md) | 同一张表怎样继续生成 DAO、DTO、Proto、Service 和完整项目 |

## 两张总图

代码生成：

```text
用户参数 / UI
  → Cobra command
  → sql2code（数据库/SQL → 多类代码片段）或 protobuf 定义
  → generate/*Generator（选模板、组装替换字段）
  → replacer（复制、替换、插入标记区）
  → gofmt / 脚本修补
  → 可运行项目
```

生成项目的请求处理：

```text
HTTP: main → initial → app → server → router → middleware → handler → dao → cache/db
gRPC: main → initial → app → server → interceptor → service → dao → cache/db
```

## 阅读约定

- “模板仓库”指 `~/.sponge`。`sponge init`/`upgrade` 会准备它；生成命令默认不是直接从当前 Git 工作区复制全部文件。
- `.tpl` 是通用模板变体，`.exp` 通常代表 extended API 变体，`.mgo` 是 MongoDB 变体，`.noregistry` 是不启用服务注册的变体。
- `// todo generate ... here` 和 `// delete the templates code start/end` 不是普通注释，而是生成器定位、插入或删除代码的锚点。
- `serverNameExample`、`userExample`、示例 module/project 名是占位符。
