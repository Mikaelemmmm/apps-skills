# RPC 服务开发模式

RPC 服务（领域服务层）位于 `app/services/` 目录下。

> **注意**：仅在**方案二**和**方案三**中需要使用 gRPC 服务。**方案一（HTTP 单体）**不需要 gRPC 服务。

## 服务定位（按架构模式）

### 方案二：HTTP + gRPC（业务逻辑在 HTTP）

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP (app/apis/)                          │
│                         职责：                                │
│  - 业务逻辑编排：调用多个 gRPC 服务完成业务流程                 │
│  - 业务规则实现                                              │
│  - 可调用 gRPC 的原子服务和复用功能                            │
└─────────────────────────────────────────────────────────────┘
                              ↓ gRPC
┌─────────────────────────────────────────────────────────────┐
│                      gRPC (app/services/)                    │
│                         职责：                                │
│  - 原子服务：单一职责的细粒度操作                              │
│  - 复用功能：可被多个 HTTP 服务调用                            │
│  - 数据库操作                                                │
└─────────────────────────────────────────────────────────────┘
```

**适用**：中等规模、HTTP 层有业务逻辑、需要服务复用

---

### 方案三：BFF + 领域服务（业务逻辑在领域服务）

```
┌─────────────────────────────────────────────────────────────┐
│                      BFF (app/apis/)                         │
│                         职责：                                │
│  - 聚合裁剪：前端请求的统一入口                                │
│  - CUD 操作：透传到 gRPC（无业务逻辑）                          │
│  - Query 操作：聚合多个 gRPC 服务数据                          │
│  - 字段裁剪：按前端需求返回                                    │
│  - 不包含业务逻辑                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓ gRPC
┌─────────────────────────────────────────────────────────────┐
│                   领域服务 (app/services/)                    │
│                         职责：                                │
│  - 核心业务逻辑：按领域划分实现                                │
│  - 业务规则实现                                              │
│  - 数据库操作                                                │
└─────────────────────────────────────────────────────────────┘
```

**适用**：大团队、多端接入、领域驱动设计

## 服务类型划分

| 类型 | 目录 | 说明 | 示例 |
|-----|------|-----|------|
| **core** | `services/core/` | 核心业务领域服务 | user, order, payment |
| **shared** | `services/shared/` | 跨业务共享服务 | authz, permit, dictionary |
| **support** | `services/support/` | 支撑服务 | message, notification, file |

> **重要**：创建 RPC 服务前，务必先评估其归属类型，确保放置在正确的领域分类下。

## 开发步骤

| 步骤 | 操作 | 说明 |
|-----|------|-----|
| 1 | 定义协议 | 参考 [rpc_proto.md](../example/proto_define/rpc_proto.md) |
| 2 | 生成代码 | `make gen type=services srv=core/user` |
| 3 | 注册服务 | 在 `internal/server/grpc.go` 中注册，参考 [server_grpc.md](../example/rpc_service/server_grpc.md) |
| 4 | 定义错误码 | 参考 [error_code_proto.md](../example/proto_define/error_code_proto.md) |
| 5 | 生成 Model | 参考 [database_patterns.md](database_patterns.md) |
| 6 | 编写 Repository | 参考 [repository.md](../example/rpc_service/repository.md) |
| 7 | 配置 ServiceContext | 参考 [svc_context.md](../example/rpc_service/svc_context.md) |
| 8 | 编写 Logic/Service | 参考 [logic.md](../example/rpc_service/logic.md) |

## 服务内部结构

```
app/services/{type}/{service}/
├── cmd/
│   ├── grpc/             # gRPC 服务入口
│   ├── mq/               # 消息队列消费者入口
│   └── job/              # 定时任务入口
├── internal/
│   ├── config/           # 配置定义
│   ├── logic/            # 业务逻辑层（或 service/）
│   │   └── userv1/
│   │       ├── userservice/           # user logic
│   │       │   └── shared/            # user logic shared code
│   │       └── userthirdservice/
│   │           └── shared/            # user third logic shared code
│   ├── manager/          # Logic 复用层（跨 logic 共享）
│   ├── repository/       # 依赖注入层
│   │   ├── model/        # 数据库 Model
│   │   ├── rpc/          # RPC 客户端
│   │   ├── xredis/       # Redis 操作
│   │   ├── repository.go
│   │   └── repository_type.go
│   ├── server/           # Server 初始化
│   └── svc/              # ServiceContext
└── ...
```

## 代码生成命令

```bash
# 生成 RPC 服务代码
make gen type=services srv=core/user

