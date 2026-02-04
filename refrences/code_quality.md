# 代码质量规范

本文档规范项目的代码质量要求，包括单元测试、Lint 检查和 Code Review。
编写代码必须符合 项目根目录下的.golangci.yml中的规范

---

## 单元测试

### 覆盖率要求

| 目录             | 覆盖率要求 | 说明               |
| ---------------- | ---------- | ------------------ |
| **pkg/**         | ≥ 90%      | 公共库需要高覆盖率 |
| **app/logic/**   | ≥ 80%      | Logic 业务逻辑层   |
| **app/manager/** | ≥ 80%      | Manager 复用层     |

### 排除文件

以下文件不计入覆盖率统计：

- `*.pb.go` - Protobuf 生成的代码
- `mock_*.go` - Mock 生成的代码

### 运行测试

```bash
# 运行所有测试
make test

# 或直接运行测试脚本
bash script/test/unit_test/unit_test.sh

# 运行特定目录的测试
go test ./pkg/... -cover
go test ./app/apis/stayy/api/internal/logic/... -cover
```

### 测试规范

#### 1. 测试文件命名

测试文件与源文件放在同一目录，命名为 `{源文件名}_test.go`：

```
internal/logic/user_logic.go
internal/logic/user_logic_test.go
```

#### 2. 使用 Mock 进行依赖隔离

```go
package apiv1

import (
	"context"
	"testing"

	"apps/app/apis/stayy/api/internal/svc"
	"apps/app/apis/stayy/api/internal/repository/mocks"
	"github.com/golang/mock/gomock"
)

func TestUserLogic_WXMiniAuth(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	// 创建 Mock 依赖
	mockUserSrv := mocks.NewMockUserSrvType(ctrl)
	mockThirdAuthManager := mocks.NewMockThirdAuthManager(ctrl)

	svcCtx := &svc.ServiceContext{
		Repo: &repository.Repository{
			UserSrv:          mockUserSrv,
			WXMiniAuthClient: mockThirdAuthManager,
		},
	}

	logic := NewUserLogic(svcCtx)

	// 设置 Mock 期望
	mockThirdAuthManager.EXPECT().GetClient("wxMini").Return(nil, nil)

	// 执行测试
	resp, err := logic.WXMiniAuth(context.Background(), &pb.WXMiniAuthReq{
		Code: "test_code",
	})

	// 断言
	assert.NoError(t, err)
	assert.NotNil(t, resp)
}
```

#### 3. 测试用例命名

使用描述性的测试函数名：

```go
// ✅ 正确
func TestUserLogic_CreateUser_Success(t *testing.T) {}
func TestUserLogic_CreateUser_InvalidParam(t *testing.T) {}
func TestUserLogic_CreateUser_DuplicateUser(t *testing.T) {}

// ❌ 错误
func TestUserLogic1(t *testing.T) {}
func TestUserLogicCreateUserCase1(t *testing.T) {}
```

#### 4. 表驱动测试

```go
func TestValidateEmail(t *testing.T) {
	tests := []struct {
		name    string
		email   string
		wantErr bool
	}{
		{
			name:    "valid email",
			email:   "test@example.com",
			wantErr: false,
		},
		{
			name:    "empty email",
			email:   "",
			wantErr: true,
		},
		{
			name:    "invalid format",
			email:   "invalid-email",
			wantErr: true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			err := validateEmail(tt.email)
			if tt.wantErr {
				assert.Error(t, err)
			} else {
				assert.NoError(t, err)
			}
		})
	}
}
```

### Mock 生成

确保接口文件包含 `//go:generate` 指令：

```go
//go:generate mockgen -package rpc -destination mock_user_service.go apps/pb/services/core/user/v1 UserServiceClient
```

执行 `make mock` 生成 Mock 文件。

---

## Lint 检查

### 运行 Lint

```bash
# 运行 golangci-lint
# lint changed file,recommend
make lint-changed
# lint all file
make lint

# 或直接运行
golangci-lint run --timeout=5m
```

### 关键规则

| 规则           | 说明                             | 违规示例                                          |
| -------------- | -------------------------------- | ------------------------------------------------- |
| **禁止导入**   | 禁止使用 `go-zero/core/logx`     | `import "github.com/zeromicro/go-zero/core/logx"` |
| **函数长度**   | 函数不超过 80 行（handler 除外） | 长函数应拆分                                      |
| **自动格式化** | 使用 `goimports`, `gofumpt`      | 代码格式统一                                      |

### 禁止导入详情

**禁止使用** `github.com/zeromicro/go-zero/core/logx`

**必须使用** `apps/pkg/xlog`

```go
// ❌ 错误
import "github.com/zeromicro/go-zero/core/logx"
logx.Error("error message")

// ✅ 正确
import "apps/pkg/xlog"
xlog.Errorf(ctx, "error message")
```

### 自动修复代码格式

```bash
make format
```

这会运行：

- `goimports` - 整理 import
- `gofumpt` - 代码格式化

---

## Code Review

### Code Review 检查点

#### 1. 架构与设计

- [ ] 服务类型划分正确（core/shared/support）
- [ ] 职责清晰，无循环依赖
- [ ] 接口设计符合原子服务原则（方案二）或领域服务原则（方案三）

#### 2. 协议定义

- [ ] Proto 文件定义规范
- [ ] Service 和 RPC 方法有 `@Desc` 注解
- [ ] 命名符合约定（PascalCase for Service/Message, snake_case for fields）
- [ ] 错误码已在 `error_prefix.proto` 中定义前缀

#### 3. 代码结构

- [ ] 目录结构符合规范
- [ ] Repository 包含 `repository.go` 和 `repository_type.go`
- [ ] Model 包含 `//go:generate mockgen` 指令
- [ ] Logic/Service 结构清晰

#### 4. 错误处理

- [ ] 使用 `apps/pkg/xerr` 定义错误
- [ ] 错误返回带日志记录
- [ ] 区分业务错误和系统错误
- [ ] 不使用 `github.com/zeromicro/go-zero/core/logx`

#### 5. 日志规范

- [ ] 使用 `apps/pkg/xlog` 记录日志
- [ ] 日志包含上下文信息（如 user_id, order_id）
- [ ] 日志级别正确（Info/Error/Debug）
- [ ] 第一个参数传递 `context.Context`

#### 6. 数据库操作

- [ ] 使用生成的 Model 方法
- [ ] 自定义方法写在 `*_model.go` 中
- [ ] 局部更新调用 `updatePartial`
- [ ] 事务使用 `model.Transaction()`

#### 7. 单元测试

- [ ] Logic/Manager 层有单元测试
- [ ] 使用 Mock 隔离依赖
- [ ] 覆盖率符合要求（pkg ≥ 90%, app/logic+biz ≥ 80%）
- [ ] 测试用例命名清晰

#### 8. 代码风格

- [ ] 通过 `golangci-lint` 检查
- [ ] 代码格式符合 `gofumpt` 规范
- [ ] Import 已整理（`goimports`）
- [ ] 无未使用的代码

### Code Review 流程

1. **自检**：提交 PR 前执行 `make lint` 和 `make test`
2. **提交 PR**：确保 PR 描述清晰，包含改动说明
3. **通过 Review**：至少一人 Review 通过后合并

### 常见问题

| 问题          | 解决方案                          |
| ------------- | --------------------------------- |
| 覆盖率不足    | 补充单元测试用例                  |
| Lint 失败     | 运行 `make format` 自动修复       |
| 导入禁用包    | 替换为 `apps/pkg/xlog`            |
| Mock 生成失败 | 检查 `//go:generate` 指令是否存在 |
