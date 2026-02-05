# Job Server 示例

Job Server 负责初始化定时任务调度器并注册任务处理器。

## 文件位置

`internal/server/job.go`

## 代码示例

```go
package server

import (
	"context"
	"fmt"
	"time"

	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/job"
	"apps/app/services/core/user/internal/svc"
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
			Name:     "userJob.OneHandler",
			Type:     scheduler.Duration,      // 固定间隔类型
			ExecTime: time.Second * 5,        // 每 5 秒执行
			ExecFunc: userJob.OneHandler,
		},
		{
			Name:     "userJob.TwoHandler",
			Type:     scheduler.Cron,           // Cron 表达式类型
			ExecTime: "0 * * * * *",          // 每分钟执行
			ExecFunc: userJob.TwoHandler,
		},
	}

	// 创建调度器
	server := scheduler.MustNewServer()
	if err := server.RegisterJobs(jobs); err != nil {
		xlog.Errorw(context.Background(), "server.RegisterJobs failed", xlog.Field("err", err))
	}

	return &JobServer{
		config: c,
		svcCtx: svcCtx,
		server: server,
	}
}

// Start 启动 Job Server
func (s *JobServer) Start() {
	fmt.Println("start user-job")
	s.server.Start()
}

// Stop 停止 Job Server
func (s *JobServer) Stop() {
	fmt.Println("stop user-job")
	s.server.Stop()
	fmt.Println("stop user-job finish")
}
```

## 入口文件示例

### 独立部署模式

`cmd/job/main.go`

```go
package main

import (
	"flag"

	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/server"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/metrics"
	"apps/pkg/otel"
	"apps/pkg/xlog"
	"github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/job.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.JobConfig
	conf.MustLoad(*configFile, &c)

	// 初始化日志
	xlog.Init(c.Name, c.LogConf)

	// 初始化 OpenTelemetry
	otel.Init(c.Name, c.OtelConf)

	// 初始化 Prometheus
	metrics.Init(c.MetricsConf)

	// 初始化 ServiceContext
	svcCtx := svc.NewServiceContext(c.Config)

	// 启动 Job Server
	jobServer := server.NewJobServer(c, svcCtx)
	jobServer.Start()
}
```

### 合并部署模式（与 gRPC 合并）

`cmd/grpc/main.go`

```go
package main

import (
	"flag"

	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/server"
	"apps/app/services/core/user/internal/svc"
	"apps/pkg/metrics"
	"apps/pkg/otel"
	"apps/pkg/xlog"
	"github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/grpc.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.GrpcConfig
	conf.MustLoad(*configFile, &c)

	// 初始化日志
	xlog.Init(c.Name, c.LogConf)

	// 初始化 OpenTelemetry
	otel.Init(c.Name, c.OtelConf)

	// 初始化 Prometheus
	metrics.Init(c.MetricsConf)

	// 初始化 ServiceContext
	svcCtx := svc.NewServiceContext(c.Config)

	// 启动 gRPC Server
	grpcServer := server.NewGrpcServer(c, svcCtx)
	go grpcServer.Start()

	// 可选：启动 Job Server
	// jobConfig := config.JobConfig{...}
	// jobServer := server.NewJobServer(jobConfig, svcCtx)
	// go jobServer.Start()

	grpcServer.Wait()
}
```

## 关键点说明

### 1. Server 结构

```go
type JobServer struct {
	config config.JobConfig
	svcCtx *svc.ServiceContext
	server scheduler.Scheduler
}
```

### 2. 注册任务

```go
// 创建 Job Handler
userJob := job.NewUserJob(svcCtx)

// 定义任务列表
jobs := []scheduler.Job{
	{
		Name:     "userJob.OneHandler",
		Type:     scheduler.Duration,      // 固定间隔类型
		ExecTime: time.Second * 5,        // 每 5 秒执行
		ExecFunc: userJob.OneHandler,
	},
	{
		Name:     "userJob.TwoHandler",
		Type:     scheduler.Cron,           // Cron 表达式类型
		ExecTime: "0 * * * * *",          // 每分钟执行
		ExecFunc: userJob.TwoHandler,
	},
}

// 创建调度器
server := scheduler.MustNewServer()
server.RegisterJobs(jobs)
```

### 3. 任务类型

| 类型 | 常量 | ExecTime 类型 | 示例 |
|-----|------|--------------|------|
| 固定间隔 | `scheduler.Duration` | `time.Duration` | `time.Second * 5` |
| Cron 表达式 | `scheduler.Cron` | `string` | `"0 * * * * *"` |

### 4. Job Handler 方法签名

```go
func (j *UserJob) OneHandler(ctx context.Context) error {
	// 调用 Logic 层处理业务
	syncUserLogic := logic.NewSyncUserLogic(ctx, j.svcCtx)
	return syncUserLogic.SyncUser()
}
```

### 5. 合并部署注意事项

```go
// gRPC Server 需要在 goroutine 中启动
go grpcServer.Start()

// Job Server 需要在 goroutine 中启动
go jobServer.Start()

// 等待退出信号
grpcServer.Wait()
```

**注意**：合并部署时，Job Server 必须使用 goroutine 启动，因为 `Start()` 方法会阻塞。
