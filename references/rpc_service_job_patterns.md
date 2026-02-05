# RPC Service Job 定时任务开发模式

Job（定时任务）是 RPC 服务的重要组成部分，用于执行定时业务逻辑。

## 服务定位

Job 定时任务用于按计划执行业务逻辑：

```
┌─────────────────────────────────────────────────────────────┐
│                    Scheduler                                 │
│              apps/pkg/scheduler                              │
│                         职责：                                │
│  - 调度任务执行                                              │
│  - 支持 Cron 表达式和固定间隔                                 │
│  - 支持分布式选举（etcd）                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Job 任务                                  │
│                    internal/job/xxx_job.go                  │
│                         职责：                                │
│  - 执行定时业务逻辑                                            │
│  - 调用 Logic 层处理业务                                       │
└─────────────────────────────────────────────────────────────┘
```

## 部署模式

| 模式 | 说明 | 适用场景 |
|-----|------|---------|
| **独立部署** | Job 在独立的进程中运行 | 业务繁重、需要独立扩缩容 |
| **合并部署** | Job 与 gRPC 在同一进程中运行 | 业务量不大、资源节省 |

### 合并部署示例

**文件**：`cmd/grpc/main.go`（与 gRPC 合并）

```go
func main() {
    // ... 初始化

    // 启动 gRPC Server
    grpcServer := server.NewGrpcServer(c, svcCtx)
    go grpcServer.Start()  // 注意：使用 goroutine

    // 启动 Job Server（可选，根据业务需求）
    jobServer := server.NewJobServer(c, svcCtx)
    jobServer.Start()

    // ... 优雅关闭
    proc.AddShutdownListener(func() {
        grpcServer.Stop()
        jobServer.Stop()
    })()
}
```

## 任务类型

| 类型 | 常量 | 说明 | 示例 |
|-----|------|------|------|
| **固定间隔** | `scheduler.Duration` | 按固定时间间隔执行 | 每 5 秒执行一次 |
| **Cron 表达式** | `scheduler.Cron` | 使用 Cron 表达式 | `0 * * * * *` 每分钟执行 |

### Cron 表达式格式

```
┌─────────── 秒 (0-59)
│ ┌───────── 分 (0-59)
│` │ ┌─────── 时 (0-23)
│ │ │ ┌───── 日 (1-31)
│ │ │ │ ┌─── 月 (1-12)
│ │ │ │ │ ┌─ 星期 (0-6, 0=周日)
│ │ │ │ │ │
* * * * * *
```

常用示例：
- `0 * * * * *` - 每分钟第 0 秒执行
- `0 0 * * * *` - 每小时第 0 分 0 秒执行
- `0 0 0 * * *` - 每天零点执行
- `0 0 2 * * *` - 每天凌晨 2 点执行
- `0 0 0 1 * *` - 每月 1 号零点执行
- `0 0 0 * * 1` - 每周一零点执行

## 开发步骤

| 步骤 | 操作 | 说明 |
|-----|------|-----|
| 1 | 配置任务 | 在 `internal/config/config.go` 中添加 JobConfig |
| 2 | 创建 Job Server | 在 `internal/server/job.go` 中创建 Job Server |
| 3 | 实现 Handler | 在 `internal/job/xxx_job.go` 中实现 Handler（调用 Logic） |
| 4 | 实现 Logic | 在 `internal/logic/job/xxx_logic.go` 中实现业务逻辑 |
| 5 | 创建入口 | 在 `cmd/job/main.go` 中创建入口或合并选择 |

## 服务内部结构

```
app/services/{type}/{service}/
├── cmd/
│   ├── job/              # Job 入口（可选）
│   │   ├── main.go
│   │   └── conf.yaml
│   └── grpc/             # gRPC 入口（可合并 Job）
└── internal/
    ├── config/           # 配置定义
    │   └── config.go     # 添加 JobConfig
    ├── job/               # Job Handler（调用 Logic）
    │   └── xxxxx_job.go
    ├── logic/
    │   └── job/          # Job 业务逻辑层
    │       └── xxx_logic.go
    ├── repository/       # 依赖注入层（与 gRPC 共享）
    │   ├── model/        # 数据库 Model
    │   ├── rpc/          # RPC 客户端
    │   ├── xredis/       # Redis 操作（包装层）
    │   │   └── job_lock_model.go  # Job 相关 Redis 操作
    │   ├── repository.go
    │   └── repository_type.go
    ├── server/           # Server 初始化
    │   └── job.go        # Job Server 实现
    └── svc/              # ServiceContext
