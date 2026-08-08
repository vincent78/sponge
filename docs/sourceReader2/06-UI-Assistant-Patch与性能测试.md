# 06 UI、Assistant、Patch 与性能测试

## 1. UI 是 CLI 的 HTTP 包装层

`sponge run` 默认监听 24631，并尝试打开浏览器。`server/http.go` 嵌入 `static` 前端。默认地址直接从 embed.FS 提供；自定义地址时把静态文件复制到 `frontend`，并替换 `appConfig.js` 中的默认后端地址。

路由集中在 `/api/v1`：生成、模板信息、assistant、压测启停、上传、列数据库表、列 driver/LLM、读取最近参数记录。`NoRoute` 返回 index.html，解决 SPA history 刷新 404。

## 2. UI 生成调用链

```text
POST /api/v1/generate {arg,path}
  → GenerateCode
  → handleGenerateCode
  → strings.Split(arg, " ")
  → 拼临时 --out
  → gobash.Run(ctx, "sponge", args...)
  → 压缩输出目录
  → 返回 zip
  → 记录参数并异步清理临时文件
```

模板信息复用同一执行器，但 `OnlyPrint` 时返回 stdout 文本。两分钟生成超时，清理协程最多十分钟轮询。参数记录按 client IP + command type 保存，用于 UI 回填。

数据库表枚举分别走 MySQL/TiDB、PostgreSQL schema/catalog、SQLite、MongoDB collection。输入绑定失败返回 InvalidParams，连接/查询失败返回 InternalServerError。

安全边界：UI 实际在服务器上执行本机 `sponge`，DSN 和上传文件都来自请求。公开部署必须鉴权、限制 origin/请求体、输出路径与命令参数；当前按空格分参不理解引号，且把错误信息直接返回。复刻应传结构化 JSON options，再构造 argv，不接收一整条命令字符串。

## 3. Assistant 生成

assistant 支持 chatgpt/deepseek/gemini，命令分 chat、generate、merge、clean。`generate` 的核心不是让模型自由改仓库，而是定位源码中的 assistant marker：

1. `checkDirAndFile` 验证目录/指定文件。
2. `parseFiles` 找到待实现函数及上下文，并判断中英文。
3. `initPromptTemplate` 初始化提示模板，`getPrompt` 拼依赖文件与目标函数。
4. 为每个文件创建客户端和 `assistantTask`。
5. `WorkerPool` 以 `max-assistant-num` 并发执行。
6. 回复中的 Markdown Go code 被抽取、重组并保存为 `xxx.go.<assistant>.md`。

`--only-print-prompt` 是非常有价值的可测试/审计入口。默认 role 会根据源码语言选中英文；模型由 type 的 default map 选择。单项失败计数而不阻止其他任务收集结果。

WorkerPool 包含 Job/Task/Result、job queue、result channel、WaitGroup 和 context。`Submit` 有超时；`Stop` 负责结束。复刻时要保证 worker panic 被恢复、结果 channel 只关闭一次、取消能传播到 LLM 请求。

## 4. Assistant 合并

`assistant merge` 扫描对应后缀 Markdown，建立源 `.go` 与生成 `.md` 配对；抽取所有 Go fence，调用 `goast.MergeGoCode`。默认 `WithCoverSameFunc` 表示模型实现可覆盖 marker 中的同名空实现；DAO 等目录又配置忽略一组框架基础方法，避免模型覆盖生成器已实现的 CRUD。

写回前备份，`--is-clean` 决定是否删除模型结果。与 protoc merge 相比，Assistant merge 的覆盖意图更强，因此复刻时应展示 diff 并要求明确策略：replace-stub、keep-existing、force-replace。

主要失败点：模型输出不是合法 Go、package 不匹配、代码 fence 缺失、重复声明、跨文件依赖不完整。不要在解析失败时尝试用字符串强拼。

## 5. 通用 merge

