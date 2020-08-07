# Session相关代码调查分析

## 源代码范围

```go
module.node.session
service.public.session.*
```

## 模块1

module.node.session

**源代码**

```go
package node

import (
	"github.com/nspcc-dev/neofs-node/lib/implementations"
	"github.com/nspcc-dev/neofs-node/services/public/session"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type sessionParams struct {
	dig.In

	Logger *zap.Logger

	TokenStore session.TokenStore

	EpochReceiver implementations.EpochReceiver
}

func newSessionService(p sessionParams) (session.Service, error) {
	return session.New(session.Params{
		TokenStore:    p.TokenStore,
		Logger:        p.Logger,
		EpochReceiver: p.EpochReceiver,
	}), nil
}


```

**结构分析**

```go
1.定义sessionParams参数结构体
type sessionParams struct {
	dig.In

	Logger *zap.Logger

	TokenStore session.TokenStore

	EpochReceiver implementations.EpochReceiver
}

2.声明SessionService的构建函数
func newSessionService(p sessionParams) (session.Service, error)
创建一个Session service

```



## 模块2

a)

/service/public/session/service.go

b)

/service/public/session/create.go

**结构分析**

```go
a) service.go
1.定义sessionService结构体以及相关接口和实现
	sessionService struct {
		ts  TokenStore
		log *zap.Logger

		epochReceiver EpochReceiver
	}

    //接口
	Service interface {
		grpc.Service
		container.ServiceServer
	}
	EpochReceiver interface {
		Epoch() uint64
	}

//接口实现，实现Service接口中grpc.Service相关接口的方法
func (sessionService) Name() string 
作用：返回服务名称，"Session Server"
func (s sessionService) Register(srv *grpc.Server) 
作用：注册GRPC服务

2.参数结构体
	Params struct {
		TokenStore TokenStore

		Logger *zap.Logger

		EpochReceiver EpochReceiver
	}

3.声明构造函数New(p Params)
func New(p Params) Service
作用：创建一个SessionService服务

b) create.go
//接口实现，实现Service接口中container.ServiceServer相关接口的方法
func (s sessionService) Create(ctx context.Context, req *CreateRequest) (*CreateResponse, error)
作用：创建client和server之间的session
伪代码逻辑：
1)比较请求参数中的到期时隙和sessionService服务的当前时隙，判断是否过期
2)生成session的私钥令牌
3)生成session的公钥令牌
4)生成sessionID
5)存储session的ID，公钥，私钥令牌
6)构建response，发送ID和公钥令牌
```



## 杂项







