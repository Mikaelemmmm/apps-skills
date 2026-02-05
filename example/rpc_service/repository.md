# Repository 层示例

RPC 服务的 Repository 层管理 Logic/Service 层的所有依赖，包括 DB Model、RPC 客户端、缓存操作、MQ 事件发送等。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
├── model/                 # 数据库 Model
│   ├── xxx_model_gen.go   # 自动生成（勿编辑）
│   └── xxx_model.go       # 扩展方法
├── rpc/                   # RPC 客户端
├── xredis/                # Redis 操作
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
	AuthzSrv           AuthzSrvType
	TokenRdsModel       TokenRdsModelType
	UserEvent           UserEventType
	OrderEvent          OrderEventType
}

// NewRepository 创建 Repository
func NewRepository(c config.Config) *Repository {
	mysqlDB := gorm.MustNewGormDB(c.DB.Mysql)
	redisDB := redis.MustNewRedisClient(c.Redis)
	pusher := kafka.MustNewPusher(c.MQ.KafkaPusherConf)

	return &Repository{
		UserModel:          model.NewUserModel(mysqlDB, c.DBCache),          // 用户信息
		UserThirdAuthModel: model.NewUserThirdAuthModel(mysqlDB, c.DBCache), // 用户三方授权信息
		AuthzSrv:           rpc.NewAuthzSrv(c.RPC.AuthzSrvConf),             // 权限服务
		TokenRdsModel:       xredis.NewTokenModel(redisDB, c.Redis.Key),       // Token 管理
		UserEvent:           event.NewUserEvent(pusher, "user"),                 // 用户事件
		OrderEvent:          event.NewOrderEvent(pusher, "order"),                // 订单事件
	}
}
```

## repository_type.go

定义类型别名，供 Logic/Service 层使用：

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
	AuthzSrvType           = rpc.AuthzSrv
	TokenRdsModelType       = xredis.TokenModel
	UserEventType           = event.UserEvent
	OrderEventType          = event.OrderEvent
)
```

## model/ - 数据库 Model

### 文件说明

| 文件 | 说明 | 是否可编辑 |
|-----|------|-----------|
| `xxx_model_gen.go` | 自动生成，包含包含增删改和基础查询方法 | ❌ 勿编辑 |
| `xxx_model.go` | 手动扩展，添加自定义查询方法 | ✅ 可编辑 |

### user_model.go - 扩展方法示例

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
package model

import (
	"context"
	"time"

	"github.com/zeromicro/go-zero/core/stores/cache"
	"gorm.io/gorm"
)

var _ UserModel = (*customUserModel)(nil)

type (
	// UserModel 用户模型接口
	UserModel interface {
		userModel
		// 此处定义扩展方法接口
		FindByMobile(ctx context.Context, mobile string) ([]*User, error)
	}

	customUserModel struct {
		*defaultUserModel
	}
)

// BeforeCreate 创建前钩子
func (u *User) BeforeCreate(db *gorm.DB) error {
	now := time.Now().Unix()
	u.CreateTime = now
	u.UpdateTime = now
	return nil
}

// BeforeUpdate 更新前钩子
func (u *User) BeforeUpdate(db *gorm.DB) error {
	u.UpdateTime = time.Now().Unix()
	return nil
}

// NewUserModel 创建用户模型
func NewUserModel(conn *gorm.DB, c cache.CacheConf) UserModel {
	return &customUserModel{
		defaultUserModel: newUserModel(conn, c),
	}
}

// FindByMobile 根据手机号查询用户列表（扩展方法示例）
func (m *customUserModel) FindByMobile(ctx context.Context, mobile string) ([]*User, error) {
	var users []*User
	err := m.DB.WithContext(ctx).Where("`mobile` = ?", mobile).Find(&users).Error
	return users, err
}
```

## rpc/ - RPC 客户端

### auth_srv.go

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

## xredis/ - Redis 操作

### token_model.go

```go
package xredis

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/spf13/cast"
	"github.com/zeromicro/go-zero/core/stores/redis"
)

const tokenKey = "token:%d"

// TokenModel Token 管理接口
//
//go:generate mockgen -package xredis -destination mock_token_model.go -source token_model.go
type TokenModel interface {
	GenToken(ctx context.Context, userID int64, token string, expireSeconds int) error
	DelToken(ctx context.Context, userID int64, expireSeconds int) (bool, error)
	GetToken(ctx context.Context, userID int64) (string, error)
}

