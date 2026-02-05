# REST API 服务开发模式

REST API 服务位于 `app/apis/` 目录下，根据架构模式不同，职责有所差异。

## 服务定位（按架构模式）

### 方案一：HTTP 单体服务

```
┌─────────────────────────────────────────────────────────────┐
│                    REST API (app/apis/)                      │
│                         职责：                                │
│  - 包含所有业务逻辑                                           │
│  - 直接操作数据库                                             │
│  - 处理所有业务规则                                           │
└─────────────────────────────────────────────────────────────┘
```

**适用**：小团队、简单业务、快速开发

---

### 方案二：HTTP + gRPC（业务逻辑在 HTTP）

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP (app/apis/)                          │
│                         职责：                                │
│  - 业务逻辑编排：调用多个 gRPC 服务完成业务流程                 │
│  - 业务规则实现                                              │
│  - 可调用 gRPC 的原子服务和复用功能                            │
└─────────────────────────────────────────────────────────────┘
                              ↓ gRPC
┌─────────────────────────────────────────────────────────────┐
│                      gRPC (app/services/)                    │
│                         职责：                                │
│  - 原子服务：单一职责的细粒度操作                              │
│  - 复用功能：可被多个 HTTP 服务调用                            │
│  - 数据库操作                                                │
└─────────────────────────────────────────────────────────────┘
```

**适用**：中等规模、HTTP 层有业务逻辑、需要服务复用

---

### 方案三：BFF + 领域服务（业务逻辑在领域服务）

```
┌─────────────────────────────────────────────────────────────┐
│                    BFF (app/apis/)                           │
│                         职责：                                │
│  - 聚合裁剪：前端请求的统一入口                                │
│  - CUD 操作：透传到 gRPC（无业务逻辑）                          │
│  - Query 操作：聚合多个 gRPC 服务数据                          │
│  - 字段裁剪：按前端需求返回                                    │
│  - 不包含业务逻辑                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓ gRPC
┌─────────────────────────────────────────────────────────────┐
│                   领域服务 (app/services/)                    │
│                         职责：                                │
│  - 核心业务逻辑：按领域划分实现                                │
│  - 业务规则实现                                              │
│  - 数据库操作                                                │
└─────────────────────────────────────────────────────────────┘
```

**适用**：大团队、多端接入、领域驱动设计

## 开发步骤

| 步骤 | 操作 | 说明 |
|-----|------|-----|
| 1 | 定义协议 | 参考 [rest_proto.md](../example/proto_define/rest_proto.md) |
| 2 | 生成代码 | `make gen type=apis srv=stayy/api` |
| 3 | 定义错误码 | 参考 [error_code_proto.md](../example/proto_define/error_code_proto.md) |
| 4 | 编写 Repository | 参考 [repository.md](../example/rest_api/repository.md) |
| 5 | 配置 ServiceContext | 参考 [svc_context.md](../example/rest_api/svc_context.md) |
| 6 | 编写 Logic | 参考 [logic.md](../example/rest_api/logic.md) |

## 服务内部结构

```
app/apis/{app}/{layer}/
├── main.go              # 服务入口
├── etc/
│   └── conf.yaml         # 配置文件
└── internal/
    ├── config/           # 配置定义
    ├── handler/          # HTTP 处理器（生成）
    ├── logic/            # 业务逻辑层
    ├── manager/          # Logic 复用层
    ├── repository/       # 依赖注入层
    │   ├── adapter/      # 三方服务适配器
    │   ├── rpc/          # RPC 客户端
    │   ├── xredis/       # Redis 操作
    │   ├── repository.go
    │   └── repository_type.go
    ├── router/           # 路由定义（生成）
    └── svc/              # ServiceContext
```

## 代码生成命令

```bash
# 生成 REST API 服务代码
make gen type=apis srv=stayy/api