```

## 任务方法签名

### 标准 Handler 签名

```go
func (j *XxxJob) JobName(ctx context.Context) error {
    // 调用 Logic 层处理业务
    xxxLogic := logic.NewXxxLogic(ctx, j.svcCtx)
    return xxxLogic.Method()
}
```

**Handler 层只负责：**
- 创建 Logic 实例
- 调用 Logic 方法
- 返回结果

**业务逻辑在 Logic 层实现：** `internal/logic/job/xxx_logic.go`

| 参数 | 类型 | 说明 |
|-----|------|------|
| `ctx` | `context.Context` | 请求上下文 |

## 任务执行最佳实践

### 1. 错误处理

**Handler 层错误处理：**
- Logic 层返回的错误：直接返回

**Logic 层错误处理：**
- 业务错误：使用业务定义的错误码，`xerr.NewMsgWithInfoLog`
- 系统错误：使用系统错误码，`xerr.NewWithErrorLog`

### 2. 分页处理大批量数据（Logic 层示例）

```go
// internal/logic/job/archive_user_logic.go
func (l *ArchiveUserLogic) ArchiveUser() error {
    const batchSize = 1000
    var lastID int64 = 0

    for {
        users, err := l.userModel.FindByPage(l.ctx, lastID, batchSize)
        if err != nil {
            return xerr.NewWithErrorLog(
                xerr.SystemError_D_ERROR,
                "find users by page failed, err: %v", err,
            )
        }

        if len(users) == 0 {
            break
        }

        // 处理当前批次
        for _, user := range users {
            if err := l.archiveOneUser(user); err != nil {
                xlog.Errorw(l.ctx, "archive user failed", "user_id", user.ID, "err", err)
            }
        }

        // 更新游标
        lastID = users[len(users)-1].ID
    }

    return nil
}
```

### 3. 使用分布式锁防止并发执行（Logic 层示例）

**重要**：使用 `repository/xredis` 中包装的 `AcquireLock` 和 `ReleaseLock` 方法，不要直接操作 Redis。

```go
// internal/logic/job/sync_user_logic.go
func (l *SyncUserLogic) SyncUser() error {
    // 获取分布式锁（使用 xredis 包装方法）
    lockKey := "job:sync_user"
    locked, err := l.jobLockRdsModel.AcquireLock(l.ctx, lockKey, 10*60)
    if err != nil {
        return xerr.NewWithErrorLog(
            xerr.SystemError_D_ERROR,
            "acquire lock failed, lock_key: %s, err: %v", lockKey, err,
        )
    }
    if !locked {
        xlog.Infow(l.ctx, "job is already running", "lock_key", lockKey)
        return nil
    }

    // 确保锁被释放（使用 xredis 包装方法）
    defer l.jobLockRdsModel.ReleaseLock(l.ctx, lockKey)

    // 执行业务逻辑
    return l.doSync()
}
```

### 4. 任务超时控制（Logic 层示例）

```go
// internal/logic/job/sync_user_logic.go
func (l *SyncUserLogic) SyncUser() error {
    // 设置超时上下文
    ctx, cancel := context.WithTimeout(l.ctx, 5*time.Minute)
    defer cancel()

    // 执行业务逻辑
    return l.doSyncWithContext(ctx)
}
```

### 5. 任务执行时间记录（Logic 层示例）

```go
// internal/logic/job/sync_user_logic.go
func (l *SyncUserLogic) SyncUser() error {
    start := time.Now()
    defer func() {
        duration := time.Since(start)
        xlog.Infow(l.ctx, "job finished",
            "job_name", "SyncUserLogic",
            "duration", duration,
        )
    }()

    // 执行业务逻辑
    return l.doSync()
}
```

### 6. 设置任务状态（使用 xredis 包装方法）

```go
// internal/logic/job/sync_user_logic.go
func (l *SyncUserLogic) SetJobStatus(status string) error {
    // 使用 xredis 包装方法设置任务状态
    return l.jobLockRdsModel.SetJobStatus(l.ctx, "sync_user", status, 3600)
}

func (l *SyncUserLogic) GetJobStatus() (string, error) {
    // 使用 xredis 包装方法获取任务状态
    return l.jobLockRdsModel.GetJobStatus(l.ctx, "sync_user")
}
```

## repository/xredis 包装方法说明

在 `repository/xredis/job_lock_model.go` 中定义 Job 相关的 Redis 操作包装方法：

```go
package xredis

import (
    "context"
    "fmt"
    "time"

    "github.com/spf13/cast"
    "github.com/zeromicro/go-zero/core/stores/redis"
)

const (
    jobLockKey   = "job:lock:%s"
    jobStatusKey = "job:status:%s"
)

