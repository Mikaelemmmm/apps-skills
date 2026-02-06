# REST API 服务中 Job 嵌入模式

REST API 服务（`app/apis/`）可以嵌入 Job（定时任务），适用于轻量级场景。

## 核心原则

**REST API 中的 Job 代码结构与 RPC Service 完全一致，只有部署方式不同。**

| 层级 | REST API | RPC Service | 说明 |
|-----|----------|-------------|------|
| Config | `config.JobConfig` | `config.JobConfig` | **结构完全一致** |
| Handler | `internal/job/xxx_job.go` | `internal/job/xxx_job.go` | **写法完全一致** |
| Logic | `internal/logic/job/xxx_logic.go` | `internal/logic/job/xxx_logic.go` | **写法完全一致** |
| Repository | `internal/repository/` | `internal/repository/` | **写法完全一致** |
| Server | `internal/server/job.go` | `internal/server/job.go` | **写法基本一致** |
| 入口文件 | `app/apis/stayy/api/main.go` | `cmd/job/main.go` | **不同：HTTP 主进程** |
| 配置文件 | `etc/conf.yaml` | `cmd/job/conf.yaml` | **不同：嵌入 HTTP 配置** |
| 部署方式 | 与 HTTP 合并 | 独立或与 gRPC 合并 | **不同** |

## 服务定位

```
┌─────────────────────────────────────────────────────────────┐
│              REST API Service (单进程)                       │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  HTTP Server (主 goroutine)                          │  │
│  │  - 处理 HTTP 请求                                     │  │
│  │  - 执行 Logic 层业务逻辑                               │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Job Server (goroutine)                              │  │
│  │  - 执行定时任务                                        │  │
│  │  - 代码结构与 RPC Service 完全一致                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Shared Repository (共享)                            │  │
│  │  - 数据库连接                                         │  │
│  │  - Redis 连接                                        │  │
│  │  - RPC 客户端                                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 部署模式

| 模式 | 说明 | 适用场景 |
|-----|------|---------|
| **合并部署** | HTTP、Job 在同一进程中运行 | 业务量不大、资源节省、轻量级任务 |

> **注意**：REST API 服务中的 Job 仅支持合并部署模式，不支持独立部署。

## 开发步骤

| 步骤 | 操作 | 说明 | 参考文档 |
|-----|------|-----|---------|
| 1 | 添加配置 | 在 `internal/config/config.go` 中添加 JobConfig | [config.md](../example/rest_api/job/config.md) |
| 2 | 创建 Handler | 在 `internal/job/xxx_job.go` 中实现 Handler | [handler.md](../example/rpc_service/job/handler.md) |
| 3 | 实现 Logic | 在 `internal/logic/job/xxx_logic.go` 中实现业务逻辑 | [logic.md](../example/rpc_service/job/logic.md) |
| 4 | 创建 Server | 在 `internal/server/job.go` 中创建 Server | [server.md](../example/rest_api/job/server.md) |
| 5 | 修改入口 | 在 `main.go` 中启动 Job Server | [main.md](../example/rest_api/job/main.md) |

## 服务内部结构

```
app/apis/{app}/{layer}/
├── main.go               # HTTP 服务入口（同时启动 Job）
├── etc/
│   └── conf.yaml          # 包含 HTTP、Job 配置
└── internal/
    ├── config/           # 配置定义
    ├── job/              # Job Handler（调用 Logic）
    │   └── xxxxx_job.go
    ├── logic/            # 业务逻辑层
    │   └── job/
    │       ├── userservice/           # user job logic
    │       │   └── shared/            # user job logic shared code
    │       └── userthirdservice/
    │           └── shared/            # user third job logic shared code
    ├── manager/          # Logic 复用层（跨 logic 共享）
    ├── repository/       # 依赖注入层（HTTP、Job 共享）
    │   ├── model/        # 数据库 Model
    │   ├── rpc/          # RPC 客户端
    │   ├── xredis/       # Redis 操作（含 Job 相关方法）
    │   ├── repository.go
    │   └── repository_type.go
    ├── server/           # Server 初始化
    │   ├── http.go       # HTTP Server（生成）
    │   └── job.go        # Job Server
    └── svc/              # ServiceContext
```

## 代码结构与 RPC Service 的对比

### 1. Config 配置

#### RPC Service Job Config

```go
// app/services/core/user/internal/config/config.go
type JobConfig struct {
    service.ServiceConf  // go-zero 基础服务配置
    Config Config        // 业务配置
}