# 参数说明
# type=services : 生成 RPC 服务
# srv           : 服务路径，格式为 {类型}/{服务名}
```

## 相关示例

| 文件 | 说明 |
|-----|------|
| [server_grpc.md](../example/rpc_service/server_grpc.md) | gRPC Server 注册 |
| [repository.md](../example/rpc_service/repository.md) | Repository 层结构（共享） |
| [svc_context.md](../example/rpc_service/svc_context.md) | ServiceContext 初始化（共享） |
| [logic.md](../example/rpc_service/logic.md) | Logic/Service 层业务逻辑（gRPC） |

### Job 相关

| 文件 | 说明 |
|-----|------|
| [rpc_service_job_patterns.md](../references/rpc_service_job_patterns.md) | Job 开发模式说明 |
| [job/handler.md](../example/rpc_service/job/handler.md) | Job Handler 示例 |
| [job/logic.md](../example/rpc_service/job/logic.md) | Job Logic 示例 |
| [job/repository.md](../example/rpc_service/job/repository.md) | Job Repository 层示例 |
| [job/config.md](../example/rpc_service/job/config.md) | Job Config 示例 |
| [job/server.md](../example/rpc_service/job/server.md) | Job Server 示例 |
| [job/svc_context.md](../example/rpc_service/job/svc_context.md) | ServiceContext 初始化（与 gRPC 共享） |

### MQ 相关

| 文件 | 说明 |
|-----|------|
| [rpc_service_mq_patterns.md](../references/rpc_service_mq_patterns.md) | MQ 开发模式说明 |
| [mq/handler.md](../example/rpc_service/mq/handler.md) | MQ Handler 示例 |
| [mq/logic.md](../example/rpc_service/mq/logic.md) | MQ Logic 示例 |
| [mq/repository.md](../example/rpc_service/mq/repository.md) | MQ Repository 层示例 |
| [mq/config.md](../example/rpc_service/mq/config.md) | MQ Config 示例 |
| [mq/server.md](../example/rpc_service/mq/server.md) | MQ Server 示例 |
| [mq/svc_context.md](../example/rpc_service/mq/svc_context.md) | ServiceContext 初始化（与 gRPC 共享） |
| [mq/proto_message.md](../example/rpc_service/mq/proto_message.md) | MQ Protobuf 消息定义 |

## 设计原则

### 方案二：原子服务原则

gRPC 服务应提供细粒度的原子操作，而非复杂的业务编排：

```go
// ✅ 正确：提供原子操作
service UserService {
  rpc CreateUser(CreateUserReq) returns (CreateUserReply);
  rpc GetUser(GetUserReq) returns (GetUserReply);
  rpc UpdateUser(UpdateUserReq) returns (UpdateUserReply);
  rpc DeleteUser(DeleteUserReq) returns (Empty);
}

// ❌ 错误：包含业务编排
service UserService {
  // 复杂的注册流程应由 HTTP 层编排
  rpc RegisterWithThirdAuthAndSendEmailAndCreateCart(...) returns (...);
}
```

### 方案三：领域服务原则

gRPC 服务按领域划分，包含核心业务逻辑：

```go
// ✅ 正确：领域服务包含业务逻辑
service UserService {
  // 用户注册（包含完整业务逻辑）
  rpc Register(RegisterReq) returns (RegisterReply);
  // 用户登录
  rpc Login(LoginReq) returns (LoginReply);
  // 获取用户信息
  rpc GetUser(GetUserReq) returns (GetUserReply);
}
```

### 可复用性

gRPC 服务应设计为可被多个 HTTP/BFF 服务调用：

```
┌─────────────┐     ┌─────────────┐
│  Stayy API  │────▶│             │
│  (BFF/HTTP) │     │             │
└─────────────┘     │  User RPC   │
                    │  Service    │
┌─────────────┐     │             │
│  Admin API  │────▶│             │
│  (BFF/HTTP) │     └─────────────┘
└─────────────┘            │
                           ▼
                    ┌─────────────┐
                    │  Database   │
                    └─────────────┘
```

## 三种入口类型

每个 RPC 服务可包含三种入口：

| 类型 | 目录 | 说明 |
|-----|------|-----|
| **gRPC Server** | `cmd/grpc/` | gRPC 服务端 |
| **MQ Consumer** | `cmd/mq/` | 消息队列消费者 |
| **Scheduled Job** | `cmd/job/` | 定时任务（支持分布式选举） |

## 单元测试要求

- Logic 层覆盖率 >= 80%
- 使用 Mock 的 Repository 进行测试
- 执行 `make mock` 生成 Mock 文件
