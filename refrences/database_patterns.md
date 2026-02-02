# 数据库操作规范

本项目使用 GORM 作为 ORM 框架，支持 MySQL、MongoDB 和 Redis 缓存。

## 生成 Model

### 命令格式

```bash
make model db="{dbname}" table="{tablename}" service="{service_path}"
```

### 参数说明

| 参数 | 说明 | 示例 |
|-----|------|------|
| `db` | 数据库名称 | `admin_mgr` |
| `table` | 数据表名称 | `sys_role` |
| `service` | 服务路径 | `apis/mgr/admin` 或 `services/core/user` |

### 示例

```bash
# 为管理后台的角色表生成 Model
make model db="admin_mgr" table="sys_role" service="apis/mgr/admin"

# 为用户服务生成 Model
make model db="user_db" table="user" service="services/core/user"
```

## 生成的文件

执行命令后，会在对应服务的 `internal/repository/model/` 目录下生成两个文件：

```
internal/repository/model/
├── user_model_gen.go   # 自动生成（勿编辑）
└── user_model.go       # 可手动扩展
```

### user_model_gen.go - 自动生成

包含以下功能（**请勿编辑此文件**）：

- 基础 CRUD 操作
- 根据主键查询
- 根据唯一索引查询
- 带缓存的增删改查

```go
// 自动生成的方法示例
Insert(ctx context.Context, data *User) error
FindOne(ctx context.Context, id int64) (*User, error)
FindOneByAccount(ctx context.Context, account string) (*User, error)
FindOneByEmail(ctx context.Context, email string) (*User, error)
FindOneByMobile(ctx context.Context, mobile string) (*User, error)
Update(ctx context.Context, data *User) (int64, error)
Delete(ctx context.Context, id int64) (int64, error)
Transaction(ctx context.Context, fn func(ctx context.Context) error) error
```

### user_model.go - 可扩展

在此文件中添加自定义查询方法：

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
package model

import (
	"time"

	"github.com/zeromicro/go-zero/core/stores/cache"
	"gorm.io/gorm"
)

var _ UserModel = (*customUserModel)(nil)

type (
	// UserModel 用户模型接口
	UserModel interface {
		userModel
		// 此处定义扩展方法接口
		FindByMobile(ctx context.Context, mobile string) ([]*User, error)
		ListUsers(ctx context.Context, page, pageSize int) ([]*User, int64, error)
	}

	customUserModel struct {
		*defaultUserModel
	}
)

// BeforeCreate 创建前钩子
func (u *User) BeforeCreate(db *gorm.DB) error {
	now := time.Now().Unix()
	u.CreateTime = now
	u.UpdateTime = now
	return nil
}

// BeforeUpdate 更新前钩子
func (u *User) BeforeUpdate(db *gorm.DB) error {
	u.UpdateTime = time.Now().Unix()
	return nil
}

// NewUserModel 创建用户模型
func NewUserModel(conn *gorm.DB, c cache.CacheConf) UserModel {
	return &customUserModel{
		defaultUserModel: newUserModel(conn, c),
	}
}
```

## 数据更新操作

### 全量更新

使用自动生成的 `Update` 方法：

```go
err := userModel.Update(ctx, &model.User{
	Id:      userId,
	Account: "new_account",
	Mobile:  "13800138000",
})
```

### 局部更新

在 `user_model.go` 中添加局部更新方法：

```go
// UpdateMobile 更新手机号
func (m *customUserModel) UpdateMobile(ctx context.Context, userID int64, mobile string) error {
	return m.updatePartial(ctx, userID, map[string]interface{}{
		"mobile": mobile,
	}, map[string]interface{}{})
}
```

调用私有方法 `updatePartial`（定义在 `_gen.go` 中）实现局部字段更新。

## 事务操作

### 单个 Model 事务

```go
err := userModel.Transaction(ctx, func(txCtx context.Context) error {
	// 在事务中执行操作
	if err := userModel.Insert(txCtx, &User{...}); err != nil {
		return err  // 返回错误会自动回滚
	}
	if err := userThirdAuthModel.Insert(txCtx, &UserThirdAuth{...}); err != nil {
		return err
	}
	return nil  // 返回 nil 会提交事务
})
```

### 跨 Model 事务

```go
err := userModel.Transaction(ctx, func(txCtx context.Context) error {
	// 可以在事务中使用多个 Model
	if err := userModel.Insert(txCtx, data); err != nil {
		return err
	}
	if err := orderModel.Insert(txCtx, orderData); err != nil {
		return err
	}
	return nil
})
```

## 缓存机制

Model 自动集成 Redis 缓存，缓存键格式：

```
cache:{service}:{table}:{index}:{value}
```

例如：
```
cache:platformUser:user:id:10001
cache:platformUser:user:account:test@example.com
cache:platformUser:user:mobile:13800138000
```

### 缓存失效

更新、删除操作会自动清除相关缓存键。

## Mock 生成

每个 Model 接口都需要添加 Mock 生成指令：

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
```

执行 `make mock` 生成 Mock 文件用于单元测试。

## 相关示例

| 文件 | 说明 |
|-----|------|
| [mysql.md](../example/database/mysql.md) | MySQL Model 完整示例 |
