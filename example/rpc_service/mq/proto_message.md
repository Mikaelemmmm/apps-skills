# MQ 消息体 Protobuf 定义示例

MQ 消息体使用 Protobuf 定义，文件位于 `protos/common/mq/` 目录下。

> **注意**：每个服务一个文件，文件名就是服务名。只使用 `message` 和 `enum` 定义，其他不需要。

## 目录结构

```
protos/common/mq/
└── user.proto          # User 服务 MQ 消息定义
```

## 文件位置

`protos/common/mq/xxx.proto`

## 代码示例

### user.proto - User 服务 MQ 消息

```proto
syntax = "proto3";

package common.mq;

option go_package = "apps/pb/common/mq";

// UserCountReq 统计消息
message UserCountReq {
  // 用户ID
  int64 user_id = 1;
}

// UserUpLevelReq 升级消息
message UserUpLevelReq {
  // 用户ID
  int64 user_id = 1;
  // 目标等级
  int32 level = 2;
  // 消息ID（用于幂等性）
  string message_id = 3;
}

// UserSyncReq 同步消息
message UserSyncReq {
  // 用户ID
  int64 user_id = 1;
  // 操作类型
  string action = 2;
}
```

### order.proto - Order 服务 MQ 消息

```proto
syntax = "proto3";

package common.mq;

option go_package = "apps/pb/common/mq";

// OrderCreateReq 创建订单消息
message OrderCreateReq {
  // 订单ID
  int64 order_id = 1;
  // 用户ID
  int64 user_id = 2;
  // 商品列表
  repeated OrderItem items = 3;
}

// OrderItem 订单商品
message OrderItem {
  // 商品ID
  int64 product_id = 1;
  // 商品数量
  int32 quantity = 2;
  // 商品单价
  int64 price = 3;
}

// OrderStatus 订单状态
enum OrderStatus {
  // 待支付
  ORDER_STATUS_PENDING = 0;
  // 已支付
  ORDER_STATUS_PAID = 1;
  // 已取消
  ORDER_STATUS_CANCELLED = 2;
}
```

## 命名规范

| 规范 | 说明 |
|-------|------|
| 文件名 | 服务名（小写），如 `user.proto`、`order.proto` |
| package | `common.mq` |
| go_package | `apps/pb/common/mq` |
| Message 名 | 操作 + Req，如 `UserCountReq`、``OrderCreateReq` |
| 字段名 | snake_case，如 `user_id`、`message_id` |

## 常用消息字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `user_id` | `int64` | 用户ID |
| `message_id` | `string` | 消息ID（用于幂等性） |
| `trace_id` | `string` | 链路追踪ID |
| `timestamp` | `int64` | 时间戳 |

## 生成代码

```bash
# 更新 protobuf submodule
make update

# 构建 protobuf 文件
make build
```

生成后的 Go 代码位于：`apps/pb/common/mq/`
