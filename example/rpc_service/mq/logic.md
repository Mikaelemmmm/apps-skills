# MQ Logic 示例

MQ Logic 层负责处理具体业务逻辑，调用 Repository 层操作数据。

## 目录结构

```
internal/logic/mq/
├── count_logic.go      # 统计业务
├── up_level_logic.go    # 升级业务
└── sync_logic.go       # 同步业务
```

## 代码示例

### count_logic.go

```go
package mq

import (
	"context"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xerr"
	"apps/pkg/xlog"
)

// CountLogic 统计业务逻辑
type CountLogic struct {
	ctx       context.Context
	userModel repository.UserModelType
}

// NewCountLogic 创建统计业务逻辑
func NewCountLogic(ctx context.Context, svcCtx *svc.ServiceContext) *CountLogic {
	return &CountLogic{
		ctx:       ctx,
		userModel: svcCtx.Repo.UserModel,
	}
}

// Count 更新用户统计
func (l *CountLogic) Count(userID int64) error {
	xlog.Infow(l.ctx, "CountLogic started", "user_id", userID)

	// 查询用户
	user, err := l.userModel.FindOne(l.ctx, userID)
	if err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"find user failed, user_id: %d, err: %v", userID, err,
		)
	}

	// 更新统计
	user.Count += 1
	if err := l.userModel.Update(l.ctx, user); err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"update user count failed, user_id: %d, err: %v", userID, err,
		)
	}

	xlog.Infow(l.ctx, "CountLogic finished", "user_id", userID)

	return nil
}
```

### up_level_logic.go

```go
package mq

import (
	"context"
	"time"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xerr"
	"apps/pkg/xlog"
)

// UpLevelLogic 升级业务逻辑
type UpLevelLogic struct {
	ctx                  context.Context
	userModel            repository.UserModelType
	msgIdempotencyRdsModel repository.MsgIdempotencyRdsModelType
	notifyClient         repository.NotifyClientType
}

// NewUpLevelLogic 创建升级业务逻辑
func NewUpLevelLogic(ctx context.Context, svcCtx *svc.ServiceContext) *UpLevelLogic {
	return &UpLevelLogic{
		ctx:                  ctx,
		userModel:            svcCtx.Repo.UserModel,
		msgIdempotencyRdsModel: svcCtx.Repo.MsgIdempotencyRdsModel,
		notifyClient:         svcCtx.Repo.NotifyClient,
	}
}

// UpLevel 升级用户等级
func (l *UpLevelLogic) UpLevel(userID int64, level int, messageID string) error {
	xlog.Infow(l.ctx, "UpLevelLogic started",
		"user_id", userID,
		"level", level,
		"message_id", messageID,
	)

	// 幂等性检查（使用 xredis 包装方法）
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
			"update user level failed, user_id: %d, level: %d, err: %v", userID, level, err,
		)
	}

	// 发送通知
	if err := l.notifyClient.Send(l.ctx, user); err != nil {
		xlog.Errorw(l.ctx, "send notify failed",
			"user_id", userID,
			"err", err,
		)
	}

	// 标记消息已处理（使用 xredis 包装方法）
	if err := l.msgIdempotencyRdsModel.MarkMessageProcessed(l.ctx, messageID, 24*3600); err != nil {
		xlog.Errorw(l.ctx, "mark message processed failed",
			"message_id", messageID,
			"err", err,
		)
	}

	xlog.Infow(l.ctx, "UpLevelLogic finished",
		"user_id", userID,
		"level", level,
	)

	return nil
}
```

### sync_logic.go

```go
package mq

import (
	"context"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xerr"
	"apps/pkg/xlog"
)

// SyncLogic 同步业务逻辑
type SyncLogic struct {
	ctx          context.Context
	userModel    repository.UserModelType
	thirdService repository.ThirdServiceType
}

// NewSyncLogic 创建同步业务逻辑
func NewSyncLogic(ctx context.Context, svcCtx *svc.ServiceContext) *SyncLogic {
	return &SyncLogic{
		ctx:          ctx,
		userModel:    svcCtx.Repo.UserModel,
		thirdService: svcCtx.Repo.ThirdService,
	}
}

// Sync 同步用户数据
func (l *SyncLogic) Sync(userID int64, action string) error {
	xlog.Infow(l.ctx, "SyncLogic started",
		"user_id", userID,
		"action", action,
	)

	// 调用第三方服务
	data, err := l.thirdService.GetUser(l.ctx, userID)
	if err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"get third user failed, user_id: %d, err: %v", userID, err,
		)
	}

	// 更新本地数据
	if err := l.userModel.Sync(l.ctx, data); err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"sync user failed, user_id: %d, err: %v", userID, err,
		)
	}

	xlog.Infow(l.ctx, "SyncLogic finished", "user_id", userID)

	return nil
}
```

## 关键点说明

### 1. Logic 结构

```go
type XxxLogic struct {
	ctx                  context.Context
	userModel            repository.UserModelType
	msgIdempotencyRdsModel repository.MsgIdempotencyRdsModelType
}

func NewXxxLogic(ctx context.Context, svcCtx *svc.ServiceContext) *XxxLogic {
	return &XxxLogic{
		ctx:                  ctx,
		userModel:            svcCtx.Repo.UserModel,
		msgIdempotencyRdsModel: svcCtx.Repo.MsgIdempotencyRdsModel,
	}
}
```

**Logic 结构体规范：**
- 包含 `ctx context.Context`
- 从 `svcCtx.Repo` 提取需要的依赖
- 不包含 `svcCtx` 本身
- 使用具体的 xredis model（如 `MsgIdempotencyRdsModel`），不是通用的 `RedisClient`

### 2. 业务方法签名

```go
func (l *XxxLogic) MethodName(userID int64, ...) error {
	// 处理业务逻辑
	return nil
}
```

### 3. 错误处理

**业务错误：**使用业务定义的错误码

```go
return xerr.NewMsgWithInfoLog(
	xerr.SystemError_PARAMS_ERROR,
	"user not found",
	"user_id: %d", userID,
)
```

**系统错误：**使用系统错误码

```go
return xerr.NewWithErrorLog(
	xerr.SystemError_D_ERROR,
	"find user failed, user_id: %d, err: %v", userID, err,
)
```

### 4. 使用 xredis 包装方法

Logic 层调用 repository/xredis 中定义的包装方法，不直接操作 Redis：

```go
// ✅ 正确：使用包装方法
processed, err := l.msgIdempotencyModel.IsMessageProcessed(l.ctx, messageID)
err := l.msgIdempotencyModel.MarkMessageProcessed(l.ctx, messageID, 24*3600)

// ❌ 错误：直接操作 Redis
// redisClient.SetNX(...)
// redisClient.SetEx(...)
```

## 单元测试

```go
package mq

import (
	"context"
	"testing"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"github.com/stretchr/testify/assert"
)

func TestCountLogic_Count(t *testing.T) {
	// 创建 Mock 依赖
	mockUserModel := &mock.MockUserModel{}
	mockMsgIdempotencyModel := &mock.MockMsgIdempotencyModel{}

	mockSvcCtx := &svc.ServiceContext{
		Repo: &repository.Repository{
			UserModel:              mockUserModel,
			MsgIdempotencyRdsModel: mockMsgIdempotencyModel,
		},
	}

	countLogic := NewCountLogic(context.Background(), mockSvcCtx)

	err := countLogic.Count(123)
	assert.NoError(t, err)
}
```
