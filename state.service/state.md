# State.service相关代码调查分析

## 源代码范围

```go
module.bootstrap.healthy
service.public.state.service
```

## 模块1

module.bootstrap.healthy

**源代码**

```go
package bootstrap

import (
	"crypto/ecdsa"
	"sync"

	"github.com/nspcc-dev/neofs-node/internal"
	"github.com/nspcc-dev/neofs-node/lib/implementations"
	"github.com/nspcc-dev/neofs-node/lib/placement"
	"github.com/nspcc-dev/neofs-node/services/public/state"
	"github.com/spf13/viper"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type (
	healthyParams struct {
		dig.In

		Logger   *zap.Logger
		Viper    *viper.Viper
		Place    placement.Component
		Checkers []state.HealthChecker `group:"healthy"`

		// for ChangeState
		PrivateKey *ecdsa.PrivateKey

		MorphNetmapContract *implementations.MorphNetmapContract
	}

	healthyResult struct {
		dig.Out

		HealthyClient HealthyClient

		StateService state.Service
	}

	// HealthyClient is an interface of healthiness checking tool.
	HealthyClient interface {
		Healthy() error
	}

	healthyClient struct {
		*sync.RWMutex
		healthy func() error
	}
)

const (
	errUnhealthy = internal.Error("unhealthy")
)

func (h *healthyClient) setHandler(handler func() error) {
	if handler == nil {
		return
	}

	h.Lock()
	h.healthy = handler
	h.Unlock()
}

func (h *healthyClient) Healthy() error {
	if h.healthy == nil {
		return errUnhealthy
	}

	return h.healthy()
}

func newHealthy(p healthyParams) (res healthyResult, err error) {
	sp := state.Params{
		Stater:              p.Place,
		Logger:              p.Logger,
		Viper:               p.Viper,
		Checkers:            p.Checkers,
		PrivateKey:          p.PrivateKey,
		MorphNetmapContract: p.MorphNetmapContract,
	}

	if res.StateService, err = state.New(sp); err != nil {
		return
	}

	healthyClient := &healthyClient{
		RWMutex: new(sync.RWMutex),
	}

	healthyClient.setHandler(res.StateService.Healthy)

	res.HealthyClient = healthyClient

	return
}
```

**结构分析**

```go
1.定义health检查参数
	healthyParams struct {
		dig.In
		Logger   *zap.Logger
		Viper    *viper.Viper
		Place    placement.Component
		Checkers []state.HealthChecker `group:"healthy"`
		// for ChangeState
		PrivateKey *ecdsa.PrivateKey
		MorphNetmapContract *implementations.MorphNetmapContract
	}
2.定义health检查结果结构体
	healthyResult struct {
		dig.Out

		HealthyClient HealthyClient

		StateService state.Service
	}
3.定义healthClient结构体/接口方法和实现
	healthyClient struct {//线程安全
		*sync.RWMutex
		healthy func() error
	}
	HealthyClient interface {
		Healthy() error
	}
//方法实现
func (h *healthyClient) Healthy() error {
	if h.healthy == nil {
		return errUnhealthy
	}
	return h.healthy()
}

func (h *healthyClient) setHandler(handler func() error) {
	if handler == nil {
		return
	}
	h.Lock()
	h.healthy = handler
	h.Unlock()
}

4.定义构造函数newHealthy
func newHealthy(p healthyParams) (res healthyResult, err error)
作用：生成healthyResult
伪代码逻辑：
1)利用healthyParams构建StateService参数
2)利用StateService参数构建StateService服务
3)构建healthyClient,并设置处理函数handler
```



## 模块2

service.public.state.service

**源代码**

