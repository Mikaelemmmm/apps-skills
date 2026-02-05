# REST API 中嵌入 MQ 的 main.go 示例

展示如何在 REST API 服务中嵌入 MQ 消息队列消费者。

> **重要**：REST API 中的 MQ Handler、Logic、Repository 写法与 RPC Service 完全一致，请参考：
> - [handler.md](../../rpc_service/mq/handler.md) - MQ Handler 示例
> - [logic.md](../../rpc_service/mq/logic.md) - MQ Logic 示例
> - [repository.md](../../rpc_service/mq/repository.md) - MQ Repository 示例

## 文件位置

`app/apis/{app}/{layer}/main.go`（例如：`app/apis/stayy/api/main.go`）

## 完整示例

```go
package main

import (
	"flag"
	"context"

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

	// 初始化 ServiceContext（HTTP、MQ 共享）
	svcCtx := svc.NewServiceContext(c.Config)

	// 创建 HTTP Server
	httpServer := rest.MustNewServer(c.RestConf, rest.WithServerOption(&server{
		svcCtx: svcCtx,
	}))

	// 注册路由
	handler.RegisterHandlers(httpServer, svcCtx)

	// 启动 MQ Server（goroutine）
	if c.Mq.UserCount.Url != "" {
		mqServer := server.NewMqServer(c.Mq, svcCtx)
		go mqServer.Start()
		xlog.Infow(context.Background(), "mq server started")
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

`ApiConfig` 包含 `MqConfig` 字段，所有 MQ 配置统一在 `MqConfig` 下。

### 2. 启用控制

```go
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq, svcCtx)
    go mqServer.Start()
    xlog.Infow(context.Background(), "mq server started")
}
```

通过检查 MQ 配置的 `Url` 字段是否为空来决定是否启动 MQ Server。

**推荐做法**：检查 `MqConfig` 中的任一队列配置的 `Url` 字段。

### 3. ServiceContext 共享

```go
svcCtx := svc.NewServiceContext(c.Config)
```

HTTP 和 MQ 共享同一个 `ServiceContext`，确保数据库连接、Redis 连接等资源被正确复用。

### 4. Goroutine 启动

```go
go mqServer.Start()
```

MQ Server 必须在 goroutine 中启动，否则会阻塞 HTTP Server 的启动。

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

# RabbitMQ 配置
Mq:
  UserCount:
    Url: amqp://guest:guest@127.0.0.1:5672/
    Exchange:
      Name: user_exchange
      Kind: direct
      Durable: true
    Queue:
      Name: user_count_queue
      Durable: true
    RoutingKey: user.count
    MaxAttempts: 3

  UserUpLevel:
    Url: amqp://guest:guest@127.0.0.1:5672/
    Exchange:
      Name: user_exchange
      Kind: direct
      Durable: true
    Queue:
      Name: user_up_level_queue
      Durable: true
    RoutingKey: user.up_level
    MaxAttempts: 3
```

## Config 结构定义

`internal/config/config.go`

```go
package config

import (
    "apps/pkg/queue/rabbitmq"
    "github.com/zeromicro/go-zero/rest"
)

// ApiConfig 包含 API 服务配置
type ApiConfig struct {
    rest.RestConf
    Config Config

    // MQ 配置
    Mq MqConfig
}

// Config 包含业务配置
type Config struct {
    DB      DBConf
    DBCache cache.CacheConf
}

// MqConfig MQ 配置结构
type MqConfig struct {
    // UserCount MQ 配置
    UserCount rabbitmq.QueueConf
    // UserUpLevel MQ 配置
    UserUpLevel rabbitmq.QueueConf
}

// DBConf 数据库配置
type DBConf struct {
    Mysql string
}

// CacheConf 缓存配置
type CacheConf struct {
    Host string
    Type string
}
```

## 同时嵌入 Job 和 MQ

```go
package main

func main() {
	// ... 初始化配置和 ServiceContext

	// 启动 Job Server（可选）
	if c.JobConfig.Enable {
		jobServer := server.NewJobServer(c.JobConfig, svcCtx)
		go jobServer.Start()
		xlog.Infow(context.Background(), "job server started")
	}

	// 启动 MQ Server（可选）
	if c.Mq.UserCount.Url != "" {
		mqServer := server.NewMqServer(c.Mq, svcCtx)
		go mqServer.Start()
		xlog.Infow(context.Background(), "mq server started")
	}

	// 启动 HTTP Server（主 goroutine）
	httpServer.Start()
}
```

## 代码结构对比

与 RPC Service 的对比：

| 文件 | REST API | RPC Service | 说明 |
|-----|----------|-------------|------|
| 入口 | `app/apis/stayy/api/main.go` | `app/services/core/user/cmd/mq/main.go` | **不同** |
| 配置文件 | `etc/conf.yaml` | `cmd/mq/conf.yaml` | **不同** |
| Config | `ApiConfig` (包含 `Mq MqConfig`) | `MqConfig` (包含 `Config`) | **结构不同** |
| Handler | `internal/mq/user_mq.go` | `internal/mq/user_mq.go` | **完全一致** |
| Logic | `internal/logic/mq/xxx_logic.go` | `internal/logic/mq/xxx_logic.go` | **完全一致** |
| Repository | `internal/repository/` | `internal/repository/` | **完全一致** |
| Server | `internal/server/mq.go` | `internal/server/mq.go` | **写法一致** |
| ServiceContext | 与 HTTP 共享 | 独立创建 | **不同** |
| 部署方式 | 仅合并部署 | 支持独立部署和合并部署 | **不同** |

## 相关示例

| 文件 | 说明 |
|-----|------|
| [config.md](./config.md) | MQ 配置示例 |
| [server.md](./server.md) | MQ Server 示例 |
| [handler.md](../../rpc_service/mq/handler.md) | MQ Handler 示例（与 RPC Service 共享） |
| [logic.md](../../rpc_service/mq/logic.md) | MQ Logic 示例（与 RPC Service 共享） |
| [repository.md](../../rpc_service/mq/repository.md) | MQ Repository 层示例（与 RPC Service 共享） |