生成插件遇到已存在逻辑文件时留下 `*.go.gen20*`。`commands/merge` 根据类型扫描 handler/service/router/ecode，调用模块 AST 合并，成功后删临时文件。备份目录保留原相对路径，避免同名文件互相覆盖。

实现原则是“生成器负责提出增量，merge 负责保护用户代码”。复刻时最好引入三方合并概念：旧生成基线、用户当前文件、新生成文件；仅比较当前与新生成无法可靠判断模板改动还是用户改动。

## 6. Patch 命令族

Patch 是生成后的迁移工具集合：

- `del-omitempty`：批量移除生成 pb JSON tag 的 omitempty；
- `gen-db-init`：生成数据库初始化片段；
- `gen-types-pb`：生成公共 types pb 并替换模块字段；
- `copy-proto` / `copy-third-party-proto` / `copy-go-mod`：补齐工程资源；
- `modify-duplicate-error-code-num`：Go AST 扫描错误码编号并修复重复；
- `modify-duplicate-error-code-offset`：解析变量和值，重新安排 service offset；
- `adapt-mono-repo`：调整已有项目到 mono-repo 布局；
- `modify-proto-package`：根据模块和路径更新 `package`、`go_package`。

这些命令混合了文本替换、Go AST、文件复制和目录适配。每个都应被看作一次 migration：先枚举目标、校验输入、备份、dry-run、应用、验证，而不是无限次重复也安全的普通生成。

## 7. 性能测试架构

`perftest` 支持 HTTP/1.1、HTTP/2、HTTP/3、WebSocket、gRPC。共同模型是：解析并校验参数 → 创建并发 worker/client → 发请求并把 `Result` 送入 channel → collector 聚合 → 打印/保存/推送统计。

HTTP 实现拆成：

- `run.go`：请求参数、调度和生命周期；
- `http1/2/3.go`：不同 transport/client；
- `stats.go`：成功失败、状态码、耗时、百分位、QPS；
- `progress_bar.go`：进度；
- Prometheus pushgateway 或自定义 HTTP endpoint 推送。

分布式模式包含 collector 与 agent。Agent 先向 collector 注册，再暴露 ready/start/stop/cancel/ping；测试 ID 和 agent ID 防止旧控制请求误操作当前任务。Collector 协调多个 Agent 并聚合状态。配置文件 `agent.yml` 提供部署参数。

统计实现需要特别检查：结果 channel 是否会在 worker 前关闭；总耗时和单请求 latency 的单位；百分位要求排序；连接复用、TLS、HTTP/2/3 transport 差异；固定请求数和固定 duration 的停止条件；context cancel 后是否泄漏 goroutine。

## 8. 重实现优先级

1. UI 只做结构化调用 CLI service，不要先复制 shell 字符串方案。
2. 上传和输出放到每请求独立临时目录，校验压缩包路径，防 Zip Slip。
3. Assistant 先实现 prompt-only 和单文件，然后加并发与 merge diff。
4. Patch 统一成 `Migration{Plan, Apply, Verify, Rollback}` 接口。
5. Perftest 先做 HTTP/1.1 单机，明确 Result/Statistics；再加协议 adapter，最后加 collector-agent。
6. 为所有长任务提供 task ID、状态查询、取消和超时，不让 HTTP handler 一直持有连接。

## 9. 推荐阅读断点

- UI：`OpenUICommand.RunE → NewRouter → GenerateCode → handleGenerateCode`。
- Assistant：`GenerateCommand.RunE → parseFiles → getPrompt → WorkerPool.worker → assistantTask.Run`。
- 合并：`mergeParams.runMerge → module.ParseHandlerAndServiceCode → goast`。
- Patch：先读 command 的 RunE，再读同文件的 run/AST 函数。
- 压测：`PerfTestHTTPCMD.RunE → Run → worker → statsCollector.collect → printReport`。

沿这五条链单步调试，比从目录第一行顺序读到最后更容易建立系统模型。
