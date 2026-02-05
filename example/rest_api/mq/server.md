# REST API 中 MQ Server 示例

展示如何在 REST API 服务中创建 MQ Server。

> **重要**：MQ Server 的写法与 RPC Service 完全一致，只是包路径和 config 来源不同。

## 文件位置

`internal/server/mq.go`

## 代码示例

```go
package server

import (
	"context"
	"fmt"

	"apps/app/apis/stayy/api/internal/config"
	"apps/app/apis/stayy/api/internal/mq"
	"apps/app/apis/stayy/api/internal/svc"
	queue "apps/pkg/queue/rabbitmq"

	"github.com/zeromicro/go-zero/core/service"
)

// MqServer MQ Server
type MqServer struct {
	consumers []service.Service  // 消费者列表
	config    string             // MQ URL（用于日志）
	svcCtx    *svc.ServiceContext
}

// NewMqServer 创建 MQ Server
func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
	// 创建 MQ Handler（与 RPC Service 完全一致）
	userMQ := mq.NewUserMQ(svcCtx)

	// 创建消费者所有消费者
	consumers := []service.Service{
		queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
		queue.MustNewQueue(c.UserUpLevel, userMQ.UpLevelConsumer()),
	}

	return &MqServer{
		consumers: consumers,
		config:    c.UserCount.Url,
		svcCtx:    svcCtx,
	}
}

// Start 启动 MQ Server
func (s *MqServer) Start() {
	fmt.Printf("start mq server: %s\n", s.config)
	for _, consumer := range s.consumers {
		consumer.Start()
	}
}

// Stop 停止 MQ Server
func (s *MqServer) Stop() {
	fmt.Printf("stop mq server: %s\n", s.config)
	for _, consumer := range s.consumers {
		consumer.Stop()
	}
	fmt.Printf("stop mq server finish: %s\n", s.config)
}
```

## 多队列配置示例

如果需要配置多个队列，在 `ApiConfig` 的 `MqConfig` 中添加多个队列配置：

### Config 结构

```go
type ApiConfig struct {
	rest.RestConf
	Config Config

	// MQ 配置
	Mq MqConfig
}

type MqConfig struct {
	UserCount   rabbitmq.QueueConf
	UserUpLevel rabbitmq.QueueConf
}
```

### 配置文件

```yaml
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

## 关键点说明

### 1. Server 结构

```go
type MqServer struct {
	consumers []service.Service  // 消费者列表
	config    string             // MQ URL（用于日志）
	svcCtx    *svc.ServiceContext
}
```

### 2. 创建 MQ Handler

```go
// 创建 MQ Handler（与 RPC Service 完全一致）
userMQ := mq.NewUserMQ(svcCtx)
```

MQ Handler 位于 `internal/mq/xxx_mq.go`，写法与 RPC Service 完全一致。

### 3. 注册消费者

```go
consumers := []service.Service{
	queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
	queue.MustNewQueue(c.UserUpLevel, userMQ.UpLevelConsumer()),
}
```

每个队列通过 `queue.MustNewQueue` 注册，参数为：
- 配置对象（`c.UserCount`）
- 消费处理函数（`userMQ.CountConsumer()`）

### 4. 启动和停止

```go
func (s *MqServer) Start() {
	for _, consumer := range s.consumers {
		consumer.Start()
	}
}

func (s *MqServer) Stop() {
	for _, consumer := range s.consumers {
		consumer.Stop()
	}
}
```

### 5. 在 main.go 中启动

```go
// 启动 MQ Server（goroutine）
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq, svcCtx)
    go mqServer.Start()
}
```

通过检查 MQ 配置的 `Url` 字段是否为空来决定是否启动。

## 支持多个队列的启动方式

### 方式一：创建多个 MQ Server

```go
// 在 main.go 中
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq.UserCount, svcCtx)
    go mqServer.Start()
}

if c.Mq.UserUpLevel.Url != "" {
    mqServer := {其他MQ配置}
    go mqServer.Start()
}
```

### 方式二：在 MqConfig 中统一管理

```go
// 推荐：所有 MQ 配置统一在 MqConfig 中
type MqConfig struct {
	UserCount   rabbitmq.QueueConf
	UserUpLevel rabbitmq.QueueConf
}

// 在 main.go 中检查其中一个是否启用即可
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq, svcCtx)
    go mqServer.Start()
}
```

## 支持 RabbitMQ 和 Kafka

### RabbitMQ

```go
import queue "apps/pkg/queue/rabbitmq"

