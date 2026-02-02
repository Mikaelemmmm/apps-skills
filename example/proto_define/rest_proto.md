# REST API 协议定义示例

REST API 服务协议定义在 `protos/apis/` 目录下。

## 文件位置

```
protos/apis/{app}/{layer}/v{version}/{service}_api.proto
```

例如：民宿应用 C 端 v1 版本的用户接口：

```
protos/apis/stayy/api/v1/user_api.proto
```

## 协议示例

```proto
syntax = "proto3";

package apis.stayy.api.v1;

option go_package = "apps/pb/apis/stayy/api/v1";

import "google/api/annotations.proto";
import "google/protobuf/descriptor.proto";

// @Desc: 用户模块
service UserService {

  // @Desc: 微信小程序授权登录
  rpc WXMiniAuth(WXMiniAuthReq) returns (ThirdAuthReply) {
    option (google.api.http) = {
      post : "/v1/user/wxMiniAuth",
      body : "*",
    };
  };
}

message WXMiniAuthReq {
  // 微信小程序授权 code
  string code = 1;
  // 微信小程序授权后的 IV
  string iv = 2;
  // 微信小程序授权后的加密数据
  string encrypted_data = 3;
}

message ThirdAuthReply {
  string access_token = 1;
  int64 access_expire = 2;
  int64 refresh_expire = 3;
}
```

## 规范要点

### 1. Package 命名

```proto
package apis.stayy.api.v1;
```

严格按照目录层级格式：`apis.{app}.{layer}.{version}`

### 2. go_package 配置

```proto
option go_package = "apps/pb/apis/stayy/api/v1";
```

- 前缀固定为 `apps/pb/`
- 后续严格按照目录层级格式

### 3. @Desc 注解使用

| 位置 | 是否使用 @Desc |
|-----|---------------|
| Service | ✅ 必须 |
| RPC 方法 | ✅ 必须 |
| Message | ❌ 使用普通 `//` 注释 |
| Message 字段 | ❌ 使用普通 `//` 注释 |

### 4. HTTP 映射

```proto
rpc WXMiniAuth(WXMiniAuthReq) returns (ThirdAuthReply) {
  option (google.api.http) = {
    post : "/v1/user/wxMiniAuth",
    body : "*",
  };
};
```

- 路径格式：`/v{version}/{module}/{action}`
- POST 请求使用 `body: "*"`