//go:generate mockgen -package xredis -destination mock_job_lock_model.go -source job_lock_model.go JobLockModel
type JobLockModel interface {
    // 分布式锁相关
    AcquireLock(ctx context.Context, key string, expireSeconds int) (bool, error)
    ReleaseLock(ctx context.Context, key string) error

    // 任务状态相关
    SetJobStatus(ctx context.Context, jobName string, status string, expireSeconds int) error
    GetJobStatus(ctx context.Context, jobName string) (string, error)
}

type jobLockModel struct {
    client *redis.Redis
    prefix string
}

// NewJobLockModel 创建 Job 锁模型
func NewJobLockModel(client *redis.Redis, prefix string) JobLockModel {
    return &jobLockModel{client: client, prefix: prefix}
}

func (j *jobLockModel) formatKey(key string) string {
    return j.prefix + ":" + key
}

// AcquireLock 获取分布式锁
func (j *jobLockModel) AcquireLock(ctx context.Context, key string, expireSeconds int) (bool, error) {
    lockKey := j.formatKey(fmt.Sprintf(jobLockKey, key))
    return j.client.SetnxExCtx(ctx, lockKey, cast.ToString(time.Now().Unix()), expireSeconds)
}

// ReleaseLock 释放分布式锁
func (j *jobLockModel) ReleaseLock(ctx context.Context, key string) error {
    lockKey := j.formatKey(fmt.Sprintf(jobLockKey, key))
    return j.client.DelCtx(ctx, lockKey)
}

// SetJobStatus 设置任务状态
func (j *jobLockModel) SetJobStatus(ctx context.Context, jobName string, status string, expireSeconds int) error {
    statusKey := j.formatKey(fmt.Sprintf(jobStatusKey, jobName))
    return j.client.SetexCtx(ctx, statusKey, status, expireSeconds)
}

// GetJobStatus 获取任务状态
func (j *jobLockModel) GetJobStatus(ctx context.Context, jobName string) (string, error) {
    statusKey := j.formatKey(fmt.Sprintf(jobStatusKey, jobName))
    val, err := j.client.GetCtx(ctx, statusKey)
    if err != nil && !errors.Is(err, redis.Nil) {
        return "", err
    }
    return val, nil
}
```

**在 repository_type.go 中定义类型别名（XXXRdsModel 规则）：**

```go
package repository

import "apps/app/services/core/user/internal/repository/xredis"

type (
    // JobLockRdsModelType Job 锁 Redis 模型类型别名
    JobLockRdsModelType = xredis.JobLockModel
)
```

**在 repository.go 中注入：**

```go
package repository

import (
    "apps/app/services/core/user/internal/config"
    "apps/app/services/core/user/internal/repository/xredis"
    "github.com/zeromicro/go-zero/core/stores/redis"
)

type Repository struct {
    JobLockRdsModel JobLockRdsModelType
}

func NewRepository(c config.Config) *Repository {
    redisDB := redis.MustNewRedis(c.Redis.RedisConf)
    return &Repository{
        JobLockRdsModel: xredis.NewJobLockModel(redisDB, c.Redis.Key),
    }
}
```

## 单元测试

### Handler 测试

```go
func TestUserJob_SyncUserJob(t *testing.T) {
    mockSvcCtx := &svc.ServiceContext{
        // 设置 Mock 依赖
    }

    userJob := NewUserJob(mockSvcCtx)
    ctx := context.Background()

    err := userJob.SyncUserJob(ctx)
    assert.NoError(t, err)
}
```

### Logic 测试

```go
func TestSyncUserLogic_SyncUser(t *testing.T) {
    // 创建 Mock 依赖
    mockUserModel := &mock.MockUserModel{}
    mockJobLockModel := &mock.MockJobLockModel{}

    mockSvcCtx := &svc.ServiceContext{
        Repo: &repository.Repository{
            UserModel:      mockUserModel,
            JobLockRdsModel: mockJobLockModel,
        },
    }

    syncUserLogic := NewSyncUserLogic(context.Background(), mockSvcCtx)

    err := syncUserLogic.SyncUser()
    assert.NoError(t, err)
}
```

## 相关示例

| 文件 | 说明 |
|-----|------|
| [handler.md](../example/rpc_service/job/handler.md) | Job Handler 示例 |
| [logic.md](../example/rpc_service/job/logic.md) | Job Logic 示例 |
| [repository.md](../example/rpc_service/job/repository.md) | Job Repository 层示例 |
| [config.md](../example/rpc_service/job/config.md) | Job Config 示例 |
| [server.md](../example/rpc_service/job/server.md) | Job Server 示例 |
| [svc_context.md](../example/rpc_service/job/svc_context.md) | ServiceContext 初始化（与 gRPC 共享） |
