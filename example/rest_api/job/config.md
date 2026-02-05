# REST API 中 Job 配置示例

展示如何在 REST API 服务中配置 Job 定时任务。

## 文件位置

`internal/config/config.go`

## Config 结构定义

```go
package config

import (
    "github.com/zeromicro/go-zero/rest"
)

// ApiConfig 包含 API 服务配置
type ApiConfig struct {
    rest.RestConf
    Config Config

    // Job 配置（嵌入在 ApiConfig 中）
    JobConfig JobConfig
}

// Config 包含业务配置
type Config struct {
    DB      DBConf
    DBCache cache.CacheConf
    RPC     RPCConf
}

// JobConfig 包含 Job 定时任务配置
type JobConfig struct {
    Enable bool          // 是否启用 Job
    Name   string        // Job 名称（用于日志和监控）
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

JobConfig:
  Enable: true
  Name: user-api
```

## 配置字段说明

### JobConfig

| 字段 | 类型 | 说明 | 示例 |
|-----|------|------|------|
| `Enable` | `bool` | 是否启用 Job Server | `true` / `false` |
| `Name` | `string` | Job 服务名称（用于日志和监控） | `"user-api"` |

### 启用条件

```go
// 在 main.go 中检查
if c.JobConfig.Enable {
    jobServer := server.NewJobServer(c.JobConfig, svcCtx)
    go jobServer.Start()
}
```

## 不启用 Job 的配置

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
  Enable: false      # 不启用 Job
  Name: user-api
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

## 与 RPC Service Job 配置的区别

### RPC Service Job Config

```go
// app/services/core/user/internal/config/config.go
type JobConfig struct {
    service.ServiceConf  // go-zero 基础服务配置
    Config Config        // 业务配置
}
```

**配置文件**：`cmd/job/conf.yaml`

```yaml
Name: user-job
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "..."
```

### REST API Job Config

```go
// app/apis/stayy/api/internal/config/config.go
type ApiConfig struct {
    rest.RestConf
    Config Config
    JobConfig JobConfig  // 嵌入 Job 配置
}

type JobConfig struct {
    Enable bool
    Name   string
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

JobConfig:
  Enable: true
  Name: user-api
```

## 关键点说明

1. **配置嵌入**：REST API 的 Job 配置嵌入在 `ApiConfig` 中，与 HTTP 配置在同一文件

2. **启用控制**：通过 `JobConfig.Enable` 字段控制是否启动 Job Server

3. **共享配置**：HTTP 和 Job 共享 `Config` 中的业务配置（数据库、Redis、RPC 客户端等）

4. **不创建独立配置文件**：REST API 不创建 `cmd/job/conf.yaml`，所有配置都在 `etc/conf.yaml` 中

5. **ServiceContext 共享**：HTTP 和 Job 使用同一个 `ServiceContext` 实例

## 相关示例

| 文件 | 说明 |
|-----|------|
| [config.md](../rpc_service/job/config.md) | RPC Service Job 配置（完整版） |
