# Metrics.service相关代码调查分析

## 源代码范围

```go
module.node.metrics
service.metrics.service
```

## 模块1

module.node.metrics

**源代码**

```go
package node

import (
	"github.com/nspcc-dev/neofs-node/lib/core"
	"github.com/nspcc-dev/neofs-node/lib/metrics"
	mService "github.com/nspcc-dev/neofs-node/services/metrics"
	"github.com/spf13/viper"
	"go.uber.org/atomic"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type (
	metricsParams struct {
		dig.In

		Logger  *zap.Logger
		Options []string `name:"node_options"`
		Viper   *viper.Viper
		Store   core.Storage
	}

	metricsServiceParams struct {
		dig.In

		Logger    *zap.Logger
		Collector metrics.Collector
	}
)

func newObjectCounter() *atomic.Float64 { return atomic.NewFloat64(0) }

func newMetricsService(p metricsServiceParams) (mService.Service, error) {
	return mService.New(mService.Params{
		Logger:    p.Logger,
		Collector: p.Collector,
	})
}

func newMetricsCollector(p metricsParams) (metrics.Collector, error) {
	store, err := p.Store.GetBucket(core.SpaceMetricsStore)
	if err != nil {
		return nil, err
	}

	return metrics.New(metrics.Params{
		Options:      p.Options,
		Logger:       p.Logger,
		Interval:     p.Viper.GetDuration("metrics_collector.interval"),
		MetricsStore: store,
	})
}

```

**结构分析**

```go
1.定义metricsParams参数和metricsServiceParams参数
	metricsParams struct {
		dig.In

		Logger  *zap.Logger
		Options []string `name:"node_options"`
		Viper   *viper.Viper
		Store   core.Storage
	}

	metricsServiceParams struct {
		dig.In

		Logger    *zap.Logger
		Collector metrics.Collector
	}

2.声明MetricsService构造函数
func newMetricsService(p metricsServiceParams) (mService.Service, error)
作用：创建一个Metrics服务实例

3.声明newMetricsCollector构造函数
func newMetricsCollector(p metricsParams) (metrics.Collector, error)
作用：创建一个MetricsCollector实例,获取Metrics相关的存储bucket


```



## 模块2

service.metrics.service

**源代码**

```go
package metrics

import (
	"context"

	"github.com/nspcc-dev/neofs-node/internal"
	"github.com/nspcc-dev/neofs-node/lib/metrics"
	"github.com/nspcc-dev/neofs-node/modules/grpc"
	"go.uber.org/zap"
)

type (
	// Service is an interface of the server of Metrics service.
	Service interface {
		MetricsServer
		grpc.Service
	}

	// Params groups the parameters of Metrics service server's constructor.
	Params struct {
		Logger    *zap.Logger
		Collector metrics.Collector
	}

	serviceMetrics struct {
		log *zap.Logger
		col metrics.Collector
	}
)

const (
	errEmptyLogger    = internal.Error("empty logger")
	errEmptyCollector = internal.Error("empty metrics collector")
)

// New is a Metrics service server's constructor.
func New(p Params) (Service, error) {
	switch {
	case p.Logger == nil:
		return nil, errEmptyLogger
	case p.Collector == nil:
		return nil, errEmptyCollector
	}

	return &serviceMetrics{
		log: p.Logger,
		col: p.Collector,
	}, nil
}

func (s *serviceMetrics) ResetSpaceCounter(_ context.Context, _ *ResetSpaceRequest) (*ResetSpaceResponse, error) {
	s.col.UpdateSpaceUsage()
	return &ResetSpaceResponse{}, nil
}

func (s *serviceMetrics) Name() string { return "metrics" }

func (s *serviceMetrics) Register(srv *grpc.Server) {
	RegisterMetricsServer(srv, s)
}


```

**结构分析**

```go
1.定义MetricsServer相关的服务接口以及实现
	Service interface {
		MetricsServer
		grpc.Service
	}

//grpc.Service服务实现
func (s *serviceMetrics) Name() string
作用：声明服务名称，"metrics"

func (s *serviceMetrics) Register(srv *grpc.Server)
作用：注册服务接口

2.错误常量
const (
	errEmptyLogger    = internal.Error("empty logger")
	errEmptyCollector = internal.Error("empty metrics collector")
)

3.声明构造函数
func New(p Params) (Service, error)
作用：创建MetricsServer服务实例
```



## 杂项

负责提供 空间坐标查询 等操作的GRPC服务





