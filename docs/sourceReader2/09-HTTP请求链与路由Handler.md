# 09 HTTP 请求链：从端口到数据库再到响应

本章以 `POST /api/v1/userExample` 和 `GET /api/v1/userExample/:id` 为例。普通 HTTP 使用 `internal/types`，Proto HTTP 使用 `api/...pb.go`，两者数据层相同。

## 1. 完整调用图

```text
TCP -> net/http.Server -> gin.Engine
 -> 全局 middleware
 -> /api/v1 路由组 -> /userExample 路由
 -> handler.Create/GetByID
 -> request bind + validate
 -> copier.Copy(request, model)
 -> dao -> cache / gorm -> database
 -> copier.Copy(model, reply)
 -> response.Success/Error -> JSON
```

## 2. server.NewHTTPServer

构造器先合并 `HTTPOption`：`isProd` 控制 Gin mode，`tls` 决定 plain HTTP、自签名、ACME 自动证书或外部证书，registry/instance 用于注册发现。随后 `routers.NewRouter()` 得到 `http.Handler`，装进标准库 `http.Server`。模板设置 60 秒 IdleTimeout 和 1 MiB MaxHeaderBytes；读写超时留作按业务开启。

## 3. 中间件顺序就是执行语义

`routers.NewRouter` 的顺序为：

1. Gin Recovery：panic 转 500，避免进程退出；
2. CORS；
3. 可选全局 Timeout；
4. RequestID；
5. Logging，从 context 提取 request ID；
6. 可选 Metrics；
7. 可选 RateLimit；
8. 可选 CircuitBreaker；
9. 可选 Tracing；
10. 可选 pprof。

中间件像洋葱：请求按注册顺序进入、响应逆序退出。RequestID 必须早于日志，否则日志无法关联。全局 timeout 简单但可能不适合上传/流式请求，源码也提示可移动到具体路由。

## 4. 系统路由与业务路由

固定路由包括 `/health`、`/ping`、`/codes`。非 prod 额外暴露 `/config` 与 `/swagger/*any`；这也是为什么环境配置错误会泄漏配置。业务路由通过包级切片 `apiV1RouterFns` 聚合：每个实体文件在 `init()` 里 append 注册函数，`NewRouter` 最后统一执行。

`userExampleRouter` 建立 `/api/v1/userExample` 子组，并绑定 Create/Delete/Update/Get/List。鉴权示例默认注释，生产必须主动启用。

## 5. 普通 Handler 的五步模板

以 Create 为例：

1. `ShouldBindJSON(&types.CreateUserExampleRequest{})` 同时反序列化并执行 binding 标签。
2. 参数错误写 `ecode.InvalidParams` 并 return。
3. `copier.Copy` 把 DTO 转成 `model.UserExample`；复杂字段需手工赋值。
4. `middleware.WrapCtx(c)` 取得标准 context，保留 request ID/trace，再调用 DAO。
5. 成功返回 `response.Success(c, gin.H{"id": model.ID})`。

Get 的不同点：区分 `database.ErrRecordNotFound` 与真正内部错误；模型转 `UserExampleObjDetail` 时刻意不输出 Password。Delete/Update 先用 `utils.StrToUint64E` 验证 path id 非零。List 把 `types.Params` 传给 DAO，返回 records 与 total。

## 6. DTO、Model 不能合并

`internal/types/userExample_types.go` 是 HTTP 契约：请求 binding、对外字段、分页包装。`internal/model/userExample.go` 是存储契约：表名、GORM 列、软删除/时间字段、查询白名单。分开可防止密码和内部字段意外暴露，也允许 API 字段与数据库演进速度不同。

`UpdateUserExampleByIDRequest` 的零值与“未提供”混在一起；DAO 又用 `>0`/`!=""` 选择更新，所以不能把数值改为 0、字符串改为空。若业务需要，应改成指针字段或显式 field mask，而不是盲目沿用模板。

## 7. Proto 驱动的 Gin HTTP

`routers_pbExample.go` 使用由 proto 插件生成的 router/service glue；`handler/userExample_logic.go` 实现 `UserExampleLogicer`。请求先由生成代码从 URI/query/body 绑定为 protobuf message，再调用 `req.Validate()`（规则来自 `validate.proto`），最后进入与普通 handler 相同的 DAO。

与纯 HTTP 的关键差异：

- 请求/响应类型来自 `api/serverNameExample/v1`；
- 校验规则定义在 `.proto` 而非 Go binding tag；
- 业务函数返回 `(reply, error)`，胶水层把 error 转 HTTP；
- path 字段依赖 `tagger.tags = "uri:\"id\""`；
- `google.api.http` 既服务 gateway，也表达 REST 映射。

## 8. 错误响应路径

参数错使用公开的业务错误；内部 DB 错先记录详细日志，再只输出通用 InternalServerError，避免泄漏 DSN/SQL。`response.Error` 接收 errcode；`response.Output` 可接已转换的 HTTP code 结构。每次输出后必须 return，否则可能重复写响应。

## 9. 测试与重实现清单

单测推荐把接口拆成表驱动案例：合法请求、JSON 语法错、binding 失败、id=0、DAO not found、DAO error、copier error、成功。DAO 应以接口注入，避免 handler 单测连接真实数据库；当前无参构造器适合运行模板，重实现时可增加 `NewUserExampleHandlerWithDao(mock)`。

重实现顺序：先建 model；再定义最小 DTO；实现 DAO 接口假对象；写 handler；注册单一路由；用 `httptest` 验证 JSON；最后加入中间件、真实 DAO、缓存、Swagger。每一步都能独立运行，定位问题最容易。
