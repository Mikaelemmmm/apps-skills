# apps skills

企业级内部微服务基础框架开发规范，覆盖 REST API、RPC、Job、MQ、数据库操作、日志与错误码处理，专为 AI Agent 优化。

![使用效果](./images/use_effect.gif)

---

## 快速开始

完整规范、架构选型（HTTP 单体 / HTTP+gRPC / BFF+领域服务）、常用命令和使用指南见 **[SKILL.md](SKILL.md)**。

---

## 文档索引

| 模块                   | 规范                                           | 示例              |
| ---------------------- | ---------------------------------------------- | ----------------- |
| Protobuf 协议定义      | [references/proto_patterns.md](references/proto_patterns.md)                           | example/proto_define |
| REST API 开发          | [references/rest_api_patterns.md](references/rest_api_patterns.md)                     | example/rest_api     |
| REST MQ / REST Job     | [references/rest_api_mq_patterns.md](references/rest_api_mq_patterns.md) / [job](references/rest_api_job_patterns.md) | example/mq           |
| RPC 服务开发           | [references/rpc_service_patterns.md](references/rpc_service_patterns.md)               | example/rpc_service  |
| RPC MQ / RPC Job       | [references/rpc_service_mq_patterns.md](references/rpc_service_mq_patterns.md) / [job](references/rpc_service_job_patterns.md) | —                   |
| 数据库操作             | [references/database_patterns.md](references/database_patterns.md)                     | example/database     |
| 错误码与日志           | [references/error_code_and_log_patterns.md](references/error_code_and_log_patterns.md) | example/error_log    |
| 代码质量规范           | [references/code_quality.md](references/code_quality.md)                               | —                   |

---

## 代码质量

| 项目          | 要求                                        |
| ------------- | ------------------------------------------- |
| 单元测试      | pkg ≥ 90%, app/logic+manager ≥ 80%          |
| Lint 检查     | golangci-lint v2 必须通过                   |
| 代码规范      | 遵循项目根目录 `.golangci.yml`              |

