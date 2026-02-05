# Job ServiceContext 初始化示例

Job 服务的 ServiceContext 与 gRPC 服务共享，负责初始化和注入所有依赖。

> **注意**：Job 和 gRPC 共享同一个 ServiceContext，不需要单独创建。

## 文件位置

`internal/svc/service_context.go`

## 代码示例

```go
package svc

import (
	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/repository"
)

// ServiceContext 服务上下文
type ServiceContext struct {
	Config config.Config
	Repo   *repository.Repository
}

// NewServiceContext 创建服务上下文
func NewServiceContext(c config.Config) *ServiceContext {
	return &ServiceContext{
		Config: c,
		Repo:   repository.NewRepository(c),
	}
}
```

## 说明

| 字段 | 说明 |
|-----|------|
| `Config` | 服务配置，从配置文件加载 |
| `Repo` | Repository 依赖，包含 DB Model、RPC 客户端等 |

## 依赖注入流程

```
config.JobConfig.Config
       │
       ▼
repository.NewRepository(c)
       │
       ▼
repository.Repository
       │
       ▼
svc.ServiceContext
       │
       ▼
job.NewUserJob(svcCtx)
```

## Job Handler 使用

```go
// internal/job/user_job.go

package job

import (
	"context"

	"apps/app/services/core/user/internal/svc"
	"apps/app/services/core/user/internal/logic/job"
)

type UserJob struct {
	svcCtx *svc.ServiceContext
}

func NewUserJob(svcCtx *svc.ServiceContext) *UserJob {
	return &UserJob{
		svcCtx: svcCtx,
	}
}

// OneHandler 任务一示例（每 5 秒执行）
func (j *UserJob) OneHandler(ctx context.Context) error {
	// 从 svcCtx.Repo 获取依赖
	syncUserLogic := logic.NewSyncUserLogic(ctx, j.svcCtx)
	return syncUserLogic.SyncUser()
}

// TwoHandler 任务二示例（每分钟执行）
func (j *UserJob) TwoHandler(ctx context.Context) error {
	// 从 svcCtx.Repo 获取依赖
	cleanExpiredLogic := logic.NewCleanExpiredLogic(ctx, j.svcCtx)
	return cleanExpiredLogic.CleanExpired()
}
```
