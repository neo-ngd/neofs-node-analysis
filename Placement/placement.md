# Placement相关代码调查分析

## 源代码范围

```go
module.network.placement
lib.placement.*
```

## 模块1

module.network.placement

**源代码**

```go
package network

import (
	"github.com/nspcc-dev/neofs-node/lib/blockchain/event"
	netmapevent "github.com/nspcc-dev/neofs-node/lib/blockchain/event/netmap"
	libcnr "github.com/nspcc-dev/neofs-node/lib/container"
	"github.com/nspcc-dev/neofs-node/lib/netmap"
	"github.com/nspcc-dev/neofs-node/lib/peers"
	"github.com/nspcc-dev/neofs-node/lib/placement"
	"github.com/nspcc-dev/neofs-node/modules/morph"
	"github.com/nspcc-dev/neofs-node/services/public/state"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type (
	placementParams struct {
		dig.In

		Log     *zap.Logger
		Peers   peers.Store
		Fetcher libcnr.Storage

		MorphEventListener event.Listener

		NetMapStorage netmap.Storage

		MorphEventHandlers morph.EventHandlers
	}

	placementOutput struct {
		dig.Out

		Placement placement.Component
		Healthy   state.HealthChecker `group:"healthy"`
	}
)

const defaultChronologyDuraion = 2

func newPlacement(p placementParams) placementOutput {
	place := placement.New(placement.Params{
		Log:                p.Log,
		Peerstore:          p.Peers,
		Fetcher:            p.Fetcher,
		ChronologyDuration: defaultChronologyDuraion,
	})

	if handlerInfo, ok := p.MorphEventHandlers[morph.ContractEventOptPath(
		morph.NetmapContractName,
		morph.NewEpochEventType,
	)]; ok {
		handlerInfo.SetHandler(func(ev event.Event) {
			nmRes, err := p.NetMapStorage.GetNetMap(netmap.GetParams{})
			if err != nil {
				p.Log.Error("could not get network map",
					zap.String("error", err.Error()),
				)
				return
			}

			if err := place.Update(
				ev.(netmapevent.NewEpoch).EpochNumber(),
				nmRes.NetMap(),
			); err != nil {
				p.Log.Error("could not update network map in placement component",
					zap.String("error", err.Error()),
				)
			}
		})

		p.MorphEventListener.RegisterHandler(handlerInfo)
	}

	return placementOutput{
		Placement: place,
		Healthy:   place.(state.HealthChecker),
	}
}
```

**结构分析**

```go
1.定义placement的输入参数和输出结构体
	placementParams struct {
		dig.In

		Log     *zap.Logger
		Peers   peers.Store
		Fetcher libcnr.Storage

		MorphEventListener event.Listener

		NetMapStorage netmap.Storage

		MorphEventHandlers morph.EventHandlers
	}

	placementOutput struct {
		dig.Out

		Placement placement.Component
		Healthy   state.HealthChecker `group:"healthy"`
	}

2.声明placement的构建函数
func newPlacement(p placementParams) placementOutput
伪代码逻辑：
a)利用placementParams参数生成Component
b)构建并注册一个netmap合约的new_epoch事件的回调函数，
当一个new_epoch事件发生时，会获取当前epoch的多对应的netmap图，并更新已有netmap图

```



## 模块2

lib.placement.*

**源代码**

a) interface.go

b) graph.go

c) neighbours.go

d) placement.go

e) store.go

**结构分析**

