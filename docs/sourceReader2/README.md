# Sponge 源码精读与重实现指南

这是一套面向 Go 新手的源码教材，目标是让读者不仅“知道 Sponge 有哪些目录”，还能够依据文档重新实现一个精简版 Sponge，再逐步补齐生产能力。

## 这套文档怎样使用

建议先按顺序阅读基础篇，再从两条主线中选择一条跟代码：

```text
代码生成主线：sponge CLI → 数据库/Proto → 中间模型 → 代码片段 → 工程模板 → 输出项目
服务运行主线：生成项目 main → 初始化 → server → middleware/interceptor → 业务层 → 数据层
```

阅读每章时，最好同时打开文中提到的源码，并亲手完成章末的“重实现检查”。不要一开始逐行阅读 600 多个 Go 文件；先掌握控制流和数据流，再深入具体组件。

## 章节导航

> 并行阅读期间会持续补齐本目录。最终章节以此导航为准。

### 第一部分：建立全局模型

- [00-阅读说明与源码全景.md](00-阅读说明与源码全景.md)：仓库身份、源码规模、目录边界、阅读方法。
- [01-总体架构与两条主链.md](01-总体架构与两条主链.md)：生成期架构、运行期架构、控制流和数据流。
- [02-新手必备Go知识与源码惯用法.md](02-新手必备Go知识与源码惯用法.md)：接口、init、functional options、context、embed、模板和 AST。

### 第二部分：代码生成器

- [03-CLI入口与命令树.md](03-CLI入口与命令树.md)：初始化、升级、插件、Cobra 命令树和共同执行骨架。
- [04-代码生成器与模板写入机制.md](04-代码生成器与模板写入机制.md)：工程 profile、文件变体、替换字段和安全写入。
- [05-Protoc插件与增量合并.md](05-Protoc插件与增量合并.md)：三个自定义插件、临时生成文件和 Go AST 合并。
- [06-UI-Assistant-Patch与性能测试.md](06-UI-Assistant-Patch与性能测试.md)：本地 UI、AI 工作流、修补命令与压测系统。
- [07-数据库到代码的核心算法.md](07-数据库到代码的核心算法.md)：四种数据库的元数据抽取、统一 DDL、AST、类型映射和代码片段。

### 第三部分：生成项目的运行时

- [08-运行时启动初始化与优雅退出.md](08-运行时启动初始化与优雅退出.md)：多种 main、initial 组合根和资源关闭。
- [09-HTTP请求链与路由Handler.md](09-HTTP请求链与路由Handler.md)：Gin 中间件、路由注册、普通/Proto Handler。
- [10-gRPC服务网关与RPC客户端.md](10-gRPC服务网关与RPC客户端.md)：拦截器、Service、Gateway 和服务发现客户端。
- [11-数据层Model-DAO-Cache-数据库.md](11-数据层Model-DAO-Cache-数据库.md)：Model、DAO、缓存、事务和一致性。
- [12-配置错误码依赖注入测试与复刻路线.md](12-配置错误码依赖注入测试与复刻路线.md)：配置、错误边界、可测试组装与运行时复刻。

### 第四部分：公共基础库

- [13-pkg总览与应用生命周期.md](13-pkg总览与应用生命周期.md)：公共包分层和 `pkg/app`。
- [14-pkg-Gin与HTTP请求链.md](14-pkg-Gin与HTTP请求链.md)：Gin middleware、response、validator 和辅助 transport。
- [15-pkg-gRPC与服务注册发现.md](15-pkg-gRPC与服务注册发现.md)：gRPC client/server、拦截器、TLS、registry/discovery。
- [16-pkg数据库缓存与查询构造.md](16-pkg数据库缓存与查询构造.md)：GORM、MongoDB、Redis、cache 与查询构造。
- [17-pkg日志追踪限流与熔断.md](17-pkg日志追踪限流与熔断.md)：日志、OpenTelemetry、指标、限流、熔断和 profile。
- [18-pkg其余包分类索引与重实现清单.md](18-pkg其余包分类索引与重实现清单.md)：其余一级包的全量职责索引。

### 第五部分：验证和重实现

- [19-测试构建部署与质量验证.md](19-测试构建部署与质量验证.md)：测试分层、Makefile、生成物验证和外部依赖。
- [20-从零重实现Sponge的路线图.md](20-从零重实现Sponge的路线图.md)：按可运行里程碑重写，附每阶段验收条件。

## 文档中的术语

- **Sponge 工具**：`cmd/sponge` 编译出的 CLI 和本地 UI 服务。
- **模板仓库**：运行 `sponge init` 后位于 `~/.sponge` 的项目素材。
- **示例模板代码**：本仓库的 `internal`、`api`、`cmd/serverNameExample_*` 等占位工程。
- **生成项目**：用户执行 `sponge web http`、`sponge micro rpc` 等命令后得到的独立工程。
- **字段代码片段**：由数据库 schema 动态生成的 model、DTO、proto、DAO update 等文本。
- **工程骨架模板**：main、server、完整 DAO/handler/service、脚本和部署文件等相对稳定的素材。

## 最重要的理解

Sponge 不是运行时依赖注入“大框架”。它的核心价值主要发生在编译前：把数据库表、Proto 和模板组合成普通 Go 工程。生成项目运行时仍是熟悉的 Gin、gRPC、GORM、Redis 和 OpenTelemetry。重新实现时也应先把“代码生成引擎”和“被生成服务框架”分成两个子系统。