type tokenModel struct {
	client *redis.Redis
	prefix string
}

// NewTokenModel 创建 Token 模型
func NewTokenModel(client *redis.Redis, prefix string) TokenModel {
	return &tokenModel{client: client, prefix: prefix}
}

func (m *tokenModel) formatKey(key string) string {
	return m.prefix + ":" + key
}

func (m *tokenModel) GenToken(ctx context.Context, userID int64, token string, expireSeconds int) error {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	return m.client.client.SetexCtx(ctx, key, token, expireSeconds)
}

func (m *tokenModel) DelToken(ctx context.Context, userID int64, expireSeconds int) (bool, error) {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	return m.client.SetnxExCtx(ctx, key, cast.ToString(time.Now().Unix()), expireSeconds)
}

func (m *tokenModel) GetToken(ctx context.Context, userID int64) (string, error) {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	token, err := m.client.GetCtx(ctx, key)
	if err != nil && !errors.Is(err, redis.Nil) {
		return "", err
	}
	return token, nil
}
```

## event/ - MQ 事件发送

MQ 事件发送按业务定义在 `repository/event/` 下，每个文件一个业务模块。

参考：[event.md](./event.md)

### user_event.go - 用户事件

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
	userRegisterKey = "user.register"
	userLoginKey    = "user.login"
	userLogoutKey   = "user.logout"
	userPasswordKey = "user.password"
)

// UserEvent 用户事件接口
//
//go:generate mockgen -package event -destination mock_user_event.go -source user_event.go UserEvent
type UserEvent interface {
	// SendRegister 发送注册事件
	SendRegister(ctx context.Context, userID int64, mobile string) error

	// SendLogin 发送登录事件
	SendLogin(ctx context.Context, userID int64, ip string) error

	// SendLogout 发送登出事件
	SendLogout(ctx context.Context, userID int64) error

	// SendPasswordChange 发送密码修改事件
	SendPasswordChange(ctx context.Context, userID int64) error

	// SendLoginDelay 延迟发送登录事件
	SendLoginDelay(ctx context.Context, userID int64, ip string, delay time.Duration) error

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

// SendRegister 发送注册事件
func (u *userEvent) SendRegister(ctx context.Context, userID int64, mobile string) error {
	req := &userMqPb.UserRegisterReq{
		UserId: userID,
		Mobile:  mobile,
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user register message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := u.formatKey(userRegisterKey)
	return u.pusher.SendWithKey(ctx, key, data)
}

// SendLogin 发送登录事件
func (u *userEvent) SendLogin(ctx context.Context, userID int64, ip string) error {
	req := &userMqPb.UserLoginReq{
		UserId:    userID,
		Ip:        ip,
		Timestamp:  time.Now().Unix(),
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user login message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendWithKey 根据 key 发送到不同分区
	key := u.formatKey(userLoginKey)
	return u.pusher.SendWithKey(ctx, key, data)
}

// SendLogout 发送登出事件
func (u *userEvent) SendLogout(ctx context.Context, userID int64) error {
	req := &userMqPb.UserLogoutReq{
		UserId:   userID,
		Timestamp: time.Now().Unix(),
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user logout message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 Send 发送消息
	return u.pusher.Send(ctx, data)
}

// SendPasswordChange 发送密码修改事件
func (u *userEvent) SendPasswordChange(ctx context.Context, userID) error {
	req := &userMqPb.UserPasswordChangeReq{
		UserId:   userID,
		Timestamp: time.Now().Unix(),
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user password change message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 Send 发送
	return u.pusher.Send(ctx, data)
}

// SendLoginDelay 延迟发送登录事件
func (u *userEvent) SendLoginDelay(ctx context.Context, userID int64, ip string, delay time.Duration) error {
	req := &userMqPb.UserLoginReq{
		UserId:   userID,
		Ip:       ip,
		Timestamp: time.Now().Unix(),
	}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal user login message failed, user_id: %d, err: %v", userID, err)
	}

	// 使用 SendDelayAfter 发送延迟消息
	return u.pusher.SendDelayAfter(ctx, data, delay)
}

// Close 关闭事件发送器
func (u *userEvent) Close() {
	u.pusher.Close()
}
```

## Mock 生成

每个接口文件都需要添加 `//go:generate` 指令：

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
```

执行 `make mock` 生成 Mock 文件，用于单元测试。
