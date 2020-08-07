# Container相关代码调查分析

## 源代码范围

```go
module.node.container
service.public.container.*
```

## 模块1

module.node.container

**源代码**

```go
package node

import (
	"github.com/nspcc-dev/neofs-node/lib/acl"
	libcnr "github.com/nspcc-dev/neofs-node/lib/container"
	svc "github.com/nspcc-dev/neofs-node/modules/bootstrap"
	"github.com/nspcc-dev/neofs-node/services/public/container"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type cnrParams struct {
	dig.In

	Logger *zap.Logger

	Healthy svc.HealthyClient

	ExtendedACLStore acl.BinaryExtendedACLStore

	ContainerStorage libcnr.Storage
}

func newContainerService(p cnrParams) (container.Service, error) {
	return container.New(container.Params{
		Logger:           p.Logger,
		Healthy:          p.Healthy,
		Store:            p.ContainerStorage,
		ExtendedACLStore: p.ExtendedACLStore,
	})
}

```

**结构分析**

```go
1.定义cnrParams参数结构体
type cnrParams struct {
	dig.In

	Logger *zap.Logger

	Healthy svc.HealthyClient

	ExtendedACLStore acl.BinaryExtendedACLStore

	ContainerStorage libcnr.Storage
}

2.声明ContainerService的构建函数
func newContainerService(p cnrParams) (container.Service, error)
创建一个Container service

```



## 模块2

a)

/service/public/container/service

b)

/service/public/container/alias

c)

/service/public/container/delete

/service/public/container/get

/service/public/container/list

/service/public/container/put

/service/public/container/acl



**结构分析**

```go
a) service.go
1.定义container service 接口
    //接口
	Service interface {
		grpc.Service
		container.ServiceServer
	}
    //实现类
	cnrService struct {
		log *zap.Logger

		healthy HealthChecker

		cnrStore libcnr.Storage

		aclStore acl.BinaryExtendedACLStore
	}
//接口实现，实现Service中grpc.Service相关接口的方法
func (cnrService) Name() string { return "ContainerService" }
作用：返回服务名称，"ContainerService"
func (s cnrService) Register(g *grpc.Server) { container.RegisterServiceServer(g, s) }
作用：注册GRPC服务

2.参数结构体
	// HealthChecker is an interface of node healthiness checking tool.
	HealthChecker interface {
		Healthy() error
	}
    //Container服务构造函数所需要的参数结构体
	Params struct {
		Logger *zap.Logger
		Healthy HealthChecker
		Store libcnr.Storage
		ExtendedACLStore acl.BinaryExtendedACLStore
	}

3.定义构造函数
func New(p Params) (Service, error)
作用：构造函数

b)alias.go
定义3个别名
type CID = refs.CID//容器ID别名
type OwnerID = refs.OwnerID//ownerID别名
type Container = container.Container//容器别名

c)delete/put/get/list/acl.go
实现service接口中container.ServiceServer相关方法的实现

func (s cnrService) Get(ctx context.Context, req *container.GetRequest) (*container.GetResponse, error)
作用：获取container实例

func (s cnrService) Put(ctx context.Context, req *container.PutRequest) (*container.PutResponse, error)
作用：向内环节点中添加新的容器，如果有足够的存储容量

func (s cnrService) Delete(ctx context.Context, req *container.DeleteRequest) (*container.DeleteResponse, error)
作用：从内环节点container存储中删除某个容器

func (s cnrService) List(ctx context.Context, req *container.ListRequest) (*container.ListResponse, error)
作用：返回所有容器

func (s cnrService) SetExtendedACL(ctx context.Context, req *container.SetExtendedACLRequest) (*container.SetExtendedACLResponse, error)
作用：设置container的扩展ACL规则
伪代码逻辑：
1)健康状态检查
2)参数检查
3)创建key/value，并序列化
4)存储键值对，并返回response结果

func (s cnrService) GetExtendedACL(ctx context.Context, req *container.GetExtendedACLRequest) (*container.GetExtendedACLResponse, error) 
作用：获取container的扩展ACL规则
伪代码逻辑：
1)健康状态检查
2)参数检查
3)构建key，查询value
4)构建response结果
```



## 杂项

### ContainerService组件

提供container的CRUD操作

Container是用于定位系统中对象的位置

一个基础的容器至少包含：

对象uuid（包含存储对象的盐）

容器所有人

存储策略：

​       放置规则

​       冗余系数

最大容量



Container描述的是一个完整NetWorkMap的子图（子集）。用户通过定义存储规则（select,filter）会从完整NetWorkMap划分出一个范围，这样的范围都被归集到容器中。



