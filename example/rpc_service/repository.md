# Repository 层示例

RPC 服务的 Repository 层管理 Logic/Service 层的所有依赖，包括 DB Model、RPC 客户端、缓存操作等。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
├── model/                 # 数据库 Model
│   ├── xxx_model_gen.go   # 自动生成（勿编辑）
│   └── xxx_model.go       # 扩展方法
├── rpc/                   # RPC 客户端
└── xredis/                # Redis 操作
```

## repository.go

定义 Repository 结构并初始化所有依赖：

```go
package repository

import (
	"apps/app/services/core/user/internal/config"
	"apps/app/services/core/user/internal/repository/model"
	"apps/app/services/core/user/internal/repository/rpc"
	"apps/pkg/store/mysql/gorm"
)

// Repository 依赖容器
type Repository struct {
	UserModel          UserModelType
	UserThirdAuthModel UserThirdAuthModelType
	AuthzSrv           AuthzSrvType
}

// NewRepository 创建 Repository
func NewRepository(c config.Config) *Repository {
	mysqlDB := gorm.MustNewGormDB(c.DB.Mysql)

	return &Repository{
		UserModel:          model.NewUserModel(mysqlDB, c.DBCache),          // 用户信息
		UserThirdAuthModel: model.NewUserThirdAuthModel(mysqlDB, c.DBCache), // 用户三方授权信息
		AuthzSrv:           rpc.NewAuthzSrv(c.RPC.AuthzSrvConf),             // 权限服务
	}
}
```

## repository_type.go

定义类型别名，供 Logic/Service 层使用：

```go
package repository

import (
	"apps/app/services/core/user/internal/repository/model"
	"apps/app/services/core/user/internal/repository/rpc"
)

// 类型别名，方便 Logic 层使用
type (
	UserModelType          = model.UserModel
	UserThirdAuthModelType = model.UserThirdAuthModel
	AuthzSrvType           = rpc.AuthzSrv
)
```

## model/ - 数据库 Model

### 文件说明

| 文件 | 说明 | 是否可编辑 |
|-----|------|-----------|
| `xxx_model_gen.go` | 自动生成，包含增删改和基础查询方法 | ❌ 勿编辑 |
| `xxx_model.go` | 手动扩展，添加自定义查询方法 | ✅ 可编辑 |

### user_model.go - 扩展方法示例

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
package model

import (
	"time"

	"github.com/zeromicro/go-zero/core/stores/cache"
	"gorm.io/gorm"
)

var _ UserModel = (*customUserModel)(nil)

type (
	// UserModel 用户模型接口
	UserModel interface {
		userModel
		// 此处定义扩展方法接口
		FindByMobile(ctx context.Context, mobile string) ([]*User, error)
	}

	customUserModel struct {
		*defaultUserModel
	}
)

// BeforeCreate 创建前钩子
func (u *User) BeforeCreate(db *gorm.DB) error {
	now := time.Now().Unix()
	u.CreateTime = now
	u.UpdateTime = now
	return nil
}

// BeforeUpdate 更新前钩子
func (u *User) BeforeUpdate(db *gorm.DB) error {
	u.UpdateTime = time.Now().Unix()
	return nil
}

// NewUserModel 创建用户模型
func NewUserModel(conn *gorm.DB, c cache.CacheConf) UserModel {
	return &customUserModel{
		defaultUserModel: newUserModel(conn, c),
	}
}

// FindByMobile 根据手机号查询用户列表（扩展方法示例）
func (m *customUserModel) FindByMobile(ctx context.Context, mobile string) ([]*User, error) {
	var users []*User
	err := m.DB.WithContext(ctx).Where("`mobile` = ?", mobile).Find(&users).Error
	return users, err
}
```

## rpc/ - RPC 客户端

### auth_srv.go

```go
package rpc

import (
	authzV1Pb "apps/pb/services/shared/authz/v1"

	"github.com/zeromicro/go-zero/zrpc"
)

//go:generate mockgen -package rpc -destination mock_authz_service.go apps/pb/services/shared/authz/v1 AuthzServiceClient

type AuthzSrv struct {
	AuthzServicev1Client authzV1Pb.AuthzServiceClient
}

// NewAuthzSrv 创建鉴权服务客户端
func NewAuthzSrv(authzSrvConf zrpc.RpcClientConf) *AuthzSrv {
	return &AuthzSrv{
		AuthzServicev1Client: authzV1Pb.NewAuthzServiceClient(zrpc.MustNewClient(authzSrvConf).Conn()),
	}
}
```

## xredis/ - Redis 操作

### token_model.go

```go
package xredis

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/spf13/cast"
	"github.com/zeromicro/go-zero/core/stores/redis"
)

const tokenKey = "token:%d"

// TokenModel Token 管理接口
//
//go:generate mockgen -package xredis -destination mock_token_model.go -source token_model.go
type TokenModel interface {
	GenToken(ctx context.Context, userID int64, token string, expireSeconds int) error
	DelToken(ctx context.Context, userID int64, expireSeconds int) (bool, error)
	GetToken(ctx context.Context, userID int64) (string, error)
}

type tokenModel struct {
	client *redis.Redis
	prefix string
}

// NewTokenModel 创建 Token 模型
func NewTokenModel(client *redis.Redis, prefix string) TokenModel {
	return &tokenModel{client: client, prefix: prefix}
}

func (m *tokenModel) formatKey(key string) string {
	return m.prefix + ":" + key
}

func (m *tokenModel) GenToken(ctx context.Context, userID int64, token string, expireSeconds int) error {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	return m.client.SetexCtx(ctx, key, token, expireSeconds)
}

func (m *tokenModel) DelToken(ctx context.Context, userID int64, expireSeconds int) (bool, error) {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	return m.client.SetnxExCtx(ctx, key, cast.ToString(time.Now().Unix()), expireSeconds)
}

func (m *tokenModel) GetToken(ctx context.Context, userID int64) (string, error) {
	key := m.formatKey(fmt.Sprintf(tokenKey, userID))
	token, err := m.client.GetCtx(ctx, key)
	if err != nil && !errors.Is(err, redis.Nil) {
		return "", err
	}
	return token, nil
}
```

## Mock 生成

每个接口文件都需要添加 `//go:generate` 指令：

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
```

执行 `make mock` 生成 Mock 文件，用于单元测试。