```go
a)interface.go
1.定义Component组件接口和相关结构体
	Component interface {
		NetworkState() *bootstrap.SpreadMap
		Neighbours(seed, epoch uint64, full bool) []peers.ID
		Update(epoch uint64, nm *netmap.NetMap) error
		Query(ctx context.Context, opts ...QueryOption) (Graph, error)
	}

	QueryOptions struct {
		CID      refs.CID //容器ID
		Previous int//时隙
		Excludes []multiaddr.Multiaddr//排除的节点地址
	}

	Params struct {	// component所需的参数的结构体
		Log                *zap.Logger
		Netmap             *netmap.NetMap
		Peerstore          peers.Store
		Fetcher            container.Storage
		ChronologyDuration uint64 // storing number of past epochs states
	}

	placement struct {//Component实现类结构体
		log *zap.Logger
		cnr container.Storage

		chronologyDur uint64
		nmStore       *netMapStore

		ps peers.Store
		healthy *atomic.Bool
	}

2.定义graph接口和结构体
    //图的结构体
	graph struct {
		roots []*netmap.Bucket//图中所包含的bucket,bucket代表一个范围，例如国家、城市，bucket可以包含子bucket,呈树形。
		items []bootstrap.NodeInfo//图中所有节点的节点信息，由grpc自动生成
		place *netmap.PlacementRule//存储规则
	}
	Graph interface {
		Filter(rule FilterRule) Graph//过滤器，按照过滤规则（自定义函数），生成新的子图
		Exclude(list []multiaddr.Multiaddr) Graph//排除某些节点后的子图
		NodeList() ([]multiaddr.Multiaddr, error)//获取图中涉及节点的Multiaddr地址
		NodeInfo() ([]bootstrap.NodeInfo, error)//获取图中涉及节点的信息
	}

3.定义networkState结构体和方法，代表某个时隙的netmap
	networkState struct {
		nm    *netmap.NetMap//netmap
		epoch uint64  //时隙
	}
func (ns networkState) Copy() *networkState

4.定义其他方法
func ExcludeNodes(list []multiaddr.Multiaddr) QueryOption
作用:创建排除某些节点的查询参数
func ContainerID(cid refs.CID) QueryOption
作用：创建是否包含某个containerId的查询参数
func UsePreviousNetmap(diff int) QueryOption
作用：查询前一时隙的NetMap的查询参数



b)graph.go //graph接口和结构体的实现类
实现Graph接口中的方法
function Filter(rule FilterRule) Graph
作用：按照过滤规则，过滤原始图中的元素
伪代码逻辑：
1）如果规则为空，直接返回原始图
2）遍历原始图中设置的过滤器组，并将bucket元素使用这些过滤器过滤
3）利用过滤后的bucket，构建新的图

function Exclude(list []multiaddr.Multiaddr) Graph
作用：将原始图中的元素排除指定元素后生成新的子图

function NodeList() ([]multiaddr.Multiaddr, error)
作用：获取图中每个容器的节点Multiaddr地址列表
伪代码逻辑：
for 遍历图中所有的bucket{
    for遍历每个bucket的节点列表
       关联节点信息，并使用转换函数将节点地址转换成multiaddr格式地址 
}
输出节点multiaddr格式地址数组。

function NodeInfo() ([]bootstrap.NodeInfo, error)
作用：获取图中每个容器的节点信息
伪代码逻辑：
for 遍历图中所有的bucket{
    for遍历每个bucket的节点列表
       关联节点信息  
}
输出节点关联信息数组[]。
例如：图中有4个节点，有A、B 2个bucket，A包含1，3号节点，B包含2，4节点
则输出【1号节点信息、3号节点信息、2号节点信息、4号节点信息】



c)placement.go
placement接口方法实现
1.实现Component接口方法
func (p *placement) Update(epoch uint64, nm *netmap.NetMap) error
func (p *placement) NetworkState() *bootstrap.SpreadMap
func (p *placement) Query(ctx context.Context, opts ...QueryOption) (Graph, error)

2.定义函数ContainerGraph
func ContainerGraph(nm *netmap.NetMap, rule *netmap.PlacementRule, ignore []uint32, cid refs.CID) (Graph, error)
作用：通过在netMap中使用placementRule规则来生成容器地图
伪代码逻辑：
1)从netmap中按照分组规则，生成相应的bucket组


d)neighbours.go
1.定义函数calculateCount
func calculateCount(n int) int
作用：用于计算临近节点个数
伪代码逻辑：count=取整(1.4*log(n)+9)+1

2.定义函数Neighbours
func (p *placement) Neighbours(seed, epoch uint64, full bool) []peers.ID
作用：按照hrw计算临近节点
伪代码逻辑：
1)获取当前时隙的netmap
2)通过listPeers生成metmap中节点的编号
3)使用hrw对节点编号排序
4)利用calculateCount对节点编号截取

3.定义函数listPeers
func (p *placement) listPeers(nodes netmap.Nodes, exclSelf bool) []peers.ID
作用：利用netmap中节点的公钥生成PerrsID节点编号
伪代码逻辑：
for 遍历netmap中所有节点
   节点公钥生成ID，并判断是否剔除自身的ID
输出ID数组

e)store.go
1.定义NetMap的存储结构体，线程安全，<时隙，netMap>格式存储
	netMapStore struct {
		*sync.RWMutex
		items map[uint64]*NetMap
		curEpoch uint64
	}

2.定义netMapStore的相关操作方法
func newNetMapStore() *netMapStore
作用：构建NetMapStore实例
func (s *netMapStore) put(epoch uint64, nm *NetMap)
作用：指定某个时隙的netMap
func (s *netMapStore) get(epoch uint64) *NetMap
作用：获取某个时隙的netMap
func (s *netMapStore) trim(epoch uint64)
作用：删除指定时隙之前的netMap
func (s *netMapStore) epoch() uint64
作用：返回NetMapStore实例的当前时隙
```



## 杂项

### 组件结构

placement是最外层结构，是一个Component.

placement内部会存有一个netmap

netmap是反应每个时隙，整个网络拓扑状态的东西

netMap呈树状结构，用bucket来划分片区，一个bucket可以代表满足某个标签条件的节点，

一个bucket可以有子bucket,可以递归区分下去。bucket还拥有node,node只能做bucket的叶子节点。



