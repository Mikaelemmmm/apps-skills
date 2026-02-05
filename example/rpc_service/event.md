# REST API Event 层示例

REST API Event 层用于在业务逻辑完成后发送消息到消息队列，支持 Kafka、RabbitMQ、RocketMQ 等多种实现。

> **注意**：REST API 层通常只做聚合裁剪，发送事件应该在调用的 RPC 服务中完成。如确需在 API 层发送事件，可参考以下示例。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
└── event/                # MQ 消息发送
    ├── notify_event.go  # 通知相关事件
    └── log_event.go     # 日志相关事件
```

## event/notify_event.go - 通知事件

```go
package event

import (
	"context"
	"fmt"

	notifyMqPb "apps/pb/common/mq"
	"apps/pkg/queue"
)

const (
	// TopicKey 主题 key
	notifyTopic = "notify_topic"

	// MessageKeys 消息 key
	emailNotifyKey  = "notify.email"
	smsNotifyKey    = "notify.sms"
	pushNotifyKey   = "notify.push"
)

// NotifyEvent 通知事件接口
//
//go:generate mockgen -package event -destination mock_notify_event.go -source notify_event.go NotifyEvent
type NotifyEvent interface {
	// SendEmail 发送邮件通知
	SendEmail(ctx context.Context, to string, subject string, content string) error

	// SendSMS 发送短信通知
	SendSMS(ctx context.Context, mobile string, template string, params map[string]string) error

	// SendPush 发送推送通知
	SendPush(ctx context.Context, userID int64, title string, message string) error

	// Close 关闭事件发送器
	Close()
}

type notifyEvent struct {
	pusher queue.Pusher
	topic  string
}

// NewNotifyEvent 创建通知事件发送器
func NewNotifyEvent(pusher queue.Pusher, topic string) NotifyEvent {
	return &notifyEvent{
		pusher: pusher,
		topic:  topic,
	}
}

func (n *notifyEvent) formatKey(key string) string {
	return n.topic + ":" + key
}

