# MQ MQ Handler 示例

MQ Handler 层负责解析消息参数并调用 Logic 层处理业务。

## 文件位置

`internal/mq/xxx_mq.go`

## 代码示例

```go
package mq

import (
	"context"
    "apps/app/services/core/user/internal/logic/mq"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/queue"
    "apps/pkg/xerr"
    "apps/pkg/xlog"
    "google.golang.org/protobuf/proto"
    userMqPb "apps/pb/common/mq"
)

// UserMQ 用户 MQ Handler
type UserMQ struct {
	svcCtx *svc.ServiceContext
}

// NewUserMQ 创建用户 MQ Handler
func NewUserMQ(svcCtx *svc.ServiceContext) *UserMQ {
	return &UserMQ{
		svcCtx: svcCtx,
	}
}

// CountConsumer 统计消费者
func (s *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
	return func(ctx context.Context, key string, payload []byte) error {
		// 1. 解析消息参数（protobuf）
		var req userMqPb.UserCountReq
		if err := proto.Unmarshal(payload, &req); err != nil {
			return xerr.NewMsgWithErrorLog(
				xerr.SystemError_PARAMS_ERROR,
				"unmarshal message failed",
				"key: %s, payload: %s, err: %v",
				key, string(payload), err,
			)
		}
		// 2. 调用 Logic 层处理业务
		countLogic := logic.NewCountLogic(ctx, s.svcCtx)
		if err := countLogic.Count(req.UserId); err != nil {
			return err
		}
		return nil
	}
}

// UpLevelConsumer 升级消费者
func (s *UserMQ) UpLevelConsumer() queue.ConsumeHandlerFunc {
	return func(ctx context.Context, key string, payload []byte) error {
		// 1. 解析消息参数（protobuf）
		var req userMqPb.UserUpLevelReq
		if err := proto.Unmarshal(payload, &req); err != nil {
			return xerr.NewMsgWithErrorLog(
				xerr.SystemError_PARAMS_ERROR,
				"unmarshal message failed",
				"key: %s, payload: %s. err: %v",
				key, string(payload), err,
			)
		}
		// 2. 调用 Logic 层处理业务
		upLevelLogic := logic.NewUpLevelLogic(ctx, s.svcCtx)
		if err := upLevelLogic.UpLevel(req.UserId, int(req.Level), req.MessageId); err != nil {
			return err
		}
		return nil
	}
}

// SyncConsumer 同步消费者（Kafka）
func (s *UserMQ) SyncConsumer() queue.ConsumeHandlerFunc {
	return func(ctx context.Context, key string, payload []byte) error {
		// 1. 解析消息参数（protobuf）
		var req userMqPb.UserSyncReq
		if err := proto.Unmarshal(payload, &req); err != nil {
			return xerr.NewMsgWithErrorLog(
				xerr.SystemError_PARAMS_ERROR,
				"unmarshal message failed",
				"key: %s, payload: %s, err: %v",
				key, string(payload), err,
			)
		}
		// 2. 调用 Logic 层处理业务
		syncLogic := logic.NewSyncLogic(ctx, s.svcCtx)
		if err := syncLogic.Sync(req.UserId, req.Action); err != nil {
			return err
		}
		return nil
	}
}
```

## 关键点说明

### 1. Handler 职责

**Handler 层只负责：**
- 解析消息参数（Protobuf）
- 调用 Logic 层
- 返回结果

### 2. Handler 结构

```go
type UserMQ struct {
	svcCtx *svc.ServiceContext
}

func NewUserMQ(svcCtx *svc.ServiceContext) *UserMQ {
	return &UserMQ{
		svcCtx: svcCtx,
	}
}
```

### 3. Protobuf 导入

```go
import userMqPb "apps/pb/common/mq"
```

### 4. 消费者方法签名

```go
func (s *XxxMQ) ConsumerName() queue.ConsumeHandlerFunc {
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
		xxxLogic := logic.NewXxxLogic(ctx, s.svcCtx)
		return xxxLogic.Method(...)
	}
}
```

### 5. 错误处理

**Handler 层错误处理：**
- 参数解析失败：使用 `xerr.NewMsgWithErrorLog` 返回参数错误
- Logic 层返回的错误：直接返回

**Logic 层错误处理：**
- 业务错误：使用业务定义的错误码，`xerr.NewMsgWithInfoLog`
- 系统错误：使用系统错误码，`xerr.NewWithErrorLog`

### 6. 幂等性处理（Logic 层示例）

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
	if err != nil {
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
| [proto_message.md](../example/mq/proto_message.md) | MQ Protobuf 消息定义 |
| [logic.md](../example/mq/logic.md) | MQ Logic 示例 |
| [repository.md](../example/mq/repository.md) | MQ Repository 层示例 |
| [config.md](../example/mq/config.md) | MQ Config 示例 |
| [server.go](../example/mq/server.md) | MQ Server 示例 |
| [svc_context.md](../example/mq/svc_context.md) | ServiceContext 初始化（与 gRPC 共享） |
