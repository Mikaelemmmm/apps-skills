# 错误码定义示例

## 错误码结构

错误码采用 **8 位数字编码**：

```
| 前 4 位：服务前缀 | 后 4 位：具体错误 |
```

- **前 4 位**：在 `protos/common/errors/error_prefix.proto` 中定义，确保全局唯一
- **后 4 位**：在各服务的 `*_error.proto` 中定义，从 1 开始递增

## 定义步骤

### 步骤 1：添加服务前缀

每次新建服务时，在 `protos/common/errors/error_prefix.proto` 中添加该服务的前缀：

```proto
syntax = "proto3";

package common.errors;

option go_package = "apps/pb/common/errors";

// @Desc: 错误前缀枚举（保持全局唯一）
enum ErrorPrefix {
  // ... 其他服务前缀 ...

  // @Desc: 民宿 API 前缀
  ERROR_PREFIX_APIS_STAYY_API = 5001;
}
```

### 步骤 2：定义具体错误码

在对应服务的 `errors/` 目录下创建 `{service}_error.proto` 文件。

**REST API 服务示例**：`protos/apis/stayy/api/errors/user_error.proto`

```proto
syntax = "proto3";

package apis.stayy.api.errors;

option go_package = "apps/pb/apis/stayy/api/errors";

// @Desc: 用户模块错误码
enum UserError {

  // @Desc: 前缀（必须与 error_prefix.proto 中定义的一致，值必须为 0）
  ERROR_PREFIX_APIS_STAYY_API = 0;

  // @Desc: 三方授权失败
  USER_ERROR_THIRD_AUTH_FAILED = 1;
  // @Desc: 三方返回数据异常
  USER_ERROR_THIRD_RETURN_ERROR = 2;
  // @Desc: 用户不存在
  USER_ERROR_NOT_FOUND = 3;
}
```

## 规范要点

### 1. 枚举命名

```
{SERVICE}_{ERROR_DESCRIPTION}
```

- 使用大写蛇形命名（UPPER_SNAKE_CASE）
- 必须包含 `@Desc` 描述

### 2. 第一个字段规则

```proto
enum UserError {
  // @Desc: 前缀
  ERROR_PREFIX_APIS_STAYY_API = 0;  // 值必须为 0

  // 具体错误从 1 开始
  USER_ERROR_THIRD_AUTH_FAILED = 1;
}
```

- 名称必须与 `error_prefix.proto` 中定义的前缀名称一致
- 值**必须为 0**
- 备注标注为"前缀"

### 3. Package 命名

**REST API 服务**：

```proto
package apis.stayy.api.errors;
option go_package = "apps/pb/apis/stayy/api/errors";
```

**RPC 服务**：

```proto
package services.core.user.errors;
option go_package = "apps/pb/services/core/user/errors";
```

## 错误码生成流程

1. 在 `protos/common/errors/error_prefix.proto` 中添加服务前缀
2. 在对应服务的 `errors/` 目录下创建 `{service}_error.proto`
3. 执行 `make build` 生成 Go 代码
4. 执行 `make errcode` 生成错误码映射