consumers := []service.Service{
	queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
}
```

### Kafka

如果使用 Kafka，需要修改 `MqConfig` 结构和导入：

```go
import queue "apps/pkg/queue/kafka"

type MqConfig struct {
	UserCount kafka.QueueConf
}

consumers := []service.Service{
	queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
}
```

## 与 RPC Service MQ Server 的对比

| 特性 | REST API MQ Server | RPC Service MQ Server |
|-----|-------------------|----------------------|
| 包路径 | `app/apis/stayy/api/...` | `app/services/core/user/...` |
| 配置来源 | `etc/conf.yaml` | `cmd/mq/conf.yaml` |
| Config 类型 | `config.MqConfig` | `config.MqConfig` |
| Handler 写法 | **完全一致** | - |
| Logic 写法 | **完全一致** | - |
| Repository 写法 | **完全一致** | - |
| Server 写法 | **完全一致** | - |
| 入口文件 | `app/apis/stayy/api/main.go` | `app/services/core/user/cmd/mq/main.go` |
| ServiceContext | 与 HTTP 共享 | 独立创建 |
| 部署方式 | 仅合并部署 | 支持独立部署和合并部署 |

## 代码对比

### RPC Service MQ Server

```go
// app/services/core/user/internal/server/mq.go
package server

import (
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/mq"
    "apps/app/services/core/user/internal/svc"
    queue "apps/pkg/queue/rabbitmq"
)

type MqServer struct {
    consumers []service.Service
    config    string
    svcCtx    *svc.ServiceContext
}

func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
    userMQ := mq.NewUserMQ(svcCtx)

    consumers := []service.Service{
        queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
        queue.MustNewQueue(c.UserUpLevel, userMQ.UpLevelConsumer()),
    }

    return &MqServer{
        consumers: consumers,
        config:    c.UserCount.Url,
        svcCtx:    svcCtx,
    }
}

func (s *MqServer) Start() {
    for _, consumer := range s.consumers {
        consumer.Start()
    }
}

func (s *MqServer) Stop() {
    for _, consumer := range s.consumers {
        consumer.Stop()
    }
}
```

### REST API MQ Server

```go
// app/apis/stayy/api/internal/server/mq.go
package server

import (
    "apps/app/apis/stayy/api/internal/config"
    "apps/app/apis/stayy/api/internal/mq"
    "apps/app/apis/stayy/api/internal/svc"
    queue "apps/pkg/queue/rabbitmq"
)

type MqServer struct {
    consumers []service.Service
    config    string
    svcCtx    *svc.ServiceContext
}

func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
    userMQ := mq.NewUserMQ(svcCtx)

    consumers := []service.Service{
        queue.MustNewQueue(c.UserCount, userMQ.CountConsumer()),
        queue.MustNewQueue(c.UserUpLevel, userMQ.UpLevelConsumer()),
    }

    return &MqServer{
        consumers: consumers,
        config:    c.UserCount.Url,
        svcCtx:    svcCtx,
    }
}

func (s *MqServer) Start() {
    for _, consumer := range s.consumers {
        consumer.Start()
    }
}

func (s *MqServer) Stop() {
    for _, consumer := range s.consumers {
        consumer.Stop()
    }
}
```

**差异**：只是包路径和 config 类型来源不同，写法完全一致。

## MQ Handler 方法签名

```go
// internal/mq/user_mq.go（与 RPC Service 完全一致）
func (s *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
    return func(ctx context.Context, key string, payload []byte) error {
        // 1. 解析消息参数（protobuf）
        var req userMqPb.UserCountReq
        if err := proto.Unmarshal(payload, &req); err != nil {
            return xerr.NewMsgWithErrorLog(...)
        }

        // 2. 调用 Logic 层处理业务
        countLogic := logic.NewCountLogic(ctx, s.svcCtx)
        return countLogic.Count(req.UserId)
    }
}
```

## 相关示例

| 文件 | 说明 |
|-----|------|
| [config.md](./config.md) | MQ 配置示例 |
| [handler.md](../../rpc_service/mq/handler.md) | MQ Handler 示例（与 RPC Service 共享） |
| [logic.md](../../rpc_service/mq/logic.md) | MQ Logic 示例（与 RPC Service 共享） |
| [repository.md](../../rpc_service/mq/repository.md) | MQ Repository 层示例（与 RPC Service 共享） |
