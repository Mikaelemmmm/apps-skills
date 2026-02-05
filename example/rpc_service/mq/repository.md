# MQ Repository 层示例

MQ 消费者的 Repository 层与 gRPC 服务共享，管理 Handler 和 Logic 层的所有依赖。

> **注意**：MQ 和 gRPC 共享同一个 Repository 层，不需要单独创建。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
├── model/                 # 数据库 Model
│   ├── xxx_model_gen.go   # 自动生成（勿编辑）
│   └── xxx_model.go       # 扩展方法
├── rpc/                   # RPC 客户端
├── xredis/                # Redis 操作（按业务分包）
└── event/                 # MQ 事件发送（按业务分包）
```

## repository.go

定义 Repository 结构并初始化所有依赖：

```go
package repository

import (
	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/repository/event"
	"apps/app/services/core/user/internal/repository/model"
	"apps/app/services/core/user/internal/repository/rpc"
	"apps/app/services/core/user/internal/repository/xredis"
	"apps/pkg/queue/kafka"
	"apps/pkg/store/mysql/gorm"
	"apps/pkg/store/redis"
)

// Repository 依赖容器
type Repository struct {
	UserModel          UserModelType
	UserThirdAuthModel UserThirdAuthModelType
	AuthMsgModel       AuthMsgModelType
	MsgIdempotencyRdsModel MsgIdempotencyRdsModelType
	UserEvent          UserEventType
	OrderEvent         OrderEventType
	AuthzSrv           AuthzSrvType
}

// NewRepository 创建 Repository
func NewRepository(c config.Config) *Repository {
	mysqlDB := gorm.MustNewGormDB(c.DB.Mysql)
	redisClient := redis.MustNewRedisClient(c.Redis)
	pusher := kafka.MustNewPusher(c.MQ.KafkaPusherConf)

	return &Repository{
		UserModel:          model.NewUserModel(mysqlDB, c.DBCache),
		UserThirdAuthModel: model.NewUserThirdAuthModel(mysqlDB, c.DBCache),
		AuthMsgModel:       model.NewAuthMsgModel(mysqlDB, c.DBCache),
		MsgIdempotencyRdsModel: xredis.NewMsgIdempotencyModel(redisClient, "user"),
		UserEvent:         event.NewUserEvent(pusher, "user"),
		OrderEvent:        event.NewOrderEvent(pusher, "order"),
		AuthzSrv:           rpc.NewAuthzSrv(c.RPC.AuthzSrvConf),
	}
}
```

## repository_type.go

定义类型别名，供 Logic 层使用：

```go
package repository

import (
	"apps/app/services/core/user/internal/repository/event"
	"apps/app/services/core/user/internal/repository/model"
	"apps/app/services/core/user/internal/repository/rpc"
	"apps/app/services/core/user/internal/repository/xredis"
)

// 类型别名，方便 Logic 层使用
type (
	UserModelType          = model.UserModel
	UserThirdAuthModelType = model.UserThirdAuthModel
	AuthMsgModelType       = model.AuthMsgModel
	MsgIdempotencyRdsModelType = xredis.MsgIdempotencyModel
	UserEventType          = event.UserEvent
	OrderEventType         = event.OrderEvent
	AuthzSrvType           = rpc.AuthzSrv
)
```

## model/ - 数据库 Model

### 文件说明

| 文件 | 说明 | 是否可编辑 |
|-----|------|-----------|
| `xxx_model_gen.go` | 自动生成，包含增删改和基础查询方法 | ❌ 勿编辑 |
| `xxx_model.go` | 手动扩展，添加自定义查询方法 | ✅ 可编辑 |

### MQ 相关 Model 示例

**auth_msg_model.go - 消息已处理记录**

```go
package model

import (
	"context"
	"time"

	"github.com/zeromicro/go-zero/core/stores/cache"
	"gorm.io/gorm"
)

var _ AuthMsgModel = (*customAuthMsgModel)(nil)

type (
	// AuthMsgModel 消息模型接口
	AuthMsgModel interface {
		authMsgModel
		// 此处定义扩展方法接口
		FindByMessageID(ctx context.Context, messageID string) (*AuthMsg, error)
		MarkAsProcessed(ctx context.Context, messageID string) error
	}

	customAuthMsgModel struct {
		*defaultAuthMsgModel
	}
)

// FindByMessageID 根据 MessageID 查询
func (m *customAuthMsgModel) FindByMessageID(ctx context.Context, messageID string) (*AuthMsg, error) {
	var msg AuthMsg
	err := m.DB.WithContext(ctx).
		Where("message_id = ?", messageID).
		First(&msg).Error
	if err != nil {
		return nil, err
	}
	return &msg, nil
}

