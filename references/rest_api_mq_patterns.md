# REST API 服务中 MQ 嵌入模式

REST API 服务（`app/apis/`）可以嵌入 MQ（消息队列消费者），适用于轻量级场景。

## 核心原则

**REST API 中的 MQ 代码结构与 RPC Service 完全一致，只有部署方式不同。**

| 层级       | REST API                         | RPC Service                      | 说明                     |
| ---------- | -------------------------------- | -------------------------------- | ------------------------ |
| Config     | `config.MqConfig`                | `config.MqConfig`                | **结构完全一致**         |
| Handler    | `internal/mq/xxx_mq.go`          | `internal/mq/xxx_mq.go`          | **写法完全一致**         |
| Logic      | `internal/logic/mq/xxx_logic.go` | `internal/logic/mq/xxx_logic.go` | **写法完全一致**         |
| Repository | `internal/repository/`           | `internal/repository/`           | **写法完全一致**         |
| Server     | `internal/server/mq.go`          | `internal/server/mq.go`          | **写法基本一致**         |
| 入口文件   | `app/apis/stayy/api/main.go`     | `cmd/mq/main.go`                 | **不同：HTTP 主进程**    |
| 配置文件   | `etc/conf.yaml`                  | `cmd/mq/conf.yaml`               | **不同：嵌入 HTTP 配置** |
| 部署方式   | 与 HTTP 合并                     | 独立或与 gRPC 合并               | **不同**                 |

## 服务定位

```
┌─────────────────────────────────────────────────────────────┐
│              REST API Service (单进程)                       │
│                                                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  HTTP Server (主 goroutine)                          │  │
│  │  - 处理 HTTP 请求                                     │  │
│  │  - 执行 Logic 层业务逻辑                               │  │
│  └───────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  MQ Server (goroutine)                              │  │
│  │  - 消费消息队列                                       │  │
│  │  - 代码结构与 RPC Service 完全一致                     │  │
│  └───────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Shared Repository (共享)                            │  │
│  │  - 数据库连接                                         │  │
│  │  - Redis 连接                                        │  │
│  │  - RPC 客户端                                        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 部署模式

| 模式         | 说明                      | 适用场景                           |
| ------------ | ------------------------- | ---------------------------------- |
| **合并部署** | HTTP、MQ 在同一进程中运行 | 业务量不大、资源节省、轻量级消费者 |

> **注意**：REST API 服务中的 MQ 仅支持合并部署模式，不支持独立部署。

## 支持的消息队列

| 队列类型     | 包路径                    | 说明            |
| ------------ | ------------------------- | --------------- |
| **RabbitMQ** | `apps/pkg/queue/rabbitmq` | RabbitMQ 消费者 |
| **Kafka**    | `apps/pkg/queue/kafka`    | Kafka 消费者    |

## 开发步骤

| 步骤 | 操作         | 说明                                                              | 参考文档                                                       |
| ---- | ------------ | ----------------------------------------------------------------- | -------------------------------------------------------------- |
| 1    | 定义消息体   | 在 `protos/common/mq/xxx.proto` 中定义消息                        | [proto_message.md](../example/rpc_service/mq/proto_message.md) |
| 2    | 添加配置     | 在 `internal/config/config.go` 中添加 MQ 配置                     | [config.md](../example/rest_api/mq/config.md)                  |
| 3    | 创建 Handler | 在 `internal/mq/xxx_mq.go` 中实现 Handler（解析参数，调用 Logic） | [handler.md](../example/rpc_service/mq/handler.md)             |
| 4    | 实现 Logic   | 在 `internal/logic/mq/xxx_logic.go` 中实现业务逻辑                | [logic.md](../example/rpc_service/mq/logic.md)                 |
| 5    | 创建 Server  | 在 `internal/server/mq.go` 中创建 Server                          | [server.md](../example/rest_api/mq/server.md)                  |
| 6    | 修改入口     | 在 `main.go` 中启动 MQ Server                                     | [main.md](../example/rest_api/mq/main.md)                      |

## 服务内部结构

```
app/apis/{app}/{layer}/
├── main.go               # HTTP 服务入口（同时启动 MQ）
├── etc/
│   └── conf.yaml          # 包含 HTTP、MQ 配置
└── internal/
    ├── config/           # 配置定义
    ├── mq/               # MQ Handler（解析参数，调用 Logic）
    │   └── xxxxx_mq.go
    ├── logic/            # 业务逻辑层
    │   └── mq/
    │       ├── userservice/           # user mq logic
    │       │   └── shared/            # user mq logic shared code
    │       └── userthirdservice/
    │           └── shared/            # user third mq logic shared code
    ├── manager/          # Logic 复用层（跨 logic 共享）
    ├── repository/       # 依赖注入层（HTTP、MQ 共享）
    │   ├── model/        # 数据库 Model
    │   ├── rpc/          # RPC 客户端
    │   ├── xredis/       # Redis 操作
    │   ├── repository.go
    │   └── repository_type.go
    ├── server/           # Server 初始化
    │   ├── http.go       # HTTP Server（生成）
    │   └── mq.go         # MQ Server
    └── svc/              # ServiceContext
