# 08. Struct 之外的 CRUD 代码如何生成

数据库解析并不只返回 model。`parser.ParseSQL` 对同一个 `tmplData` 一次生成多种片段，避免每层重复解释 schema。

## 1. `codes` 输出字典

每张表的 `makeCode` 返回 `codeText`，多张表的结果最后合并为：

| key | 内容 | 最终主要去向 |
|---|---|---|
| `model` | 带 package/import 的 model Go 文件 | `internal/model` |
| `json` | 按字段生成的示例 JSON | 模板/辅助输出 |
| `dao` | Update 时构造 `map[column]value` 的判断代码 | `internal/dao` 标记区 |
| `handler` | Create/Update/Detail HTTP struct | `internal/types` |
| `proto` | service、RPC、message、HTTP 注解 | `api/.../*.proto` |
| `service` | gRPC service 使用的 Create/Update/Detail Go struct 片段 | service 模板相关位置 |
| `crud_info` | 主键和表名的 JSON 元数据 | 选择模板、替换 ID/方法名 |
| `table_info` | 完整表/列 JSON 元数据 | 自定义模板 |
| `__table_name__` | Go Camel 表名列表 | 替换 `UserExample` 等名称 |

这里要区分“字段驱动生成”和“骨架模板复制”：parser 只生成随表字段变化的片段；完整 DAO 方法、handler 方法、路由、测试、配置和部署文件来自仓库模板，再把片段插进去。

## 2. DAO 更新字段片段

`getUpdateFieldsCode` 过滤 ID、创建/更新时间、软删除字段，然后执行 `updateFieldTmpl`：

```go
if table.Name != "" {
    update["name"] = table.Name
}
```

不同 Go 类型通过 `tmplField.ConditionZero()` 决定“非零”条件：数字非 0、字符串非空、时间非 nil 且非零、JSON 非空等。该设计支持部分更新，但也意味着普通值类型无法表达“明确更新成零值”；需要这类语义时通常应使用指针 DTO 或显式字段集合。

完整的 Create、Delete、Get、List 等 DAO 方法不由 parser 从零拼出，而是来自 `internal/dao/userExample*.go[.tpl/.exp/.mgo]`。generator 将 update 片段和实体/主键信息替换进去。

## 3. HTTP request/reply struct

`getHandlerStructCodes` 先为 transport 层调整类型和 JSON 名，再执行三个模板：

- `CreateXxxRequest`：排除系统保留字段；
- `UpdateXxxByIDRequest`：保留/处理 ID，排除不应由用户更新的字段；
- `XxxObjDetail`：输出详情所需字段。

MongoDB ObjectID 在 HTTP 层表示为 string；嵌套 model 类型会加 `model.` 包前缀。生成结果写入 `internal/types` 模板的 mark 区。

完整 handler 方法来自模板，它们负责 Gin binding、copier、日志、ecode、DAO 调用和 response。`codes["handler"]` 只提供真正随数据库列变化的 DTO 定义。

## 4. Proto 与 gRPC 数据结构

`goTypeToProto` 将统一字段再次变换，例如：

```text
int → int32
uint → uint32
float64 → double
time.Time → string（部分固定 ID/时间标记随后还会适配）
[]string → repeated string
JSON / decimal → string
MongoDB ObjectID → string ID 相关字段
```

`getProtoFileCode` 根据两个维度选四套主要模板：

| `IsWebProto` | `IsExtendedAPI` | 结果 |
|---|---|---|
| false | false | 纯 gRPC，基础 5 API |
| false | true | 纯 gRPC，扩展 9 API |
| true | false | 带 HTTP/OpenAPI 注解，基础 5 API |
| true | true | 带 HTTP/OpenAPI 注解，扩展 9 API |

基础 API 是 Create、DeleteByID、UpdateByID、GetByID、List；扩展再加入 DeleteByIDs、GetByCondition、ListByIDs、ListByLastID。

模板先生成 service/RPC 外壳，再把 Create/Update/Detail message 插入固定 mark。最后 `adaptedDbType` 根据数据库和主键模式替换 ID message：标准关系型 ID 使用 uint64；MongoDB 路径使用 string 并配套 string validate/tagger 规则；Web 版本增加 URI/form tag。

通用主键模式走 `commonTemplate.go`，方法名可变成 `GetBy订单号` 一类由真实主键命名的形式，而非硬编码 ByID。

## 5. Service 片段

`getServiceStructCode` 生成 gRPC service 在 copier/业务处理时使用的字段结构，字段过滤与 handler 类似，但面向 protobuf/service 类型。

完整 `internal/service/userExample*.go` 仍来自模板，包括：

- 包级 `init()` 注册 service；
- 构造 DAO/cache；
- `req.Validate()`；
- protobuf 与 model 互转；
- DAO 调用；
- ecode 到 gRPC status 的映射；
- 单测和 client test。

## 6. `crud_info` 为什么不可少

