# Logic 层业务逻辑示例

Logic 层是 REST API 服务的业务逻辑层，负责编排 RPC 调用和处理业务流程。

## 文件位置

`internal/logic/apiv1/{service}_logic.go`

## 代码示例

```go
package apiv1

import (
	"context"

	"apps/app/apis/stayy/api/internal/repository"
	"apps/app/apis/stayy/api/internal/repository/adapter/thirdauth"
	"apps/app/apis/stayy/api/internal/svc"
	"apps/pb/apis/stayy/api/errors"
	pb "apps/pb/apis/stayy/api/v1"
	userV1Pb "apps/pb/services/core/user/v1"
	"apps/pkg/xerr"
)

// UserLogic 用户逻辑
type UserLogic struct {
	svcCtx           *svc.ServiceContext
	thirdAuthManager *repository.ThirdAuthManager
	userSrv          *repository.UserSrvType
	tokenRdsModel    repository.TokenRdsModelType
}

// NewUserLogic 创建用户逻辑
func NewUserLogic(svcCtx *svc.ServiceContext) *UserLogic {
	return &UserLogic{
		svcCtx:           svcCtx,
		thirdAuthManager: svcCtx.Repo.WXMiniAuthClient,
		userSrv:          svcCtx.Repo.UserSrv,
		tokenRdsModel:    svcCtx.Repo.TokenRdsModel,
	}
}

// WXMiniAuth 微信小程序授权登录
func (l *UserLogic) WXMiniAuth(ctx context.Context, req *pb.WXMiniAuthReq) (*pb.ThirdAuthReply, error) {
	// 1. 获取三方授权客户端并解析用户数据
	wxMiniAuthClient, err := l.thirdAuthManager.GetClient(thirdauth.ThirdAuthClientWXMini)
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(xerr.SystemError_S_ERROR,
			"get wxmini auth client failed",
			"err: %v, req: %+v", err, req)
	}

	userData, err := wxMiniAuthClient.ParseUserData(ctx,
		thirdauth.WithCode(req.Code),
		thirdauth.WithIV(req.Iv),
		thirdauth.WithEncryptedData(req.EncryptedData))
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(errors.UserError_ERROR_PREFIX_APIS_STAYY_API,
			"parse user data failed",
			"err: %v, req: %+v", err, req)
	}

	// 2. 调用用户服务进行登录或注册
	resp, err := l.userSrv.UserServiceV1Client.Register(ctx, &userV1Pb.RegisterReq{
		UserAuthType: userV1Pb.UserAuthType_USER_AUTH_TYPE_THIRD,
		RegisterInfo: &userV1Pb.RegisterReq_ThirdRegisterReq{
			ThirdRegisterReq: &userV1Pb.ThirdRegisterReq{
				ThirdType:     userV1Pb.ThirdAuthType_THIRD_AUTH_TYPE_WX_MINI,
				ThirdUnionKey: userData.ThirdUnionKey,
				ThirdKey:      userData.ThirdKey,
				Mobile:        userData.Mobile,
				Avatar:        userData.Avatar,
				NickName:      userData.NickName,
			},
		},
	})
	if err != nil {
		return nil, err
	}

	// 3. 返回响应
	return &pb.ThirdAuthReply{
		AccessToken:   resp.GetToken().GetAccessToken(),
		AccessExpire:  resp.GetToken().AccessExpire,
		RefreshExpire: resp.GetToken().GetRefreshExpire(),
	}, nil
}
```

## Redis 操作示例

```go
// VerifyToken 验证 Token
func (l *UserLogic) VerifyToken(ctx context.Context, req *pb.VerifyTokenReq) (*pb.VerifyTokenReply, error) {
	// 使用 TokenRdsModel 查询 Token
	token, err := l.tokenRdsModel.GetToken(ctx, req.UserId)
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_S_ERROR,
			"get token failed",
			"user_id: %d, err: %v", req.UserId, err,
		)
	}

	if token != req.Token {
		return nil, xerr.NewMsgWithErrorLog(
			errors.UserError_ERROR_PREFIX_APIS_STAYY_API,
			"token mismatch",
			"user_id: %d", req.UserId,
		)
	}

	return &pb.VerifyTokenReply{Valid: true}, nil
}

// RevokeToken 撤销 Token
func (l *UserLogic) RevokeToken(ctx context.Context, req *pb.RevokeTokenReq) (*pb.RevokeTokenReply, error) {
	// 使用 TokenRdsModel 删除 Token（加入黑名单）
	_, err := l.tokenRdsModel.DelToken(ctx, req.UserId, 86400) // 黑名单有效期 24 小时
	if err != nil {
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_S_ERROR,
			"revoke token failed",
			"user_id: %d, err: %v", req.UserId, err,
		)
	}

	return &pb.RevokeTokenReply{Success: true}, nil
}
```

## 代码结构

### 1. Logic 结构体

```go
type XxxLogic struct {
	svcCtx *svc.ServiceContext  // 服务上下文
	// 从 svcCtx.Repo 中提取需要的依赖
	dep1   *repository.Dep1Type
	dep2   *repository.Dep2Type
}
```

### 2. 构造函数

```go
func NewXxxLogic(svcCtx *svc.ServiceContext) *XxxLogic {
	return &XxxLogic{
		svcCtx: svcCtx,
		// 提取依赖，方便后续使用
		dep1:   svcCtx.Repo.Dep1,
		dep2:   svcCtx.Repo.Dep2,
	}
}
```

### 3. 业务方法

```go
func (l *XxxLogic) MethodName(ctx context.Context, req *pb.Request) (*pb.Reply, error) {
	// 1. 参数校验（如有必要）
	// 2. 调用 RPC 服务
	// 3. 处理响应
	// 4. 返回结果
}
```

## 错误处理

### 业务错误

使用业务定义的错误码：

```go
return nil, xerr.NewMsgWithErrorLog(
	errors.UserError_ERROR_PREFIX_APIS_STAYY_API,
	"parse user data failed",
	"err: %v, req: %+v", err, req)
```

### 系统错误

使用系统级错误码：

```go
return nil, xerr.NewMsgWithErrorLog(
	xerr.SystemError_S_ERROR,
	"get wxmini auth client failed",
	"err: %v, req: %+v", err, req)
```

## 依赖注入

| 来源 | 说明 |
|-----|------|
| `svcCtx.Config` | 服务配置 |
| `svcCtx.Repo.Xxx` | 从 Repository 获取的依赖 |

## 单元测试

```go
func TestUserLogic_WXMiniAuth(t *testing.T) {
	// 创建 Mock 依赖
	mockRepo := &repository.Repository{
		WXMiniAuthClient: &mockThirdAuthClient{},
		UserSrv:         &mockUserSrv{},
		TokenRdsModel:    &mock.MockTokenModel{},
	}

	svcCtx := &svc.ServiceContext{
		Repo: mockRepo,
	}

	logic := NewUserLogic(svcCtx)

	// 测试...
}
```