```

## Protobuf 消息定义

MQ 消息体使用 Protobuf 定义，文件位于 `protos/common/mq/` 目录下。

> **注意**：每个服务一个文件，文件名就是服务名。只使用 `message` 和 `enum` 定义，其他不需要。

### 目录结构

```
protos/common/mq/
└── user.proto          # User 服务 MQ 消息定义
```

### 消息定义示例

**user.proto**

```proto
syntax = "proto3";

package common.mq;

option go_package = "apps/pb/common/mq";

// UserCountReq 统计消息
message UserCountReq {
  // 用户ID
  int64 user_id = 1;
}

// UserUpLevelReq 升级消息
message UserUpLevelReq {
  // 用户ID
  int64 user_id = 1;
  // 目标等级
  int32 level = 2;
  // 消息ID（用于幂等性）
  string message_id = 3;
}
```

### 命名规范

| 规范       | 说明                                            |
| ---------- | ----------------------------------------------- |
| 文件名     | 服务名（小写），如 `user.proto`、`order.proto`  |
| package    | `common.mq`                                     |
| go_package | `apps/pb/common/mq`                             |
| Message 名 | 操作 + Req，如 `UserCountReq`、`OrderCreateReq` |
| 字段名     | snake_case，如 `user_id`、`message_id`          |

### 生成代码

```bash
# 更新 protobuf submodule
make update

# 构建 protobuf 文件
make build
```

生成后的 Go 代码位于：`apps/pb/common/mq/`

## 代码结构与 RPC Service 的对比

### 1. Config 配置

#### RPC Service MQ Config

```go
// app/services/core/user/internal/config/config.go
type MqConfig struct {
    service.ServiceConf  // go-zero 基础服务配置
    Config Config        // 业务配置
}

// cmd/mq/conf.yaml
Name: user-mq
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."
  DBCache:
    Host: "..."

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
```

#### REST API MQ Config

```go
// app/apis/stayy/api/internal/config/config.go
type ApiConfig struct {
    rest.RestConf
    Config Config

    // MQ 配置
    Mq MqConfig
}

type MqConfig struct {
    // UserCount MQ 配置
    UserCount rabbitmq.QueueConf
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
}
```

**差异**：

- RPC Service: 使用独立的 `service.ServiceConf`，有独立配置文件 `cmd/mq/conf.yaml`
- REST API: 使用独立的 `MqConfig` 结构，所有 MQ 配置集中在 `Mq` 下

### 2. Handler 层

#### RPC Service MQ Handler

```go
// app/services/core/user/internal/mq/user_mq.go
package mq

import (
    "context"
    "google.golang.org/protobuf/proto"
    userMqPb "apps/pb/common/mq"
    "apps/app/services/core/user/internal/logic/mq"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/xerr"
)

type UserMQ struct {
    svcCtx *svc.ServiceContext
}

func NewUserMQ(svcCtx *svc.ServiceContext) *UserMQ {
    return &UserMQ{svcCtx: svcCtx}
}

