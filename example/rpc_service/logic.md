# Logic/Service 层业务逻辑示例

RPC 服务的业务逻辑层通常命名为 `service`，包含具体的业务实现。

## 文件位置

`internal/service/{version}/{service}_service.go`

## 代码示例

```go
package authzservice

import (
	"context"
	"time"

	"apps/app/services/shared/authz/internal/svc"
	pb "apps/pb/services/shared/authz/v1"
	"apps/pkg/xcontext"
	"apps/pkg/xerr"

	"github.com/golang-jwt/jwt"
)

const (
	// TokenBufferTimeSeconds token 向前兼容时间，防止服务器时间不同步
	TokenBufferTimeSeconds = 5
	// JwtKey jwt 信息 key
	JwtKey = "seafun_client_jwt"
)

// GenTokenService 生成 Token 服务
type GenTokenService struct {
	ctx          context.Context
	svcCtx       *svc.ServiceContext
	tokenRdsModel repository.TokenRdsModelType
}

// NewGenTokenService 创建生成 Token 服务
func NewGenTokenService(ctx context.Context, svcCtx *svc.ServiceContext) *GenTokenService {
	return &GenTokenService{
		ctx:          ctx,
		svcCtx:       svcCtx,
		tokenRdsModel: svcCtx.Repo.TokenRdsModel,
	}
}

// GenToken 生成 Token
func (s *GenTokenService) GenToken(req *pb.GenTokenReq) (*pb.GenTokenReply, error) {
	tokenStartTime := time.Now().Unix() - TokenBufferTimeSeconds // 防止多台机器时间不同步
	accessExpire := s.svcCtx.Config.Jwt.AccessExpire

	// 生成 JWT token
	accessToken, err := s.getJwtToken(tokenStartTime, req)
	if err != nil {
		return nil, err
	}

	// 保存 Token 到 Redis
	if err := s.tokenRdsModel.GenToken(s.ctx, req.Uid, accessToken, int(accessExpire)); err != nil {
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_D_ERROR,
			"save token to redis failed",
			"user_id: %d, err: %v", req.Uid, err,
		)
	}

	return &pb.GenTokenReply{
		AccessToken:   accessToken,
		AccessExpire:  tokenStartTime + int64(accessExpire),
		RefreshExpire: tokenStartTime + int64(accessExpire/2),
	}, nil
}

// getJwtToken 生成 JWT Token
func (s *GenTokenService) getJwtToken(startTime int64, req *pb.GenTokenReq) (string, error) {
	expire := startTime + int64(s.svcCtx.Config.Jwt.AccessExpire)

	claims := make(jwt.MapClaims)
	claims["exp"] = expire
	claims["iat"] = startTime
	claims[JwtKey] = xcontext.UserJWT{
		UserID: req.GetUid(),
		Expire: expire,
	}

	jwtToken := jwt.New(jwt.SigningMethodHS256)
	jwtToken.Claims = claims

	result, err := jwtToken.SignedString([]byte(s.svcCtx.Config.Jwt.Secret))
	if err != nil {
		return "", xerr.NewMsgWithErrorLog(
			xerr.SystemError_T_ERROR,
			"gen jwt token failed",
			"golang-jwt gen token failed, req:%+v, err:%s",
			req, err.Error(),
		)
	}

	return result, nil
}
```

## Redis 操作示例

