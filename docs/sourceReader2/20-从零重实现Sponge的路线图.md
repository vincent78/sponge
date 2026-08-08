# 20. 从零重实现 Sponge 的路线图

本章不是按仓库目录，而是按“每一步都有可运行成果”组织。不要直接复制全部源码；先建立小而清晰的内核，再用测试逐项扩展。

## 阶段 0：明确目标和非目标

第一版目标：输入 MySQL DDL，生成一个可编译的 Gin CRUD 项目。非目标：UI、在线数据库、gRPC、MongoDB、注册发现、AI、压测、Kubernetes。

建议新工程结构：

```text
cmd/gen
internal/schema       统一 Table/Column
internal/parser       DDL parser adapter
internal/naming       命名规则
internal/mapper       DB→Go/Proto 类型
internal/render       字段代码片段
internal/assemble     工程模板选择和写入
templates/http        最小 Gin 项目
testdata
```

## 阶段 1：统一 schema 模型

定义 `Table`、`Column`、`PrimaryKey`、`DBType`，不能让 AST 类型泄露到 renderer。实现验证：空表、重复列、未知类型、非法标识符。

验收：手写 Table 能稳定序列化为 JSON，单测覆盖标准/字符串/复合主键。

## 阶段 2：DDL 解析

接入 SQL parser，把 `CREATE TABLE` AST 转为统一模型。保留原始列名、nullable、default、comment、primary/unique/auto increment。

验收：至少 20 个 DDL fixture；解析后结构与 golden JSON 一致；错误包含语句位置。

## 阶段 3：命名和类型映射

实现纯函数：snake→Camel、lowerCamel、复数、ID 缩写、前缀裁剪。类型 mapper 同时返回 Go 类型、import、transport 类型和零值策略。

验收：所有命名/类型组合 table-driven 测试；未知类型产生明确诊断。

## 阶段 4：只生成 model

用 `text/template` 渲染 package/import/struct/TableName/列白名单，输出前 `go/format.Source`。

验收：生成文件可 `go test`；同一输入字节级确定；JSON/decimal/time imports 正确。

## 阶段 5：生成 DTO 和 DAO 片段

分离 database model 与 HTTP DTO。先实现 Create/Get/List，Update 使用指针或 update mask，避免 Sponge 当前非零判断的表达限制。

验收：零值更新、敏感字段不返回、nullable 字段三条测试通过。

## 阶段 6：工程装配器

模板内只放稳定骨架，动态代码使用唯一命名的占位节点。写入流程：先在同父目录创建临时输出 → 完整渲染 → fmt/build 验证 → 原子改名。失败自动清理，避免半成品。

验收：生成最小 Gin 服务，health 和一个 CRUD endpoint 编译运行；重复输出有明确策略。

## 阶段 7：CLI

用 Cobra 或标准 flag 实现：`model`、`http`、`version`。参数验证与生成逻辑分开；业务层接收结构体 options，不直接读取全局 flag。

验收：help、缺参、非法 driver、输出已存在、取消信号都有测试。

## 阶段 8：在线数据库适配器

按接口实现 MySQL、PostgreSQL、SQLite。优先直接转统一模型，不必伪造 MySQL DDL。所有查询参数化，并支持 schema/database 名。

验收：相同逻辑表在不同 DB 产生等价模型；连接失败/权限不足/表不存在错误可区分。

## 阶段 9：gRPC/Proto

先选择事实来源：若数据库只用于首次生成 proto，后续以 proto 为准。实现稳定 field number 策略、validate 和 HTTP annotations，再接 protoc。

验收：基础 5 API、gateway 可编译；已有字段删除不会复用编号；兼容性检查通过。

## 阶段 10：项目变体

用显式 feature matrix 选择模板：HTTP/gRPC/gateway、basic/extended、SQL/Mongo、mono/multi repo。不要在各文件散落嵌套 if。

验收：每个受支持组合都有生成 smoke test；不支持组合在生成前报错。

## 阶段 11：运行时生产能力

按实际需求依次加入配置、日志、优雅退出、metrics、trace、TLS、限流/熔断、注册发现、消息系统。每加一个能力必须明确生命周期和故障降级。

## 阶段 12：UI 与高级工具

UI 只调用稳定的内部生成 API或 CLI。之后再做 zip、生成记录、assistant、自定义模板、merge/patch、性能测试。

## 建议的关键接口

```go
type SchemaSource interface {
    Load(context.Context, Input) ([]Table, error)
}

type Renderer interface {
    Render(ProjectModel) (Fragments, error)
}

type Assembler interface {
    Assemble(context.Context, Blueprint, Fragments, string) error
}

type Verifier interface {
    Verify(context.Context, string) error
}
```

这些接口把外部数据库、纯生成算法、文件系统副作用和编译验证分开，单测成本低于直接在 Cobra `RunE` 中完成全部工作。

## 与 Sponge 保持兼容时的顺序

1. 先复刻命名与类型映射 golden；
2. 再兼容 `codes` map 的 key 和片段格式；
3. 再兼容模板占位符/mark；
4. 最后兼容 CLI 参数和输出目录命名。

这样可以逐层对比，而不是一次生成整个项目后面对上千行 diff。

## 最终完成标准

- 核心算法不依赖全局可变状态；
- 同一输入产生确定输出；
- 所有文件写入具备失败回滚；
- schema、renderer、assembler 可独立测试；
- 生成项目通过 fmt、vet、test、build；
- 支持矩阵有文档和自动测试；
- 生命周期资源成对关闭；
- 错误可定位到输入、阶段和文件；
- 模板升级不会覆盖用户业务区域；
- 文档能让未参与开发的人完成一次新 driver 或新 transport 扩展。
