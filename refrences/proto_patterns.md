# Protobuf 协议定义规范

本项目采用 **Protocol-First** 开发模式，所有 API 均通过 Protobuf 定义，再通过代码生成工具生成 REST API 和 RPC 服务代码。

## 协议目录结构

Protobuf 协议作为独立仓库管理，通过 Git Submodule 引入到项目根目录的 `protos/` 目录下。

```
protos/
├── apis/              # REST API 协议
│   └── {app}/         # 应用名（如 stayy: 民宿）
│       ├── api/       # C 端 API
│       │   ├── v1/    # 版本号
│       │   ├── v2/    # 差异较大时独立新版本
│       │   └── errors/ # 错误码定义（仅一个目录）
│       └── admin/     # 后台管理端 API
│           ├── v1/
│           └── errors/
├── services/          # gRPC 服务协议
│   ├── core/          # 核心领域服务
│   ├── shared/        # 共享领域服务
│   └── support/       # 支撑领域服务
├── common/            # 公共协议
│   ├── enum/          # 公共枚举
│   ├── errors/        # 错误前缀定义
│   └── message/       # 公共 Message（分页、状态等）
│       └── mq/        # MQ 消息体定义
└── third_party/       # 三方 proto（无需修改）
```

## 服务类型划分

| 类型        | 说明             | 示例                        |
| ----------- | ---------------- | --------------------------- |
| **core**    | 核心业务领域服务 | user, order, payment        |
| **shared**  | 跨业务共享服务   | authz, permit, dictionary   |
| **support** | 支撑服务         | message, notification, file |

> **重要**：定义 RPC 服务时，务必先评估其归属类型，确保放置在正确的领域分类下。

## 命名规范

### Package 命名

- **REST API**: `apis.{app}.{layer}.{version}`
- **RPC 服务**: `services.{type}.{service}.{version}`
- **公共**: `common.{category}`

```proto
// REST API 示例
package apis.stayy.api.v1;
option go_package = "apps/pb/apis/stayy/api/v1";

// RPC 服务示例
package services.shared.permit.v1;
option go_package = "apps/pb/services/shared/permit/v1";
```

### @Desc 注解规范

`@Desc` 注解**仅用于** Service 和 RPC 方法的描述，**必须添加**。

```proto
// @Desc: 用户模块
service UserService {
  // @Desc: 微信小程序授权登录
  rpc WXMiniAuth(WXMiniAuthReq) returns (ThirdAuthReply);
}
```

Message 中的字段使用普通 `//` 注释即可：

```proto
message WXMiniAuthReq {
  string code = 1;          // 微信小程序授权 code
  string iv = 2;            // 微信小程序授权后的 IV
  string encrypted_data = 3; // 微信小程序授权后的加密数据
}
```

## 相关示例

| 场景              | 示例文档                                                           |
| ----------------- | ------------------------------------------------------------------ |
| REST API 协议定义 | [rest_proto.md](../example/proto_define/rest_proto.md)             |
| RPC 服务协议定义  | [rpc_proto.md](../example/proto_define/rpc_proto.md)               |
| 错误码定义        | [error_code_proto.md](../example/proto_define/error_code_proto.md) |
| 公共协议定义      | [common_proto.md](../example/proto_define/common_proto.md)         |

## 开发流程

1. 执行 `make update` 更新子模块
2. 在 `protos/` 对应目录下创建/修改 `.proto` 文件
3. 执行 `make build` 编译 protobuf 生成 Go 代码
4. 执行 `make gen` 生成业务代码
5. 执行 `make errcode` 生成错误码映射
