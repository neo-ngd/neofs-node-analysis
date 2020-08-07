# localStore相关代码调查分析

## 源代码范围

modules.node.localstore

lib.localstore.*

### 模块1

modules.node.localstore

源代码

```go
package node

import (
	"github.com/nspcc-dev/neofs-node/lib/core"
	"github.com/nspcc-dev/neofs-node/lib/localstore"
	"github.com/nspcc-dev/neofs-node/lib/meta"
	"github.com/nspcc-dev/neofs-node/lib/metrics"
	"go.uber.org/atomic"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type (
	localstoreParams struct {
		dig.In

		Logger    *zap.Logger
		Storage   core.Storage
		Counter   *atomic.Float64
		Collector metrics.Collector
	}

	metaIterator struct {
		iter localstore.Iterator
	}
)

func newMetaIterator(iter localstore.Iterator) meta.Iterator {
	return &metaIterator{iter: iter}
}

func (m *metaIterator) Iterate(handler meta.IterateFunc) error {
	return m.iter.Iterate(nil, func(objMeta *localstore.ObjectMeta) bool {
		return handler == nil || handler(objMeta.Object) != nil
	})
}

func newLocalstore(p localstoreParams) (localstore.Localstore, error) {
	metaBucket, err := p.Storage.GetBucket(core.MetaStore)
	if err != nil {
		return nil, err
	}

	blobBucket, err := p.Storage.GetBucket(core.BlobStore)
	if err != nil {
		return nil, err
	}

	local, err := localstore.New(localstore.Params{
		BlobBucket: blobBucket,
		MetaBucket: metaBucket,
		Logger:     p.Logger,
		Collector:  p.Collector,
	})
	if err != nil {
		return nil, err
	}

	iter := newMetaIterator(local)
	p.Collector.SetCounter(local)
	p.Collector.SetIterator(iter)

	return local, nil
}

```

结构分析

```go
1.定义localstoreParams参数结构体
	localstoreParams struct {
		dig.In

		Logger    *zap.Logger
		Storage   core.Storage
		Counter   *atomic.Float64
		Collector metrics.Collector
	}

2.声明函数newLocalstore
func newLocalstore(p localstoreParams) (localstore.Localstore, error)
作用：构建Localstore接口实例
伪代码逻辑：
a)利用params中的storage组件构建MetaStore、BlobStore等参数
b)利用这些参数构建Localstore接口实例
```

### 模块2

lib.localstore.*

源代码

a) interface.go

​    put.go

​    get.go

​    del.go

​    meta.go

​    list.go

​    has.go

​    range.go

b) alias.go

c) filter.go

​    filter_func.go

d) localstore.proto

结构分析

