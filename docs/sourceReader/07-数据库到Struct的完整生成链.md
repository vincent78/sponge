# 07. 数据库到 Struct 的完整生成链

本章只追踪一件事：执行基于数据库的生成命令后，Sponge 怎样查看表结构，最后写出 Go model struct。

## 1. 从哪个命令进入

完整 HTTP 项目入口是 `cmd/sponge/commands/generate/http.go` 的 `HTTPCommand()`；完整 gRPC 项目入口是 `rpc.go` 的 `RPCCommand()`。单独生成 model 则从 `model.go` 进入。

它们都会构造相同的参数：

```go
sqlArgs := sql2code.Args{
    Package:  "model",
    JSONTag:  true,
    GormType: true,
}
```

随后填入 `DBDriver`、`DBDsn`、`DBTable` 等，再调用：

```go
codes, err := sql2code.Generate(&sqlArgs)
```

因此数据库到代码的公共入口不是 generator 自己，而是 `pkg/sql2code/sql2code.go`。

## 2. `sql2code.Args` 控制什么

重要字段可以分成四组：

| 类别 | 字段 | 含义 |
|---|---|---|
| schema 来源 | `SQL`、`DDLFile`、`DBDsn`、`DBTable` | 直接 SQL、DDL 文件或在线数据库，优先级依次下降 |
| 数据库 | `DBDriver` | mysql、tidb、postgresql、sqlite、mongodb |
| struct 风格 | `Package`、`GormType`、`JSONTag`、`JSONNamedType`、`IsEmbed` | 包名、tag、JSON 命名、是否嵌入公共 Model |
| 类型/命名 | `NullStyle`、`NoNullType`、`TablePrefix`、`ColumnPrefix` | nullable 表示和前缀裁剪 |

`checkValid()` 会检查至少提供一种 schema 来源、SQLite 文件存在、表名不能以 `_test` 结尾，并在 driver 为空时默认 MySQL。

## 3. `getSQL`：把不同数据库统一成 DDL

这是设计上的关键转折点。后面的 parser 只想处理一种 AST，所以各数据库先被转换为 MySQL 风格 `CREATE TABLE`。

### MySQL / TiDB

调用链：

```text
getSQL
  → utils.AdaptiveMysqlDsn
  → parser.GetMysqlTableInfo
  → database/sql.Open("mysql", dsn)
  → SHOW CREATE TABLE `表名`
  → 返回数据库原始 DDL
```

实现位于 `pkg/sql2code/parser/mysql.go`。它不是查询 `information_schema.columns`，而是直接取得数据库认可的完整建表语句，因此列类型、默认值、注释、索引/主键信息都在 DDL 中。

注意：表名被直接拼进 SQL，并用反引号包裹；调用方应只传可信的表名。连接通过 `defer db.Close()` 释放。

### PostgreSQL

PostgreSQL 没有等价的 `SHOW CREATE TABLE`。调用链：

```text
getSQL
  → AdaptivePostgresqlDsn
  → GetPostgresqlTableInfo
  → GORM + pg_catalog / information_schema 查询
  → PGFields
  → ConvertToSQLByPgFields
  → 临时 MySQL 风格 DDL + pgTypeMap
```

元数据查询从 `pg_class`、`pg_attribute`、`pg_type`、`pg_description` 和主键约束取出列名、原生类型、长度、nullable、注释、主键。

`ConvertToSQLByPgFields` 将 PostgreSQL 类型映射为 parser 能理解的 MySQL 类型，例如 integer→int、bigint→bigint、jsonb→json、boolean→bit(1)。同时返回 `map[column]真实PG类型`。后续生成 GORM `type:` tag 和 bool 修正时使用这个 map，所以临时转换不是最终类型真相。

当前实现按表名查询，未显式携带 schema 名；同名表或非默认 schema 是阅读和扩展时需要留意的边界。

### SQLite

调用链：

```text
getSQL
  → GetSqliteTableInfo(dbFile, table)
  → sqlite.Init
  → PRAGMA table_info('表名')
  → SqliteFields
  → convertToSQLBySqliteFields
  → 临时 MySQL 风格 DDL
```

SQLite 元数据包括 cid、name、type、notnull、default value、pk。类型被粗略映射：integer→INT、text→TEXT、real→FLOAT、datetime→DATETIME、boolean→TINYINT，未知类型→VARCHAR(100)。如果名为 `id` 的 text 列则改为 VARCHAR(50)。

这里生成的临时 DDL没有保留所有 SQLite 语义，例如复杂 default、复合约束和严格表属性；它的目标只是支撑 CRUD struct 生成。

### MongoDB

MongoDB 没有固定 schema，Sponge 采用“抽样推断”：

```text
getSQL
  → AdaptiveMongodbDsn
  → GetMongodbTableInfo
  → collection.FindOne({}, sort _id desc)
  → 读取最新一条 document 的 BSON elements
  → MgoField[]
  → ConvertToSQLByMgoFields
  → 临时 DDL + mongoTypeMap + 子结构文本
```

BSON 类型映射在 `mongodb.go` 的 `mgoTypeToGo`，例如 ObjectID→`primitive.ObjectID`、Int32→`int`、DateTime→`time.Time`、Array/EmbeddedDocument 递归推断。

嵌套对象会额外生成 Go 子 struct 和 Proto message，放在特殊 map key `_sub_struct_`、`_proto_sub_struct_` 中。缺少 `created_at`/`updated_at` 时会自动补上时间字段。`_id` 在临时 DDL 中改写为 `id varchar(24)` 主键，但真实类型仍由 `mongoTypeMap` 保存。

这意味着 MongoDB 生成结果只反映被抽样的最新 document：字段稀疏、同字段混合类型、空数组、collection 为空都会影响或阻止推断。它不是 collection 全量 schema 合并。

