# Placement相关代码调查分析

## 源代码范围

```go
lib.implementation.placement
```

## 模块1

lib.implementation.placement

**源代码**

```go
package implementations

import (
	"context"

	"github.com/multiformats/go-multiaddr"
	"github.com/nspcc-dev/neofs-api-go/bootstrap"
	"github.com/nspcc-dev/neofs-api-go/container"
	"github.com/nspcc-dev/neofs-api-go/object"
	"github.com/nspcc-dev/neofs-api-go/refs"
	"github.com/nspcc-dev/neofs-node/internal"
	"github.com/nspcc-dev/neofs-node/lib/netmap"
	"github.com/nspcc-dev/neofs-node/lib/placement"
	"github.com/pkg/errors"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

/*
	File source code includes implementations of placement-related solutions.
	Highly specialized interfaces give the opportunity to hide placement implementation in a black box for the reasons:
	  * placement is implementation-tied entity working with graphs, filters, etc.;
	  * NeoFS components are mostly needed in a small part of the solutions provided by placement;
	  * direct dependency from placement avoidance helps other components do not touch crucial changes in placement.
*/

type (
	// CID is a type alias of
	// CID from refs package of neofs-api-go.
	CID = refs.CID

	// SGID is a type alias of
	// SGID from refs package of neofs-api-go.
	SGID = refs.SGID

	// ObjectID is a type alias of
	// ObjectID from refs package of neofs-api-go.
	ObjectID = refs.ObjectID

	// Object is a type alias of
	// Object from object package of neofs-api-go.
	Object = object.Object

	// Address is a type alias of
	// Address from refs package of neofs-api-go.
	Address = refs.Address

	// Netmap is a type alias of
	// NetMap from netmap package.
	Netmap = netmap.NetMap

	// ObjectPlacer is an interface of placement utility.
	ObjectPlacer interface {
		ContainerNodesLister
		ContainerInvolvementChecker
		GetNodes(ctx context.Context, addr Address, usePreviousNetMap bool, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error)
		Epoch() uint64
	}

	// ContainerNodesLister is an interface of container placement vector builder.
	ContainerNodesLister interface {
		ContainerNodes(ctx context.Context, cid CID) ([]multiaddr.Multiaddr, error)
		ContainerNodesInfo(ctx context.Context, cid CID, prev int) ([]bootstrap.NodeInfo, error)
	}

	// ContainerInvolvementChecker is an interface of container affiliation checker.
	ContainerInvolvementChecker interface {
		IsContainerNode(ctx context.Context, addr multiaddr.Multiaddr, cid CID, previousNetMap bool) (bool, error)
	}

	objectPlacer struct {
		pl placement.Component
	}
)

const errEmptyPlacement = internal.Error("could not create storage lister: empty placement component")

// NewObjectPlacer wraps placement.Component and returns ObjectPlacer interface.
func NewObjectPlacer(pl placement.Component) (ObjectPlacer, error) {
	if pl == nil {
		return nil, errEmptyPlacement
	}

	return &objectPlacer{pl}, nil
}

func (v objectPlacer) ContainerNodes(ctx context.Context, cid CID) ([]multiaddr.Multiaddr, error) {
	graph, err := v.pl.Query(ctx, placement.ContainerID(cid))
	if err != nil {
		return nil, errors.Wrap(err, "objectPlacer.ContainerNodes failed on graph query")
	}

	return graph.NodeList()
}

func (v objectPlacer) ContainerNodesInfo(ctx context.Context, cid CID, prev int) ([]bootstrap.NodeInfo, error) {
	graph, err := v.pl.Query(ctx, placement.ContainerID(cid), placement.UsePreviousNetmap(prev))
	if err != nil {
		return nil, errors.Wrap(err, "objectPlacer.ContainerNodesInfo failed on graph query")
	}

	return graph.NodeInfo()
}

func (v objectPlacer) GetNodes(ctx context.Context, addr Address, usePreviousNetMap bool, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error) {
	queryOptions := make([]placement.QueryOption, 1, 2)
	queryOptions[0] = placement.ContainerID(addr.CID)

	if usePreviousNetMap {
		queryOptions = append(queryOptions, placement.UsePreviousNetmap(1))
	}

	graph, err := v.pl.Query(ctx, queryOptions...)
	if err != nil {
		if st, ok := status.FromError(errors.Cause(err)); ok && st.Code() == codes.NotFound {
			return nil, container.ErrNotFound
		}

		return nil, errors.Wrap(err, "placer.GetNodes failed on graph query")
	}

	filter := func(group netmap.SFGroup, bucket *netmap.Bucket) *netmap.Bucket {
		return bucket
	}

	if !addr.ObjectID.Empty() {
		filter = func(group netmap.SFGroup, bucket *netmap.Bucket) *netmap.Bucket {
			return bucket.GetSelection(group.Selectors, addr.ObjectID.Bytes())
		}
	}

	return graph.Exclude(excl).Filter(filter).NodeList()
}

func (v objectPlacer) IsContainerNode(ctx context.Context, addr multiaddr.Multiaddr, cid CID, previousNetMap bool) (bool, error) {
	nodes, err := v.GetNodes(ctx, Address{
		CID: cid,
	}, previousNetMap)
	if err != nil {
		return false, errors.Wrap(err, "placer.FromContainer failed on placer.GetNodes")
	}

	for i := range nodes {
		if nodes[i].Equal(addr) {
			return true, nil
		}
	}

	return false, nil
}

func (v objectPlacer) Epoch() uint64 { return v.pl.NetworkState().Epoch }

```

