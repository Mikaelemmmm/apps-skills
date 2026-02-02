# Repository 层示例

Repository 层负责管理 Logic 层的所有依赖，包括 RPC 客户端、三方服务适配器、缓存操作等。

## 目录结构

```
internal/repository/
├── repository.go          # Repository 结构定义和初始化
├── repository_type.go     # 类型别名定义
├── adapter/               # 三方服务适配器
│   └── thirdauth/         # 三方授权适配器
├── rpc/                   # RPC 客户端
└── xredis/                # Redis 操作
```

## repository.go

定义 Repository 结构并初始化所有依赖：

```go
package repository

import (
	"apps/app/apis/stayy/api/internal/config"
	"apps/app/apis/stayy/api/internal/repository/adapter/thirdauth"
	"apps/app/apis/stayy/api/internal/repository/rpc"
)

// Repository 依赖容器
type Repository struct {
	AuthSrv          *AuthzSrvType
	UserSrv          *UserSrvType
	WXMiniAuthClient *ThirdAuthManager
}

// NewRepository 创建 Repository
func NewRepository(c config.Config) *Repository {
	return &Repository{
		AuthSrv:          rpc.NewAuthzSrv(c.RPC.AuthzSrvConf), // 鉴权服务
		UserSrv:          rpc.NewUserSrv(c.RPC.UserSrvConf),   // 用户服务
		WXMiniAuthClient: thirdauth.NewManager(c),             // 三方授权管理
	}
}
```

## repository_type.go

定义类型别名，供 Logic 层使用：

```go
package repository

import (
	"apps/app/apis/stayy/api/internal/repository/adapter/thirdauth"
	"apps/app/apis/stayy/api/internal/repository/rpc"
)

// 类型别名，方便 Logic 层使用
type (
	AuthzSrvType     = rpc.AuthzSrv
	UserSrvType      = rpc.UserSrv
	ThirdAuthManager = thirdauth.Manager
)
```

## adapter/thirdauth/ - 三方服务适配器

### client.go - 客户端接口定义

```go
package thirdauth

import (
	"context"
	"apps/app/apis/stayy/api/internal/config"
	"apps/pkg/xerr"
)

// ThirdUserData 三方用户数据统一结构
type ThirdUserData struct {
	ThirdKey      string
	ThirdUnionKey string
	NickName      string
	Mobile        string
	Avatar        string
}

// Client 三方授权客户端接口
//
//go:generate mockgen -package thirdauth -destination mock_client.go -source client.go
type Client interface {
	ParseUserData(ctx context.Context, opts ...thirdAuthOption) (*ThirdUserData, error)
}

// Manager 管理所有三方授权客户端
type Manager struct {
	c config.Config
}

// NewManager 创建 Manager
func NewManager(c config.Config) *Manager {
	return &Manager{c: c}
}

// GetClient 获取指定名称的三方授权客户端
func (m *Manager) GetClient(name string) (Client, error) {
	if factory, ok := clientFactory[name]; ok {
		return factory(m.c), nil
	}
	return nil, xerr.NewMsgWithErrorLog(xerr.SystemError_S_ERROR,
		"third auth client not found", "client: %s", name)
}
```

### wx_mini.go - 微信小程序实现

```go
package thirdauth

import (
	"context"
	"fmt"

	"apps/app/apis/stayy/api/internal/config"

	"github.com/silenceper/wechat/v2"
	"github.com/silenceper/wechat/v2/cache"
	"github.com/silenceper/wechat/v2/miniprogram"
	miniConfig "github.com/silenceper/wechat/v2/miniprogram/config"
)

const ThirdAuthClientWXMini = "wxMini"

type wxMini struct {
	wxMiniProgram *miniprogram.MiniProgram
}

func init() {
	// 注册微信小程序客户端
	registerClient(ThirdAuthClientWXMini, newWXMini)
}

func newWXMini(c config.Config) Client {
	return &wxMini{
		wxMiniProgram: wechat.NewWechat().GetMiniProgram(&miniConfig.Config{
			AppID:     c.WXMiniConfig.AppID,
			AppSecret: c.WXMiniConfig.Secret,
			Cache:     cache.NewMemory(),
		}),
	}
}

func (l *wxMini) ParseUserData(ctx context.Context, opts ...thirdAuthOption) (*ThirdUserData, error) {
	option := &thirdAuthOptions{}
	for _, o := range opts {
		o(option)
	}

	// 1. 微信小程序登录
	authResult, err := l.wxMiniProgram.GetAuth().Code2Session(option.Code)
	if err != nil || authResult.ErrCode != 0 || authResult.OpenID == "" {
		return nil, fmt.Errorf("wxMini Code2Session failed: err=%v, result=%+v", err, authResult)
	}

	// 2. 解密用户数据
	userData, err := l.wxMiniProgram.GetEncryptor().Decrypt(authResult.SessionKey, option.EncryptedData, option.IV)
	if err != nil {
		return nil, fmt.Errorf("wxMini decrypt failed: err=%v", err)
	}

	return &ThirdUserData{
		ThirdKey:      userData.OpenID,
		ThirdUnionKey: userData.UnionID,
		NickName:      userData.NickName,
		Mobile:        userData.PhoneNumber,
		Avatar:        userData.AvatarURL,
	}, nil
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
//go:generate mockgen -package rpc -destination mock_xxx.go -source xxx.go
```

执行 `make mock` 生成 Mock 文件，用于单元测试。