// cmd/job/conf.yaml
Name: user-job
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."
  DBCache:
    Host: "..."
```

#### REST API Job Config

```go
// app/apis/stayy/api/internal/config/config.go
type ApiConfig struct {
    rest.RestConf
    Config Config

    // Job 配置（嵌入在 ApiConfig 中）
    JobConfig JobConfig
}

type JobConfig struct {
    Enable bool          // 是否启用 Job
    Name   string        // Job 名称
}

// etc/conf.yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."
  DBCache:
    Host: "..."

JobConfig:
  Enable: true
  Name: user-api
```

**差异**：
- RPC Service: 使用独立的 `service.ServiceConf`，有独立配置文件 `cmd/job/conf.yaml`
- REST API: 配置嵌入在 `ApiConfig` 中，配置在 `etc/conf.yaml`

### 2. Handler 层

#### RPC Service Job Handler

```go
// app/services/core/user/internal/job/user_job.go
package job

import (
    "context"
    "apps/app/services/core/user/internal/logic/job"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/xlog"
)

type UserJob struct {
    svcCtx *svc.ServiceContext
}

func NewUserJob(svcCtx *svc.ServiceContext) *UserJob {
    return &UserJob{svcCtx: svcCtx}
}

func (j *UserJob) OneHandler(ctx context.Context) error {
    syncUserLogic := logic.NewSyncUserLogic(ctx, j.svcCtx)
    return syncUserLogic.SyncUser()
}
```

#### REST API Job Handler

```go
// app/apis/stayy/api/internal/job/user_job.go
package job

import (
    "context"
    "apps/app/apis/stayy/api/internal/logic/job"
    "apps/app/apis/stayy/api/internal/svc"
    "apps/pkg/xlog"
)

type UserJob struct {
    svcCtx *svc.ServiceContext
}

func NewUserJob(svcCtx *svc.ServiceContext) *UserJob {
    return &UserJob{svcCtx: svcCtx}
}

func (j *UserJob) OneHandler(ctx context.Context) error {
    syncUserLogic := logic.NewSyncUserLogic(ctx, j.svcCtx)
    return syncUserLogic.SyncUser()
}
```

**差异**：**无差异**，只是包路径不同

### 3. Logic 层

#### RPC Service Job Logic

```go
// app/services/core/user/internal/logic/job/sync_user_logic.go
package job

import (
    "context"
    "time"
    "apps/app/services/core/user/internal/repository"
    "apps/pkg/xerr"
    "apps/pkg/xlog"
)

type SyncUserLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    Repo   *repository.Repository
}

func NewSyncUserLogic(ctx context.Context, svcCtx *svc.ServiceContext) *SyncUserLogic {
    return &SyncUserLogic{
        ctx:    ctx,
        svcCtx: svcCtx,
        Repo:   svcCtx.Repo,
    }
}

func (l *SyncUserLogic) SyncUser() error {
    // 业务逻辑
    return nil
}
```

#### REST API Job Logic

```go
// app/apis/stayy/api/internal/logic/job/sync_user_logic.go
package job

import (
    "context"
    "time"
    "apps/app/apis/stayy/api/internal/repository"
    "apps/pkg/xerr"
    "apps/pkg/xlog"
)

type SyncUserLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    Repo   *repository.Repository
}

func NewSyncUserLogic(ctx context.Context, svcCtx *svc.ServiceContext) *SyncUserLogic {
    return &SyncUserLogic{
        ctx:    ctx,
        svcCtx: svcCtx,
        Repo:   svcCtx.Repo,
    }
}

func (l *SyncUserLogic) SyncUser() error {
    // 业务逻辑
    return nil
}
```

**差异**：**无差异**，只是包路径不同

### 4. Server 层

#### RPC Service Job Server

```go
// app/services/core/user/internal/server/job.go
package server

import (
    "context"
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

    jobs := []scheduler.Job{
        {
            Name:     "userJob.OneHandler",
            Type:     scheduler.Duration,
            ExecTime: time.Second * 5,
            ExecFunc: userJob.OneHandler,
        },
    }

    server := scheduler.MustNewServer()
    server.RegisterJobs(jobs)

    return &JobServer{
        config: c,
        svcCtx: svcCtx,
        server: server,
    }
}

func (s *JobServer) Start() {
    s.server.Start()
}