```go
package state

import (
	"context"
	"crypto/ecdsa"
	"encoding/hex"
	"strconv"

	"github.com/nspcc-dev/neofs-api-go/bootstrap"
	"github.com/nspcc-dev/neofs-api-go/refs"
	"github.com/nspcc-dev/neofs-api-go/service"
	"github.com/nspcc-dev/neofs-api-go/state"
	crypto "github.com/nspcc-dev/neofs-crypto"
	"github.com/nspcc-dev/neofs-node/internal"
	"github.com/nspcc-dev/neofs-node/lib/core"
	"github.com/nspcc-dev/neofs-node/lib/implementations"
	"github.com/nspcc-dev/neofs-node/modules/grpc"
	"github.com/pkg/errors"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/spf13/viper"
	"go.uber.org/zap"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type (
	// Service is an interface of the server of State service.
	Service interface {
		state.StatusServer
		grpc.Service
		Healthy() error
	}

	// HealthChecker is an interface of node healthiness checking tool.
	HealthChecker interface {
		Name() string
		Healthy() bool
	}

	// Stater is an interface of the node's network state storage with read access.
	Stater interface {
		NetworkState() *bootstrap.SpreadMap
	}

	// Params groups the parameters of State service server's constructor.
	Params struct {
		Stater Stater

		Logger *zap.Logger

		Viper *viper.Viper

		Checkers []HealthChecker

		PrivateKey *ecdsa.PrivateKey

		MorphNetmapContract *implementations.MorphNetmapContract
	}

	stateService struct {
		state    Stater
		config   *viper.Viper
		checkers []HealthChecker
		private  *ecdsa.PrivateKey
		owners   map[refs.OwnerID]struct{}

		stateUpdater *implementations.MorphNetmapContract
	}

	// HealthRequest is a type alias of
	// HealthRequest from state package of neofs-api-go.
	HealthRequest = state.HealthRequest
)

const (
	errEmptyViper         = internal.Error("empty config")
	errEmptyLogger        = internal.Error("empty logger")
	errEmptyStater        = internal.Error("empty stater")
	errUnknownChangeState = internal.Error("received unknown state")
)

const msgMissingRequestInitiator = "missing request initiator"

var requestVerifyFunc = core.VerifyRequestWithSignatures

// New is an State service server's constructor.
func New(p Params) (Service, error) {
	switch {
	case p.Logger == nil:
		return nil, errEmptyLogger
	case p.Viper == nil:
		return nil, errEmptyViper
	case p.Stater == nil:
		return nil, errEmptyStater
	case p.PrivateKey == nil:
		return nil, crypto.ErrEmptyPrivateKey
	}

	svc := &stateService{
		config:   p.Viper,
		state:    p.Stater,
		private:  p.PrivateKey,
		owners:   fetchOwners(p.Logger, p.Viper),
		checkers: make([]HealthChecker, 0, len(p.Checkers)),

		stateUpdater: p.MorphNetmapContract,
	}

	for i, checker := range p.Checkers {
		if checker == nil {
			p.Logger.Debug("ignore empty checker",
				zap.Int("index", i))
			continue
		}

		p.Logger.Info("register health-checker",
			zap.String("name", checker.Name()))

		svc.checkers = append(svc.checkers, checker)
	}

	return svc, nil
}

func fetchOwners(l *zap.Logger, v *viper.Viper) map[refs.OwnerID]struct{} {
	// if config.yml used:
	items := v.GetStringSlice("node.rpc.owners")

	for i := 0; ; i++ {
		item := v.GetString("node.rpc.owners." + strconv.Itoa(i))

		if item == "" {
			l.Info("stat: skip empty owner", zap.Int("idx", i))
			break
		}

		items = append(items, item)
	}

	result := make(map[refs.OwnerID]struct{}, len(items))

	for i := range items {
		var owner refs.OwnerID

		if data, err := hex.DecodeString(items[i]); err != nil {
			l.Warn("stat: skip wrong hex data",
				zap.Int("idx", i),
				zap.String("key", items[i]),
				zap.Error(err))

			continue
		} else if key := crypto.UnmarshalPublicKey(data); key == nil {
			l.Warn("stat: skip wrong key",
				zap.Int("idx", i),
				zap.String("key", items[i]))
			continue
		} else if owner, err = refs.NewOwnerID(key); err != nil {
			l.Warn("stat: skip wrong key",
				zap.Int("idx", i),
				zap.String("key", items[i]),
				zap.Error(err))
			continue
		}

		result[owner] = struct{}{}

		l.Info("rpc owner added", zap.Stringer("owner", owner))
	}

	return result
}

func nonForwarding(ttl uint32) error {
	if ttl != service.NonForwardingTTL {
		return status.Error(codes.InvalidArgument, service.ErrInvalidTTL.Error())
	}

	return nil
}

func requestInitiator(req service.SignKeyPairSource) *ecdsa.PublicKey {
	if signKeys := req.GetSignKeyPairs(); len(signKeys) > 0 {
		return signKeys[0].GetPublicKey()
	}

	return nil
}

// ChangeState allows to change current node state of node.
// To permit access, used server config options.
// The request should be signed.
func (s *stateService) ChangeState(ctx context.Context, in *state.ChangeStateRequest) (*state.ChangeStateResponse, error) {
	// verify request structure
	if err := requestVerifyFunc(in); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	}

	// verify change state permission
	if key := requestInitiator(in); key == nil {
		return nil, status.Error(codes.InvalidArgument, msgMissingRequestInitiator)
	} else if owner, err := refs.NewOwnerID(key); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	} else if _, ok := s.owners[owner]; !ok {
		return nil, status.Error(codes.PermissionDenied, service.ErrWrongOwner.Error())
	}

	// convert State field to NodeState
	if in.GetState() != state.ChangeStateRequest_Offline {
		return nil, status.Error(codes.InvalidArgument, errUnknownChangeState.Error())
	}

	// set update state parameters
	p := implementations.UpdateStateParams{}
	p.SetState(implementations.StateOffline)
	p.SetKey(
		crypto.MarshalPublicKey(&s.private.PublicKey),
	)

	if err := s.stateUpdater.UpdateState(p); err != nil {
		return nil, status.Error(codes.Aborted, err.Error())
	}

	return new(state.ChangeStateResponse), nil
}

// DumpConfig request allows dumping settings for the current node.
// To permit access, used server config options.
// The request should be signed.
func (s *stateService) DumpConfig(_ context.Context, req *state.DumpRequest) (*state.DumpResponse, error) {
	if err := service.ProcessRequestTTL(req, nonForwarding); err != nil {
		return nil, err
	} else if err = requestVerifyFunc(req); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	} else if key := requestInitiator(req); key == nil {
		return nil, status.Error(codes.InvalidArgument, msgMissingRequestInitiator)
	} else if owner, err := refs.NewOwnerID(key); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	} else if _, ok := s.owners[owner]; !ok {
		return nil, status.Error(codes.PermissionDenied, service.ErrWrongOwner.Error())
	}

	return state.EncodeConfig(s.config)
}

// Netmap returns SpreadMap from Stater (IRState / Place-component).
func (s *stateService) Netmap(_ context.Context, req *state.NetmapRequest) (*bootstrap.SpreadMap, error) {
	if err := service.ProcessRequestTTL(req); err != nil {
		return nil, err
	} else if err = requestVerifyFunc(req); err != nil {
		return nil, err
	}

	if s.state != nil {
		return s.state.NetworkState(), nil
	}

	return nil, status.New(codes.Unavailable, "service unavailable").Err()
}

func (s *stateService) healthy() error {
	for _, svc := range s.checkers {
		if !svc.Healthy() {
			return errors.Errorf("service(%s) unhealthy", svc.Name())
		}
	}

	return nil
}

// Healthy returns error as status of service, if nil service healthy.
func (s *stateService) Healthy() error { return s.healthy() }

// Check that all checkers is healthy.
func (s *stateService) HealthCheck(_ context.Context, req *HealthRequest) (*state.HealthResponse, error) {
	if err := service.ProcessRequestTTL(req); err != nil {
		return nil, err
	} else if err = requestVerifyFunc(req); err != nil {
		return nil, err
	}

	var (
		err  = s.healthy()
		resp = &state.HealthResponse{Healthy: true, Status: "OK"}
	)

	if err != nil {
		resp.Healthy = false
		resp.Status = err.Error()
	}

	return resp, nil
}

func (*stateService) Metrics(_ context.Context, req *state.MetricsRequest) (*state.MetricsResponse, error) {
	if err := service.ProcessRequestTTL(req); err != nil {
		return nil, err
	} else if err = requestVerifyFunc(req); err != nil {
		return nil, err
	}

	return state.EncodeMetrics(prometheus.DefaultGatherer)
}

func (s *stateService) DumpVars(_ context.Context, req *state.DumpVarsRequest) (*state.DumpVarsResponse, error) {
	if err := service.ProcessRequestTTL(req, nonForwarding); err != nil {
		return nil, err
	} else if err = requestVerifyFunc(req); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	} else if key := requestInitiator(req); key == nil {
		return nil, status.Error(codes.InvalidArgument, msgMissingRequestInitiator)
	} else if owner, err := refs.NewOwnerID(key); err != nil {
		return nil, status.Error(codes.InvalidArgument, err.Error())
	} else if _, ok := s.owners[owner]; !ok {
		return nil, status.Error(codes.PermissionDenied, service.ErrWrongOwner.Error())
	}

	return state.EncodeVariables(), nil
}

// Name of the service.
func (*stateService) Name() string { return "StatusService" }

// Register service on gRPC server.
func (s *stateService) Register(g *grpc.Server) { state.RegisterStatusServer(g, s) }

```