## 4. 统一进入 `parser.ParseSQL`

`Generate()` 把 DDL 和额外 `fieldTypes` 转成 options 后调用 `pkg/sql2code/parser.ParseSQL`。

解析器使用 `github.com/zhufuyi/sqlparser` 把 SQL 解析为 AST。它遍历每个 statement，只处理 `*ast.CreateTableStmt`，每张表调用一次 `makeCode(stmt, opt)`。

`makeCode` 首先创建中间数据 `tmplData`：

```text
表原名、去前缀后的 Go 名、首字母小写名、表注释
数据库 driver
字段列表 []tmplField
主键/CRUD 信息 CrudInfo
MongoDB 子 struct / 子 message
```

`tmplField` 是每一列的统一表示，保存：数据库列名、Go 字段名、Go 类型、JSON 名、tag、注释、是否主键、driver，以及某些特殊类型的 rewrite 信息。

## 5. 每一列怎样变成 Go 字段

对 `stmt.Cols` 逐列执行：

1. 取数据库列名；按 `ColumnPrefix` 可选裁剪 Go 字段名前缀。
2. `toCamel` 生成导出字段名，例如 `user_id → UserID`。
3. 按 `JSONNamedType` 生成 snake_case 或 camelCase JSON 名。
4. 从 table constraint 和 column option 判断主键。
5. 拼接 tag：column、原生 type、primary_key、AUTO_INCREMENT、default、unique、not null。
6. 提取列 comment。
7. 计算 Go 类型。

关系型数据库的类型入口是 `mysqlToGoType(col.Tp, nullStyle)`。核心映射：

| 数据库类型类别 | 默认 Go 类型 |
|---|---|
| tinyint(1) | `bool` |
| tiny/small/medium/int | `int` 或 unsigned 的 `uint` |
| bigint | `int64` 或 `uint64` |
| float/double | `float64` |
| char/varchar/text/blob | `string` |
| timestamp/datetime/date | `time.Time` |
| enum/set/geometry | `string` |
| JSON | model 中可重写为 `*datatypes.JSON` |
| bit(1) | model 中可重写为 Sponge bool 类型 |
| decimal | model 中可重写为 `*decimal.Decimal` |
| 未知类型 | `UnknownCustomType` |

nullable 有三种表达：禁用 nullable 包装、`database/sql.Null*`、指针。`NoNullType` 会强制禁用；否则 `NullStyle=sql/ptr` 控制策略。列明确不允许 null 时仍使用普通值类型。

## 6. 主键并不一定叫 ID

`newCrudInfo(data)` 的选择顺序：

1. 数据库声明的 primary key；
2. 第一个名字以 `_id` 结尾且类型合适的列；
3. 第一个 int/uint/string 等适合 CRUD 的列；
4. 第一列兜底。

随后保存主键的原名、Camel 名、复数形式、Go/Proto 类型。标准模式要求主键名为 `id` 且为常用整数类型；否则 `IsCommonType=true`，最终 generator 会选择 `.tpl` 通用主键模板。

当前 AST 逻辑在 table constraint 中只取 `con.Keys[0]`，因此复合主键没有被完整建模为多列 CRUD key，这是重要限制。

## 7. 真正生成 model struct

`getModelStructCode` 使用 `template.go` 的 `modelStructTmpl`：

```go
type {{.TableName}} struct {
    {{range .Fields}}{{.Name}} {{.GoType}} `{{.Tag}}`{{end}}
}
```

它还会执行以下修正：

- `IsEmbed=true`：过滤 id/created_at/updated_at/deleted_at，嵌入 `sgorm.Model` 或 `Model2`；
- 普通关系型模型的 `ID` 默认强制成 `uint64`，但通用主键模式保留真实主键类型；
- 时间字段通常变成 `*time.Time`；
- JSON、decimal、特殊 bool 按 rewrite 信息替换并补 import；
- MongoDB 的 ID 改为 `primitive.ObjectID`，BSON tag 从临时 column 语法恢复为真实字段；
- 有 MongoDB 嵌套对象时追加子 struct；
- 如表名和 GORM 默认复数推断不一致，生成 `TableName()`；
- 生成 `XxxColumnNames` 白名单，供动态查询防止列名 SQL 注入。

单张表片段最终进入 `getModelCode`，统一补 package/import，并经过 `go/format.Source`。因此 parser 输出的 `codes["model"]` 已经是一份可编译的完整 Go model 文件内容。

## 8. 最后怎样写进文件

以完整 HTTP 生成为例，`httpGenerator.addFields` 创建 replacement field：

```text
model 模板中的示例 model 区域
  ← codes[parser.CodeTypeModel]
```

随后：

```text
r.SetSubDirsAndFiles(...)    只选需要的模板
r.SetOutputDir(...)          确定输出目录
r.SetReplacementFields(...) 注册全部替换规则
r.SaveFiles()                复制并替换模板文件
```

单独执行 `sponge web model` 走 `modelGenerator`，只选择 `internal/model/userExample.go`，仍使用相同的 `CodeTypeModel` 替换内容。

## 9. 建议断点顺序

```text
HTTPCommand.RunE
sql2code.Generate
sql2code.getSQL
GetMysqlTableInfo / GetPostgresqlTableInfo / GetSqliteTableInfo / GetMongodbTableInfo
parser.ParseSQL
parser.makeCode
mysqlToGoType
newCrudInfo
getModelStructCode
getModelCode
httpGenerator.addFields（或 modelGenerator.addFields）
replacer.SaveFiles
```

观察变量时重点看 `sql`、`fieldTypes`、AST 的 `stmt.Cols`、`tmplData.Fields`、`CrudInfo`、`codes["model"]`，这样能看到每一步的信息是否丢失。
