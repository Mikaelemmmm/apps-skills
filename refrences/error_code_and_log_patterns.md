# 错误码与日志使用规范

## 错误码

### 错误码定义

错误码采用 **8 位数字编码**：

```
| 前 4 位：服务前缀 | 后 4 位：具体错误 |
```

定义步骤请参考 [error_code_proto.md](../example/proto_define/error_code_proto.md)。

---

## 错误码返回规则（重要）

### 核心原则

**错误返回必须使用 `xerr.New` 系列方法，禁止直接返回原始错误**

```go
// ✅ 正确：使用 xerr.New 系列方法
return nil, xerr.NewMsgWithErrorLog(errors.CommentError_COMMENT_ERROR_NOT_FOUND, "评论不存在", "comment_id: %d", req.CommentId)

// ❌ 错误：直接返回原始错误
return nil, err
```

### 系统错误码使用规则

系统错误码是**预定义的全局错误码**，仅用于表示**底层基础设施错误**。**严禁用于业务逻辑错误**。

| 错误码 | 说明 | 正确使用场景 | 禁止使用场景 |
|-------|------|-------------|-------------|
| `S_ERROR` | 系统内部错误 | 未知的内部系统错误、配置错误、类型转换失败等 | 业务错误、已知错误场景 |
| `D_ERROR` | 数据库错误 | Repository 层 MySQL 连接失败、查询失败等底层 DB 错误 | 业务状态错误（如用户不存在） |
| `R_ERROR` | Redis 错误 | Redis 连接失败、缓存操作失败等底层缓存错误 | 业务状态错误 |
| `E_ERROR` | 事件错误 | MQ 发送失败、事件处理失败等底层消息错误 | 业务状态错误 |
| `T_ERROR` | 三方错误 | 第三方服务调用失败（如微信支付、短信服务）等 | gRPC 内部调用失败、业务错误 |
| `PARAMS_ERROR` | 参数错误 | 参数绑定失败、参数校验失败（Handler 层常见） | 业务逻辑错误 |

### 业务错误码使用规则

**业务错误码**必须在对应的 `*_error.proto` 中定义，用于表示**业务逻辑错误**。

| 场景 | 应使用错误码 | 示例 |
|------|-------------|------|
| 数据不存在 | 业务定义的错误码 | `CommentError_COMMENT_ERROR_NOT_FOUND` |
| 数据已存在 | 业务定义的错误码 | `CommentError_COMMENT_ERROR_ALREADY_EXISTS` |
| 业务校验失败 | 业务定义的错误码 | `CommentError_COMMENT_ERROR_INVALID_RATING` |
| 无权操作 | 业务定义的错误码 | `CommentError_COMMENT_ERROR_UNAUTHORIZED` |
| gRPC 调用失败 | **使用当前服务自己的错误码包装** | 见下方说明 |

### gRPC 调用错误处理

**重要**：内部 gRPC 调用失败时，**必须使用当前服务自己的错误码包装**，不要：
1. 直接返回原始错误
2. 包装为系统错误码（如 T_ERROR）

**原因**：
- BFF/HTTP 层调用 gRPC 服务时，应该返回 HTTP 服务自己的错误码，而不是下游服务的错误码
- 这样错误码能够正确映射到 API 文档和前端展示
- 便于错误追踪和问题定位

```go
// ✅ 正确：使用当前服务的错误码包装
resp, err := s.svcCtx.CommentRPC.CreateComment(ctx, req)
if err != nil {
    return nil, xerr.NewMsgWithErrorLog(
        errors.StayyError_STAYY_ERROR_THIRD_AUTH_FAILED,
        "创建评论失败",
        "comment_id: %d, err: %v", req.CommentId, err,
    )
}

// ❌ 错误：保留原始错误
resp, err := s.svcCtx.CommentRPC.CreateComment(ctx, req)
if err != nil {
    return nil, err  // 会返回下游服务的错误码，前端无法正确识别
}

// ❌ 错误：包装为 T_ERROR
resp, err := s.svcCtx.CommentRPC.CreateComment(ctx, req)
if err != nil {
    return nil, xerr.Wrap(err, xerr.SystemError_T_ERROR)
}
```

**API 层示例**：

```go
// app/apis/stayy/api/internal/logic/apiv1/comment_logic.go
import (
    "apps/pb/apis/stayy/api/errors"
    "apps/pkg/xerr"
)

// CreateComment 创建评论
func (l *CommentLogic) CreateComment(ctx context.Context, req *pb.CreateCommentReq) (*pb.CreateCommentReply, error) {
    // 调用评论服务创建评论
    grpcResp, err := l.svcCtx.CommentSrv.CommentServiceV1Client.CreateComment(ctx, grpcReq)
    if err != nil {
        xlog.Errorf(ctx, "CreateComment failed, user_id: %d, err: %v", userJwt.UserID, err)
        return nil, xerr.NewMsgWithErrorLog(
            errors.StayyError_STAYY_ERROR_THIRD_AUTH_FAILED,
            "创建评论失败",
            "user_id: %d, err: %v", userJwt.UserID, err,
        )
    }
    // ...
}
```

