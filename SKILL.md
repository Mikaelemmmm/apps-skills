---
name: apps-skills
description: |
  构建的企业级内部微服务基础框架，适用于单体应用和微服务架构，支持企业内部开箱即用。
  所有服务代码都是基于protobuf，HTTP、RPC、MQ必须先定义protobuf。

  **适用场景：**
  - 创建/编写 HTTP 服务（app/apis 下的 REST API 服务）
  - 创建/编写 RPC 服务（app/services 下的 gRPC 服务）
  - 数据库操作（GORM、MongoDB、Redis 缓存）
  - 熔断、限流、负载保护
  - 问题排查与框架约定理解
  - 生成生产级微服务代码

  **特性：**
  - 完整的规则与代码示例
  - 最佳实践
  - 常见问题与解决

license: MIT
allowed-tools:
  - Read
  - Grep
  - Glob
---

# apps skills

本技能包提供企业级内部微服务基础框架开发规范，专为 AI Agent 优化，帮助开发者构建生产级服务。覆盖内容：REST API、RPC 服务、数据库操作、问题排查、日志与错误码处理。

---

## 架构模式选择

本项目支持 **三种架构模式**，根据团队规模、业务复杂度和复用需求选择：

| 模式       | 架构                                     | 适用场景                       | 复杂度 |
| ---------- | ---------------------------------------- | ------------------------------ | ------ |
| **方案一** | HTTP 单体服务                            | 小团队、业务简单、快速迭代     | 低     |
| **方案二** | HTTP（业务逻辑编排）+ gRPC（原子、复用） | 中、大团队、需要服务复用       | 高     |
| **方案三** | BFF + 领域服务层                         | 中、大团队、多端接入、领域清晰 | 中     |

### 方案对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 方案一：HTTP 单体                                                         │
│                                                                         │
│  ┌─────────────┐                                                         │
│  │ HTTP 服务   │  → 直接操作数据库，包含所有业务逻辑                       │
│  └─────────────┘                                                         │
│                                                                         │
│  适用：小团队、简单业务、快速开发                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 方案二：HTTP + gRPC                                                      │
│                                                                         │
│  ┌─────────────┐     gRPC      ┌─────────────┐                          │
│  │ HTTP 服务   │  ──────────▶  │ gRPC 服务   │  → 数据库                │
│  │ (业务逻辑)   │               │ (原子服务)   │                          │
│  └─────────────┘               └─────────────┘                          │
│                                                                         │
│  - HTTP：业务逻辑编排、调用多个 gRPC 服务                                  │
│  - gRPC：原子操作、可被多个 HTTP 服务复用的功能                              │
│                                                                         │
│  适用：中、大团队、需要服务复用、HTTP 层有业务逻辑                              │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 方案三：BFF + 领域服务                                                    │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                               │
│  │ BFF-Web │  │ BFF-App │  │ BFF-Admin│                               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                               │
│       │            │            │                                       │
│       └────────────┴────────────┘                                       │
│                    │                                                     │
│                    ▼                                                     │
│           ┌─────────────┐                                               │
│           │ 领域服务层   │  → 按领域划分（user、order、payment 等）         │
│           │  (gRPC)     │                                               │
│           └─────────────┘                                               │
│                                                                         │
│  - BFF：“查询”聚合裁剪、“增删改”透传写入（无业务逻辑）                                     │
│  - 领域服务：核心业务逻辑、可被多个 BFF 复用                                │
│                                                                         │
│  适用：中、大团队、多端接入、领域驱动设计                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### HTTP 与 gRPC 职责划分

| 层级     | 方案二                                                             | 方案三                                                               |
| -------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **HTTP** | • 业务逻辑编排<br>• 调用多个 gRPC 完成业务流程<br>• 可包含业务规则 | • 无业务逻辑<br>• 聚合裁剪、"增删改"透传写入<br>• "查询"跨服务时聚合 |
| **gRPC** | • 原子服务<br>• 可被复用的功能<br>• 数据操作                       | • 核心业务逻辑<br>• 按领域划分<br>• 业务规则实现                     |

---

## 使用指南

当在项目中开发后端业务需求时，参考以下对应模块的规范文档：

