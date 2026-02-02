# 错误码与日志使用规范

## 错误码

### 错误码定义

错误码采用 **8 位数字编码**：

```
| 前 4 位：服务前缀 | 后 4 位：具体错误 |
```

定义步骤请参考 [error_code_proto.md](../example/proto_define/error_code_proto.md)。

### 错误码使用

在 Logic/Service 层中直接使用定义的错误码。

### 系统错误码类型

| 错误码 | 说明 | 使用场景 |
|-------|------|---------|
| `S_ERROR` | 系统内部错误 | 未知系统错误 |
| `D_ERROR` | 数据库错误 | 数据库操作失败 |
| `R_ERROR` | Redis 错误 | 缓存操作失败 |
| `E_ERROR` | 事件错误 | 事件处理失败 |
| `T_ERROR` | 三方错误 | 第三方服务调用失败 |
| `PARAMS_ERROR` | 参数错误 | 请求参数校验失败 |

### 错误返回方法

`apps/pkg/xerr` 包提供了以下错误返回方法：

#### 不带消息

```go
// 返回默认错误消息
xerr.New(err Error)
```

#### 带日志

```go
// 返回默认错误消息，记录错误日志
xerr.NewWithErrorLog(err Error, logFormat string, logParams ...any)

// 返回默认错误消息，记录信息日志
xerr.NewWithInfoLog(err Error, logFormat string, logParams ...any)
```

#### 自定义消息

```go
// 返回自定义消息，不记录日志
xerr.NewMsg(err Error, msg string)

// 返回自定义消息，记录错误日志
xerr.NewMsgWithErrorLog(err Error, msg, logFormat string, logParams ...any)

// 返回自定义消息，记录信息日志
xerr.NewMsgWithInfoLog(err Error, msg, logFormat string, logParams ...any)
```

### 使用示例

#### 业务错误

使用业务定义的错误码，返回自定义消息给前端：

```go
import (
	"apps/pb/apis/stayy/api/errors"
	"apps/pkg/xerr"
)

// 用户不存在
return nil, xerr.NewMsgWithErrorLog(
	errors.UserError_ERROR_PREFIX_APIS_STAYY_API,
	"用户不存在",
	"user_id: %d", req.UserId,
)
```

#### 系统错误

使用系统级错误码，通常发生在中间件或底层组件：

```go
import "apps/pkg/xerr"

// 数据库查询失败
return nil, xerr.NewMsgWithErrorLog(
	xerr.SystemError_D_ERROR,
	"database query failed",
	"table: user, err: %v", err,
)

// Redis 连接失败
return nil, xerr.NewMsgWithErrorLog(
	xerr.SystemError_R_ERROR,
	"redis connection failed",
	"addr: %s, err: %v", addr, err,
)

// 三方服务调用失败
return nil, xerr.NewMsgWithErrorLog(
	xerr.SystemError_T_ERROR,
	"third party service error",
	"service: wechat, err: %v", err,
)
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

## 相关示例

| 文件 | 说明 |
|-----|------|
| [error_code.md](../example/error_log/error_code.md) | 错误码使用示例 |
| [log.md](../example/error_log/log.md) | 日志使用示例 |
