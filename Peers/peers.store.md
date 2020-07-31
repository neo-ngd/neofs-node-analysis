## Peers.Store

用来保存peer的信息

### Peers.Storage

是一个interface, 存储Peers信息，具体实现使用内存，保存在map中

#### ID 

Peer ID string 类型，由Peer publickey创建

##### 函数

* *IDFromPublicKey(pk \*ecdsa.PublicKey) ID*

  将pubickey转为bytes然后在调用IDFromBinary

* *IDFromBinary(b []byte) ID* 

  先计算hash256,然后使用
github.com/multiformats/go-multihash 自我描述哈希方法 有c#包
```
fn code  dig size hash digest
-------- -------- ------------------------------------
00010001 00000100 101101100 11111000 01011100 10110101
sha1     4 bytes  4 byte sha1 digest
```
计算哈希，返回base58string

##### 方法

* *String() string*
* *Bytes() []byte*
* *Equal(id ID) bool*

#### Peer

连接的节点信息

|Field|Type|Decription|
|-|-|-|
|id|ID|id|
|pub|*ecdsa.PublicKey|节点的公钥|
|key|*ecdsa.PrivateKey|节点的私钥|
|addr|multiaddr.Multiaddr|节点的网络地址|

##### 函数

* *NewLocalPeer(addr multiaddr.Multiaddr, key \*ecdsa.PrivateKey) Peer*

  创建一个Peer实例，publickey由PrivateKey计算，只针对本地节点，使用私钥

* *NewPeer(addr multiaddr.Multiaddr, pub \*ecdsa.PublicKey, key \*ecdsa.PrivateKey) Peer*

  创建一个Peer实例，id由ID.IDFromPublicKey使用PublicKey计算而得。私钥可以为*nil*


##### 方法

* *SetPrivateKey(key \*ecdsa.PrivateKey) error*

  设置私钥

* *ID() ID*

  获取ID

* *Address() multiaddr.Multiaddr*

  获取网络地址

* *PublicKey() \*ecdsa.PublicKey*

  获取公钥

* *PrivateKey() (*ecdsa.PrivateKey, error)*

  获取私钥

#### 函数

* *NewSimpleStorage(capacity int, l \*zap.Logger) Storage*

  创建一个Peer.Storage实例，使用配置或者默认设置

#### 方法

* *List(id ID, seed int64, count int) ([]ID, error)*
  
  除id之外的Peers根据距离seed由近到远排列,然后取出前count个

* *Get(id ID) (Peer, error)*

  根据id返回Peer

* *Set(id ID, p Peer) error*

  设置id对应的Peer

* *Rem(id ID) error*

  根据id删除Peer

* *Update(nm \*netmap.NetMap)*

  根据Netmap中items的信息更新所有Peers信息


* *Filter(filter PeerFilter)*

  使用filter函数过滤符合条件的Peers

### Peers.Store

基于Peers.Storage封装的管理Peers

#### 函数

* *NewStore(p StoreParams) (Store, error)*

  使用*NewSimpleStorage*穿件Peers.Storage实例，然后使用配置信息，创建本地Peer然后创建一个Peers.Store实例

* *maddrFilter(addr multiaddr.Multiaddr) PeerFilter*

  返回根据网络地址获取Peers时使用的条件函数 func(p Peer) bool { return addr.Equal(p.Address()) }

#### 方法

* *AddressID(addr multiaddr.Multiaddr) (ID, error)*

  根据网络地址返回ID

* *SelfID() ID*

  返回本地节点的ID

* *AddPeer(addr multiaddr.Multiaddr, pub \*ecdsa.PublicKey, key \*ecdsa.PrivateKey) (ID, error)*

  根据参数addr, pub, key创建Peer实例，ID由ID.IDFromPublicKey计算，使用Storage.Set方法添加到Peers.Storage中，返回ID

* *DeletePeer(id ID)*

  根据ID从Peers.Storage中删除Peer信息，调用*Storage.Rem*方法

* *Update(nm *netmap.NetMap) error*

  使用Netmap调用*Storage.Update*更新Peers信息，并检查本地信息是否正确，不正确重新设置

* *GetAddr(id ID) (multiaddr.Multiaddr, error)*

  根据ID获取网络地址，调用*Storage.Get*方法，并返回Peer.addr

* *GetPublicKey(id ID) (\*ecdsa.PublicKey, error)*

  根据ID返回公钥，调用*Storage.Get*方法获取Peer, 然后返回Peer.PublicKey

* *GetPrivateKey(id ID) (\*ecdsa.PrivateKey, error)*

  根据ID返回公钥，调用*Storage.Get*方法获取Peer, 然后返回Peer.PrivateKey

* *Sign(data []byte) ([]byte, error)*

  使用本地的私钥签名数据

* *Verify(id ID, data, sign []byte) error*

  根据ID获取Peer的公钥*GetPublicKey*，并校验data的签名sign

* *Neighbours(seed int64, count int) ([]ID, error)*

  调用*Storage.List*方法，参数为本地节点ID，seed， count, 所有Peer中排除本地节点后，根据Peer与seed的距离，由小到大排序，取出前count个

*  *Check(min int) error* <未完成实现>

