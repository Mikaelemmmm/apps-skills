# REST API 中 MQ 配置示例

展示如何在 REST API 服务中配置 MQ 消息队列消费者。

## 文件位置

`internal/config/config.go`

## Config 结构定义

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
    RPC     RPCConf
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

// RPCConf RPC 客户端配置
type RPCConf struct {
    UserSrvConf ServiceConf
}

// ServiceConf RPC 服务配置
type ServiceConf struct {
    Endpoints []string
}
```

## 配置文件示例

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
  RPC:
    UserSrvConf:
      Endpoints:
        - 127.0.0.1:10001

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

## RabbitMQ QueueConf 字段说明

| 字段 | 类型 | 说明 | 示例 |
|-----|------|------|------|
| `Url` | `string` | RabbitMQ 连接地址 | `amqp://guest:guest@127.0.0.1:5672/` |
| `Exchange.Name` | `string` | 交换机名称 | `user_exchange` |
| `Exchange.Kind` | `string` | 交换机类型 | `direct` / `topic` / `fanout` |
| `Exchange.Durable` | `bool` | 交换机是否持久化 | `true` |
| `Queue.Name` | `string` | 队列名称 | `user_count_queue` |
| `Queue.Durable` | `bool` | 队列是否持久化 | `true` |
| `RoutingKey` | `string` | 路由键 | `user.count` |
| `MaxAttempts` | `int` | 最大重试次数 | `3` |

## 启用条件

```go
// 在 main.go 中检查
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq, svcCtx)
    go mqServer.Start()
}
```

通过检查 MQ 配置的 `Url` 字段是否为空来决定是否启动 MQ Server。

## 不启用 MQ 的配置

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

# MQ 配置留空或注释
# Mq:
#   UserCount:
#     Url: ""
```

## 支持多种消息队列

### RabbitMQ 配置

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
```

### Kafka 配置

如果使用 Kafka，需要修改 `MqConfig` 结构：

```go
package config

import (
    "apps/pkg/queue/kafka"
    "github.com/zeromicro/go-zero/rest"
)

type ApiConfig struct {
    rest.RestConf
    Config Config

    // Kafka 配置
    Mq KafkaMqConfig
}

type KafkaMqConfig struct {
    UserCount kafka.QueueConf
}
```

**配置文件**：

```yaml
Mq:
  UserCount:
    Url: localhost:9092
    Topic: user_count_topic
    GroupId: user_api_group
```

## 同时配置 Job 和 MQ

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
```

## 与 RPC Service MQ 配置的区别

### RPC Service MQ Config

```go
// app/services/core/user/internal/config/config.go
type MqConfig struct {
    service.ServiceConf  // go-zero 基础服务配置
    Config Config        // 业务配置
    UserCountMq rabbitmq.QueueConf
}
```

**配置文件**：`cmd/mq/conf.yaml`

```yaml
Name: user-mq
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."

UserCountMq:
  Url: amqp://guest:guest@127.0.0.1:5672/
  # ...
```

### REST API MQ Config

```go
// app/apis/stayy/api/internal/config/config.go
type ApiConfig struct {
    rest.RestConf
    Config Config

    // MQ 配置（使用独立的 MqConfig 结构）
    Mq MqConfig
}

type MqConfig struct {
    UserCount rabbitmq.QueueConf
    UserUpLevel rabbitmq.QueueConf
}
```

**配置文件**：`etc/conf.yaml`（与 HTTP 配置在同一文件）

```yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."

Mq:
  UserCount:
    Url: amqp://guest:guest@127.0.0.1:5672/
    # ...
```

## 关键点说明

1. **独立 MqConfig 结构**：所有 MQ 配置集中在 `MqConfig` 结构下，每个队列作为字段定义

2. **配置文件层级**：在 YAML 配置文件中，所有 MQ 配置在 `Mq:` 下，每个队列作为子字段

3. **启用控制**：通过检查 MQ 配置的 `Url` 字段是否为空来控制是否启动 MQ Server

4. **共享配置**：HTTP 和 MQ 共享 `Config` 中的业务配置（数据库、Redis、RPC 客户端等）

5. **不创建独立配置文件**：REST API 不创建 `cmd/mq/conf.yaml`，所有配置都在 `etc/conf.yaml` 中

6. **多队列支持**：一个服务可以配置多个`MqConfig` 中的字段，如 `UserCount`、`UserUpLevel`

7. **ServiceContext 共享**：HTTP 和 MQ 使用同一个 `ServiceContext` 实例

## 相关示例

| 文件 | 说明 |
|-----|------|
| [config.md](../rpc_service/mq/config.md) | RPC Service MQ 配置（完整版） |