parser 将 `CrudInfo` JSON 放进 `codes["crud_info"]`。generator 再反序列化，用它决定：

- 标准模板还是 `.tpl` 通用主键模板；
- CRUD 方法中的 ID 类型；
- ByID 是否改成 ByXxx；
- 单数/复数实体命名；
- cache key 和参数名；
- Proto ID 类型和校验规则；
- extended API 的方法与测试替换。

换句话说，`model/handler/proto` 是要写入文件的代码，`crud_info` 是指导“怎样组装文件”的控制数据。

## 7. 完整 HTTP 项目怎样拼起来

`httpGenerator.generateCode()` 的顺序可以拆成：

```text
1. 创建 replacer，模板根目录为 ~/.sponge
2. 建立 subDirs/subFiles/selectFiles 白名单
3. SetSelectFiles 按 DB driver 选择普通或 MongoDB 文件
4. 读取 crud_info，判断 common style
5. 按 common / extended / mgo 组合 .tpl、.exp、.mgo 文件
6. replaceFilesContent 用主键元数据改写模板内部内容
7. 把 model、dao、handler 片段及 module/server/table 等加入 fields
8. 删除仅供模板展示的 start/end 标记区
9. 加入配置、数据库、部署、镜像、mono-repo 等替换字段
10. SetSubDirsAndFiles → SetOutputDir → SetReplacementFields → SaveFiles
11. saveGenInfo 保存生成信息
```

关键代码消费关系：

```text
CodeTypeModel   → modelFileMark
CodeTypeDAO     → daoFileMark
CodeTypeHandler → handlerFileMark（先 adjustmentOfIDType）
TableName       → userExample/UserExample 等占位替换
CrudInfo        → 模板变体和主键相关全局替换
```

完整 gRPC 项目的 `rpcGenerator` 多消费 `CodeTypeProto` 和 `CodeTypeService`，并选择 api、third_party、protoc 脚本、grpc server 与 service 模板。

## 8. 多表生成的特殊流程

完整 `web http`/`micro rpc` 命令收到逗号分隔多表时：

1. 第一张表生成完整项目骨架；
2. 记住返回的真实 `outPath`；
3. 后续每张表重新执行 `sql2code.Generate`；
4. HTTP 使用 `handlerGenerator` 向同一项目追加模块；
5. gRPC 使用 `serviceGenerator` 向同一项目追加模块。

这样 main、server、配置、部署文件只生成一次，但每张表有自己的 model/DAO/cache/transport/测试。

## 9. 自定义模板模式

`IsCustomTemplate=true` 时，`makeCode` 不生成内置 CRUD 片段，而是通过 `newTableInfo(data)` 输出 `table_info` JSON。其中包含：

- 表的原名、Camel/小驼峰/复数/snake 名；
- 表注释、driver；
- 每列的各种命名、注释、Go 类型、tag、主键标记；
- 主键的类型和各种命名；
- MongoDB 子结构与子 message。

自定义模板引擎可把这份结构化元数据用于任意文本模板，因此不局限于 Go 后端。

## 10. 从现象反查源码

| 现象 | 第一检查点 | 第二检查点 |
|---|---|---|
| 数据库列没读到 | 各 driver 的 `Get*TableInfo` | 临时 DDL 内容 |
| struct 类型错误 | `mysqlToGoType` / `fieldTypes` | `getModelStructCode` rewrite |
| 字段名/tag 错误 | `makeCode` 的命名/tag 拼装 | generator 二次替换 |
| 主键方法名或类型错误 | `newCrudInfo` | common `.tpl` 选择 |
| DTO 少字段 | `tmplExecuteWithFilter` / ignore fields | handler/service 模板 mark |
| Proto 类型错误 | `goTypeToProto` | `adaptedDbType` |
| DAO 方法存在但更新不了零值 | `ConditionZero` | request 是否使用指针/字段集合 |
| 输出缺文件 | generator 的 `selectFiles` | `SetSubDirsAndFiles` |
| 文件有占位符残留 | generator `addFields` | `replacer.SaveFiles` |

## 11. 最小实验：不连接数据库也能观察核心

可以直接给 `sql2code.Args.SQL` 传 DDL，绕过数据库：

```go
codes, err := sql2code.Generate(&sql2code.Args{
    SQL: `CREATE TABLE users (
        id bigint unsigned NOT NULL AUTO_INCREMENT,
        name varchar(64) NOT NULL COMMENT '姓名',
        balance decimal(10,2) NULL,
        created_at datetime NOT NULL,
        PRIMARY KEY (id)
    );`,
    DBDriver:  "mysql",
    Package:   "model",
    JSONTag:   true,
    GormType:  true,
})
```

依次打印 `model`、`dao`、`handler`、`proto`、`service`、`crud_info`。这能把“schema 解析问题”和“模板文件组装问题”分开，是调试生成器最有效的第一步。
