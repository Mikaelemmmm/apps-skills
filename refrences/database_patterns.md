# 数据库操作规范

本项目使用 GORM 作为 ORM 框架，支持 MySQL、MongoDB 和 Redis 缓存。

## ⚠️ 重要：Model 生成流程

**必须严格按照以下顺序操作，否则生成的 Model 代码可能不正确：**

1. **确保 MySQL 服务正在运行**
2. **创建数据库表**（通过迁移脚本或手动执行 SQL）
3. **验证表已存在**
4. **执行 `make model` 命令生成代码**

### ❌ 错误做法

```bash
# 不要这样做！如果数据库未运行或表不存在，生成的代码是错误的
make model db="xxx" table="xxx" service="xxx"
```

```go
// ❌ 绝对不要手动编辑 *_model_gen.go 或 vars.go！
// 这些文件由 make model 命令自动生成，任何手动编辑都会被覆盖
```

### ✅ 正确做法

```bash
# 1. 确保 MySQL 运行中
mysql -h127.0.0.1 -P3306 -uroot -p -e "SELECT 1;"

# 2. 创建表（如果不存在）
mysql -h127.0.0.1 -P3306 -uroot -p mydb < deploy/migration/my_service.sql

# 3. 验证表存在
mysql -h127.0.0.1 -P3306 -uroot -p -e "USE mydb; DESC mytable;"

# 4. 生成 Model（从真实的表结构读取）
make model db="mydb" table="mytable" service="services/my/service"
```

### ⚠️ 核心规则

1. **不要编辑 *_model_gen.go 文件** - 这是自动生成的代码，任何修改都会被下次 `make model` 覆盖
2. **修改表结构后必须重新执行 `make model`** - 表结构变化后，必须重新生成 Model 以获取最新的字段定义
3. **只在 *_model.go 中添加自定义代码** - 自定义查询方法、GORM hooks 等应在此文件中添加

## 生成 Model

### 命令格式

```bash
make model db="{dbname}" table="{tablename}" service="{service_path}"
```

### 参数说明

| 参数 | 说明 | 示例 |
|-----|------|------|
| `db` | 数据库名称（必须存在） | `admin_mgr` |
| `table` | 数据表名称（必须存在） | `sys_role` |
| `service` | 服务路径 | `apis/mgr/admin` 或 `services/core/user` |

### 完整示例

```bash
# 示例：为评论服务生成 Model

# 1. 创建数据库（如果不存在）
mysql -h127.0.0.1 -P3306 -uroot -p -e "CREATE DATABASE IF NOT EXISTS services_support_comment;"

# 2. 执行迁移脚本创建表
mysql -h127.0.0.1 -P3306 -uroot -p services_support_comment < deploy/migration/services_support_comment.sql

# 3. 验证表结构
mysql -h127.0.0.1 -P3306 -uroot -p -e "USE services_support_comment; DESC comment;"

# 4. 生成 Model 代码
make model db="services_support_comment" table="comment" service="services/support/comment"
```

## 生成的文件

执行命令后，会在对应服务的 `internal/repository/model/` 目录下生成以下文件：

```
internal/repository/model/
├── comment_model_gen.go  # 自动生成（DO NOT EDIT）- 包含 Model 结构体和 CRUD 方法
├── comment_model.go       # 可手动扩展 - 自定义查询方法和钩子
└── vars.go               # 自动生成（DO NOT EDIT）- 接口扩展定义
```

### *_model_gen.go - 自动生成（DO NOT EDIT）

此文件由 `make model` 命令自动生成，**请勿手动编辑**。

包含以下内容：

- **Model 结构体定义**：从实际表结构读取，包含正确的 GORM 标签
- **基础 CRUD 操作**：Insert、FindOne、Update、Delete
- **唯一索引查询方法**：根据表的唯一索引自动生成
- **缓存集成**：自动处理 Redis 缓存
- **事务支持**：Transaction 方法

```go
// 自动生成的代码示例（不要编辑！）
type Comment struct {
    Id          int64  `gorm:"column:id"`
    HouseId     int64  `gorm:"column:house_id"`
    OrderId     int64  `gorm:"column:order_id"`
    UserId      int64  `gorm:"column:user_id"`
    // ... 其他字段，从实际表结构读取
}

// 自动生成的方法（不要编辑！）
func (m *defaultCommentModel) Insert(ctx context.Context, data *Comment) error
func (m *defaultCommentModel) FindOne(ctx context.Context, id int64) (*Comment, error)
func (m *defaultCommentModel) FindOneByOrderId(ctx context.Context, orderId int64) (*Comment, error)
func (m *defaultCommentModel) Update(ctx context.Context, data *Comment) (int64, error)
func (m *defaultCommentModel) Delete(ctx context.Context, id int64) (int64, error)
func (m *defaultCommentModel) Transaction(ctx context.Context, fn func(ctx context.Context) error) error
```

### *_model.go - 可扩展

在此文件中添加自定义查询方法和 GORM 钩子：

```go
//go:generate mockgen -package model -destination mock_user_model.go . UserModel
package model

import (
	"context"
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

### vars.go - 自动生成（DO NOT EDIT）

此文件包含接口扩展定义，**请勿手动编辑**。

```go
var ErrNotFound = gorm.ErrRecordNotFound

var _ CommentModel = (*customCommentModel)(nil)

type (
	CommentModel interface {
		commentModel
		FindByHouseID(ctx context.Context, houseID uint64, page, pageSize int) ([]*Comment, int64, error)
		FindByUserID(ctx context.Context, userID uint64, page, pageSize int) ([]*Comment, int64, error)
	}

	customCommentModel struct {
		*defaultCommentModel
	}
)
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

在 `*_model.go` 中添加局部更新方法：

```go
// UpdateMobile 更新手机号
func (m *customUserModel) UpdateMobile(ctx context.Context, userID int64, mobile string) error {
	return m.updatePartial(ctx, userID, map[string]interface{}{
		"mobile": mobile,
	}, map[string]interface{}{})
}
```

调用私有方法 `updatePartial`（定义在 `*_model_gen.go` 中）实现局部字段更新。

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
