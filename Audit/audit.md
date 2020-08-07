# Audit

//描述

依赖

|Field|Type|Description|
|-|-|-|
||\*viper.Viper|  读取配置|
| |\*zap.Logger|  日志输出|
| Placer |implementations.ObjectPlacer |**lib.implements.ObjectPlacer**|
| PeerStore |peers.Store|[Peers.Store](../Peers/peers.store.md)|
| Peers |peers.Interface|[Peers.Interface](../Peers/peers.interface.md)|
| TimeoutsPrefix | string ||
| key| \*ecdsa.PrivateKey| 自身的priate key|
| TokenStore |session.PrivateTokenStore| **module: session.NewMapTokenStore** |

### AddressStore

依赖peers.Store通过网络地址address后去public key

| Field | Type        | Description |
| ----- | ----------- | ----------- |
|log|\*zap.Logger|日志输出|
| ps    | peers.Store | peers存储   |

#### 方法
 * *SelfAddr() (multiaddr.Multiaddr, error)*

   调用*peers.Store.SelfID()*获取自身的ID，然后使用*peers.Store.GetAddr()*获取自身的地址

 * *PublicKey(mAddr multiaddr.Multiaddr)*

   先调用*peers.AddressID*根据mAddr获取ID，然后调用*peers.GetPublicKey*使用ID获取public key

#### 函数

* *NewAddressStore(ps peers.Store, log \*zap.Logger) (AddressStoreComponent, error)*

  使用peers.Store和log创建一个AddressStore实例

### ObjectTransport

//描述

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

#### RemoteService

| Field | Type        | Description |
| ----- | ----------- | ----------- |
|log|\*zap.Logger|日志输出|
| ps    | peers.Interface | 管理连接   |

##### 方法

* *Remote(ctx context.Context, addr multiaddr.Multiaddr) (object.ServiceClient, error)*
  创建grpc连接addr, 返回object.ServiceClient

##### 函数

* *NewRemoteService(ps peers.Interface) RemoteService*
  创建一个RemoteService实例

### ContainerTraverseExecutor

### SelectiveContainerExecutor



