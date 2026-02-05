# 公共协议定义

`protos/common/` 目录下定义 APIs 和 Services 共用的协议。

## 目录结构

```
protos/common/
├── enum/          # 公共枚举类型
├── errors/        # 错误前缀定义
├── message/       # 公共 Message
│   ├── empty.proto
│   ├── page.proto
│   └── ...
└── mq/           # MQ 消息体定义（按服务名独立文件）
```

## 各目录用途

### 1. enum/ - 公共枚举

定义全局共用的枚举类型，如状态、类型等。

```
common/enum/
├── status.proto       # 通用状态
├── enable_status.proto # 启用/禁用状态
└── ...
```

### 2. errors/ - 错误前缀

- 定义每个服务的错误码前缀，确保全局唯一
- 每新建服务时，需在此添加对应前缀

```proto
// common/errors/error_prefix.proto
enum ErrorPrefix {
  // ... 其他服务前缀 ...

  // @Desc: 用户服务前缀
  ERROR_PREFIX_SERVICES_CORE_USER = 1001;
}
```

### 3. message/ - 公共 Message

#### 3.1 基础类型

| 文件 | 说明 |
|-----|------|
| `empty.proto` | 空返回值 |
| `page.proto` | 分页请求/响应 |
| `enable_status.proto` | 启用/禁用状态 |

#### 3.2 MQ 消息体

所有 MQ 消息体定义在 `common/mq/` 下，按服务名独立文件。

```
common/mq/
├── user.proto    # 用户相关 MQ 消息
├── order.proto   # 订单相关 MQ 消息
└── ...
```

**使用 Proto 定义 MQ 消息体的优势**：
- 序列化性能优于 JSON
- 类型安全
- 代码生成自动化

## 使用示例

### 导入公共协议

```proto
import "common/message/empty.proto";
import "common/message/page.proto";
import "common/enum/enable_status.proto";
import "common/mq/user.proto";
```

### 使用 Empty 返回值

```proto
rpc DeleteUser(DeleteUserReq) returns (common.message.Empty) {}
```

### 使用分页请求

```proto
import "common/message/page.proto";

message ListUsersReq {
  // 继承公共分页字段
  common.message.PageReq page = 1;
  string keyword = 2;
}

message ListUsersReply {
  // 继承公共分页响应字段
  common.message.PageReply page = 1;
  repeated User users = 2;
}
```
