# Job Logic 示例

Job Logic 层负责处理具体业务逻辑，调用 Repository 层操作数据。

## 目录结构

```
internal/logic/job/
├── sync_user_logic.go    # 同步用户业务
├── archive_user_logic.go # 归档用户业务
└── clean_expired_logic.go  # 清理过期数据业务
```

## 代码示例

### sync_user_logic.go

```go
package job

import (
	"context"
	"time"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xerr"
	"apps/pkg/xlog"
)

// SyncUserLogic 同步用户业务逻辑
type SyncUserLogic struct {
	ctx              context.Context
	userModel        repository.UserModelType
	jobLockRdsModel   repository.JobLockRdsModelType
	thirdService     repository.ThirdServiceType
}

// NewSyncUserLogic 创建同步用户业务逻辑
func NewSyncUserLogic(ctx context.Context, svcCtx *svc.ServiceContext) *SyncUserLogic {
	return &SyncUserLogic{
		ctx:              ctx,
		userModel:        svcCtx.Repo.UserModel,
		jobLockRdsModel:   svcCtx.Repo.JobLockRdsModel,
		thirdService:     svcCtx.Repo.ThirdService,
	}
}

// SyncUser 同步用户数据
func (l *SyncUserLogic) SyncUser() error {
	start := time.Now()
	defer func() {
		duration := time.Since(start)
		xlog.Infow(l.ctx, "SyncUserLogic finished", "duration", duration)
	}()

	xlog.Infow(l.ctx, "SyncUserLogic started")

	// 获取分布式锁
	lockKey := "job:sync_user"
	locked, err := l.jobLockRdsModel.AcquireLock(l.ctx, lockKey, 10*60)
	if err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"acquire lock failed, lock_key: %s, err: %v", lockKey, err,
		)
	}
	if !locked {
		xlog.Infow(l.ctx, "job is already running", "lock_key", lockKey)
		return nil
	}

	// 确保锁被释放
	defer l.jobLockRdsModel.ReleaseLock(l.ctx, lockKey)

	// 查询需要同步的用户
	users, err := l.userModel.FindPendingSync(l.ctx, 100)
	if err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"find pending sync users failed, err: %v", err,
		)
	}

	if len(users) == 0 {
		xlog.Infow(l.ctx, "no users need sync")
		return nil
	}

	// 同步用户
	successCount := 0
	for _, user := range users {
		if err := l.syncOneUser(user); err != nil {
			xlog.Errorw(l.ctx, "sync user failed",
				"user_id", user.ID,
				"err", err,
			)
			continue
		}
		successCount++
	}

	xlog.Infow(l.ctx, "sync users completed",
		"total", len(users),
		"success", successCount,
	)

	return nil
}

// syncOneUser 同步单个用户
func (l *SyncUserLogic) syncOneUser(user *model.User) error {
	// 调用第三方服务同步
	synced, err := l.thirdService.GetUser(l.ctx, user.ID)
	if err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"sync user to third service failed, user_id: %d, err: %v",
			user.ID, err,
		)
	}

	// 更新同步状态
	user.SyncStatus = 1
	user.SyncTime = time.Now()
	user.SyncData = synced
	return l.userModel.UpdateSyncStatus(l.ctx, user.ID, synced)
}
```

### archive_user_logic.go

```go
package job

import (
	"context"
	"time"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xerr"
	"apps/pkg/xlog"
)

// ArchiveUserLogic 归档用户业务逻辑
type ArchiveUserLogic struct {
	ctx              context.Context
	userModel        repository.UserModelType
	userArchiveModel repository.UserArchiveModelType
}

// NewArchiveUserLogic 创建归档用户业务逻辑
func NewArchiveUserLogic(ctx context.Context, svcCtx *svc.ServiceContext) *ArchiveUserLogic {
	return &ArchiveUserLogic{
		ctx:              ctx,
		userModel:        svcCtx.Repo.UserModel,
		userArchiveModel: svcCtx.Repo.UserArchiveModel,
	}
}

// ArchiveUser 归档用户数据
func (l *ArchiveUserLogic) ArchiveUser() error {
	start := time.Now()
	defer func() {
		duration := time.Since(start)
		xlog.Infow(l.ctx, "ArchiveUserLogic finished", "duration", duration)
	}()

	xlog.Infow(l.ctx, "ArchiveUserLogic started")

	const batchSize = 1000
	var lastID int64 = 0

	for {
		users, err := l.userModel.FindDeletedBefore(l.ctx, lastID, batchSize)
		if err != nil {
			return xerr.NewWithErrorLog(
				xerr.SystemError_D_ERROR,
				"find deleted users failed, err: %v", err,
			)
		}

		if len(users) == 0 {
			break
		}

		// 处理当前批次
		for _, user := range users {
			if err := l.archiveOneUser(user); err != nil {
				xlog.Errorw(l.ctx, "archive user failed",
					"user_id", user.ID,
					"err", err,
				)
			}
		}

		// 更新游标
		lastID = users[len(users)-1].ID
	}

	return nil
}

// archiveOneUser 归档单个用户
func (l *ArchiveUserLogic) archiveOneUser(user *model.User) error {
	// 归档到历史表
	archive := &model.UserArchive{
		ID:       user.ID,
		Name:     user.Name,
		Email:    user.Email,
		Data:      user.Data,
		DeletedAt: time.Now(),
	}

	if err := l.userArchiveModel.Insert(l.ctx, archive); err != nil {
		return xerr.NewWithErrorLog(
			xerr.SystemError_D_ERROR,
			"insert user archive failed, user_id: %d, err: %v",
			user.ID, err,
		)
	}

	// 真实删除用户表数据
	return l.userModel.Delete(l.ctx, user.ID)
}
```

