# Job Repository 层示例

Job 定时任务的 Repository 层与 gRPC 服务共享，管理 Handler 和 Logic 层的所有依赖。

> **注意**：Job 和 gRPC 共享同一个 Repository 层，不需要单独创建。

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
	UserModel         UserModelType
	UserArchiveModel UserArchiveModelType
	LoginLogModel     LoginLogModelType
	JobLockRdsModel   JobLockRdsModelType
	UserEvent         UserEventType
	OrderEvent        OrderEventType
	AuthzSrv          AuthzSrvType
}

// NewRepository 创建 Repository
func NewRepository(c config.Config) *Repository {
	mysqlDB := gorm.MustNewGormDB(c.DB.Mysql)
	redisClient := redis.MustNewRedisClient(c.Redis)
	pusher := kafka.MustNewPusher(c.MQ.KafkaPusherConf)

	return &Repository{
		UserModel:         model.NewUserModel(mysqlDB, c.DBCache),
		UserArchiveModel: model.NewUserArchiveModel(mysqlDB, c.DBCache),
		LoginLogModel:     model.NewLoginLogModel(mysqlDB, c.DBCache),
		JobLockRdsModel:   xredis.NewJobLockModel(redisClient, "user"),
		UserEvent:         event.NewUserEvent(pusher, "user"),
		OrderEvent:        event.NewOrderEvent(pusher, "order"),
		AuthzSrv:          rpc.NewAuthzSrv(c.RPC.AuthzSrvConf),
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
	UserModelType         = model.UserModel
	UserArchiveModelType = model.UserArchiveModel
	LoginLogModelType     = model.LoginLogModel
	JobLockRdsModelType   = xredis.JobLockModel
	UserEventType         = event.UserEvent
	OrderEventType        = event.OrderEvent
	AuthzSrvType          = rpc.AuthzSrv
)
```

## model/ - 数据库 Model

### 文件说明

| 文件 | 说明 | 是否可编辑 |
|-----|------|-----------|
| `xxx_model_gen.go` | 自动生成，包含增删改和基础查询方法 | ❌ 勿编辑 |
| `xxx_model.go` | 手动扩展，添加自定义查询方法 | ✅ 可编辑 |

### Job 相关 Model 示例

**user_model.go - 扩展同步状态查询**

```go
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
		// Job 相关扩展方法
		FindPendingSync(ctx context.Context, limit int) ([]*User, error)
		UpdateSyncStatus(ctx context.Context, userID int64, syncData string) error
		FindDeletedBefore(ctx context.Context, lastID int64, limit int) ([]*User, error)
	}

	customUserModel struct {
		*defaultUserModel
	}
)

// FindPendingSync 查询需要同步的用户
func (m *customUserModel) FindPendingSync(ctx context.Context, limit int) ([]*User, error) {
	var users []*User
	err := m.DB.WithContext(ctx).
		Where("sync_status = ?", 0).
		Limit(limit).
		Find(&users).Error
	return users, err
}

// UpdateSyncStatus 更新同步状态
func (m *customUserModel) UpdateSyncStatus(ctx context.Context, userID int64, syncData string) error {
	user, err := m.FindOne(ctx, userID)
	if err != nil {
		return err
	}

	user.SyncStatus = 1
	user.SyncTime = time.Now().Unix()
	user.SyncData = syncData

	return m.Update(ctx, user)
}

// FindDeletedBefore 查询已删除的用户（分页）
func (m *customUserModel) FindDeletedBefore(ctx context.Context, lastID int64, limit int) ([]*User, error) {
	var users []*User
	err := m.DB.WithContext(ctx).
		Where("deleted = ? AND id > ?", 1, lastID).
		Limit(limit).
		Order("id ASC").
		Find(&users).Error
	return users, err
}
```

**login_log_model.go - 扩展过期清理方法**

```go
package model

import (
	"context"
	"time"

	"github.com/zeromicro/go-zero/core/stores/cache"
)

var _ LoginLogModel = (*customLoginLogModel)(nil)

type (
	// LoginLogModel 登录日志模型接口
	LoginLogModel interface {
		loginLogModel
		// Job 相关扩展方法
		DeleteExpired(ctx context.Context, expiredBefore time.Time) (int64, error)
	}

	customLoginLogModel struct {
		*defaultLoginLogModel
	}
)

