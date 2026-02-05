# MQ Event 层示例

MQ Event 层用于发送消息到消息队列，支持 Kafka、RabbitMQ、RocketMQ 等多种实现。

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
	"github.com/zeromicro/go-zero/core/stores/redis"
)

const (
	// TopicKey 主题 key
	userTopic = "user_topic"

	// MessageKeys 消息 key
	userCountKey   = "user.count"
	userUpLevelKey = "user.up_level"
	userSyncKey    = "user.sync"
)

// UserEvent 用户事件接口
//
//go:generate mockgen -package event -destination mock_user_event.go -source user_event.go UserEvent
type UserEvent interface {
	// SendCount 发送统计消息
	SendCount(ctx context.Context, userID int64) error

	// SendUpLevel 发送升级消息
	SendUpLevel(ctx context.Context, userID int64, level int, messageID string) error

	// SendSync 发送同步消息
	SendSync(ctx context.Context, userID int64, action string) error

	// SendUpLevelDelay 延迟发送升级消息
	SendUpLevelDelay(ctx context.Context, userID int64, level int, messageID string, delay time.Duration) error

	// SendUpLevelAsync 异步发送升级消息
	SendUpLevelAsync(ctx context.Context, userID int64, level int, messageID string) error

	// Flush 刷新异步消息
	Flush() error

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

// SendCount 发送统计消息（同步发送）
func (u *userEvent) SendCount(ctx context.Context, userID int64) error {
	req := &userMqPb.UserCountReq{
		UserId: userID,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user count message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := u.formatKey(userCountKey)
	return u.pusher.SendWithKey(ctx, key, data)
}

// SendUpLevel 发送升级消息（同步发送）
func (u *userEvent) SendUpLevel(ctx context.Context, userID int64, level int, messageID string) error {
	req := &userMqPb.UserUpLevelReq{
		UserId:    userID,
		Level:      int32(level),
		MessageId:  messageID,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user up level message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := u.formatKey(userUpLevelKey)
	return u.pusher.SendWithKey(ctx, key, data)
}

// SendSync 发送同步消息（同步发送）
func (u *userEvent) SendSync(ctx context.Context, userID int64, action string) error {
	req := &userMqPb.UserSyncReq{
		UserId: userID,
		Action: action,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user sync message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 Send 发送消息
	return u.pusher.Send(ctx, data)
}

// SendUpLevelDelay 延迟发送升级消息
func (u *userEvent) SendUpLevelDelay(ctx context.Context, userID int64, level int, messageID string, delay time.Duration) error {
	req := &userMqPb.UserUpLevelReq{
		UserId:   userID,
		Level:     int32(level),
		MessageId: messageID,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user up level message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendDelayAfter 发送延迟消息
	return u.pusher.SendDelayAfter(ctx, data, delay)
}

// SendUpLevelAsync 异步发送升级消息
func (u *userEvent) SendUpLevelAsync(ctx context.Context, userID int64, level int, messageID string) error {
	req := &userMqPb.UserUpLevelReq{
		UserId:   userID,
		Level:     int32(level),
		MessageId: messageID,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user up level message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendAsync 异步发送消息
	return u.pusher.SendAsync(ctx, data, func(result *queue.SendResult) {
		if result.Err != nil {
			// 处理发送失败
			fmt.Printf("send user up level async failed, user_id: %d, err: %v", userID, result.Err)
		}
	})
}

// Flush 刷新异步消息到服务器
func (u *userEvent) Flush() error {
	return u.pusher.Flush()
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
	orderCreateKey  = "order.create"
	orderCancelKey  = "order.cancel"
	orderPayKey     = "order.pay"
)

// OrderEvent 订单事件接口
//
//go:generate mockgen -package event -destination mock_order_event.go -source order_event.go OrderEvent
type OrderEvent interface {
	// SendCreate 发送创建订单消息
	SendCreate(ctx context.Context, orderID int64, userID int64, items []*orderMqPb.OrderItem) error

	// SendCancel 发送取消订单消息
	SendCancel(ctx context.Context, orderID int64, userID int64, reason string) error

	// SendPay 发送支付消息
	SendPay(ctx context.Context, orderID int64, userID int64, amount int64) error

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

// SendCreate 发送创建订单消息
func (o *orderEvent) SendCreate(ctx context.Context, orderID int64, userID int64, items []*orderMqPb.OrderItem) error {
	req := &orderMqPb.OrderCreateReq{
		OrderId: orderID,
		UserId:  userID,
		Items:   items,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal order create message failed, order_id: %d, err: %v", orderID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := o.formatKey(orderCreateKey)
	return o.pusher.SendWithKey(ctx, key, data)
}

// SendCancel 发送取消订单消息
func (o *orderEvent) SendCancel(ctx context.Context, orderID int64, userID int64, reason string) error {
	req := &orderMqPb.OrderCancelReq{
		OrderId: orderID,
		UserId:  userID,
		Reason:  reason,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal order cancel message failed, order_id: %d, err: %v", orderID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := o.formatKey(orderCancelKey)
	return o.pusher.SendWithKey(ctx, key, data)
}

// SendPay 发送支付消息
func (o *orderEvent) SendPay(ctx context.Context, orderID int64, userID int64, amount int64) error {
	req := &orderMqPb.OrderPayReq{
		OrderId: orderID,
		UserId:  userID,
		Amount:  amount,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal order pay message failed, order_id: %d, err: %v", orderID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := o.formatKey(orderPayKey)
	return o.pusher.SendWithKey(ctx, key, data)
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
| **批量发送** | `SendBatch` | 批量发送多条消息 | 高吞吐量场景 |
| **延迟发送** | `SendDelayAfter` | 延迟指定时间后发送 | 定时任务、延迟处理 |
| **指定时刻发送** | `SendDelayAt` | 在指定时刻发送 | 定时任务 |
| **异步发送** | `SendAsync` | 非阻塞发送 | 高性能、可容忍少量丢失 |

## 注意事项

1. **消息体序列化**：使用 `proto.Marshal` 序列化 Protobuf 消息
2. **错误处理**：序列化失败需要记录日志，不能直接忽略
3. **异步发送 Flush**：使用 `SendAsync` 后需调用 `Flush` 确保消息发送
4. **资源释放**：进程退出前调用 `Close` 释放资源
5. **幂等性**：消息体中应包含 `message_id` 用于幂等性处理
