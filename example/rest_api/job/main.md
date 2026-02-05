# REST API 中嵌入 Job 的 main.go 示例

展示如何在 REST API 服务中嵌入 Job 定时任务。

> **重要**：REST API 中的 Job Handler、Logic、Repository 写法与 RPC Service 完全一致，请参考：
> - [handler.md](../../rpc_service/job/handler.md) - Job Handler 示例
> - [logic.md](../../rpc_service/job/logic.md) - Job Logic 示例
> - [repository.md](../../rpc_service/job/repository.md) - Job Repository 示例

## 文件位置

`app/apis/{app}/{layer}/main.go`（例如：`app/apis/stayy/api/main.go`）

## 完整示例

```go
package main

import (
	"flag"

	"apps/app/apis/stayy/api/internal/config"
	"apps/app/apis/stayy/api/internal/handler"
	"apps/app/apis/stayy/api/internal/svc"
	"apps/app/apis/stayy/api/internal/server"
	"apps/pkg/metrics"
	"apps/pkg/otel"
	"apps/pkg/xlog"
	"github.com/zeromicro/go-zero/rest"
	"github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/conf.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.ApiConfig
	conf.MustLoad(*configFile, &c)

	// 初始化日志
	xlog.Init(c.Name, c.LogConf)

	// 初始化 OpenTelemetry
	otel.Init(c.Name, c.OtelConf)

	// 初始化 Prometheus
	metrics.Init(c.MetricsConf)

	// 初始化 ServiceContext（HTTP、Job 共享）
	svcCtx := svc.NewServiceContext(c.Config)

	// 创建 HTTP Server
	httpServer := rest.MustNewServer(c.RestConf, rest.WithServerOption(&server{
		svcCtx: svcCtx,
	}))

	// 注册路由
	handler.RegisterHandlers(httpServer, svcCtx)

	// 启动 Job Server（可选）
	if c.JobConfig.Enable {
		jobServer := server.NewJobServer(c.JobConfig, svcCtx)
		go jobServer.Start()
		xlog.Infow(context.Background(), "job server started")
	}

	// 启动 HTTP Server（主 goroutine）
	httpServer.Start()
}
```

## 关键点说明

### 1. 配置加载

```go
var c config.ApiConfig
conf.MustLoad(*configFile, &c)
```

`ApiConfig` 包含 `JobConfig` 字段，用于 Job 相关配置。

### 2. 启用控制

```go
if c.JobConfig.Enable {
    jobServer := server.NewJobServer(c.JobConfig, svcCtx)
    go jobServer.Start()
}
```

通过 `JobConfig.Enable` 控制是否启动 Job Server。

### 3. ServiceContext 共享

```go
svcCtx := svc.NewServiceContext(c.Config)
```

HTTP 和 Job 共享同一个 `ServiceContext`，确保数据库连接、Redis 连接等资源被正确复用。

### 4. Goroutine 启动

```go
go jobServer.Start()
```

Job Server 必须在 goroutine 中启动，否则会阻塞 HTTP Server 的启动。

### 5. HTTP Server 为主 goroutine

```go
httpServer.Start()
```

HTTP Server 在主 goroutine 中运行，当接收到退出信号时会通知所有子 goroutine。

## 配置示例

`etc/conf.yaml`

```yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "root:password@tcp(127.0.0.1:3306)/services_stayy_api?charset=utf8mb4&parseTime=true"
  DBCache:
    Host: "127.0.0.1:6379"
    Type: node

JobConfig:
  Enable: true
  Name: user-api
```

## Config 结构定义

`internal/config/config.go`

```go
package config

import (
    "github.com/zeromicro/go-zero/rest"
)

// ApiConfig 包含 API 服务配置
type ApiConfig struct {
    rest.RestConf
    Config Config

    // Job 配置
    JobConfig JobConfig
}

// Config 包含业务配置
type Config struct {
    DB      DBConf
    DBCache cache.CacheConf
}

// JobConfig 包含 Job 定时任务配置
type JobConfig struct {
    Enable bool          // 是否启用 Job
    Name   string        // Job 名称
}
```

## 代码结构对比

与 RPC Service 的对比：

| 文件 | REST API | RPC Service | 说明 |
|-----|----------|-------------|------|
| 入口 | `app/apis/stayy/api/main.go` | `app/services/core/user/cmd/job/main.go` | **不同** |
| 配置文件 | `etc/conf.yaml` | `cmd/job/conf.yaml` | **不同** |
| Config | `config.JobConfig`（嵌入） | `config.JobConfig`（独立） | **不同** |
| Handler | `internal/job/user_job.go` | `internal/job/user_job.go` | **完全一致** |
| Logic | `internal/logic/job/xxx_logic.go` | `internal/logic/job/xxx_logic.go` | **完全一致** |
| Repository | `internal/repository/` | `internal/repository/` | **完全一致** |
| Server | `internal/server/job.go` | `internal/server/job.go` | **写法一致** |