// DeleteExpired 删除过期的登录日志
func (m *customLoginLogModel) DeleteExpired(ctx context.Context, expiredBefore time.Time) (int64, error) {
	result := m.DBDB.WithContext(ctx).
		Where("create_time < ?", expiredBefore).
		Delete(&LoginLog{})
	return result.RowsAffected, result.Error
}
```

## xredis/ - Redis 操作

Job 相关的 Redis 操作按业务定义在 `repository/xredis/` 下，每个文件一个业务模块。

**job_lock_model.go - Job 分布式锁和状态管理**

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

const (
	jobLockKey   = "job:lock:%s"
	jobStatusKey = "job:status:%s"
)

//go:generate mockgen -package xredis -destination mock_job_lock_model.go -source job_lock_model.go JobLockModel
type JobLockModel interface {
	// 分布式锁相关
	AcquireLock(ctx context.Context, key string, expireSeconds int) (bool, error)
	ReleaseLock(ctx context.Context, key string) error

	// 任务状态相关
	SetJobStatus(ctx context.Context, jobName string, status string, expireSeconds int) error
	GetJobStatus(ctx context.Context, jobName string) (string, error)
}

type jobLockModel struct {
	client *redis.Redis
	prefix string
}

// NewJobLockModel 创建 Job 锁模型
func NewJobLockModel(client *redis.Redis, prefix string) JobLockModel {
	return &jobLockModel{client: client, prefix: prefix}
}

func (j *jobLockModel) formatKey(key string) string {
	return j.prefix + ":" + key
}

// AcquireLock 获取分布式锁
func (j *jobLockModel) AcquireLock(ctx context.Context, key string, expireSeconds int) (bool, error) {
	lockKey := j.formatKey(fmt.Sprintf(jobLockKey, key))
	return j.client.SetnxExCtx(ctx, lockKey, cast.ToString(time.Now().Unix()), expireSeconds)
}

// ReleaseLock 释放分布式锁
func (j *jobLockModel) ReleaseLock(ctx context.Context, key string) error {
	lockKey := j.formatKey(fmt.Sprintf(jobLockKey, key))
	return j.client.DelCtx(ctx, lockKey)
}

// SetJobStatus 设置任务状态
func (j *jobLockModel) SetJobStatus(ctx context.Context, jobName string, status string, expireSeconds int) error {
	statusKey := j.formatKey(fmt.Sprintf(jobStatusKey, jobName))
	return j.client.SetexCtx(ctx, statusKey, status, expireSeconds)
}

// GetJobStatus 获取任务状态
func (j *jobLockModel) GetJobStatus(ctx context.Context, jobName string) (string, error) {
	statusKey := j.formatKey(fmt.Sprintf(jobStatusKey, jobName))
	val, err := j.client.GetCtx(ctx, statusKey)
	if err != nil && !errors.Is(err, redis.Nil) {
		return "", err
	}
	return val, nil
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
	userSyncKey = "user.sync"
)

type UserEvent interface {
	SendSync(ctx context.Context, userID int64, action string) error
	Close()
}

type userEvent struct {
	pusher queue.Pusher
	topic  string
}

func NewUserEvent(pusher queue.Pusher, topic string) UserEvent {
	return &userEvent{pusher: pusher, topic: topic}
}

func (u *userEvent) SendSync(ctx context.Context, userID int64, action string) error {
	req := &userMqPb.UserSyncReq{UserId: userID, Action: action}
	data, err := proto.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal failed, err: %v", err)
	}
	return u.pusher.SendWithKey(ctx, u.topic+":"+userSyncKey, data)
}

func (u *userEvent) Close() {
	u.pusher.Close()
}
```

## rpc/ - RPC 客户端

Job 可能需要调用其他 RPC 服务：

```go
package rpc

import (
	authzV1Pb "apps/pb/services/shared/authz/v1"
	thirdV1Pb "apps/pb/services/support/third/v1"
	"github.com/zeromicro/go-zero/zrpc"
)

//go:generate mockgen -package rpc -destination mock_authz_service.go apps/pb/services/shared/authz/v1 AuthzServiceClient
type AuthzSrv struct {
	AuthzServicev1Client authzV1Pb.AuthzServiceClient
}

// NewAuthzSrv 创建鉴权服务客户端
func NewAuthzSrv(authzSrvConf zrpc.RpcClientConf) *AuthzSrv {
	return &AuthzSrv{
		AuthzServicev1Client: authzV1Pb.NewAuthzazServiceClient(zrpc.MustNewClient(authzSrvConf).Conn()),
	}
}

//go:generate mockgen -package rpc -destination mock_third_service.go apps/pb/services/support/support/v1 ThirdServiceClient
type ThirdSrv struct {
	ThirdServicev1Client thirdV1Pb.ThirdServiceClient
}

// NewThirdSrv 创建第三方服务客户端
func NewThirdSrv(thirdSrvConf zrpc.RpcClientConf) *ThirdSrv {
	return &ThirdSrv{
		ThirdServicev1Client: thirdV1Pb.NewThirdServiceClient(zrpc.MustNewClient(thirdSrvConf).Conn()),
	}
}
```