// SendEmail 发送邮件通知（同步发送）
func (n *notifyEvent) SendEmail(ctx context.Context, to string, subject string, content string) error {
	req := &notifyMqPb.EmailNotifyReq{
		To:      to,
		Subject:  subject,
		Content:  content,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal email notify message failed, to: %s, err: %v", to, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := n.formatKey(emailNotifyKey)
	return n.pusher.SendWithKey(ctx, key, data)
}

// SendSMS 发送短信通知（同步发送）
func (n *notifyEvent) SendSMS(ctx context.Context, mobile string, template string, params map[string]string) error {
	req := &notifyMqPb.SMSNotifyReq{
		Mobile:   mobile,
		Template: template,
		Params:   params,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal sms notify message failed, mobile: %s, err: %v", mobile, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := n.formatKey(smsNotifyKey)
	return n.pusher.SendWithKey(ctx, key, data)
}

// SendPush 发送推送通知（同步发送）
func (n *notifyEvent) SendPush(ctx context.Context, userID int64, title string, message string) error {
	req := &notifyMqPb.PushNotifyReq{
		UserId:  userID,
		Title:    title,
		Message:  message,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal push notify message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := n.formatKey(pushNotifyKey)
	return n.pusher.SendWithKey(ctx, key, data)
}

// Close 关闭事件发送器
func (n *notifyEvent) Close() {
	n.pusher.Close()
}
```

## event/log_event.go - 日志事件

```go
package event

import (
	"context"
	"fmt"

	logMqPb "apps/pb/common/mq"
	"apps/pkg/queue"
)

const (
	// TopicKey 主题 key
	logTopic = "log_topic"

	// MessageKeys 消息 key
	userActionKey = "log.user_action"
	apiErrorKey   = "log.api_error"
)

// LogEvent 日志事件接口
//
//go:generate mockgen -package event -destination mock_log_event.go -source log_event.go LogEvent
type LogEvent interface {
	// LogUserAction 记录用户操作
	LogUserAction(ctx context.Context, userID int64, action string, detail string) error

	// LogAPIError 记录 API 错误
	LogAPIError(ctx context.Context, path string, method string, err string) error

	// Close 关闭事件发送器
	Close()
}

type logEvent struct {
	pusher queue.Pusher
	topic  string
}

// NewLogEvent 创建日志事件发送器
func NewLogEvent(pusher queue.Pusher, topic string) LogEvent {
	return &logEvent{
		pusher: pusher,
		topic:  topic,
	}
}

func (l *logEvent) formatKey(key string) string {
	return l.topic + ":" + key
}

// LogUserAction 记录用户操作
func (l *logEvent) LogUserAction(ctx context.Context, userID int64, action string, detail string) error {
	req := &logMqPb.UserActionLogReq{
		UserId: userID,
		Action:  action,
		Detail:  detail,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user action log message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 Send 发送消息
	return l.pusher.Send(ctx, data)
}

// LogAPIError 记录 API 错误
func (l *logEvent) LogAPIError(ctx context.Context, path string, method string, err string) error {
	req := &logMqPb.APIErrorLogReq{
{
		Path:   path,
		Method: method,
		Error:  err,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal api error log message failed, path: %s, err: %v", path, err)
	}

	// 使用 Send 发送消息
	return l.pusher.Send(ctx, data)
}

// Close 关闭事件发送器
func (l *logEvent) Close() {
	l.pusher.Close()
}
```

## 命名规范

| 位置 | 命名 |
|------|------|
| event 文件 | `{业务名}_event.go`，如 `notify_event.go`、`log_event.go` |
| 接口名 | `{业务名}Event`，如 `NotifyEvent`、`LogEvent` |
| 结构体名 | 小写 `{业务名}event`，如 `notifyEvent`、`logEvent` |
| 构造函数 | `New{业务名}Event`，如 `NewNotifyEvent`、`NewLogEvent` |
| repository_type.go 类型别名 | `{业务名}EventType`，如 `NotifyEventType`、`LogEventType` |
| repository.go 字段名 | `{业务名}Event`，如 `NotifyEvent`、`LogEvent` |

## 发送方式对比

| 方式 | 方法 | 说明 | 适用场景 |
|-----|------|------|---------|
| **同步发送** | `Send` | 阻塞直到消息被确认 | 需要确保消息发送成功 |
| **Key 分区发送** | `SendWithKey` | 根据 key 发送到不同分区 | 需要保证同一业务的消息顺序 |
| **批量发送** | `SendBatch` | 批量发送多条消息 | 高吞吐量场景 |
| **异步发送** | `SendAsync` | 非阻塞发送 | 高性能、可容忍少量丢失 |

## 注意事项

1. **消息体序列化**：使用 `proto.Marshal` 序列化 Protobuf 消息
2. **错误处理**：序列化失败需要记录日志，不能直接忽略
3. **资源释放**：进程退出前调用 `Close` 释放资源
4. **幂等性**：消息体中应包含 `message_id` 用于幂等性处理

## 在 Logic 层使用

```go
type UserLogic struct {
	svcCtx         *svc.ServiceContext
	userSrv        *repository.UserSrvType
	notifyEvent    repository.NotifyEventType
}

func NewUserLogic(svcCtx *svc.ServiceContext) *UserLogic {
	return &UserLogic{
		svcCtx:      svcCtx,
		userSrv:     svcCtx.Repo.UserSrv,
		notifyEvent:  svcCtx.Repo.NotifyEvent,
	}
}

// WXMiniAuth 微信小程序授权登录
func (l *UserLogic) WXMiniAuth(ctx context.Context, req *pb.WXMiniAuthReq) (*pb.ThirdAuthReply, error) {
	// 1. 调用用户服务进行登录或注册
	resp, err := l.userSrv.Register(ctx, &userV1Pb.RegisterReq{...})
	if err != nil {
		return nil, err
	}

	// 2. 发送欢迎消息（异步通知，不影响主流程）
	go func() {
		if err := l.notifyEvent.SendEmail(ctx, "user@example.com", "欢迎", "欢迎使用"); err != nil {
			xlog.Errorw(ctx, "send welcome email failed", "err", err)
		}
	}()

	// 3. 返回响应
	return &pb.ThirdAuthReply{
		AccessToken:  resp.GetToken().GetAccessToken(),
		AccessExpire: resp.GetToken().AccessExpire,
	}, nil
}
```
