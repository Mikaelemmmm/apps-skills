# 日志使用示例

## 基本用法

```go
package apiv1

import (
	"context"

	"apps/pkg/xlog"
)

func (l *UserLogic) GetUser(ctx context.Context, req *pb.GetUserReq) (*pb.GetUserReply, error) {
	// 记录请求开始
	xlog.Infof(ctx, "get user start, user_id: %d", req.UserId)

	// 查询用户
	user, err := l.userModel.FindOne(ctx, req.UserId)
	if err != nil {
		// 记录错误
		xlog.Errorf(ctx, "find user failed, user_id: %d, err: %v", req.UserId, err)
		return nil, err
	}

	// 记录调试信息
	xlog.Debugf(ctx, "user found: %+v", user)

	// 记录成功
	xlog.Infof(ctx, "get user success, user_id: %d", req.UserId)

	return &pb.GetUserReply{User: user}, nil
}
```

## 日志级别

### Info - 正常业务流程

```go
// 用户登录成功
xlog.Infof(ctx, "user login success, user_id: %d, ip: %s", userID, clientIP)

// 订单创建成功
xlog.Infof(ctx, "order created, order_id: %s, user_id: %d, amount: %d", orderID, userID, amount)

// 支付成功
xlog.Infof(ctx, "payment success, order_id: %s, amount: %d", orderID, amount)
```

### Error - 错误记录

```go
// 数据库查询失败
xlog.Errorf(ctx, "database query failed, table: user, user_id: %d, err: %v", userID, err)

// 三方服务调用失败
xlog.Errorf(ctx, "third auth failed, provider: wechat, code: %s, err: %v", code, err)

// 缓存操作失败
xlog.Errorf(ctx, "redis set failed, key: %s, err: %v", key, err)
```

### Debug - 调试信息

```go
// 请求参数
xlog.Debugf(ctx, "request params: %+v", req)

// 响应数据
xlog.Debugf(ctx, "response data: %+v", resp)

// 缓存键
xlog.Debugf(ctx, "cache keys: %+v", cacheKeys)

// SQL 查询（调试时）
xlog.Debugf(ctx, "sql query: %s, args: %+v", sql, args)
```

## Context 传递

### 有 Context 时

```go
func (l *UserLogic) GetUser(ctx context.Context, req *pb.GetUserReq) (*pb.GetUserReply, error) {
	// 使用传入的 ctx
	xlog.Infof(ctx, "get user start, user_id: %d", req.UserId)
	// ...
}
```

### 无 Context 时

```go
func init() {
	// 使用 context.Background()
	xlog.Infof(context.Background(), "service initialized")
}
```

### HTTP Handler 中

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	// 使用 r.Context()，包含请求链路追踪信息
	xlog.Infof(r.Context(), "get user request, uri: %s", r.RequestURI)
	// ...
}
```

## 日志格式建议

### 关键业务节点

```go
// 包含关键业务标识
xlog.Infof(ctx, "order paid, order_id: %s, user_id: %d, amount: %d, pay_method: %s",
	orderID, userID, amount, payMethod)
```

### 错误记录

```go
// 包含错误上下文
xlog.Errorf(ctx, "create order failed, user_id: %d, product_id: %d, err: %v",
	userID, productID, err)
```

### 性能记录

```go
import "time"

func (l *UserLogic) GetUser(ctx context.Context, req *pb.GetUserReq) (*pb.GetUserReply, error) {
	start := time.Now()

	// 业务逻辑...

	// 记录耗时
	xlog.Infof(ctx, "get user completed, user_id: %d, cost: %dms",
		req.UserId, time.Since(start).Milliseconds())

	return resp, nil
}
```

## 日志与错误码配合

### 优先使用错误码的日志功能

```go
// ✅ 推荐：错误码已包含日志
return nil, xerr.NewMsgWithErrorLog(
	errors.UserError_USER_NOT_FOUND,
	"用户不存在",
	"user_id: %d", req.UserID,
)

// ❌ 不推荐：重复记录日志
xlog.Errorf(ctx, "user not found, user_id: %d", req.UserID)
return nil, xerr.NewMsg(errors.UserError_USER_NOT_FOUND, "用户不存在")
```

### 仅在需要时单独记录日志

```go
// 记录请求开始
xlog.Infof(ctx, "request start, method: %s, path: %s", r.Method, r.URL.Path)

// 业务处理（使用错误码）
if err != nil {
	return nil, xerr.NewMsgWithErrorLog(...)
}

// 记录请求结束
xlog.Infof(ctx, "request end, cost: %dms", time.Since(start).Milliseconds())
```

## 禁止事项

```go
// ❌ 禁止使用 go-zero 的 logx
import "github.com/zeromicro/go-zero/core/logx"
logx.Error("error message")

// ❌ 禁止不传 context
xlog.Error("error message")

// ❌ 禁止使用 fmt.Printf 记录日志
fmt.Printf("user_id: %d\n", userID)

// ✅ 正确使用
import "apps/pkg/xlog"
xlog.Errorf(ctx, "error message, user_id: %d", userID)
```