---

## 错误返回方法

`apps/pkg/xerr` 包提供了以下错误返回方法：

### 方法列表

| 方法 | 说明 | 使用场景 |
|------|------|---------|
| `xerr.New(err)` | 返回默认错误消息 | 简单业务错误 |
| `xerr.NewMsg(err, msg)` | 返回自定义消息，不记录日志 | 不需要记录日志的错误 |
| `xerr.NewWithErrorLog(err, logFormat, ...logParams)` | 返回默认错误消息，记录错误日志 | 需要记录错误日志 |
| `xerr.NewMsgWithErrorLog(err, msg, logFormat, ...logParams)` | 返回自定义消息，记录错误日志 | **推荐使用**：自定义消息 + 错误日志 |
| `xerr.NewWithInfoLog(err, logFormat, ...logParams)` | 返回默认错误消息，记录信息日志 | 需要记录信息日志 |
| `xerr.NewMsgWithInfoLog(err, msg, logFormat, ...logParams)` | 返回自定义消息，记录信息日志 | 自定义消息 + 信息日志 |

---

## 使用示例

### 业务错误

使用业务定义的错误码，返回自定义消息给前端：

```go
import (
    "apps/pb/services/support/comment/errors"
    "apps/pkg/xerr"
)

// 评论不存在
return nil, xerr.NewMsgWithErrorLog(
    errors.CommentError_COMMENT_ERROR_NOT_FOUND,
    "评论不存在",
    "comment_id: %d", req.CommentId,
)

// 评论已存在
return nil, xerr.NewMsg(errors.CommentError_COMMENT_ERROR_ALREADY_EXISTS, "评论已存在")
```

### 数据库底层错误（D_ERROR）

仅在 Repository 层或真正数据库操作失败时使用：

```go
import "apps/pkg/xerr"

// Repository 层：数据库查询失败
comment, err := m.defaultCommentModel.FindOne(ctx, commentId)
if err != nil {
    switch err {
    case model.ErrNotFound:
        // 返回业务错误码
        return nil, xerr.NewMsgWithErrorLog(
            errors.CommentError_COMMENT_ERROR_NOT_FOUND,
            "评论不存在",
            "comment_id: %d", commentId,
        )
    default:
        // 真正的数据库错误，使用 D_ERROR
        return nil, xerr.NewMsgWithErrorLog(
            xerr.SystemError_D_ERROR,
            "数据库查询失败",
            "table: comment, comment_id: %d, err: %v", commentId, err,
        )
    }
}
```

### Redis 错误（R_ERROR）

```go
// Redis 连接/操作失败
token, err := l.svcCtx.BizRedis.Get(ctx, redisKey)
if err != nil {
    return nil, xerr.NewMsgWithErrorLog(
        xerr.SystemError_R_ERROR,
        "Redis获取失败",
        "key: %s, err: %v", redisKey, err,
    )
}
```

### 第三方服务错误（T_ERROR）

```go
// 微信支付失败
resp, err := l.wechatPayClient.Pay(ctx, req)
if err != nil {
    return nil, xerr.NewMsgWithErrorLog(
        xerr.SystemError_T_ERROR,
        "微信支付失败",
        "order_id: %s, err: %v", req.OrderId, err,
    )
}
```

### 系统内部错误（S_ERROR）

```go
// 未知的内部错误
if someError {
    return nil, xerr.NewMsgWithErrorLog(
        xerr.SystemError_S_ERROR,
        "系统内部错误",
        "context: %+v", context,
    )
}
```

### 参数错误（PARAMS_ERROR）

```go
// 参数校验失败
if req.UserId <= 0 {
    return nil, xerr.NewMsg(
        xerr.SystemError_PARAMS_ERROR,
        "用户ID不能为空",
    )
}
```

---

## 常见错误示例

### ❌ 错误：业务错误使用 D_ERROR

```go
// ❌ 错误
user, err := m.userModel.FindOne(ctx, userId)
if err != nil {
    return nil, xerr.NewMsgWithErrorLog(
        xerr.SystemError_D_ERROR,
        "用户不存在",
        "user_id: %d", userId,
    )
}

// ✅ 正确
user, err := m.userModel.FindOne(ctx, userId)
if err != nil {
    if errors.Is(err, model.ErrNotFound) {
        return nil, xerr.NewMsgWithErrorLog(
            errors.UserError_USER_NOT_FOUND,
            "用户不存在",
            "user_id: %d", userId,
        )
    }
    return nil, xerr.NewMsgWithErrorLog(
        xerr.SystemError_D_ERROR,
        "数据库查询失败",
        "user_id: %d, err: %v", userId, err,
    )
}
```

