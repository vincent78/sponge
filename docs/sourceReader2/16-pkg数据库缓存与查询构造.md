# 16. `pkg/sgorm`、`mgo`、`cache`：数据访问基础

## 1. SQL 初始化不是“只打开一个连接”

`sgorm/mysql`、`postgresql`、`sqlite` 都返回 `*gorm.DB`，但驱动和 option 不同。共同过程是：

```text
配置 DSN
 → 创建 database/sql pool
 → 设置 idle/open/lifetime
 → gorm.Open(driver, gorm.Config)
 → 安装日志、trace、业务 plugin
 → 可选读写分离
 → 返回共享的 *gorm.DB
```

`*gorm.DB` 是并发安全的会话入口，底层 `*sql.DB` 才是连接池；不要每个请求 Init/Close。应用启动时创建一次，退出时通过 `dbclose.Close` 或各驱动 `Close` 关闭。

### MySQL

`mysql.Init` 先 `sql.Open` 并设置池参数，再用该连接创建 GORM；默认 singular table，迁移时默认禁用外键，并设置 utf8mb4 table option。`WithEnableTrace` 安装 `otelgorm`；`WithRWSeparation` 使用 `dbresolver`，replicas 随机选择；`WithGormPlugin` 安装扩展插件。

### PostgreSQL / SQLite

结构与 MySQL 对齐：日志、慢 SQL、池、trace、plugin。SQLite 的“连接池”语义和文件锁不同于网络数据库；内存库和多连接还可能看到不同数据库，应根据 DSN/测试场景限制 open conns。PostgreSQL DSN 由 `utils.AdaptivePostgresqlDsn` 辅助归一化。

### 日志

`sgorm/glog.NewCustomGormLogger` 把 GORM logger 接到 zap，并从 context 取 request ID；慢查询配置会创建标准库 logger。若同时配置 logging 与 slow threshold，后设置的 logger 可能覆盖前者，阅读 option 消费顺序非常重要。

## 2. 基础模型与自定义类型

`sgorm.Model`/`Model2` 提供 ID、创建/更新时间和软删除等公共字段，`DB`、`KV`、`ErrRecordNotFound` 是别名/再导出。`GetTableName` 借助 GORM statement 推断模型表名。

`user_defined.go` 的 `Bool/BitBool/TinyBool` 实现数据库扫描和值转换，用于不同驱动的 bit/tinyint 布尔兼容；`SetDriver` 改变解释方式。重实现要实现 `sql.Scanner` 与 `driver.Valuer`，测试 nil、数字、字节、非法输入，并避免全局 driver 在多数据库进程中相互干扰。

## 3. 安全的动态查询

`sgorm/query.Params` 包含 Page、Limit、Sort 和 Columns。每个 `Column` 有 Name、Exp、Value、Logic。支持 eq/neq/gt/gte/lt/lte/like/in/notin/isnull/isnotnull，以及 and/or 和括号分组。

转换流程：

1. `checkExp` 把受控表达式映射为 SQL 片段。
2. LIKE 对内部 `%`、`_` 转义，并按输入决定是否自动包 `%`。
3. IN 字符串拆分成数字或字符串 slice，占位符使用 `(?)`。
4. NULL 表达式不需要参数。
5. 校验 logic 和括号分组。
6. `ConvertToGormConditions` 产出条件字符串和独立 args，交给 `db.Where(query, args...)`。

列名不能用 `?` 参数化，因此 `WithWhitelistNames` 是防 SQL 注入的关键，不是可选优化。Sort 同样必须经过列白名单。`WithValidateFn` 可做业务级联合校验。

分页由 `NewPage`/`DefaultPage` 计算 limit、offset、sort；`SetMaxSize` 限制单页最大值。Page 采用从 0 开始，offset=`page*limit`。重实现时明确页码基准，避免前端按 1 开始导致少一页。

## 4. MongoDB 层

`mgo.Init(dsn)` 从 URI 提取库名，`Init2(uri, dbName)` 显式指定；两者连接后返回 `*mongo.Database`。`Close` 从 database 取得 client 并 disconnect。`WithOption` 返回 driver options，`NewCustomLogger` 接 zap。

