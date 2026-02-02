# ServiceContext 初始化示例

ServiceContext 是服务的全局上下文，负责初始化和注入所有依赖。

## 文件位置

`internal/svc/svc_context.go`

## 代码示例

```go
package svc

import (
	"apps/app/apis/stayy/api/internal/config"
	"apps/app/apis/stayy/api/internal/repository"
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
| `Repo` | Repository 依赖，包含所有 Logic 需要的依赖 |

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
logic.NewXxxLogic(svcCtx)
```
