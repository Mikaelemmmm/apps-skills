# Job Event 层示例

Job Event 层用于在定时任务中发送消息到消息队列，支持 Kafka、RabbitMQ、RocketMQ 等多种实现。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
└── event/                # MQ 消息发送
    ├── user_event.go     # 用户相关事件
    └── order_event.go    # 订单相关事件
```

## event/user_event.go - 用户事件

```go
package event

import (
	"context"
	"fmt"
	"time"

	userMqPb "apps/pb/common/mq"
	"apps/pkg/queue"
)

const (
	// TopicKey 主题 key
	userTopic = "user_topic"

	// MessageKeys 消息 key
	userSyncKey  = "user.sync"
	userArchiveKey = "user.archive"
)

// UserEvent 用户事件接口
//
//go:generate mockgen -package event -destination mock_user_event.go -source user_event.go UserEvent
type UserEvent interface {
	// SendSync 发送同步消息
	SendSync(ctx context.Context, userID int64, action string) error

	// SendArchive 发送归档消息
	SendArchive(ctx context.Context, userID int64, data string) error

	// SendSyncDelay 延迟发送同步消息
	SendSyncDelay(ctx context.Context, userID int64, action string, delay time.Duration) error

	// Close 关闭事件发送器
	Close()
}

type userEvent struct {
	pusher queue.Pusher
	topic  string
}

// NewUserEvent 创建用户事件发送器
func NewUserEvent(pusher queue.Pusher, topic string) UserEvent {
	return &userEvent{
		pusher: pusher,
		topic:  topic,
	}
}

func (u *userEvent) formatKey(key string) string {
	return u.topic + ":" + key
}

// SendSync 发送同步消息
func (u *userEvent) SendSync(ctx context.Context, userID int64, action string) error {
	req := &userMqPb.UserSyncReq{
		UserId: userID,
		Action: action,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user sync message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := u.formatKey(userSyncKey)
	return u.pusher.SendWithKey(ctx, key, data)
}

// SendArchive 发送归档消息
func (u *userEvent) SendArchive(ctx context.Context, userID int64, data string) error {
	req := &userMqPb.UserArchiveReq{
		UserId: userID,
		Data:   data,
	}
	dataBytes, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user archive message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 Send 发送消息
	return u.pusher.Send(ctx, dataBytes)
}

// SendSyncDelay 延迟发送同步消息
func (u *userEvent) SendSyncDelay(ctx context.Context, userID int64, action string, delay time.Duration) error {
	req := &userMqPb.UserSyncReq{
		UserId: userID,
		Action: action,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user sync message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendDelayAfter 发送延迟消息
	return u.pusher.SendDelayAfter(ctx, data, delay)
}

// Close 关闭事件发送器
func (u *userEvent) Close() {
	u.pusher.Close()
}
```

## event/order_event.go - 订单事件

```go
package event

import (
	"context"
	"fmt"

	orderMqPb "apps/pb/common/mq"
	"apps/pkg/queue"
)

const (
	// TopicKey 主题 key
	orderTopic = "order_topic"

	// MessageKeys 消息 key
	orderSyncKey = "order.sync"
	orderNotifyKey = "order.notify"
)

// OrderEvent 订单事件接口
//
//go:generate mockgen -package event -destination mock_order_event.go -source order_event.go OrderEvent
type OrderEvent interface {
	// SendSync 发送同步消息
	SendSync(ctx context.Context, orderID int64, status int32) error

	// SendNotify 发送通知消息
	SendNotify(ctx context.Context, orderID int64, userID int64, message string) error

	// Close 关闭事件发送器
	Close()
}

type orderEvent struct {
	pusher queue.Pusher
	topic  string
}

// NewOrderEvent 创建订单事件发送器
func NewOrderEvent(pusher queue.Pusher, topic string) OrderEvent {
	return &orderEvent{
		pusher: pusher,
		topic:  topic,
	}
}

func (o *orderEvent) formatKey(key string) string {
	return o.topic + ":" + key
}

// SendSync 发送同步消息
func (o *orderEvent) SendSync(ctx context.Context, orderID int64, status int32) error {
	req := &orderMqPb.OrderSyncReq{
		OrderId: orderID,
		Status:  status,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal order sync message failed, order_id: %d, err: %v", orderID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := o.formatKey(orderSyncKey)
	return o.pusher.SendWithKey(ctx, key, data)
}

// SendNotify 发送通知消息
func (o *orderEvent) SendNotify(ctx context.Context, orderID int64, userID int64, message string) error {
	req := &orderMqPb.OrderNotifyReq{
		OrderId: orderID,
		UserId:  userID,
		Message:  message,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal order notify message failed, order_id: %d, err: %v", orderID, err)
	}

	// 使用 Send 发送消息
	return o.pusher.Send(ctx, data)
}

// Close 关闭事件发送器
func (o *orderEvent) Close() {
	o.pusher.Close()
}
```

## 命名规范

| 位置 | 命名 |
|------|------|
| event 文件 | `{业务名}_event.go`，如 `user_event.go`、`order_event.go` |
| 接口名 | `{业务名}Event`，如 `UserEvent`、`OrderEvent` |
| 结构体名 | 小写 `{业务名}event`，如 `userEvent`、`orderEvent` |
| 构造函数 | `New{业务名}Event`，如 `NewUserEvent`、`NewOrderEvent` |
| repository_type.go 类型别名 | `{业务名}EventType`，如 `UserEventType`、`OrderEventType` |
| repository.go 字段名 | `{业务名}Event`，如 `UserEvent`、`OrderEvent` |

## 发送方式对比

| 方式 | 方法 | 说明 | 适用场景 |
|-----|------|------|---------|
| **同步发送** | `Send` | 阻塞直到消息被确认 | 需要确保消息发送成功 |
| **Key 分区发送** | `SendWithKey` | 根据 key 发送到不同分区 | 需要保证同一业务的消息顺序 |
| **延迟发送** | `SendDelayAfter` | 延迟指定时间后发送 | 定时任务、延迟处理 |

## 注意事项

1. **消息体序列化**：使用 `proto.Marshal` 序列化 Protobuf 消息
2. **错误处理**：序列化失败需要记录日志，不能直接忽略
3. **资源释放**：进程退出前调用 `Close` 释放资源
4. **幂等性**：消息体中应包含 `message_id` 用于幂等性处理
