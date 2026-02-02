# RPC 服务协议定义示例

RPC 服务协议定义在 `protos/services/` 目录下。

## 文件位置

```
protos/services/{type}/{service}/v{version}/{service}_service.proto
```

例如：共享服务下的鉴权服务：

```
protos/services/shared/permit/v1/permit_service.proto
```

## 协议示例

```proto
syntax = "proto3";

package services.shared.permit.v1;

option go_package = "apps/pb/services/shared/permit/v1";

import "common/message/empty.proto";

// @Desc: 后台 API 权限模块
service PermitService {

  // @Desc: 设置 API 权限
  rpc SetPermission(SetPermissionReq) returns (common.message.Empty) {}

  // @Desc: 生成 token
  rpc GenToken(GenTokenReq) returns (GenTokenReply) {}
}

message Resource {
  string path = 1;
  string method = 2;
}

message SetPermissionReq {
  // 租户
  string domain = 1;
  // 角色
  string role = 2;
  // 角色对应的资源
  repeated Resource resources = 3;
}

message GenTokenReq {
  // 租户
  string domain = 1;
  // 管理员唯一标识
  int64 uid = 2;
  // 角色
  string role = 3;
}

message GenTokenReply {
  // 访问 token
  string access_token = 1;
  // token 失效时间
  int64 access_expire = 2;
  // refresh token 时间
  int64 refresh_expire = 3;
}
```

## 规范要点

### 1. Package 命名

```proto
package services.shared.permit.v1;
```

严格按照目录层级格式：`services.{type}.{service}.{version}`

### 2. go_package 配置

```proto
option go_package = "apps/pb/services/shared/permit/v1";
```

- 前缀固定为 `apps/pb/`
- 后续严格按照目录层级格式

### 3. 服务类型选择

定义 RPC 服务前，先评估其归属类型：

| 类型 | 说明 | 示例 |
|-----|------|-----|
| **core** | 核心业务领域 | user, order, payment |
| **shared** | 跨业务共享 | authz, permit |
| **support** | 支撑服务 | message, file |

### 4. @Desc 注解使用

| 位置 | 是否使用 @Desc |
|-----|---------------|
| Service | ✅ 必须 |
| RPC 方法 | ✅ 必须 |
| Message | ❌ 使用普通 `//` 注释 |
| Message 字段 | ❌ 使用普通 `//` 注释 |

### 5. 空返回使用 common.message.Empty

```proto
import "common/message/empty.proto";

rpc SetPermission(SetPermissionReq) returns (common.message.Empty) {}
```
