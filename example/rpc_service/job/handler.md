# Job Handler 示例

Job Handler 层负责调用 Logic 层处理业务。

## 文件位置

`internal/job/xxx_job.go`

## 代码示例

```go
package job

import (
	"context"

	"apps/app/services/core/user/internal/logic/job"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/xlog"
)

// UserJob 用户 Job Handler
type UserJob struct {
	svcCtx *svc.ServiceContext
}

// NewUserJob 创建用户 Job Handler
func NewUserJob(svcCtx *svc.ServiceContext) *UserJob {
	return &UserJob{
		svcCtx: svcCtx,
	}
}

// OneHandler 任务一示例（每 5 秒执行）
func (j *UserJob) OneHandler(ctx context.Context) error {
	// 调用 Logic 层处理业务
	// syncUserLogic := logic.NewSyncUserLogic(ctx, j.svcCtx)
	// return syncUserLogic.SyncUser()
	xlog.Infof(ctx, "userJob OneHandler started")
	return nil
}

// TwoHandler 任务二示例（每分钟执行）
func (j *UserJob) TwoHandler(ctx context.Context) error {
	// 调用 Logic 层处理业务
	// cleanExpiredLogic := logic.NewCleanExpiredLogic(ctx, j.svcCtx)
	// return cleanExpiredLogic.CleanExpired()
	xlog.Infof(ctx, "userJob TwoHandler started")
	return nil
}
```

## 关键点说明

### 1. Handler 职责

**Handler 层只负责：**
- 创建 Logic 实例
- 调用 Logic 方法
- 返回结果

### 2. Handler 结构

```go
type UserJob struct {
	svcCtx *svc.ServiceContext
}

func NewUserJob(svcCtx *svc.ServiceContext) *UserJob {
	return &UserJob{
		svcCtx: svcCtx,
	}
}
```

### 3. 任务方法签名

```go
func (j *XxxJob) JobName(ctx context.Context) error {
	// 调用 Logic 层处理业务
	xxxLogic := logic.NewXxxLogic(ctx, j.svcCtx)
	return xxxLogic.Method(...)
}
```

### 4. 错误处理

**Handler 层错误处理：**
- Logic 层返回的错误：直接返回

**Logic 层错误处理：**
- 业务错误：使用业务定义的错误码，`xerr.NewMsgWithInfoLog`
- 系统错误：使用系统错误码，`xerr.NewWithErrorLog`

## 单元测试

```go
package job

import (
	"context"
	"testing"

	"apps/app/services/core/user/internal/svc"
	"github.com/stretchr/testify/assert"
)

func TestUserJob_OneHandler(t *testing.T) {
	mockSvcCtx := &svc.ServiceContext{
		// 设置 Mock 依赖
	}

	userJob := NewUserJob(mockSvcCtx)
	ctx := context.Background()

	err := userJob.OneHandler(ctx)
	assert.NoError(t, err)
}
```
