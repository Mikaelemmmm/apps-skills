# MQ Config 配置示例

MQ 消费者的配置定义。

## 文件位置

`internal/config/config.go`

## 代码示例

```go
package config

import (
	"apps/pkg/queue/kafka"
	"apps/pkg/queue/rabbitmq"

	"github.com/zeromicro/go-zero/core/service"
)

// MqConfig 包含 MQ 消费者配置
type MqConfig struct {
	service.ServiceConf  // go-zero 基础服务配置
	Config      Config   // 业务配置
	UserCountMq rabbitmq.QueueConf  // RabbitMQ 队列配置
	UserUpLevel kafka.QueueConf      // Kafka 队列配置
}

// Config 包含业务配置
type Config struct {
	DB      DBConf
	DBCache cache.CacheConf
	RPC     RPCConf
}
```

## 配置文件示例

`cmd/mq/conf.yaml`

```yaml
Name: user-mq
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "root:password@tcp(127.0.0.1:3306)/services_core_user?charset=utf8mb4&parseTime=true"
  DBCache:
    Host: "127.0.0.1:6379"
    Type: node
  RPC:
    AuthzSrvConf:
      Endpoints:
        - 127.0.0.1:10001

UserCountMq:
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
  Url: localhost:9092
  Topic: user_up_level_topic
  GroupId: user_group
```

## QueueConf 结构说明

### RabbitMQ QueueConf

| 字段 | 类型 | 说明 |
|-----|------|------|
| `Url` | `string` | RabbitMQ 连接地址 |
| `Exchange` | `ExchangeConf` | 交换机配置 |
| `Queue` | `QueueConf` | 队列配置 |
| `RoutingKey` | `string` | 路由键 |
| `MaxAttempts` | `int` | 最大重试次数 |

### Kafka QueueConf

| 字段 | 类型 | 说明 |
|-----|------|------|
| `Url` | `string` | Kafka 连接地址 |
| `Topic` | `string` | 主题名称 |
| `GroupId` | `string` | 消费者组 ID |
