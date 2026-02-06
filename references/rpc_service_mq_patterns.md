# RPC Service MQ 消息队列开发模式

MQ（消息队列）消费者是 RPC 服务的重要组成部分，用于异步处理消息。

## 服务定位

MQ 消费者用于异步处理业务逻辑，解耦服务和提高系统可靠性：

```
┌─────────────────────────────────────────────────────────────┐
│                    gRPC / HTTP 服务                          │
│                        生产消息                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  消息队列     │  (RabbitMQ / Kafka)
                    └──────┬───────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    MQ 消费者                                 │
│                    cmd/mq/main.go                            │
│                         职责：                                │
│  - 消费消息队列中的消息                                        │
│  - 执行异步业务逻辑                                            │
│  - 调用 Logic 层处理业务                                       │
└─────────────────────────────────────────────────────────────┘
```

## 部署模式

| 模式 | 说明 | 适用场景 |
|-----|------|---------|
| **独立部署** | MQ 消费者在独立的进程中运行 | 业务繁重、需要独立扩缩容 |
| **合并部署** | MQ 与 gRPC 在同一进程中运行 | 业务量不大、资源节省 |

### 合并部署示例

**文件**：`cmd/grpc/main.go`（与 gRPC 合并）

```go
func main() {
    // ... 初始化

    // 启动 gRPC Server
    grpcServer := server.NewGrpcServer(c, svcCtx)
    go grpcServer.Start()  // 注意：使用 goroutine

    // 启动 MQ Server（可选，根据业务需求）
    mqServer := server.NewMqServer(c.MqConfig, svcCtx)
    mqServer.Start()

    // ... 优雅关闭
    proc.AddShutdownListener(func() {
        grpcServer.Stop()
        mqServer.Stop()
    })()
}
```

## 支持的消息队列

| 队列类型 | 包路径 | 说明 |
|---------|--------|------|
| **RabbitMQ** | `apps/pkg/queue/rabbitmq` | RabbitMQ 消费者 |
| **Kafka** | `apps/pkg/queue/kafka` | Kafka 消费者 |

## 开发步骤

| 步骤 | 操作 | 说明 |
|-----|------|-----|
| 1 | 定义消息体 | 在 `protos/common/mq/xxx.proto` 中定义消息 |
| 2 | 配置队列 | 在 `internal/config/config.go` 中添加队列配置 |
| 3 | 创建 MQ Server | 在 `internal/server/mq.go` 中创建 MQ Server |
| 4 | 实现 Handler | 在 `internal/mq/xxx_mq.go` 中实现 Handler（解析参数，调用 Logic） |
| 5 | 实现 Logic | 在 `internal/logic/mq/xxx_logic.go` 中实现业务逻辑 |
| 6 | 创建入口 | 在 `cmd/mq/main.go` 中创建入口或合并选择 |

## 服务内部结构

```
app/services/{type}/{service}/
├── cmd/
│   ├── mq/               # MQ 消费者入口（可选）
│   │   ├── main.go
│   │   └── conf.yaml
│   └── grpc/             # gRPC 入口（可合并 MQ）
└── internal/
    ├── config/           # 配置定义
    │   └── config.go     # 添加 MqConfig
    ├── mq/               # MQ Handler（解析参数，调用 Logic）
    │   └── xxx_mq.go
    ├── logic/            # 业务逻辑层
    │   └── mq/
    │       ├── userservice/           # user mq logic
    │       │   └── shared/            # user mq logic shared code
    │       └── userthirdservice/
    │           └── shared/            # user third mq logic shared code
    ├── manager/          # Logic 复用层（跨 logic 共享）
    ├── repository/       # 依赖注入层（与 gRPC 共享）
    │   ├── model/        # 数据库 Model
    │   ├── rpc/          # RPC 客户端
    │   ├── xredis/       # Redis 操作
    │   ├── repository.go
    │   └── repository_type.go
    ├── server/           # Server 初始化
    │   └── mq.go         # MQ Server 实现
    └── svc/              # ServiceContext
```

## Protobuf 消息定义

MQ 消息体使用 Protobuf 定义，文件位于 `protos/common/mq/` 目录下。

> **注意**：每个服务一个文件，文件名就是服务名。只使用 `message` 和 `enum` 定义，其他不需要。

### 目录结构

```
protos/common/mq/
└── user.proto          # User 服务 MQ 消息定义
```

### 消息定义示例

**user.proto**

```proto
syntax = "proto3";

package common.mq;

option go_package = "apps/pb/common/mq";

// UserCountReq 统计消息
message UserCountReq {
  // 用户ID
  int64 user_id = 1;
}

// UserUpLevelReq 升级消息
message UserUpLevelReq {
  // 用户ID
  int64 user_id = 1;
  // 目标等级
  int32 level = 2;
  // 消息ID（用于幂等性）
  string message_id = 3;
}

// UserSyncReq 同步消息
message UserSyncReq {
  // 用户ID
  int64 user_id = 1;
  // 操作类型
  string action = 2;
}
```

### 命名规范

| 规范 | 说明 |
|-------|------|
| 文件名 | 服务名（小写），如 `user.proto`、`order.proto` |
| package | `common.mq` |
| go_package | `apps/pb/common/mq` |
| Message 名 | 操作 + Req，如 `UserCountReq`、`OrderCreateReq` |
| 字段名 | snake_case，如 `user_id`、`message_id` |

### 生成代码

```bash
# 更新 protobuf submodule
make update

# 构建 protobuf 文件
make build
```

生成后的 Go 代码位于：`apps/pb/common/mq/`