`mgo.Model` 提供 ObjectID、时间和软删除字段。辅助函数：

- `ExcludeDeleted(filter)` 注入未删除条件；
- `EmbedUpdatedAt(update)` 写更新时间；
- `EmbedDeletedAt(update)` 写软删除时间；
- `ConvertToObjectIDs` 把字符串 ID 转换为 ObjectID。

`mgo/query` 与 `sgorm/query` 采用相似的 Params/Column 心智模型，但输出 BSON filter 和 sort。不能把 SQL 运算符直接拼进 Mongo；实现要限制字段名和操作符，正确组合 `$and/$or` 及括号分组。数据库对象为长期资源，请求级超时通过 context 传递。

## 5. Cache 接口和两种驱动

`cache.Cache` 统一：`Set/Get/MultiSet/MultiGet/Del/SetCacheWithNotFound`。`DefaultClient` 加包级便捷函数，但全局可变对象会让测试互相影响；业务更适合显式依赖注入接口。

### Key 和编码

`BuildCacheKey(prefix,key)` 生成 `prefix:key`，空 key 报错。对象用 `pkg/encoding` 编解码，`newObject` 用于批量读取时创建目标对象。调用 `Get(ctx,key,&dst)` 时必须传可写指针；MultiGet 的目标必须是可写 map，反射类型要与 `newObject` 返回值匹配。

### Redis

`NewRedisCache` 接收 `*redis.Client`，便于测试。Set 编码后写 Redis；Get 将 `redis.Nil` 原样作为未命中；MultiSet 用 pipeline MSet 后逐 key Expire；MultiGet MGet 后反射注入 map；Del 支持多 key。Cluster 版本实现相同接口，但跨 slot 的批量命令需要特别关注 hash slot 限制。

### Memory

`InitMemory` 创建 Ristretto，关键 option 是频率计数器数量、最大 cost、buffer items。全局 client 用 lazy init；`NewMemoryCache` 包装相同 Cache 接口。Ristretto 写入是异步的，因此 Set 后调用 `Wait()` 保证紧接着 Get 可见。关闭应用时调用 `CloseGlobalMemory`。

### 缓存穿透占位符

未查到数据库记录时写 `"*"`，TTL 为 `DefaultNotFoundExpireTime`；再次 Get 返回 `ErrPlaceholder`，从而避免不存在的 key 持续击穿数据库。要严格区分：

- `CacheNotFound`/`redis.Nil`：缓存中没有，应该查数据库；
- `ErrPlaceholder`：数据库此前也没有，直接返回 not found；
- 其他 error：缓存故障，可降级查库但要记录。

占位符 TTL 应短于正常对象，写入新对象后必须覆盖占位符。

## 6. Cache-aside 的正确请求链

```text
Get cache
 ├─ hit → 返回对象
 ├─ placeholder → 返回业务 not found
 ├─ miss → 查 DB
 │    ├─ found → Set cache → 返回
 │    └─ absent → SetCacheWithNotFound → not found
 └─ cache error → 记录 → 查 DB（按可用性策略）
```

更新通常先写 DB，再删除 cache，而不是直接改 cache；并发一致性要求更高时可用延迟双删、版本 key、消息失效等方案。当前包只提供原语，不替业务决定一致性策略。

## 7. 连接客户端

- `goredis.Init` 解析 DSN；另有 single、sentinel、cluster 构造和对应 Close。
- `etcdcli`、`consulcli`、`nacoscli` 是注册发现底层客户端初始化器。
- 调用方负责设置连接超时、认证/TLS、启动探活，并在 `app.closes` 中释放。

## 8. 重实现与测试清单

先用 SQLite + memory cache 重写一个 GetByID，再替换 MySQL + Redis。测试：

- 连接池 option 真正落到底层 `sql.DB`；
- GORM not found 与普通 DB error 分开；
- query 所有 operator、括号、中文 LIKE、非法字段、非法 sort；
- cache hit/miss/placeholder/损坏数据；
- MultiGet 空输入和部分命中；
- context timeout；
- Close 可重复调用；
- 并发更新时不会永久返回旧值。