```go
a)
1.定义Localstore接口，以及实现该接口的结构体
	Localstore interface {
		Put(context.Context, *Object) error
		Get(Address) (*Object, error)
		Del(Address) error
		Meta(Address) (*ObjectMeta, error)
		Iterator
		Has(Address) (bool, error)
		ObjectsCount() (uint64, error)

		object.PositionReader
		Size() int64
	}
	
	localstore struct {
		metaBucket core.Bucket
		blobBucket core.Bucket

		log *zap.Logger
		col metrics.Collector
	}

    //方法接口方法实现put.go/get.go...
    ....

2.定义异常类型
{
    var ErrOutOfRange = errors.New("range is out of payload bounds")
    var ErrEmptyMetaHandler = errors.New("meta handler is nil")
    var errNilLogger = errors.New("logger is nil")
    var errNilCollector = errors.New("metrics collector is nil")
}

3.定义构造函数
func New(p Params) (Localstore, error)
作用：创建一个Localstore接口实例

b)alias.go
定义/neofs-api-go 内一部分类型的别名

c)过滤器
1.定义过滤器FilterPipeline属性和方法以及方法实现
    //结构体属性
	filterPipeline struct {
		*sync.RWMutex
		name     string
		pri      uint64
		filterFn FilterFunc
		maxSubPri  uint64
		mSubResult map[string]map[FilterCode]FilterCode
		subFilters []FilterPipeline
	}
    //方法
	FilterPipeline interface {
		Pass(ctx context.Context, meta *ObjectMeta) *FilterResult
		PutSubFilter(params SubFilterParams) error
		GetPriority() uint64
		SetPriority(uint64)
		GetName() string
	}
    //方法实现//使用锁线程安全
    ...

2.定义过滤结果状态结构体和相应常量和变量以及相应的生成函数
    //结构体
	FilterResult struct {
		c FilterCode
		e error
	}
    //常量
    const{
    	CodeUndefined=0
	    CodePass=1
	    CodeFail=2
	    CodeIgnore=3
    }
    //变量 FilterResult类型
	rPass
	rFail
	rIgnore
	rUndefined
    //生成方法
    ResultPass()
    ResultFail()
    ResultIgnore()
    ResultUndefined()
    ResultWithError()
    //构造方法
    func NewFilter(p *FilterParams) FilterPipeline 单过滤器
    func AllPassIncludingFilter(name string, params ...*FilterParams) (FilterPipeline, error)过滤器链，需要所有过滤器都通过

3.定义三个过滤方法
//直接通过
func SkippingFilterFunc(_ context.Context, _ *ObjectMeta) *FilterResult
//包含ContainerID的过滤器
func ContainerFilterFunc(cidList []CID) FilterFunc
//元数据存储时隙是否小于某个时隙
func StoredEarlierThanFilterFunc(epoch uint64)

d)localstore.proto
定义ObjectMeta消息类型
```

​    

### 类/接口设计

接口

```c#
interface:
ILocalstore{
		public Put(context.Context, Object) throw Exception
		public Object Get(Address) throw Exception
		public Del(Address) throw Exception
		public ObjectMeta Meta(Address) throw Exception
		public Iterator Iterate(FilterPipeline, MetaHandler) throw Exception
		public bool Has(Address) throw Exception
		public ulong ObjectsCount() throw Exception

		object.PositionReader//PRead(ctx context.Context, addr refs.Address, rng Range) ([]byte, error)
		public long Size()
	}
```

实现

```c#
Localstore:ILocalstore{
	private	IBucket metaBucket;
	private	IBucket blobBucket;
    private Collector col;
    
    //构造函数
    public Localstore(Params p)
    //ILocalstore接口方法实现

}
class Params{
    private IBucket blobBucket;
    private IBucket metaBucket;
    private Collector collector;
    
}
```

接口

```c#
IFilterPipeline{
	public FilterResult	Pass(Context, meta ObjectMeta);
	public PutSubFilter(SubFilterParams params) throw Exception
	public ulong GetPriority();
	public SetPriority(long)
	public string GetName();
}
```

实现

```c#
class FilterPipeline:IFilterPipeline
{
    private string name;
    private ulong pri;
    private function filterfunc;
    private ulong maxSubPri;
    private Map<string,Map<FilterCode,value>> mSubResult;
    private FilterPipeline subFilters;
    
    //构造函数
    public FilterPipeline(){
        
    }
    
    public static FilterPipeline AllPassIncludingFilter(name string, params ...*FilterParams)
    
    //IFilterPipeline接口实现
    ...
    
    public FilterResult SkippingFilterFunc(_ context.Context, _ *ObjectMeta)
    public FilterResult ContainerFilterFunc(CID[] cidList)
    public FilterResult  StoredEarlierThanFilterFunc(ulong epoch)
}

class FilterResult
{
	FilterCode c
	error e
}
//常量
enum {
    CodeUndefined=0,
    CodePass=1,
    CodeFail=2,
    CodeIgnore=3
}
```



## 杂项

### 结构体系

store------>localstore

FilterParam--->FilterPipeline