func (r *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
    return func(ctx context.Context, key string, payload []byte) error {
        var req userMqPb.UserCountReq
        if err := proto.Unmarshal(payload, &req); err != nil {
            return xerr.NewMsgWithErrorLog(
                xerr.SystemError_PARAMS_ERROR,
                "unmarshal message failed",
                "key: %s, payload: %s, err: %v",
                key, string(payload), err,
            )
        }

        countLogic := logic.NewCountLogic(ctx, r.svcCtx)
        return countLogic.Count(req.UserId)
    }
}
```

#### REST API MQ Handler

```go
// app/apis/stayy/api/internal/mq/user_mq.go
package mq

import (
    "context"
    "google.golang.org/protobuf/proto"
    userMqPb "apps/pb/common/mq"
    "apps/app/apis/stayy/api/internal/logic/mq"
    "apps/app/apis/stayy/api/internal/svc"
    "apps/pkg/xerr"
)

type UserMQ struct {
    svcCtx *svc.ServiceContext
}

func NewUserMQ(svcCtx *svc.ServiceContext) *UserMQ {
    return &UserMQ{svcCtx: svcCtx}
}

func (r *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
    return func(ctx context.Context, key string, payload []byte) error {
        var req userMqPb.UserCountReq
        if err := proto.Unmarshal(payload, &req); err != nil {
            return xerr.NewMsgWithErrorLog(
                xerr.SystemError_PARAMS_ERROR,
                "unmarshal message failed",
                "key: %s, payload: %s, err: %v",
                key, string(payload), err,
            )
        }

        countLogic := logic.NewCountLogic(ctx, r.svcCtx)
        return countLogic.Count(req.UserId)
    }
}
```

**差异**：**无差异**，只是包路径不同

### 3. Logic 层

#### RPC Service MQ Logic

```go
// app/services/core/user/internal/logic/mq/count_logic.go
package mq

import (
    "context"
    "apps/app/services/core/user/internal/repository"
    "apps/pkg/xerr"
    "apps/pkg/xlog"
)

type CountLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    Repo   *repository.Repository
}

func NewCountLogic(ctx context.Context, svcCtx *svc.ServiceContext) *CountLogic {
    return &CountLogic{
        ctx:    ctx,
        svcCtx: svcCtx,
        Repo:   svcCtx.Repo,
    }
}

func (l *CountLogic) Count(userID int64) error {
    // 业务逻辑
    return nil
}
```

#### REST API MQ Logic

```go
// app/apis/stayy/api/internal/logic/mq/count_logic.go
package mq

import (
    "context"
    "apps/app/apis/stayy/api/internal/repository"
    "apps/pkg/xerr"
    "apps/pkg/xlog"
)

type CountLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    Repo   *repository.Repository
}

func NewCountLogic(ctx context.Context, svcCtx *svc.ServiceContext) *CountLogic {
    return &CountLogic{
        ctx:    ctx,
        svcCtx: svcCtx,
        Repo:   svcCtx.Repo,
    }
}

func (l *CountLogic) Count(userID int64) error {
    // 业务逻辑
    return nil
}
```

**差异**：**无差异**，只是包路径不同

### 4. Server 层

#### RPC Service MQ Server

```go
// app/services/core/user/internal/server/mq.go
package server

import (
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/mq"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/queue/rabbit"
)

type MqServer struct {
    config config.MqConfig
    svcCtx *svc.ServiceContext
    server *rabbit.RabbitMQ
}

func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
    userMQ := mq.NewUserMQ(svcCtx)

    consumers := []rabbit.Consumer{
        {
            Name:    "UserCountConsumer",
            Queue:   c.UserCountMq.Queue,
            Handler: userMQ.CountConsumer(),
        },
    }

    server := rabbit.NewRabbitMQ(c.UserCountMq.Url, c.UserCountMq.Exchange, consumers)

    return &MqServer{
        config: c,
        svcCtx: svcCtx,
        server: server,
    }
}

func (s *MqServer) Start() {
    s.server.Start()
}

func (s *MqServer) Stop() {
    s.server.Stop()
}
```

#### REST API MQ Server

```go
// app/apis/stayy/api/internal/server/mq.go
package server

import (
    "apps/app/apis/stayy/api/internal/config"
    "apps/app/apis/stayy/api/internal/mq"
    "apps/app/apis/stayy/api/internal/svc"
    "apps/pkg/queue/rabbit"
)

