# 11 数据层：Model、DAO、Cache 与数据库

## 1. 层次与所有权

```text
handler/service
 -> DAO interface
    -> cache interface -> memory/redis
    -> GORM DB -> mysql/postgresql/sqlite
 -> model（表映射与查询白名单）
```

Handler 不写 SQL，Cache 不知道 HTTP/gRPC，Model 不负责连接。这个边界使同一 DAO 能被两种传输层复用。

## 2. Model

`model.UserExample` 嵌入 `sgorm.Model`，获得 ID、CreatedAt、UpdatedAt、DeletedAt；其余字段用 `gorm:"column:..."` 映射。`TableName()` 固定返回 `user_example`。`UserExampleColumnNames` 是自定义查询允许的字段集合，不是文档装饰：List 把用户提交的列名转换成 SQL 前必须检查它。

模型包含 Password，而返回 DTO 不含 Password。复制时方向必须正确，且不能直接 JSON encode model。

## 3. DB 初始化

`database.InitDB` 对 driver 小写化，支持 mysql/tidb、postgresql、sqlite，否则 panic。各实现读取 DSN、连接池最大空闲/打开数、连接最大生命周期；EnableLog 时注入 logger 和 request_id；EnableTrace 时注入 GORM tracing。PostgreSQL 额外 `sgorm.SetDriver("postgresql")` 处理方言差异。

全局 `gdb` 也支持 `GetDB` + `sync.Once` 惰性初始化，但正常启动路径是 InitApp 主动初始化。`ErrRecordNotFound` 重导出 sgorm 错误，让上层不依赖具体 ORM 包。

## 4. Cache 统一抽象

`database.CacheType` 保存类型与可选 Redis client。`cache.NewUserExampleCache` 根据类型创建 memory 或 redis 后端，JSON 编解码，未知/空类型返回 nil，DAO 因而自然进入无缓存模式。

实体缓存 key 为 `userExample:<id>`，TTL 五分钟。接口包含 Set/Get/MultiSet/MultiGet/Del，以及负缓存 `SetPlaceholder/IsPlaceholderErr`。相同业务接口屏蔽了内存与 Redis 差异。

## 5. DAO 构造与接口

`NewUserExampleDao(db, cache)` 返回接口；cache 非 nil 时同时创建 `singleflight.Group`。基础方法有 Create、DeleteByID、UpdateByID、GetByID、GetByColumns，另有接收 `*gorm.DB` 的事务版本。事务由上层开启并传入，DAO 不擅自 commit。

Create 依靠 GORM 把自增 ID 回写 model。Delete/Update 成功后删除缓存，采用 cache-aside。缓存删除失败只忽略/记录，不让主要写操作失败；这意味着短时间可能读旧值，应根据一致性需求调整。

## 6. GetByID 的缓存算法

```text
无 cache -> DB First
有 cache -> cache.Get
  命中 -> 返回
  普通 miss -> singleflight(id)
       -> DB First
          找不到 -> 写 placeholder（默认约 10 分钟）
          找到 -> 写实体缓存（5 分钟）
  placeholder -> 映射为 ErrRecordNotFound
  其他 cache error -> 返回 error
```

singleflight 防止同一个 id 高并发 miss 时同时打数据库（击穿）；placeholder 防止持续查询不存在 id（穿透）。它不能防止大量不同 id 的恶意查询，仍需限流/布隆过滤器。

一个值得审视的策略是：Redis 暂时故障时源码直接返回 cache error，而不是降级查 DB。重实现时需明确“缓存是强依赖还是可降级组件”。

## 7. Update 的零值陷阱

`updateDataByID` 建 map，仅当字符串非空、数字大于 0 才加入。这实现“部分更新”，却不能设置空字符串或 0，且示例没有更新 Status。若重新实现，推荐：

- HTTP DTO 使用 `*string/*int` 表示是否出现；或
- Proto 使用 `FieldMask` / optional；
- 根据 presence 组装 update map；
- 明确禁止更新 ID、CreatedAt 等字段。

不要直接 `Save(table)`，它可能覆盖调用方未提供的字段。

## 8. 自定义分页查询

`GetByColumns` 调用 `params.ConvertToGormConditions(query.WithWhitelistNames(...))`，得到占位符 SQL 与 args。除 `Sort == "ignore count"` 外先 Count；total=0 提前返回。随后 `ConvertToPage()` 得到 order/limit/offset，再 Find。

必须同时约束列名、操作符、排序字段和分页上限。值应始终走参数绑定，不能字符串拼接。`ignore count` 是性能选项，但客户端将得不到真实总数。

## 9. 事务方法

`CreateByTx/DeleteByTx/UpdateByTx` 使用调用方传来的 tx，并继续 `WithContext(ctx)`。典型用法：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if _, err := daoA.CreateByTx(ctx, tx, a); err != nil { return err }
    return daoB.UpdateByTx(ctx, tx, b)
})
```

缓存失效发生在事务真正 commit 之前可能产生竞态。高一致性系统应在 commit 成功后删缓存，或采用 outbox/版本号；模板适合常规 CRUD，但不是所有一致性场景的最终答案。

## 10. 数据层测试与重实现

优先用临时 SQLite 做 DAO 集成测试，覆盖软删除、ID 回写、部分更新、分页/排序、非法字段、事务 rollback。缓存用 memory 后端测试 miss-hit、placeholder、TTL、删除失效；并发启动 100 个相同 Get，断言 DB 查询近似一次。Redis 行为用容器测试 MultiGet、超时与序列化。

重实现顺序：Model/TableName -> DB 初始化 -> 无缓存 DAO CRUD -> 查询白名单/分页 -> 事务 -> Cache interface/memory -> cache-aside -> singleflight/placeholder -> Redis -> tracing/metrics。每加入一层都保持旧测试通过。