```go
// VerifyTokenService 验证 Token 服务
type VerifyTokenService struct {
	ctx          context.Context
	svcCtx       *svc.ServiceContext
	tokenRdsModel repository.TokenRdsModelType
}

// NewVerifyTokenService 创建验证 Token 服务
func NewVerifyTokenService(ctx context.Context, svcCtx *svc.ServiceContext) *VerifyTokenService {
	return &VerifyTokenService{
		ctx:          ctx,
		svcCtx:       svcCtx,
		tokenRdsModel: svcCtx.Repo.TokenRdsModel,
	}
}

// VerifyToken 验证 Token
func (s *VerifyTokenService) VerifyToken(req *pb.VerifyTokenReq) (*pb.VerifyTokenReply, error) {
	// 使用 TokenRdsModel 查询 Token
	token, err := s.tokenRdsModel.GetToken(s.ctx, req.Uid)
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_D_ERROR,
			"get token failed",
			"user_id: %d, err: %v", req.Uid, err,
		)
	}

	if token != req.Token {
		return nil, xerr.NewMsgWithErrorLog(
			errors.AuthorizationError_ERROR_PREFIX_SERVICES_AUTHZ_AUTHORIZATION,
			"token mismatch",
			"user_id: %d", req.Uid,
		)
	}

	return &pb.VerifyTokenReply{Valid: true}, nil
}

// RevokeTokenService 撤销 Token 服务
type RevokeTokenService struct {
	ctx          context.Context
	svcCtx       *svc.ServiceContext
	tokenRdsModel repository.TokenRdsModelType
}

// NewRevokeTokenService 创建撤销 Token 服务
func NewRevokeTokenService(ctx context.Context, svcCtx *svc.ServiceContext) *RevokeTokenService {
	return &RevokeTokenService{
		ctx:          ctx,
		svcCtx:       svcCtx,
		tokenRdsModel: svcCtx.Repo.TokenRdsModel,
	}
}

// RevokeToken 撤销 Token
func (s *RevokeTokenService) RevokeToken(req *pb.RevokeTokenReq) (*pb.RevokeTokenReply, error) {
	// 使用 TokenRdsModel 删除 Token（加入黑名单）
	_, err := s.tokenRdsModel.DelToken(s.ctx, req.Uid, 86400) // 黑名单有效期 24 小时
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_D_ERROR,
			"revoke token failed",
			"user_id: %d, err: %v", req.Uid, err,
		)
	}

	return &pb.RevokeTokenReply{Success: true}, nil
}
```

## 代码结构

### 1. Service 结构体

```go
type XxxService struct {
	ctx    context.Context  // 请求上下文
	svcCtx *svc.ServiceContext  // 服务上下文
	// 从 svcCtx.Repo 中提取需要的依赖
	dep1   *repository.Dep1Type
	dep2   *repository.Dep2Type
}
```

### 2. 构造函数

```go
func NewXxxService(ctx context.Context, svcCtx *svc.ServiceContext) *XxxService {
	return &XxxService{
		ctx:    ctx,
		svcCtx: svcCtx,
		// 提取依赖，方便后续使用
		dep1:   svcCtx.Repo.Dep1,
		dep2:   svcCtx.Repo.Dep2,
	}
}
```

### 3. 业务方法

```go
func (s *XxxService) MethodName(req *pb.Request) (*pb.Reply, error) {
	// 1. 参数校验（如有必要）
	// 2. 业务逻辑处理
	// 3. 数据库操作（通过 Model）
	// 4. 返回结果
}
```

## 与 REST API Logic 的区别

| 特性 | REST API Logic | RPC Service |
|-----|---------------|-------------|
| **命名** | `XxxLogic` | `XxxService` |
| **上下文** | 需要显式传入 ctx | 构造时已包含 ctx |
| **依赖注入** | `svcCtx` | `svcCtx` |
| **职责** | 业务编排，调用 RPC | 原子服务，操作数据库 |

## 错误处理

### 业务错误

使用业务定义的错误码：

```go
return nil, xerr.NewMsgWithErrorLog(
	errors.UserError_USER_NOT_FOUND,
	"user not found",
	"user_id: %d", req.UserId,
)
```

### 系统错误

使用系统级错误码：

```go
return nil, xerr.NewMsgWithErrorLog(
	xerr.SystemError_D_ERROR,
	"database query failed",
	"table: user, err: %v", err,
)
```

## 数据库操作

```go
// 通过 Model 操作数据库
user, err := s.userModel.FindOne(ctx, req.UserId)
if err != nil {
	return nil, xerr.NewMsgWithErrorLog(
		xerr.SystemError_D_ERROR,
		"find user failed",
		"user_id: %d, err: %v", req.UserId, err,
	)
}

// 创建数据
err = s.userModel.Insert(ctx, &model.User{
	Account: req.Account,
	Mobile:  req.Mobile,
})
```

## 单元测试

```go
func TestGenTokenService_GenToken(t *testing.T) {
	// 创建 Mock 依赖
	mockSvcCtx := &svc.ServiceContext{
		Config: config.Config{
			Jwt: config.JwtConfig{
				AccessExpire: 7200,
				Secret:       "test-secret",
			},
		},
	}

	ctx := context.Background()
	service := NewGenTokenService(ctx, mockSvcCtx)

	// 测试...
	req := &pb.GenTokenReq{
		Uid:  12345,
		Role: "admin",
	}

	resp, err := service.GenToken(req)
	assert.NoError(t, err)
	assert.NotEmpty(t, resp.AccessToken)
}
```