# 参数说明
# type=apis   : 生成 API 服务
# srv         : 服务路径，格式为 {应用名}/{服务名}
```

## Job 和 MQ 嵌入模式

REST API 服务可以嵌入轻量级的 Job（定时任务）和 MQ（消息队列消费者），适用于业务量不大的场景。

| 特性 | 说明 |
|-----|------|
| 部署方式 | HTTP、Job、MQ 在同一进程中运行（不同 goroutine） |
| 入口文件 | `app/apis/stayy/api/main.go` |
| 配置文件 | `etc/conf.yaml`（与 HTTP 配置在同一文件） |
| 适用场景 | 轻量级任务、业务量不大、资源节省 |

> **重要**：REST API 中的 Job、MQ Handler、Logic、Repository 写法与 RPC Service 完全一致

## 相关示例

| 文件 | 说明 |
|-----|------|
| [repository.md](../example/rest_api/repository.md) | Repository 层结构 |
| [svc_context.md](../example/rest_api/svc_context.md) | ServiceContext 初始化 |
| [logic.md](../example/rest_api/logic.md) | Logic 层业务逻辑 |

### Job 相关

| 文件 | 说明 |
|-----|------|
| [rest_api_job_patterns.md](../references/rest_api_job_patterns.md) | Job 嵌入模式说明 |
| [job/main.md](../example/rest_api/job/main.md) | REST API 中嵌入 Job 的 main.go 示例 |
| [job/config.md](../example/rest_api/job/config.md) | REST API 中 Job 配置示例 |
| [job/server.md](../example/rest_api/job/server.md) | REST API 中 Job Server 示例 |
| [job/handler.md](../example/rpc_service/job/handler.md) | Job Handler 示例（与 RPC Service 共享） |
| [job/logic.md](../example/rpc_service/job/logic.md) | Job Logic 示例（与 RPC Service 共享） |

### MQ 相关

| 文件 | 说明 |
|-----|------|
| [rest_api_mq_patterns.md](../references/rest_api_mq_patterns.md) | MQ 嵌入模式说明 |
| [mq/main.md](../example/rest_api/mq/main.md) | REST API 中嵌入 MQ 的 main.go 示例 |
| [mq/config.md](../example/rest_api/mq/config.md) | REST API 中 MQ 配置示例 |
| [mq/server.md](../example/rest_api/mq/server.md) | REST API 中 MQ Server 示例 |
| [mq/handler.md](../example/rpc_service/mq/handler.md) | MQ Handler 示例（与 RPC Service 共享） |
| [mq/logic.md](../example/rpc_service/mq/logic.md) | MQ Logic 示例（与 RPC Service 共享） |

## 设计原则

### 方案一：HTTP 单体服务

所有业务逻辑都在 HTTP 层实现，直接操作数据库：

```go
// ✅ 正确：单体服务可直接操作数据库
func (l *CreateUserLogic) CreateUser(ctx context.Context, req *pb.CreateUserReq) (*pb.CreateUserReply, error) {
    // 业务逻辑校验
    if req.Age < 18 {
        return nil, errors.New("age must be >= 18")
    }

    // 直接操作数据库
    err := l.userModel.Insert(ctx, &model.User{
        Account: req.Account,
        Age:     req.Age,
    })
    // ...
}
```

---

### 方案二：HTTP + gRPC（业务逻辑在 HTTP）

HTTP 层包含业务逻辑编排，调用 gRPC 的原子服务：

```go
// ✅ 正确：HTTP 层编排业务逻辑
func (l *RegisterLogic) Register(ctx context.Context, req *pb.RegisterReq) (*pb.RegisterReply, error) {
    // 1. 业务逻辑：校验
    if err := l.validateRegister(req); err != nil {
        return nil, err
    }

    // 2. 调用 gRPC 原子服务：创建用户
    userResp, err := l.userSrv.CreateUser(ctx, &userPb.CreateUserReq{
        Account: req.Account,
        Mobile:  req.Mobile,
    })
    if err != nil {
        return nil, err
    }

    // 3. 调用 gRPC 复用服务：发送欢迎短信
    l.smsSrv.SendWelcomeSms(ctx, &smsPb.SendSmsReq{
        Mobile: req.Mobile,
        Template: "welcome",
    })

    return &pb.RegisterReply{UserId: userResp.UserId}, nil
}
```

---

### 方案三：BFF + 领域服务（业务逻辑在领域服务）

BFF 层只做聚合裁剪，无业务逻辑：

#### CUD 操作（创建、更新、删除）

直接透传到 gRPC 服务：

```go
// ✅ 正确：BFF 层直接透传
func (l *CreateUserLogic) CreateUser(ctx context.Context, req *pb.CreateUserReq) (*pb.CreateUserReply, error) {
    return l.userSrv.CreateUser(ctx, req)
}

// ❌ 错误：BFF 层不应有业务逻辑
func (l *CreateUserLogic) CreateUser(ctx context.Context, req *pb.CreateUserReq) (*pb.CreateUserReply, error) {
    // BFF 层不应处理业务逻辑
    if req.Age < 18 {
        return nil, errors.New("age must be >= 18")
    }
    return l.userSrv.CreateUser(ctx, req)
}
```

#### Query 操作（查询）

```go
// ✅ 正确：聚合查询
func (l *GetUserDetailLogic) GetUserDetail(ctx context.Context, req *pb.GetUserDetailReq) (*pb.GetUserDetailReply, error) {
    // 并发调用多个 RPC 服务
    var (
        userResp *userPb.GetUserReply
        orderResp *orderPb.GetUserOrdersReply
        wg errgroup.Group
    )

    wg.Go(func() error {
        var err error
        userResp, err = l.userSrv.GetUser(ctx, &userPb.GetUserReq{Id: req.Id})
        return err
    })

    wg.Go(func() error {
        var err error
        orderResp, err = l.orderSrv.GetUserOrders(ctx, &orderPb.GetUserOrdersReq{UserId: req.Id})
        return err
    })

    if err := wg.Wait(); err != nil {
        return nil, err
    }

    // 聚合返回
    return &pb.GetUserDetailReply{
        User: userResp.User,
        Orders: orderResp.Orders,
    }, nil
}
```

## 单元测试要求

- Logic 层覆盖率 >= 80%
- 使用 Mock 的 Repository 进行测试
- 执行 `make mock` 生成 Mock 文件