### ❌ 错误：gRPC 调用失败使用 T_ERROR

```go
// ❌ 错误
resp, err := l.svcCtx.CommentRPC.CreateComment(ctx, req)
if err != nil {
    return nil, xerr.Wrap(err, xerr.SystemError_T_ERROR)
}

// ✅ 正确：保留原始错误
resp, err := l.svcCtx.CommentRPC.CreateComment(ctx, req)
if err != nil {
    return nil, err
}
```

### ❌ 错误：直接返回原始错误

```go
// ❌ 错误
return nil, err

// ✅ 正确
return nil, xerr.NewMsgWithErrorLog(
    errors.UserError_USER_NOT_FOUND,
    "用户不存在",
    "user_id: %d", userId,
)
```

### 日志配合使用规则

**重要**：

1. **使用 `xerr.NewXXXLog` 返回错误时**：不需要额外调用 `xlog.Errorf`，因为 `xerr.NewXXXLog` 方法内部已经记录了日志

```go
// ✅ 正确
if err != nil {
    return nil, xerr.NewMsgWithErrorLog(
        errors.StayyError_STAYY_ERROR_USER_SERVICE_FAILED,
        "用户服务调用失败",
        "err: %v", err,
    )
}

// ❌ 错误：重复记录日志
xlog.Errorf(ctx, "user service failed, err: %v", err)  // 不需要这一行
return nil, xerr.NewMsgWithErrorLog(
    errors.StayyError_STAYY_ERROR_USER_SERVICE_FAILED,
    "用户服务调用失败",
    "err: %v", err,
)
```

2. **只在需要单独记录日志的场景使用 `xlog`**：例如业务成功、请求开始/结束等

```go
// ✅ 正确：记录业务成功
xlog.Infof(ctx, "Create user success, user_id: %d", userID)

// ✅ 正确：记录请求开始
xlog.Infof(ctx, "request start, method: %s, path: %s", r.Method, r.URL.Path)
```

---

## 日志

### 日志包

**禁止使用** `github.com/zeromicro/go-zero/core/logx`

**必须使用** `apps/pkg/xlog`

### 日志方法

```go
import "apps/pkg/xlog"

// 带格式化参数
xlog.Infof(ctx, "user login success, user_id: %d", userID)
xlog.Errorf(ctx, "database query failed, err: %v", err)
xlog.Debugf(ctx, "request params: %+v", req)

// 不带格式化参数
xlog.Info(ctx, "user login success")
xlog.Error(ctx, "database query failed")
xlog.Debug(ctx, "request params received")
```

### 日志级别

| 级别 | 方法 | 使用场景 |
|-----|------|---------|
| **Info** | `xlog.Infof`, `xlog.Info` | 正常业务流程记录 |
| **Error** | `xlog.Errorf`, `xlog.Error` | 错误记录 |
| **Debug** | `xlog.Debugf`, `xlog.Debug` | 调试信息 |

### Context 传递

所有日志方法**必须**传入 `context.Context`：

```go
// ✅ 正确
xlog.Infof(ctx, "user login success, user_id: %d", userID)

// ❌ 错误
xlog.Infof("user login success, user_id: %d", userID)

// ✅ 无 context 时使用 context.Background()
xlog.Infof(context.Background(), "background task started")
```

### 日志内容规范

#### Info 日志

记录关键业务节点：

```go
xlog.Infof(ctx, "user login success, user_id: %d, ip: %s", userID, clientIP)
xlog.Infof(ctx, "order created, order_id: %s, amount: %d", orderID, amount)
```

#### Error 日志

记录错误详情，包含上下文信息：

```go
xlog.Errorf(ctx, "database query failed, table: user, user_id: %d, err: %v", userID, err)
xlog.Errorf(ctx, "third auth failed, provider: wechat, code: %s, err: %v", code, err)
```

#### Debug 日志

记录调试信息，生产环境可能关闭：

```go
xlog.Debugf(ctx, "request params: %+v", req)
xlog.Debugf(ctx, "cache keys: %+v", cacheKeys)
```

### 日志与错误码配合

优先使用错误码的日志功能，日志已在错误处理中记录：

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

仅在需要单独记录日志时使用 `xlog`：

```go
// 记录请求开始
xlog.Infof(ctx, "request start, method: %s, path: %s", r.Method, r.URL.Path)

// 业务处理...

// 记录请求结束
xlog.Infof(ctx, "request end, cost: %dms", time.Since(start).Milliseconds())
```

---

## 相关示例

| 文件 | 说明 |
|-----|------|
| [error_code.md](../example/error_log/error_code.md) | 错误码使用示例 |
| [log.md](../example/error_log/log.md) | 日志使用示例 |
