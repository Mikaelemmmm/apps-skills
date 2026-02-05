# MQ Server 示例

MQ Server 负责初始化消息队列消费者并注册消费处理器。

## 文件位置

`internal/server/mq.go`

## 代码示例

```go
package server

import (
	"fmt"

	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/mq"
	"apps/app/services/core/user/internal/svc"
	queue "apps/pkg/queue/rabbitmq"

	"github.com/zeromicro/go-zero/core/service"
)

// MqServer MQ Server
type MqServer struct {
	consumers []service.Service  // 消费者列表
	config    config.MqConfig
	svcCtx    *svc.ServiceContext
}

// NewMqServer 创建 MQ Server
func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
	// 创建 MQ Handler
	userConsumer := mq.NewUserMQ(svcCtx)

	// 创建消费者并注册处理函数
	consumers := []service.Service{
		queue.MustNewQueue(c.UserCountMq, userConsumer.CountConsumer()),
		queue.MustNewQueue(c.UserCountMq, userConsumer.UpLevelConsumer()),
	}

	return &MqServer{
		config:    c,
		svcCtx:    svcCtx,
		consumers: consumers,
	}
}

// Start 启动 MQ Server
func (c *MqServer) Start() {
	fmt.Printf("start user-mq server ")
	for _, consumer := range c.consumers {
		consumer.Start()
	}
}

// Stop 停止 MQ Server
func (c *MqServer) Stop() {
	fmt.Printf("stop user-mq ")
	for _, consumer := range c.consumers {
		consumer.Stop()
	}
	fmt.Printf("stop user-mq finish")
}
```

## 入口文件示例

### 独立部署模式

`cmd/mq/main.go`

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

var configFile = flag.String("f", "etc/mq.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.MqConfig
	conf.MustLoad(*configFile, &c)

	// 初始化日志
	xlog.Init(c.Name, c.LogConf)

	// 初始化 OpenTelemetry
	otel.Init(c.Name, c.OtelConf)

	// 初始化 Prometheus
	metrics.Init(c.MetricsConf)

	// 初始化 ServiceContext
	svcCtx := svc.NewServiceContext(c.Config)

	// 启动 MQ Server
	mqServer := server.NewMqServer(c, svcCtx)
	mqServer.Start()
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

	// 可选：启动 MQ Server
	// mqConfig := config.MqConfig{...}
	// mqServer := server.NewMqServer(mqConfig, svcCtx)
	// go mqServer.Start()

	grpcServer.Wait()
}
```

## 关键点说明

### 1. Server 结构

```go
type MqServer struct {
	consumers []service.Service  // 消费者列表
	config    config.MqConfig
	svcCtx    *svc.ServiceContext
}
```

### 2. 注册消费者

```go
// 创建 MQ Handler
userConsumer := mq.NewUserMQ(svcCtx)

// 创建消费者并注册处理函数
consumers := []service.Service{
	queue.MustNewQueue(c.UserCountMq, userConsumer.CountConsumer()),
	queue.MustNewQueue(c.UserCountMq, userConsumer.UpLevelConsumer()),
}
```

### 3. 启动和停止

```go
func (c *MqServer) Start() {
	for _, consumer := range c.consumers {
		consumer.Start()
	}
}

func (c *MqServer) Stop() {
	for _, consumer := range c.consumers {
		consumer.Stop()
	}
}
```

### 4. 支持的消息队列

| 队列类型 | 包路径 | 使用方法 |
|---------|--------|---------|
| RabbitMQ | `apps/pkg/queue/rabbitmq` | `rabbitmq.MustNewQueue(conf, handler)` |
| Kafka | `apps/pkg/queue/kafka` | `kafka.MustNewQueue(conf, handler)` |