// MarkAsProcessed 标记消息已处理
func (m *customAuthMsgModel) MarkAsProcessed(ctx context.Context, messageID string) error {
	msg, err := m.FindByMessageID(ctx, messageID)
	if err != nil {
		return err
	}

	msg.Processed = 1
	msg.ProcessTime = time.Now()
	return m.Update(ctx, msg)
}
```

## xredis/ - Redis 操作

MQ 相关的 Redis 操作按业务定义在 `repository/xredis/` 下，每个文件一个业务模块。

**msg_idempotency_model.go - 消息幂等性处理**

```go
package xredis

import (
	"context"
	"errors"
	"fmt"

	"github.com/zeromicro/go-zero/core/stores/redis"
)

const (
	msgProcessedKey = "msg:processed:%s"
)

//go:generate mockgen -package xredis -destination mock_msg_idempotency_model.go -source msg_idempotency_model.go MsgIdempotencyModel
type MsgIdempotencyModel interface {
	// 检查消息是否已处理
	IsMessageProcessed(ctx context.Context, messageID string) (bool, error)
	// 标记消息已处理
	MarkMessageProcessed(ctx context.Context, messageID string, expireSeconds int) error
}

type msgIdempotencyModel struct {
	client *redis.Redis
	prefix string
}

// NewMsgIdempotencyModel 创建消息幂等性模型
func NewMsgIdempotencyModel(client *redis.Redis, prefix string) MsgIdempotencyModel {
	return &msgIdempotencyModel{client: client, prefix: prefix}
}

func (m *msgIdempotencyModel) formatKey(key string) string {
	return m.prefix + ":" + key
}

// IsMessageProcessed 检查消息是否已处理
func (m *msgIdempotencyModel) IsMessageProcessed(ctx context.Context, messageID string) (bool, error) {
	key := m.formatKey(fmt.Sprintf(msgProcessedKey, messageID))
	val, err := m.client.GetCtx(ctx, key)
	if err != nil {
		if errors.Is(err, redis.Nil) {
			return false, nil
		}
		return false, err
	}
	return val == "1", nil
}

// MarkMessageProcessed 标记消息已处理
func (m *msgIdempotencyModel) MarkMessageProcessed(ctx context.Context, messageID string, expireSeconds int) error {
	key := m.formatKey(fmt.Sprintf(msgProcessedKey, messageID))
	return m.client.SetexCtx(ctx, key, "1", expireSeconds)
}
```

## event/ - MQ 事件发送

MQ 事件发送按业务定义在 `repository/event/` 下，每个文件一个业务模块。

参考：[event.md](./event.md)

**user_event.go - 用户事件**
```go
package event

import (
	"context"
	"fmt"

	userMqPb "apps/pb/common/mq"
	"apps/pkg/queue"
)

const (
	userTopic    = "user_topic"
	userCountKey = "user.count"
)

type UserEvent interface {
	SendCount(ctx context.Context, userID int64) error
	Close()
}

type userEvent struct {
	pusher queue.Pusher
	topic  string
}

func NewUserEvent(pusher queue.Pusher, topic string) UserEvent {
	return &userEvent{pusher: pusher, topic: topic}
}

func (u *userEvent) SendCount(ctx context.Context, userID int64) error {
	req := &userMqPb.UserCountReq{UserId: userID}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal failed, err: %v", err)
	}
	return u.pusher.SendWithKey(ctx, u.topic+":"+userCountKey, data)
}

func (u *userEvent) Close() {
	u.pusher.Close()
}
```

## rpc/ - RPC 客户端

MQ 可能需要调用其他 RPC 服务：

```go
package rpc

import (
	authzV1Pb "apps/pb/services/shared/authz/v1"
	"github.com/zeromicro/go-zero/zrpc"
)

//go:generate mockgen -package rpc -destination mock_authz_service.go apps/pb/services/shared/authz/v1 AuthzServiceClient
type AuthzSrv struct {
	AuthzServicev1Client authzV1Pb.AuthzServiceClient
}

// NewAuthzSrv 创建鉴权服务客户端
func NewAuthzSrv(authzSrvConf zrpc.RpcClientConf) *AuthzSrv {
	return &AuthzSrv{
		AuthzServicev1Client: authzV1Pb.NewAuthzServiceClient(zrpc.MustNewClient(authzSrvConf).Conn()),
	}
}
```
