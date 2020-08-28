# SelectiveContainerExecutor

# ObjectsContainerHandler

//描述

依赖

|Field|Type|Description|
|-|-|-|
||\*viper.Viper|  读取配置|
| |\*zap.Logger|  日志输出|
| Placer |implementations.ObjectPlacer |[**lib.implements.ObjectPlacer{placement.Component}**](../Placement/placement.md)|
| PeerStore |peers.Store|[Peers.Store](../Peers/peers.store.md)|
| Peers |peers.Interface|[Peers.Interface](../Peers/peers.interface.md)|
| TimeoutsPrefix | string ||
| key| \*ecdsa.PrivateKey| 自身的priate key|
| TokenStore |session.PrivateTokenStore| **module: session.NewMapTokenStore** |

## AddressStore

依赖peers.Store通过网络地址address后去找public key

| Field | Type        | Description |
| ----- | ----------- | ----------- |
|log|\*zap.Logger|日志输出|
| ps    | peers.Store | peers存储   |

### 方法
 * *SelfAddr() (multiaddr.Multiaddr, error)*

   调用*peers.Store.SelfID()*获取自身的ID，然后使用*peers.Store.GetAddr()*获取自身的地址

 * *PublicKey(mAddr multiaddr.Multiaddr)*

   先调用*peers.AddressID*根据mAddr获取ID，然后调用*peers.GetPublicKey*使用ID获取public key

### 函数

* *NewAddressStore(ps peers.Store, log \*zap.Logger) (AddressStoreComponent, error)*

  使用peers.Store和log创建一个AddressStore实例

## ObjectTransport

依赖Peers.Interface创建grpc连接，从而构建根据neofs-api-go中object/service.proto中编译生成的ObjectServiceClient

依赖
|属性|值|来源|
|-|-|-|
|AddressStore|     as|*NewAddressStore(ps peers.Store, log \*zap.Logger) (AddressStoreComponent, error)*|
|EpochReceiver|   p.Placer|module:{Constructor: newPlacementTool}|
|RemoteService|    object.NewRemoteService(p.Peers)|使用Peers.Interface创建grpc连接|
|Logger|           p.Logger||
|Key|           p.Key||
|PutTimeout|       p.Viper.GetDuration(p.TimeoutsPrefix + ".timeouts.put")|配置中读取|
|GetTimeout|      p.Viper.GetDuration(p.TimeoutsPrefix + ".timeouts.get")|配置中读取|
|HeadTimeout|     p.Viper.GetDuration(p.TimeoutsPrefix + ".timeouts.head")|配置中读取|
|SearchTimeout|    p.Viper.GetDuration(p.TimeoutsPrefix + ".timeouts.search")|配置中读取|
|RangeHashTimeout| p.Viper.GetDuration(p.TimeoutsPrefix + ".timeouts.range_hash")|配置中读取|
|DialTimeout|     p.Viper.GetDuration("object.dial_timeout")|配置中读取|
|PrivateTokenStore| p.TokenStore|module: {Constructor:session.NewMapTokenStore}|

### RemoteService

| Field | Type        | Description |
| ----- | ----------- | ----------- |
|log|\*zap.Logger|日志输出|
| ps    | peers.Interface | 管理连接   |

#### 方法

* *Remote(ctx context.Context, addr multiaddr.Multiaddr) (object.ServiceClient, error)*
  创建grpc连接addr, 返回object.ServiceClient

#### 函数

* *NewRemoteService(ps peers.Interface) RemoteService*
  创建一个RemoteService实例

### TransportComponent