type MqServer struct {
    config config.MqConfig
    svcCtx *svc.ServiceContext
    server *rabbit.RabbitMQ
}

func NewMqServer(c config.MqConfig, svcCtx *svc.ServiceContext) *MqServer {
    userMQ := mq.NewUserMQ(svcCtx)

    consumers := []rabbit.Consumer{
        {
            Name:    "UserCountConsumer",
            Queue:   c.UserCount.Queue,
            Handler: userMQ.CountConsumer(),
        },
        {
            Name:    "UserUpLevelConsumer",
            Queue:   c.UserUpLevel.Queue,
            Handler: userMQ.UpLevelConsumer(),
        },
    }

    server := rabbit.NewRabbitMQ(c.UserCount.Url, c.UserCount.Exchange, consumers)

    return &MqServer{
        config: c,
        svcCtx: svcCtx,
        server: server,
    }
}

func (s *MqServer) Start() {
    s.server.Start()
}

func (s *MqServer) Stop() {
    s.server.Stop()
}
```

**差异**：**无差异**，只是包路径和 config 类型来源不同

### 5. 入口文件

#### RPC Service MQ Entry

```go
// app/services/core/user/cmd/mq/main.go
package main

import (
    "flag"
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/server"
    "apps/app/services/core/user/internal/svc"
    "apps/pkg/xlog"
    "github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/mq.yaml", "the config file")

func main() {
    flag.Parse()

    var c config.MqConfig
    conf.MustLoad(*configFile, &c)

    xlog.Init(c.Name, c.LogConf)
    svcCtx := svc.NewServiceContext(c.Config)

    mqServer := server.NewMqServer(c, svcCtx)
    mqServer.Start()
}
```

#### REST API MQ Entry

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

    // 启动 MQ Server（goroutine）
    if c.Mq.UserCount.Url != "" {
        mqServer := server.NewMqServer(c.Mq, svcCtx)
        go mqServer.Start()
    }

    // 启动 HTTP Server（主 goroutine）
    httpServer.Start()
}
```

**差异**：

- RPC Service: 独立进程，直接启动 MQ Server
- REST API: HTTP 主进程，MQ 在在 goroutine 中启动

## 关键注意事项

### 1. Goroutine 启动

```go
// ✅ 正确：MQ 必须在 goroutine 中启动
if c.UserCountMq.Url != "" {
    mqServer := server.NewMqServer(c.UserCountMq, svcCtx)
    go mqServer.Start()  // 使用 goroutine
}

// HTTP Server 为主 goroutine，直接启动
httpServer.Start()
```

### 2. 共享 Repository

MQ 与 HTTP 共享同一个 Repository 实例，确保数据库连接池、Redis 连接等资源被正确复用。

```go
// ServiceContext 初始化时创建的 Repository
svcCtx := svc.NewServiceContext(c.Config)

// HTTP、MQ 都使用同一个 svcCtx
// 所有资源（数据库、Redis、RPC 客户端）自动共享
```

### 3. 配置启用控制

```go
// 通过检查 MQ 配置的 Url 字段是否为空来决定是否启动
if c.Mq.UserCount.Url != "" {
    mqServer := server.NewMqServer(c.Mq, svcCtx)
    go mqServer.Start()
}
```

## 相关示例

| 文件                                                           | 说明                                        |
| -------------------------------------------------------------- | ------------------------------------------- |
| [main.md](../example/rest_api/mq/main.md)                      | REST API 中嵌入 MQ 的 main.go 示例          |
| [config.md](../example/rest_api/mq/config.md)                  | REST API 中 MQ 配置示例                     |
| [server.md](../example/rest_api/mq/server.md)                  | REST API 中 MQ Server 示例                  |
| [proto_message.md](../example/rpc_service/mq/proto_message.md) | MQ Protobuf 消息定义                        |
| [handler.md](../example/rpc_service/mq/handler.md)             | MQ Handler 示例（与 RPC Service 共享）      |
| [logic.md](../example/rpc_service/mq/logic.md)                 | MQ Logic 示例（与 RPC Service 共享）        |
| [repository.md](../example/rpc_service/mq/repository.md)       | MQ Repository 层示例（与 RPC Service 共享） |
