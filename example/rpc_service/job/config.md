# Job Config 配置示例

Job 定时任务的配置定义。

## 文件位置

`internal/config/config.go`

## 代码示例

```go
package config

import (
	"github.com/zeromicro/go-zero/core/service"
)

// JobConfig 包含 Job 定时任务配置
type JobConfig struct {
	service.ServiceConf  // go-zero 基础服务配置
	Config Config        // 业务配置
}

// Config 包含业务配置
type Config struct {
	DB      DBConf
	DBCache cache.CacheConf
	RPC     RPCConf
}
```

## 配置文件示例

`cmd/job/conf.yaml`

```yaml
Name: user-job
Host: 0.0.0.0
Port: 8888
Mode: dev
Timeout: 10000

Config:
  DB:
    Mysql: "root:password@tcp(127.0.0.1:3306)/services_core_user?charset=utf8mb4&parseTime=true"
  DBCache:
    Host: "127.0.0.1:6379"
    Type: node
  RPC:
    AuthzSrvConf:
      Endpoints:
        - 127.0.0.1:10001
```

## 任务注册说明

任务不是通过配置文件定义，而是在 `internal/server/job.go` 中通过代码注册：

```go
jobs := []scheduler.Job{
	{
		Name:     "userJob.OneHandler",
		Type:     scheduler.Duration,    // 固定间隔类型
		ExecTime: time.Second * 5,      // 每 5 秒执行
		ExecFunc: userJob.OneHandler,
	},
	{
		Name:     "userJob.TwoHandler",
		Type:     scheduler.Cron,         // Cron 表达式类型
		ExecTime: "0 * * * * *",        // 每分钟执行
		ExecFunc: userJob.TwoHandler,
	},
}
```

## Job 结构说明

| 字段 | 类型 | 说明 |
|-----|------|------|
| `Name` | `string` | 任务名称（唯一标识） |
| `Type` | `JobType` | 任务类型：`scheduler.Duration` 或 `scheduler.Cron` |
| `ExecTime` | `interface{}` | 执行时间：`time.Duration` 或 `string` (Cron 表达式) |
| `ExecFunc` | `func(ctx context.Context) error` | 任务执行函数 |

## Cron 表达式格式

```
┌─────────── 秒 (0-59)
│ ┌───────── 分 (0-59)
│ │ ┌─────── 时 (0-23)
│ │ │ ┌───── 日 (1-31)
│ │ │ │ ┌─── 月 (1-12)
│ │ │ │ │ ┌─ 星期 (0-6, 0=周日)
│ │ │ │ │ │
* * * * * *
```

### 常用 Cron 表达式

| 表达式 | 说明 |
|--------|------|
| `0 * * * * *` | 每分钟第 0 秒执行 |
| `0 */5 * * * *` | 每 5 分钟执行 |
| `0 0 * * * *` | 每小时第 0 分 0 秒执行 |
| `0 0 */2 * * *` | 每 2 小时执行 |
| `0 0 0 * * *` | 每天零点执行 |
| `0 0 2 * * *` | 每天凌晨 2 点执行 |
| `0 0 0 * * 1` | 每周一零点执行 |
| `0 0 0 1 * *` | 每月 1 号零点执行 |
