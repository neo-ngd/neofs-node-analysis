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
1.定义FilterPipeline接口
	FilterPipeline interface {
		Pass(ctx context.Context, meta *ObjectMeta) *FilterResult
		PutSubFilter(params SubFilterParams) error
		GetPriority() uint64
		SetPriority(uint64)
		GetName() string
	}

d)localstore.proto
定义ObjectMeta消息类型
```

​    

## 杂项

### 结构体系

store----(取出Bucket)----->localstore