func (s *JobServer) Stop() {
    s.server.Stop()
}
```

#### REST API Job Server

```go
// app/apis/stayy/api/internal/server/job.go
package server

import (
    "context"
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

    jobs := []scheduler.Job{
        {
            Name:     "userJob.OneHandler",
            Type:     scheduler.Duration,
            ExecTime: time.Second * 5,
            ExecFunc: userJob.OneHandler,
        },
    }

    server := scheduler.MustNewServer()
    server.RegisterJobs(jobs)

    return &JobServer{
        config: c,
        svcCtx: svcCtx,
        server: server,
    }
}

func (s *JobServer) Start() {
    s.server.Start()
}

func (s *JobServer) Stop() {
    s.server.Stop()
}
```

**差异**：**无差异**，只是包路径和 config 类型来源不同

### 5. 入口文件

#### RPC Service Job Entry

```go
// app/services/core/user/cmd/job/main.go
package main

import (
    "flag"
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/server"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/xlog"
    "github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/job.yaml", "the config file")

func main() {
    flag.Parse()

    var c config.JobConfig
    conf.MustLoad(*configFile, &c)

    xlog.Init(c.Name, c.LogConf)
    svcCtx := svc.NewServiceContext(c.Config)

    jobServer := server.NewJobServer(c, svcCtx)
    jobServer.Start()
}
```

#### REST API Job Entry

```go
// app/apis/stayy/api/main.go
package main

import (
    "flag"
    "apps/app/apis/stayy/api/internal/config"
    "apps/app/apis/stayy/api/internal/handler"
    "apps/app/apis/stayy/api/internal/svc"
    "apps/app/apis/stayy/api/internal/server"
    "apps/pkg/xlog"
    "github.com/zeromicro/go-zero/rest"
    "github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/conf.yaml", "the config file")

func main() {
    flag.Parse()

    var c config.ApiConfig
    conf.MustLoad(*configFile, &c)

    xlog.Init(c.Name, c.LogConf)
    svcCtx := svc.NewServiceContext(c.Config)

    // 创建 HTTP Server
    httpServer := rest.MustNewServer(c.RestConf, rest.WithServerOption(&server{
        svcCtx: svcCtx,
    }))
    handler.RegisterHandlers(httpServer, svcCtx)

    // 启动 Job Server（goroutine）
    if c.JobConfig.Enable {
        jobServer := server.NewJobServer(c.JobConfig, svcCtx)
        go jobServer.Start()
    }

    // 启动 HTTP Server（主 goroutine）
    httpServer.Start()
}
```

**差异**：
- RPC Service: 独立进程，直接启动 Job Server
- REST API: HTTP 主进程，Job 在 goroutine 中启动

## 关键注意事项

### 1. Goroutine 启动

```go
// ✅ 正确：Job 必须在 goroutine 中启动
if c.JobConfig.Enable {
    jobServer := server.NewJobServer(c.JobConfig, svcCtx)
    go jobServer.Start()  // 使用 goroutine
}

// HTTP Server 为主 goroutine，直接启动
httpServer.Start()
```

### 2. 共享 Repository

Job 与 HTTP 共享同一个 Repository 实例，确保数据库连接池、Redis 连接等资源被正确复用。

```go
// ServiceContext 初始化时创建的 Repository
svcCtx := svc.NewServiceContext(c.Config)

// HTTP、Job 都使用同一个 svcCtx
// 所有资源（数据库、Redis、RPC 客户端）自动共享
```

### 3. 配置启用控制

```go
// 通过配置控制是否启动
if c.JobConfig.Enable {
    jobServer := server.NewJobServer(c.JobConfig, svcCtx)
    go jobServer.Start()
}
```

## 相关示例

| 文件 | 说明 |
|-----|------|
| [main.md](../example/rest_api/job/main.md) | REST API 中嵌入 Job 的 main.go 示例 |
| [config.md](../example/rest_api/job/config.md) | REST API 中 Job 配置示例 |
| [server.md](../example/rest_api/job/server.md) | REST API 中 Job Server 示例 |
| [handler.md](../example/rpc_service/job/handler.md) | Job Handler 示例（与 RPC Service 共享） |
| [logic.md](../example/rpc_service/job/logic.md) | Job Logic 示例（与 RPC Service 共享） |
| [repository.md](../example/rpc_service/job/repository.md) | Job Repository 层示例（与 RPC Service 共享） |