| Field | Type | Description |
|-|-|-|
|reqSender| requestSender|接口类型，实现为[coreRequestSender](#coreRequestSender)|
|resTracker |resultTracker|接口类型，实现为[idleResultTracker](#idleResultTracker)|
|getCaller  |     remoteProcessCaller|接口类型，实现为[getCaller](#getCaller)|
|putCaller   |    remoteProcessCaller|接口类型，实现为[putCaller](#putCaller)|
|headCaller      |remoteProcessCaller|接口类型，实现为[headCaller](#headCaller)|
|rangeCaller   |  remoteProcessCaller|接口类型，实现为[rangeCaller](#rangeCaller)|
|rangeHashCaller| remoteProcessCaller|接口类型，实现为[rangeHashCaller](#rangeHashCaller)|
|searchCaller |   remoteProcessCaller|接口类型，实现为[searchCaller](#searchCaller)|

#### coreRequestSender

|Field|Type|Description|
|-|-|-|
|requestPrep |  transportRequestPreparer| 接口类型，实现为[coreRequestPreparer](#coreRequestPreparer) |
|addressStore | implementations.AddressStoreComponent|[AddressStore](#AddressStore)|
|remoteService| RemoteService|[RemoteService](#RemoteService)|
|putTimeout   |    time.Duration| 配置中读取 |
|getTimeout    |   time.Duration|配置中读取 |
|searchTimeout  |  time.Duration|配置中读取 |
|headTimeout  |    time.Duration|配置中读取 |
|rangeHashTimeout| time.Duration|配置中读取 |
|dialTimeout   |   time.Duration|配置中读取 |

##### coreRequestPreparer

|Field|Type|Description|
| - | - | - |
| epochRecv | EpochReceiver | [**lib.implements.ObjectPlacer{placement.Component}**](../Placement/placement.md) |
| key | \*ecdsa.PrivateKey |本地配置|
| signingFunc | signingFunc |neofs-api-go.service.SignRequestData|
| privateTokenStore | session.PrivateTokenSource | neofs-api-go.session.PrivateTokenStore|

* *prepareRequest(req transport.MetaInfo) (serviceRequest, error)*

  * MetaInfo

  ```go
  MetaInfo interface {
  		GetTTL() uint32
  		GetTimeout() time.Duration
  		service.SessionTokenSource
  		GetRaw() bool
  		Type() object.RequestType
  		service.BearerTokenSource //interface neofs-api-go.service.BearTokenSource
  		service.ExtendedHeadersSource // interface neofs-api-go.service.ExtendHeader
  }
  
  // SessionTokenSource is an interface of the container of a SessionToken with read access.
  type SessionTokenSource interface {
  	GetSessionToken() SessionToken
  }
  
  // ACLRulesSource is an interface of the container of binary extended ACL rules with read access.
  type ACLRulesSource interface {
  	GetACLRules() []byte
  }
  
  // ACLRulesContainer is an interface of the container of binary extended ACL rules.
  type ACLRulesContainer interface {
  	ACLRulesSource
  	SetACLRules([]byte)
  }
  
  // ExpirationEpochSource is an interface of the container of an expiration epoch number with read access.
  type ExpirationEpochSource interface {
  	ExpirationEpoch() uint64
  }
  
  // ExpirationEpochContainer is an interface of the container of an expiration epoch number.
  type ExpirationEpochContainer interface {
  	ExpirationEpochSource
  	SetExpirationEpoch(uint64)
  }
  
  // OwnerIDSource is an interface of the container of an OwnerID value with read access.
  type OwnerIDSource interface {
  	GetOwnerID() OwnerID
  }
  
  // OwnerIDContainer is an interface of the container of an OwnerID value.
  type OwnerIDContainer interface {
  	OwnerIDSource
  	SetOwnerID(OwnerID)
  }
  
  // OwnerKeySource is an interface of the container of owner key bytes with read access.
  type OwnerKeySource interface {
  	GetOwnerKey() []byte
  }
  
  // OwnerKeyContainer is an interface of the container of owner key bytes.
  type OwnerKeyContainer interface {
  	OwnerKeySource
  	SetOwnerKey([]byte)
  }
  
  // OwnerIDSource is an interface of the container of an OwnerID value with read access.
  type OwnerIDSource interface {
  	GetOwnerID() OwnerID
  }
  
  // OwnerIDContainer is an interface of the container of an OwnerID value.
  type OwnerIDContainer interface {
  	OwnerIDSource
  	SetOwnerID(OwnerID)
  }
  
  // SignatureSource is an interface of the container of signature bytes with read access.
  type SignatureSource interface {
  	GetSignature() []byte
  }
  
  // SignatureContainer is an interface of the container of signature bytes.
  type SignatureContainer interface {
  	SignatureSource
  	SetSignature([]byte)
  }
  
  // BearerTokenInfo is an interface of a fixed set of Bearer token information value containers.
  // Contains:
  // - binary extended ACL rules;
  // - expiration epoch number;
  // - ID of the token's owner.
  type BearerTokenInfo interface {
  	ACLRulesContainer
  	ExpirationEpochContainer
  	OwnerIDContainer
  }
  
  // BearerToken is an interface of Bearer token information and key-signature pair.
  type BearerToken interface {
  	BearerTokenInfo
  	OwnerKeyContainer
  	SignatureContainer
  }
  
  type BearerTokenSource interface {
  	GetBearerToken() BearerToken
  }
  
  type ExtendedHeader interface {
  	Key() string
  	Value() string
  }
  
  type ExtendedHeadersSource interface {
  	ExtendedHeaders() []ExtendedHeader
  }
  
  rawMetaInfo struct {
  		raw     bool  //get, set
  		ttl     uint32 //get, set
  		timeout time.Duration //get, set
  		token   service.SessionToken //get, set
  		rt      object.RequestType //get, set
  		bearer  service.BearerToken //get, set
  		extHdrs []service.ExtendedHeader //get, set
  }
  ```
  基于MetaInfo扩展了不同的请求类型
  ```go
  // SearchInfo is an interface of the container of object Search operation parameters.
  SearchInfo interface {
  	MetaInfo
  	GetCID() refs.CID
  	GetQuery() []byte
  }
  
  // PutInfo is an interface of the container of object Put operation parameters.
  PutInfo interface {
  	MetaInfo
  	GetHead() *object.Object
  	Payload() io.Reader
  	CopiesNumber() uint32
  }
  
  // AddressInfo is an interface of the container of object request by Address.
  AddressInfo interface {
  	MetaInfo
  	GetAddress() refs.Address
  }
  
  // GetInfo is an interface of the container of object Get operation parameters.
  GetInfo interface {
  	AddressInfo
  }
  
  // HeadInfo is an interface of the container of object Head operation parameters.
  HeadInfo interface {
  	GetInfo
  	GetFullHeaders() bool
  }
  
  // RangeInfo is an interface of the container of object GetRange operation parameters.
  RangeInfo interface {
  	AddressInfo
  	GetRange() object.Range
  }
  
  // RangeHashInfo is an interface of the container of object GetRangeHash operation parameters.
  RangeHashInfo interface {
  	AddressInfo
  	GetRanges() []object.Range
  	GetSalt() []byte
  }
  ```
  根据不同的请求类型，创建neofs-api-go/object/service.proto定义的消息类型
  ```go
  func prepareSearchRequest(req transport.SearchInfo) serviceRequest 
  func prepareGetRequest(req transport.GetInfo) serviceRequest 
  func prepareHeadRequest(req transport.HeadInfo) serviceRequest 
  func preparePutRequest(req transport.PutInfo) serviceRequest 
  func prepareRangeHashRequest(req transport.RangeHashInfo) serviceRequest 
  func prepareRangeRequest(req transport.RangeInfo) serviceRequest 
  ```
  
#### idleResultTracker

* *trackResult(context.Context, resultItems)*

  {}

#### getCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.Get获取object数据

#### putCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.Put存储object数据

#### headCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.Head获取指定object head

#### rangeCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.Range向指定节点获取数据段

#### rangeHashCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.RangeHash向指定节点获取数据段的tzhash

#### searchCaller

* *call(ctx context.Context, r serviceRequest, c \*clientInfo) (interface{}, error)*

  调用ObjectServiceClient.Search从指定container查询object

#### 方法

* *Transport(ctx context.Context, p transport.ObjectTransportParams)*

  向指定节点发送消息，然后使用执行函数处理返回结果

### ContainerTraverseExecutor

|Field|Type|Description|
|-|-|-|
|transport|transport.ObjectTransport||

* *Execute(ctx context.Context, p TraverseParams)*

  向p.Traverser中的每个节点，使用transport发送请求，并使用p.Handler的处理函数处理返回结果

### SelectiveContainerExecutor

|Field|Type|Description|
|-|-|-|
|cnl|ContainerNodesLister||
|Executor|ContainerTraverseExecutor||
|log|\*zap.Logger||

* *Get(ctx context.Context, p \*GetParams) error*

* *Head(ctx context.Context, p \*HeadParams) error*

* *Search(ctx context.Context, p \*SearchParams) error*

* *RangeHash(ctx context.Context, p \*RangeHashParams) error*

* *exec(ctx context.Context, p \*execItems) error*

  首先验证调用参数，然后调用*prepareNodes*确定请求节点哪些，然后针对每个节点调用*prepareAddrList*获取object列表，然后构建TransportInfo调用Executor执行Execute

* *prepareNodes(ctx context.Context, p \*SelectiveParams) ([]multiaddr.Multiaddr, error)*

  准备请求的节点，如果调用参数里面的有节点信息，直接返回，如果允许请求localnode，填一个nil，如果前两者都不满足，使用ContainerNodesLister根据cid获取节点地址

* *prepareAddrList(ctx context.Context, p \*SelectiveParams, node multiaddr.Multiaddr) []refs.Address*

  准备objectId列表，如果请求参数里面有直接返回，如果没有向node发送Ojbect.RequestSearch