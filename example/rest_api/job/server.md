# REST API 中 Job Server 示例

展示如何在 REST API 服务中创建 Job Server。

> **重要**：Job Server 的写法与 RPC Service 完全一致，只是包路径和 config 来源不同。

## 文件位置

`internal/server/job.go`

## 代码示例

```go
package server

import (
	"context"
	"fmt"
	"time"

	"apps/app/apis/stayy/api/internal/config"
	"apps/app/apis/stayy/api/internal/job"
	"apps/app/apis/stayy/api/internal/svc"
	"apps/pkg/scheduler"
	"apps/pkg/xlog"
)

// JobServer Job Server
type JobServer struct {
	config config.JobConfig
	svcCtx *svc.ServiceContext
	server scheduler.Scheduler
}

// NewJobServer 创建 Job Server
func NewJobServer(c config.JobConfig, svcCtx *svc.ServiceContext) *JobServer {
	// 创建 Job Handler
	userJob := job.NewUserJob(svcCtx)

	// 定义任务列表
	jobs := []scheduler.Job{
		{
			Name:     "userJob.CleanupExpiredTokens",
			Type:     scheduler.Duration,      // 固定间隔类型
			ExecTime: time.Hour,               // 每小时执行
			ExecFunc: userJob.CleanupExpiredTokens,
		},
		{
			Name:     "userJob.UpdateUserStatistics",
			Type:     scheduler.Cron,           // Cron 表达式类型
			ExecTime: "0 0 2 * * *",          // 每天凌晨 2 点执行
			ExecFunc: userJob.UpdateUserStatistics,
		},
	}

	// 创建调度器
	schedulerServer := scheduler.MustNewServer()
	if err := schedulerServer.RegisterJobs(jobs); err != nil {
		xlog.Errorw(context.Background(), "register jobs failed", "err", err)
	}

	return &JobServer{
		config: c,
		svcCtx: svcCtx,
		server: schedulerServer,
	}
}

// Start 启动 Job Server
func (s *JobServer) Start() {
	fmt.Printf("start %s job server\n", s.config.Name)
	s.server.Start()
}

// Stop 停止 Job Server
func (s *JobServer) Stop() {
	fmt.Printf("stop %s job server\n", s.config.Name)
	s.server.Stop()
	fmt.Printf("stop %s job server finish\n", s.config.Name)
}
```

## 关键点说明

### 1. Server 结构

```go
type JobServer struct {
	config config.JobConfig     // Job 配置
	svcCtx *svc.ServiceContext  // ServiceContext（与 HTTP 共享）
	server scheduler.Scheduler   // 调度器实例
}
```

### 2. 创建 Job Handler

```go
// 创建 Job Handler（与 RPC Service 完全一致）
userJob := job.NewUserJob(svcCtx)
```

Job Handler 位于 `internal/job/xxx_job.go`，写法与 RPC Service 完全一致。

### 3. 定义任务列表

```go
jobs := []scheduler.Job{
	{
		Name:     "userJob.CleanupExpiredTokens",
		Type:     scheduler.Duration,
		ExecTime: time.Hour,
		ExecFunc: userJob.CleanupExpiredTokens,
	},
	{
		Name:     "userJob.UpdateUserStatistics",
		Type:     scheduler.Cron,
		ExecTime: "0 0 2 * * *",
		ExecFunc: userJob.UpdateUserStatistics,
	},
}
```

### 4. 任务类型

| 类型 | 常量 | ExecTime 类型 | 示例 |
|-----|------|--------------|------|
| 固定间隔 | `scheduler.Duration` | `time.Duration` | `time.Hour` |
| Cron 表达式 | `scheduler.Cron` | `string`` `"0 0 2 * * *"` |

### 5. Cron 表达式格式

```
┌─────────── 秒 (0-59)
│ ┌───────── 分 (0-59)
│ │ ┌─────── 时 (0-23)
│ │ │ ┌───── 日 (1-31)
│ │ │ │ ┌─── 月 (1-12)
│ │ │ │ │ ┌─ 星期 (0-6, 0=周日)
│ │ │ │ │ │
* * * * *
```

### 常用 Cron 表达式

| 表达式 | 说明 |
|--------|------|
| `0 * * * * *` | 每分钟第 0 秒执行 |
| `0 */5 * * * *` | 每 5 分钟执行 |
| `0 0 * * * *` | 每小时第 0 分 0 秒执行 |
| `0 0 */2 * * *` | 每 2 小时执行 |
| `0 0 0 * * *` | 每天零点执行 |
| `0 0 2 * * *` | 每天凌晨 2 点执行 |
| `0 0 0 * * 1` | 每周一零点执行 |
| `0 0 0 1 * *` | 每月 1 号零点执行 |

### 6. Job Handler 方法签名

```go
// internal/job/user_job.go（与 RPC Service 完全一致）
func (j *UserJob) CleanupExpiredTokens(ctx context.Context) error {
	// 调用 Logic 层处理业务
	cleanupLogic := logic.NewCleanupExpiredTokensLogic(ctx, j.svcCtx)
	return cleanupLogic.CleanupExpiredTokens()
}
```

### 7. 在 main.go 中启动

```go
if c.JobConfig.Enable {
    jobServer := server.NewJobServer(c.JobConfig, svcCtx)
    go jobServer.Start()
}
```

## 与 RPC Service Job Server 的对比

| 特性 | REST API Job Server | RPC Service Job Server |
|-----|---------------------|------------------------|
| 包路径 | `app/apis/stayy/api/...` | `app/services/core/user/...` |
| 配置结构 | `config.JobConfig`（嵌入） | `config.JobConfig`（独立） |
| 配置来源 | `etc/conf.yaml` | `cmd/job/conf.yaml` |
| Handler 写法 | **完全一致** | - |
| Logic 写法 | **完全一致** | - |
| Repository 写法 | **完全一致** | - |
| Server 写法 | **完全一致** | - |
| 入口文件 | `app/apis/stayy/api/main.go` | `app/services/core/user/cmd/job/main.go` |
| ServiceContext | 与 HTTP 共享 | 独立创建 |

## 代码对比

### RPC Service Job Server

```go
// app/services/core/user/internal/server/job.go
package server

import (
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/job"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/scheduler"
)

type JobServer struct {
    config config.JobConfig
    svcCtx *svc.ServiceContext
    server scheduler.Scheduler
}

func NewJobServer(c config.JobConfig, svcCtx *svc.ServiceContext) *JobServer {
    userJob := job.NewUserJob(svcCtx)
    // ... 任务注册
    return &JobServer{...}
}
```

### REST API Job Server

```go
// app/apis/stayy/api/internal/server/job.go
package server

import (
    "apps/app/apis/stayy/api/internal/config"
    "apps/app/apis/stayy/api/internal/job"
    "apps/app/apis/stayy/api/internal/svc"
    "apps/pkg/scheduler"
)

type JobServer struct {
    config config.JobConfig
    svcCtx *svc.ServiceContext
    server scheduler.Scheduler
}

func NewJobServer(c config.JobConfig, svcCtx *svc.ServiceContext) *JobServer {
    userJob := job.NewUserJob(svcCtx)
    // ... 任务注册
    return &JobServer{...}
}
```

**差异**：只是包路径不同，写法完全一致

## 相关示例

| 文件 | 说明 |
|-----|------|
| [job_server.md](../../rpc_service/job/server.md) | RPC Service Job Server（完整版） |
| [handler.md](../../rpc_service/job/handler.md) | Job Handler 示例（与 RPC Service 共享） |
| [logic.md](../../rpc_service/job/logic.md) | Job Logic 示例（与 RPC Service 共享） |
