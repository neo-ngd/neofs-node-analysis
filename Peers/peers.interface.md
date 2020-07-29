## Peers.Interface

建立管理p2p连接，包含普通网络连接和grpc连接

### Transport

#### 依赖

* github.com/rubyist/circuitbreaker

  断路器，如果请求达到了一定失败条件，比如失败n次，连续失败n次等，就会中断，所有请求都会阻断。但是期间会允许一些请求重试，如果这些请求成功，将重启链路。
  ```
  // Creates a circuit breaker that will trip if the function fails 10 times
  cb := circuit.NewThresholdBreaker(10)

  events := cb.Subscribe()
  go func() {
    for {
      e := <-events
      // Monitor breaker events like BreakerTripped, BreakerReset, BreakerFail, BreakerReady
    }
  }()

  cb.Call(func() error {
    // This is where you'll do some remote call
    // If it fails, return an error
  }, 0)
  ```
  
* ipfs github.com/multiformats/go-multiaddr

  可以自我描述的地址格式，比如`127.0.0.1:9090`我们无法判断是tcp还是udp还是什么其他协议，使用multiaddr格式可以具体表示为`/ip4/127.0.0.1/udp/9090/quic`

* ipfs github.com/multiformats/go-multiaddr-net

  针对multiaddr封装的net包，将net包中的方法都重新进行了封装，以使用multiaddr

#### 属性

| Field     | Type           | Description              |
| --------- | -------------- | ------------------------ |
| threshold | int64          | 触发断路器的连续失败次数 |
| timeout   | time.Duration  | 请求的超时时间           |
| panel     | *circuit.Panel | 断路器管理面板           |

#### 方法

* *Dial(ctx context.Context, addr multiaddr.Multiaddr, reset bool) (Connection, error)*

  使用addr指定的网络协议，连接addr指定的地址，返回一个Connection

* *Listen(addr multiaddr.Multiaddr) (manet.Listener, error)*

  采用addr中指定的协议监听addr中地址，返回一个manet.Listener

* *breakerLookup(addr fmt.Stringer) \*circuit.Breaker*

  从panel中获取一个断路器，如果查找不到，根据addr创建一个name，来创建一个断路器

### Peers.Interface

Peers.Interface是一个接口定义，具体实现为iface

#### iface

iface为Peers.Interface的实现

#### 属性

| Field          | Type                                                         | Description                  |
| -------------- | ------------------------------------------------------------ | ---------------------------- |
| log            |                                                              | 日志输出                     |
| addr           | multiaddr.Multiaddr                                          | 本地地址                     |
| tr             | transport.Transport                                          |                              |
| tick           | time.Duration                                                |                              |
| idle           | time.Duration                                                |                              |
| keepAlive      | time.Duration                                                |                              |
| pingTTL        | time.Duration                                                |                              |
| metricsTimeout | time.Duration                                                |                              |
| grpc           | struct {<br>	// globalMutex used by garbage collector and other high<br> globalMutex \*sync.RWMutex <br>  // bookMutex resolves concurrent access to the new connection<br> bookMutex \*sync.RWMutex<br> // connBook contains connection info<br> // it's mutex resolves concurrent access to existed<br> connection connBook map[string]*connItem<br> } | 保存地址和对应的gprc链接实例 |
| cons           | struct {<br> *sync.RWMutex items<br> map[string]transport.Connection<br>} | 保存除grpc连接之外的连接     |
| lis            | struct {<br> *sync.RWMutex<br> items map[string]manet.Listener<br> } | 保存所有对地址的监听         |

#### 方法

* *Shutdown() error*

  关闭iface，关闭和删除所有连接

* *RemoveConnection(maddr multiaddr.Multiaddr) error*

  关闭和删除某个地址的所有连接包括普通网络连接和grpc连接

* *removeListener(addr string) error*

  关闭并删除对所有地址的监听

* *removeNetConnection(addr string) error*

  关闭和删除所有网络连接，除grpc连接

* *removeGRPCConnection(addr string) error*

  关闭并删除所有gprc连接

* *Connect(ctx context.Context, maddr multiaddr.Multiaddr) (manet.Conn, error)*

  根据maddr中描述的协议和地址进行连接，并保存在cons中，如果已经存在连接直接返回

* *Listen(maddr multiaddr.Multiaddr) (manet.Listener, error)*

  建立对Multiaddr的监听，协议地址根据Mutiaddr中的描述确定

* *Address() multiaddr.Multiaddr*

  返回配置的地址

* *newConnection(ctx context.Context, addr multiaddr.Multiaddr, reset bool) (transport.Connection, error)*

  使用transport的Dial方法建立Multiaddr中定义的连接

* *GRPCConnection(ctx context.Context, maddr multiaddr.Multiaddr, reset bool) (\*grpc.ClientConn, error)*

  根据地址建立GPRC链接。首先从grpc.connBook中查找，如果已经存在连接，直接返回，如果不存在，建立连接并保存到grpc.connBook中
  
* *Job(ctx context.Context)*

  启动两个定时器和监听循环，更新grpc连接状态统计和连接清理(关闭删除异常的连接)

#### 函数

* *isGRPCClosed(con \*grpc.ClientConn) bool*

  根据grpc连接的状态判断是否已经关闭的连接

* *gRPCKeepAlive(ping, ttl time.Duration) grpc.DialOption*

  创建grpcKeepAlive参数，返回grpc.DialOption类型

* *convertAddress(maddr multiaddr.Multiaddr) (string, error)*

  将Multiaddr格式的地址，转成普通格式地址
  