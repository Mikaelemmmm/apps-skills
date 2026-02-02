# 错误码使用示例

## 业务错误

使用业务定义的错误码，返回自定义消息给前端：

```go
package apiv1

import (
	"context"

	"apps/app/apis/stayy/api/internal/repository/adapter/thirdauth"
	"apps/pb/apis/stayy/api/errors"
	"apps/pkg/xerr"
)

func (l *UserLogic) WXMiniAuth(ctx context.Context, req *pb.WXMiniAuthReq) (*pb.ThirdAuthReply, error) {
	// 获取三方授权客户端
	wxMiniAuthClient, err := l.thirdAuthManager.GetClient(thirdauth.ThirdAuthClientWXMini)
	if err != nil {
		// 系统错误：获取客户端失败
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_S_ERROR,
			"get wxmini auth client failed",
			"err: %v, req: %+v", err, req,
		)
	}

	// 解析三方用户数据
	userData, err := wxMiniAuthClient.ParseUserData(ctx,
		thirdauth.WithCode(req.Code),
		thirdauth.WithIV(req.Iv),
		thirdauth.WithEncryptedData(req.EncryptedData),
	)
	if err != nil {
		// 业务错误：三方授权失败
		return nil, xerr.NewMsgWithErrorLog(
			errors.UserError_ERROR_PREFIX_APIS_STAYY_API,
			"parse user data failed",
			"err: %v, req: %+v", err, req,
		)
	}

	// ...
}
```

## 系统错误

使用系统级错误码，通常发生在中间件或底层组件：

```go
package apiv1

import (
	"context"

	"apps/pkg/xerr"
)

func (l *OrderLogic) CreateOrder(ctx context.Context, req *pb.CreateOrderReq) (*pb.CreateOrderReply, error) {
	// 数据库操作
	order, err := l.orderModel.FindOne(ctx, req.UserId)
	if err != nil {
		// 系统错误：数据库查询失败
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_D_ERROR,
			"database query failed",
			"table: order, user_id: %d, err: %v", req.UserId, err,
		)
	}

	// Redis 操作
	token, err := l.tokenModel.GetToken(ctx, req.UserId)
	if err != nil {
		// 系统错误：Redis 操作失败
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_R_ERROR,
			"redis get token failed",
			"user_id: %d, err: %v", req.UserId, err,
		)
	}

	// 参数校验
	if req.Amount <= 0 {
		// 参数错误
		return nil, xerr.NewMsgWithErrorLog(
			xerr.SystemError_PARAMS_ERROR,
			"invalid amount",
			"amount: %d", req.Amount,
		)
	}

	// ...
}
```

## 错误码方法对照表

| 方法 | 错误码 | 消息 | 日志 |
|-----|-------|------|------|
| `xerr.New` | ✅ | 默认 | ❌ |
| `xerr.NewWithErrorLog` | ✅ | 默认 | ✅ Error |
| `xerr.NewWithInfoLog` | ✅ | 默认 | ✅ Info |
| `xerr.NewMsg` | ✅ | 自定义 | ❌ |
| `xerr.NewMsgWithErrorLog` | ✅ | 自定义 | ✅ Error |
| `xerr.NewMsgWithInfoLog` | ✅ | 自定义 | ✅ Info |

## 参数说明

```go
// NewMsgWithErrorLog 参数
func NewMsgWithErrorLog(
    err Error,              // 错误码枚举
    msg string,             // 返回给前端的错误消息
    logFormat string,       // 日志格式化字符串
    logParams ...any,       // 日志格式化参数
) error
```

## 常见错误码

### 系统级错误码

```go
import "apps/pkg/xerr"

// 系统内部错误
xerr.SystemError_S_ERROR

// 数据库错误
xerr.SystemError_D_ERROR

// Redis 错误
xerr.SystemError_R_ERROR

// 事件错误
xerr.SystemError_E_ERROR

// 三方错误
xerr.SystemError_T_ERROR

// 参数错误
xerr.SystemError_PARAMS_ERROR
```

### 业务错误码

```go
import (
	"apps/pb/apis/stayy/api/errors"
	userV1Errors "apps/pb/services/core/user/errors"
)

// API 服务错误码
errors.UserError_ERROR_PREFIX_APIS_STAYY_API

// RPC 服务错误码
userV1Errors.UserError_ERROR_PREFIX_SERVICES_CORE_USER
```
