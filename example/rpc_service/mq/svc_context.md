# MQ ServiceContext 初始化示例

MQ 服务的 ServiceContext 与 gRPC 服务共享，负责初始化和注入所有依赖。

> **注意**：MQ 和 gRPC 共享同一个 ServiceContext，不需要单独创建。

## 文件位置

`internal/svc/service_context.go`

## 代码示例

```go
package svc

import (
	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/repository"
)

// ServiceContext 服务上下文
type ServiceContext struct {
	Config config.Config
	Repo   *repository.Repository
}

// NewServiceContext 创建服务上下文
func NewServiceContext(c config.Config) *ServiceContext {
	return &ServiceContext{
		Config: c,
		Repo:   repository.NewRepository(c),
	}
}
```

## 说明

| 字段 | 说明 |
|-----|------|
| `Config` | 服务配置，从配置文件加载 |
| `Repo` | Repository 依赖，包含 DB Model、RPC 客户端等 |

## 依赖注入流程

```
config.MqConfig.Config
       │
       ▼
repository.NewRepository(c)
       │
       ▼
repository.Repository
       │
       ▼
svc.ServiceContext
       │
       ▼
mq.NewUserMQ(svcCtx)
```

## MQ Handler 使用

```go
// internal/mq/user_mq.go

package mq

import (
	"apps/app/services/core/user/internal/svc"
	"apps/app/services/core/user/internal/logic/mq"
)

type UserMQ struct {
	svcCtx *svc.ServiceContext
}

func NewUserMQ(svcCtx *svc.ServiceContext) *UserMQ {
	return &UserMQ{
		svcCtx: svcCtx,
	}
}

func (s *UserMQ) CountConsumer() queue.ConsumeHandlerFunc {
	return func(ctx context.Context, key string, payload []byte) error {
		// 从 svcCtx.Repo 获取依赖
		countLogic := logic.NewCountLogic(ctx, s.svcCtx)
		return countLogic.Count(userID)
	}
}
```