**结构分析**

```go
1.定义stateService结构体和接口接口实现
	stateService struct {
		state    Stater
		config   *viper.Viper
		checkers []HealthChecker
		private  *ecdsa.PrivateKey
		owners   map[refs.OwnerID]struct{}

		stateUpdater *implementations.MorphNetmapContract
	}
	Service interface {
		state.StatusServer
		grpc.Service
		Healthy() error
	}
//grpc.Service接口实现
func (*stateService) Name()
作用：GRPC服务名称，"StatusService"
func (s *stateService) Register(g *grpc.Server)
作用：注册GRPC服务

//state.StatusServer接口方法实现
Netmap(context.Context, *NetmapRequest) (*bootstrap.SpreadMap, error)
作用：获取服务当前的network状态
伪代码逻辑：
1)跟新请求的TTL
2)验证请求结构体
3)获取stateService中network的状态
4)如果无法获取network状态，则返回无服务异常

Metrics(context.Context, *MetricsRequest) (*MetricsResponse, error)
作用：改变当前节点的坐标
伪代码逻辑：
1)跟新请求的TTL
2)验证请求结构体
3)调用链上合约，跟新坐标

HealthCheck(context.Context, *HealthRequest) (*HealthResponse, error)
作用：检查所有检查项是否都正常
伪代码逻辑：
1)跟新请求的TTL
2)验证请求结构体
3)执行检查
4)输出检查结果

DumpConfig(context.Context, *DumpRequest) (*DumpResponse, error)
作用：改变当前节点的配置
伪代码逻辑：
1)跟新请求的TTL
2)验证请求结构体
3)获取公钥并验证权限
4)验证所有人
5)调用链上合约，跟新Config

DumpVars(context.Context, *DumpVarsRequest) (*DumpVarsResponse, error)
作用：
伪代码逻辑：
1)跟新请求的TTL
2)验证请求结构体
3)获取公钥并验证权限
4)验证所有人
5)将状态中的值转json,并输出

ChangeState(context.Context, *ChangeStateRequest) (*ChangeStateResponse, error)
作用：改变当前节点的状态
伪代码逻辑：
1)验证请求结构体
2)验证权限
3)把请求State转换为NodeState
4)构造State跟新参数
5)调用链上合约，跟新State

2.定义异常
	errEmptyViper         = internal.Error("empty config")
	errEmptyLogger        = internal.Error("empty logger")
	errEmptyStater        = internal.Error("empty stater")
	errUnknownChangeState = internal.Error("received unknown state")
    msgMissingRequestInitiator = "missing request initiator"

3.声明构造函数func New(p Params) (Service, error) 
作用：构造State service实例
伪代码逻辑：
1)Params参数参数检查
2)构造StateService服务实例
3)向StateService服务实例中追加HealthChecker
```



## 杂项

State.service是负责节点State状态查询、更改等操作的GRPC服务