### clean_expired_logic.go

```go
package job

import (
	"context"
	"time"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xlog"
)

// CleanExpiredLogic 清理过期数据业务逻辑
type CleanExpiredLogic struct {
	ctx           context.Context
	loginLogModel repository.LoginLogModelType
}

// NewCleanExpiredLogic 创建清理过期数据业务逻辑
func NewCleanExpiredLogic(ctx context.Context, svcCtx *svc.ServiceContext) *CleanExpiredLogic {
	return &CleanExpiredLogic{
		ctx:           ctx,
		loginLogModel: svcCtx.Repo.LoginLogModel,
	}
}

// CleanExpired 清理过期的登录日志
func (l *CleanExpiredLogic) CleanExpired() error {
	start := time.Now()
	defer func() {
		duration := time.Since(start)
		xlog.Infow(l.ctx, "CleanExpiredLogic finished", "duration", duration)
	}()

	xlog.Infow(l.ctx, "CleanExpiredLogic started")

	// 删除 90 天前的登录日志
	expiredBefore := time.Now().AddDate(0, 0, -90)
	deleted, err := l.loginLogModel.DeleteExpired(l.ctx, expiredBefore)
	if err != nil {
		return err
	}

	xlog.Infow(l.ctx, "clean expired login logs completed",
		"deleted", deleted,
	)

	return nil
}
```

## 关键点说明

### 1. Logic 结构

```go
type XxxLogic struct {
	ctx              context.Context
	userModel        repository.UserModelType
	jobLockRdsModel   repository.JobLockRdsModelType
}

func NewXxxLogic(ctx context.Context, svcCtx *svc.ServiceContext) *XxxLogic {
	return &XxxLogic{
		ctx:              ctx,
		userModel:        svcCtx.Repo.UserModel,
		jobLockRdsModel:   svcCtx.Repo.JobLockRdsModel,
	}
}
```

**Logic 结构体规范：**
- 包含 `ctx context.Context`
- 从 `svcCtx.Repo` 提取需要的依赖
- 使用具体的 xredis model（如 `JobLockRdsModel`），不是通用的 `RedisClient`

### 2. 业务方法签名

```go
func (l *XxxLogic) MethodName() error {
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
locked, err := l.jobLockRdsModel.AcquireLock(l.ctx, lockKey, 10*60)
defer l.jobLockRdsModel.ReleaseLock(l.ctx, lockKey)

// ❌ 错误：直接操作 Redis
// redisClient.SetNX(...)
// redisClient.Del(...)
```

## 单元测试

```go
package job

import (
	"context"
	"testing"

	"apps/app/services/core/user/internal/repository"
	"apps/app/services/core/user/internal/svc"
	"github.com/stretchr/testify/assert"
)

func TestSyncUserLogic_SyncUser(t *testing.T) {
	// 创建 Mock 依赖
	mockUserModel := &mock.MockUserModel{}
	mockJobLockModel := &mock.MockJobLockModel{}

	mockSvcCtx := &svc.ServiceContext{
		Repo: &repository.Repository{
			UserModel:      mockUserModel,
			JobLockRdsModel: mockJobLockModel,
		},
	}

	syncUserLogic := NewSyncUserLogic(context.Background(), mockSvcCtx)

	err := syncUserLogic.SyncUser()
	assert.NoError(t, err)
}
```
