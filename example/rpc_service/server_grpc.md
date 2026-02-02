# gRPC Server 注册示例

RPC 服务需要将服务实现注册到 gRPC Server。

## 文件位置

`internal/server/grpc.go`

## 代码示例

```go
package server

import (
	"fmt"

	"apps/app/services/shared/authz/internal/config"
	"apps/app/services/shared/authz/internal/service/authzv1"
	"apps/app/services/shared/authz/internal/svc"
	authzV1Pb "apps/pb/services/shared/authz/v1"
	"apps/pkg/grpc/interceptor/server"

	"github.com/zeromicro/go-zero/core/service"
	"github.com/zeromicro/go-zero/zrpc"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

// GrpcServer gRPC 服务端
type GrpcServer struct {
	svcCtx *svc.ServiceContext
	server *zrpc.RpcServer
	conf   config.GrpcConfig
}

// NewGrpcServer 创建 gRPC 服务端
func NewGrpcServer(conf config.GrpcConfig, svcCtx *svc.ServiceContext) *GrpcServer {
	return &GrpcServer{
		svcCtx: svcCtx,
		conf:   conf,
	}
}

// Start 启动服务
func (s *GrpcServer) Start() {
	s.server = zrpc.MustNewServer(s.conf.RpcServerConf, func(grpcServer *grpc.Server) {
		// 注册服务
		authzV1Pb.RegisterAuthzServiceServer(grpcServer, authzv1.NewAuthzService(s.svcCtx))

		// 开发/测试环境开启反射
		if s.conf.Mode == service.DevMode || s.conf.Mode == service.TestMode {
			reflection.Register(grpcServer)
		}
	})

	// 添加拦截器
	s.server.AddUnaryInterceptors(server.UnaryServerInterceptors()...)

	// 启动服务
	fmt.Printf("start srv [%s:%s] success\n", s.conf.Name, s.conf.ListenOn)
	s.server.Start()
}

// Stop 停止服务
func (s *GrpcServer) Stop() {
	fmt.Printf("stop srv [%s:%s] ...\n", s.conf.Name, s.conf.ListenOn)
	s.server.Stop()
	fmt.Printf("stop srv [%s:%s] success\n", s.conf.Name, s.conf.ListenOn)
}
```

## 关键点说明

### 1. 服务注册

```go
authzV1Pb.RegisterAuthzServiceServer(grpcServer, authzv1.NewAuthzService(s.svcCtx))
```

这是唯一需要根据服务名称修改的部分。

| 模板 | 说明 |
|-----|------|
| `{PbPackage}.Register{ServiceName}Server` | Protobuf 生成的注册函数 |
| `grpcServer` | gRPC Server 实例 |
| `{serviceDir}.New{ServiceName}(svcCtx)` | 服务实现工厂函数 |

### 2. 反射服务

开发/测试环境开启 gRPC 反射，方便使用 grpcurl 等工具调试：

```go
if s.conf.Mode == service.DevMode || s.conf.Mode == service.TestMode {
	reflection.Register(grpcServer)
}
```

### 3. 拦截器

添加全局一元拦截器：

```go
s.server.AddUnaryInterceptors(server.UnaryServerInterceptors()...)
```

## 服务注册对照表

| 服务 | PbPackage | ServiceName | ServiceDir |
|-----|-----------|-------------|------------|
| 用户服务 | `userV1Pb` | `UserService` | `service/userv1` |
| 鉴权服务 | `authzV1Pb` | `AuthzService` | `service/authzv1` |
| 权限服务 | `permitV1Pb` | `PermitService` | `service/permitv1` |