## 消费者方法签名

### 标准 Handler 签名

```go
func (r *XxxMQ) ConsumerName() queue.ConsumeHandlerFunc {
    return func(ctx context.Context, key string, payload []byte) error {
        // 1. 解析消息参数（protobuf）
        var req xxxMqPb.XxxReq
        if err := proto.Unmarshal(payload, &req); err != nil {
            return xerr.NewMsgWithErrorLog(
                xerr.SystemError_PARAMS_ERROR,
                "unmarshal message failed",
                "key: %s, payload: %s, err: %v",
                key, string(payload), err,
            )
        }

        // 2. 调用 Logic 层处理业务
        xxxLogic := logic.NewXXXLogic(ctx, r.svcCtx)
        return xxxLogic.Method(...)
    }
}
```

**Handler 层只负责：**
- 解析消息参数（Protobuf）
- 调用 Logic 层
- 返回结果

**业务逻辑在 Logic 层实现：** `internal/logic/mq/xxx_logic.go`

| 参数 | 类型 | 说明 |
|-----|------|------|
| `ctx` | `context.Context` | 请求上下文 |
| `key` | `string` | 消息 key / routing key |
| `payload` | `[]byte` | 消息内容（protobuf 二进制） |

## 消息处理最佳实践

### 1. 解析消息（Protobuf）

```go
import userMqPb "apps/pb/common/mq"

func (r *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
    return func(ctx context.Context, key string, payload []byte) error {
        var req userMqPb.UserCountReq
        if err := proto.Unmarshal(payload, &req); err != nil {
            return xerr.NewMsgWithErrorLog(
                xerr.SystemError_PARAMS_ERROR,
                "unmarshal message failed",
                "key: %s, payload: %s, err: %v",
                key, string(payload), err,
            )
        }
        // 调用 Logic 层处理
        countLogic := logic.NewCountLogic(ctx, r.svcCtx)
        return countLogic.Count(req.UserId)
    }
}
```

### 2. 错误处理

**Handler 层错误处理：**
- 参数解析失败：使用 `xerr.NewMsgWithErrorLog` 返回参数错误
- Logic 层返回的错误：直接返回

**Logic 层错误处理：**
- 业务错误：使用业务定义的错误码，`xerr.NewMsgWithInfoLog`
- 系统错误：使用系统错误码，`xerr.NewWithErrorLog`

### 3. 重试机制

消息队列通常内置重试机制，消费者返回错误时会自动重试：
- RabbitMQ: 根据配置的 `max_attempts` 重试
- Kafka: 手动提交 offset，失败不提交即可重试

### 4. 幂等性处理（Logic 层示例）

防止重复消费导致数据不一致，幂等性处理应该在 Logic 层实现，使用 repository/xredis 中包装的方法：

```go
// internal/logic/mq/up_level_logic.go
func (l *UpLevelLogic) UpLevel(userID int64, level int, messageID string) error {
    // 使用消息 ID 实现幂等性
    processed, err := l.msgIdempotencyRdsModel.IsMessageProcessed(l.ctx, messageID)
    if err != nil {
        return xerr.NewWithErrorLog(
            xerr.SystemError_D_ERROR,
            "check message processed failed, message_id: %s, err: %v", messageID, err,
        )
    }
    if processed {
        xlog.Infow(l.ctx, "message already processed", "message_id", messageID)
        return nil
    }

    // 查询用户
    user, err := l.userModel.FindOne(l.ctx, userID)
    if err !=) nil {
        return xerr.NewWithErrorLog(
            xerr.SystemError_D_ERROR,
            "find user failed, user_id: %d, err: %v", userID, err,
        )
    }

    // 更新等级
    user.Level = level
    user.LevelTime = time.Now()
    if err := l.userModel.Update(l.ctx, user); err != nil {
        return xerr.NewWithErrorLog(
            xerr.SystemError_D_ERROR,
            "update user level failed, user_id: %d, err: %v", userID, err,
        )
    }

    // 发送通知
    if err := l.notifyClient.Send(l.ctx, user); err != nil {
        xlog.Errorw(l.ctx, "send notify failed",
            "user_id", userID,
            "err", err,
        )
    }

    // 标记消息已处理
    return l.msgIdempotencyRdsModel.MarkMessageProcessed(l.ctx, messageID, 24*3600)
}
```

## 单元测试

```go
func TestUserMQ_CountConsumer(t *testing.T) {
    mockSvcCtx := &svc.ServiceContext{
        // 设置 Mock 依赖
    }

    userMQ := NewUserMQ(mockSvcCtx)
    handler := userMQ.CountConsumer()

    ctx := context.Background()
    req := &userMqPb.UserCountReq{UserId: 123}
    payload, _ := proto.Marshal(&req)

    err := handler(ctx, "test-key", payload)
    assert.NoError(t, err)
}
```

## 相关示例

| 文件 | 说明 |
|-----|------|
| [proto_message.md](../example/rpc_service/mq/proto_message.md) | MQ Protobuf 消息定义 |
| [handler.md](../example/rpc_service/mq/handler.md) | MQ Handler 示例 |
| [logic.md](../example/rpc_service/mq/logic.md) | MQ Logic 示例 |
| [repository.md](../example/rpc_service/mq/repository.md) | MQ Repository 层示例 |
| [config.md](../example/rpc_service/mq/config.md) | MQ Config 示例 |
| [server.md](../example/rpc_service/mq/server.md) | MQ Server 示例 |
| [svc_context.md](../example/rpc_service/mq/svc_context.md) | ServiceContext 初始化（与 gRPC 共享） |