| 开发场景              | 参考文档                                                                   |
| --------------------- | -------------------------------------------------------------------------- |
| **Protobuf 协议定义** | [proto_patterns.md](refrences/proto_patterns.md)                           |
| **REST API 开发**     | [rest_api_patterns.md](refrences/rest_api_patterns.md)                     |
| **RPC 服务开发**      | [rpc_service_patterns.md](refrences/rpc_service_patterns.md)               |
| **数据库操作**        | [database_patterns.md](refrences/database_patterns.md)                     |
| **错误码与日志**      | [error_code_and_log_patterns.md](refrences/error_code_and_log_patterns.md) |
| **代码质量规范**      | [code_quality.md](refrences/code_quality.md)                               |

---

## 文档结构

```
apps-skills/
├── SKILL.md                          # 技能包主入口（本文件）
├── refrences/                        # 模式参考文档（规范说明）
│   ├── proto_patterns.md             # Protobuf 协议定义规范
│   ├── rest_api_patterns.md          # REST API 开发模式
│   ├── rpc_service_patterns.md       # RPC 服务开发模式
│   ├── database_patterns.md          # 数据库操作规范
│   ├── error_code_and_log_patterns.md # 错误码与日志规范
│   └── code_quality.md              # 代码质量规范（单测、Lint、Review）
└── example/                          # 代码示例
    ├── proto_define/                 # Protobuf 定义示例
    │   ├── rest_proto.md             # REST API 协议示例
    │   ├── rpc_proto.md              # RPC 服务协议示例
    │   ├── error_code_proto.md       # 错误码定义示例
    │   └── common_proto.md           # 公共协议示例
    ├── rest_api/                     # REST API 代码示例
    │   ├── repository.md             # Repository 层示例
    │   ├── svc_context.md            # ServiceContext 示例
    │   └── logic.md                  # Logic 层示例
    ├── rpc_service/                  # RPC 服务代码示例
    │   ├── server_grpc.md            # gRPC Server 注册示例
    │   ├── repository.md             # Repository 层示例
    │   ├── svc_context.md            # ServiceContext 示例
    │   └── logic.md                  # Logic 层示例
    ├── database/                     # 数据库操作示例
    │   └── mysql.md                  # MySQL Model 示例
    └── error_log/                    # 错误与日志示例
        ├── error_code.md             # 错误码使用示例
        └── log.md                    # 日志使用示例
```

---

## 服务类型划分

RPC 服务（领域服务层）按类型划分：

| 类型        | 目录                | 说明             | 示例                        |
| ----------- | ------------------- | ---------------- | --------------------------- |
| **Core**    | `services/core/`    | 核心业务领域服务 | user, order, payment        |
| **Shared**  | `services/shared/`  | 跨业务共享服务   | authz, permit, dictionary   |
| **Support** | `services/support/` | 支撑服务         | message, notification, file |

> **注意**：创建 RPC 服务时，务必先评估其归属类型，确保放置在正确的领域分类下。

---

## 常用命令

### 协议与代码生成

```bash
# 更新 protobuf 子模块
make update

# 编译 protobuf 生成 Go 代码
make build

# 生成业务代码
make gen type=apis srv=stayy/api       # 生成 HTTP API 服务
make gen type=services srv=core/user   # 生成 gRPC 服务

# 生成错误码映射
make errcode

# 生成数据库 Model
make model db=admin_mgr table=sys_role service=apis/mgr/admin

# 生成 Swagger 文档
make doc type=apis

# 生成 Mock 文件
make mock
```

### 代码质量检查

```bash
# 运行 golangci-lint 检查
make lint

# 运行单元测试
make test

# 自动修复代码格式
make format
```

---

## 代码质量规范

| 项目            | 要求                               | 说明                       |
| --------------- | ---------------------------------- | -------------------------- |
| **单元测试**    | pkg ≥ 90%, app/logic+manager ≥ 80% | 排除 \_.pb.go, mock\_\_.go |
| **Lint 检查**   | 必须通过 golangci-lint             | 见下方规则                 |
| **代码 Review** | 必须通过 Code Review               | 见下方检查点               |

详细规范参考：[code_quality.md](refrences/code_quality.md)

---