**结构分析**

```go
1.定义objectPlacer结构体、接口和相关实现
    //结构体，objectPlacer是placement组件的封装类
	objectPlacer struct {
		pl placement.Component
	}
    //接口
	ObjectPlacer interface {
		ContainerNodesLister
		ContainerInvolvementChecker
		GetNodes(ctx context.Context, addr Address, usePreviousNetMap bool, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error)
		Epoch() uint64
	}
	ContainerNodesLister interface {
		ContainerNodes(ctx context.Context, cid CID) ([]multiaddr.Multiaddr, error)
		ContainerNodesInfo(ctx context.Context, cid CID, prev int) ([]bootstrap.NodeInfo, error)
	}

	ContainerInvolvementChecker interface {
		IsContainerNode(ctx context.Context, addr multiaddr.Multiaddr, cid CID, previousNetMap bool) (bool, error)
	}
    //接口实现
func (v objectPlacer) ContainerNodes(ctx context.Context, cid CID) ([]multiaddr.Multiaddr, error) 
作用：获取objectPlacer内部placement所包含的节点列表

func (v objectPlacer) ContainerNodesInfo(ctx context.Context, cid CID, prev int) ([]bootstrap.NodeInfo, error)
作用：获取objectPlacer内部placement所包含的节点信息列表

func (v objectPlacer) GetNodes(ctx context.Context, addr Address, usePreviousNetMap bool, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error)
作用：获取子图中有效节点列表
伪代码逻辑：
1)构建查询参数
2)查询获取子图
3)子图排除节点并执行过滤后，返回剩余节点列表

func (v objectPlacer) IsContainerNode(ctx context.Context, addr multiaddr.Multiaddr, cid CID, previousNetMap bool) (bool, error)
作用：判读objectPlacer是否包含指定节点
伪代码逻辑：
1)从objectPlacer中获取所有涉及的节点
2)遍历节点，判断是否包含指定节点的地址

func (v objectPlacer) Epoch() uint64
作用：返回objectPlacer的时隙

2.别名定义
	CID = refs.CID//容器ID
	SGID = refs.SGID//
	ObjectID = refs.ObjectID//
	Object = object.Object//
	Address = refs.Address//对象地址 (container id + object id)
	Netmap = netmap.NetMap//节点网络拓扑
```



## 杂项

### 组件结构

ObjectPlacer是placement组件的包裹类，内置一个placement组件。

```c#
ObjectPlacer{//单例
   Placement{
      nmStore    map<时隙，NetMap> 保存各时隙的NetMap
      interface 与NetMap相关的CRUD操作{
          a)跟新某个时隙对应的NetMap
          b)查询NetMap
          c)获取某个时隙下，与自身节点有关联关系的节点列表
          d)获取某个时隙的NetMap
      }
   }
   interface  container/node相关的查找操作{
              a)查询某个容器涉及的节点地址列表,返回Multiaddr[]
              b)查询某个容器涉及的节点信息列表,返回NodeInfo[]
              c)查询某个地址对应的某个容器涉及的节点列表,返回Multiaddr[]
              c)判断某个节点是否是某个容器相关的节点,返回bool
              d)查询当前时隙，返回uint64
   }   
}

Graph{//普通对象，非单例
  roots  Bucket[]      图涉及的顶层标签
  items  NodeInfo[]    
  place  PlacementRule 存储规则(过滤器)
  
  interface Graph中node和子图的查询方法{
      a)使用过滤器生成子图
      b)踢出某些节点生成子图
      c)获取图涉及的节点列表
      d)获取图涉及的节点信息列表
  }
}
```





