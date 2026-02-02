# ServiceContext 初始化示例

RPC 服务的 ServiceContext 与 REST API 类似，负责初始化和注入所有依赖。

## 文件位置

`internal/svc/svc_context.go`

## 代码示例

```go
package svc

import (
	"apps/app/services/shared/authz/internal/config"
	"apps/app/services/shared/authz/internal/repository"
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
config.Config
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
service.NewXxxService(svcCtx)
```
