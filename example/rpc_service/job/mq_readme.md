# Job MQ Skills 文档完整性检查

## 文档文件清单

### MQ
- `example/mq/handler.md` ✅
- `example/mq/logic.md` ✅
- `example/mq/repository.md` ✅
- `example/mq/config.md` ✅
- `example/mq/server.md` ✅
- `example/mq/svc_context.md` ✅

### Job
- `example/job/handler.md` ✅
- `example/job/logic.md` ✅
- `example/job/repository.md` ✅
- `example/job/config.md` ✅
- `example/job/server.md` ✅
- `example/job/svc_context.md` ✅

## 问题记录与修复

### 1. 目录结构说明
- MQ Logic 目录：`internal/logic/mq/`
- Job Logic 目录：`internal/logic/job/`

### 2. 代码示例问题
- ✅ 已修复 `l.ctx.ctx` 笔误
- ✅ 配置结构基于实际 `services/core/user` 代码
- ✅ Server 实现基于实际代码

## 开发完整流程

### MQ 消费者开发流程

1. **配置定义** (`internal/config/config.go`)
   - 添加 `MqConfig` 结构
   - 包含 `service.ServiceConf` 和 `Config`

2. **Repository 层** (`internal/repository/`)
   - 与 gRPC 共享，不需要单独创建
   - 根据 MQ 需求添加 Model/Redis/RPC 客户端

3. **MQ Handler** (`internal/mq/xxx_mq.go`)
   - 解析消息参数
   - 调用 Logic 层
   - 返回结果

4. **MQ Logic** (`internal/logic/mq/xxx_logic.go`)
   - 实现业务逻辑
   - 调用 Repository 层

5. **MQ Server** (`internal/server/mq.go`)
   - 创建消费者列表 `[]service.Service`
   - 使用 `queue.MustNewQueue` 创建消费者

6. **入口文件** (`cmd/mq/main.go`)
   - 初始化 ServiceContext
   - 启动 MQ Server

### Job 定时任务开发流程

1. **配置定义** (`internal/config/config.go`)
   - 添加 `JobConfig` 结构
   - 包含 `service.ServiceConf` 和 `Config`

2. **Repository 层** (`internal/repository/`)
   - 与 gRPC 共享，不需要单独创建
   - 根据 Job 需求添加 Model/Redis/RPC 客户端

3. **Job Handler** (`internal/job/xxx_job.go`)
   - 定义任务方法
   - 调用 Logic 层

4. **Job Logic** (`internal/logic/job/xxx_logic.go`)
   - 实现业务逻辑
   - 调用 Repository 层

5. **Job Server** (`internal/server/job.go`)
   - 定义任务列表 `[]scheduler.Job`
   - 使用 `scheduler.MustNewServer` 创建调度器

6. **入口文件** (`cmd/job/main.go`)
   - 初始化 ServiceContext
   - 启动 Job Server
