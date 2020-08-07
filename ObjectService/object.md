# Object相关代码调查分析

## 源代码范围

```go
module.node.objectmanager
service.public.object.*
```

## 模块1

module.node.objectmanager

**源代码**

```go
package node

import (
	"crypto/ecdsa"

	"github.com/nspcc-dev/neofs-api-go/bootstrap"
	"github.com/nspcc-dev/neofs-api-go/hash"
	apiobj "github.com/nspcc-dev/neofs-api-go/object"
	"github.com/nspcc-dev/neofs-api-go/session"
	libacl "github.com/nspcc-dev/neofs-node/lib/acl"
	"github.com/nspcc-dev/neofs-node/lib/container"
	"github.com/nspcc-dev/neofs-node/lib/core"
	"github.com/nspcc-dev/neofs-node/lib/implementations"
	"github.com/nspcc-dev/neofs-node/lib/ir"
	"github.com/nspcc-dev/neofs-node/lib/localstore"
	"github.com/nspcc-dev/neofs-node/lib/peers"
	"github.com/nspcc-dev/neofs-node/lib/placement"
	"github.com/nspcc-dev/neofs-node/lib/transformer"
	"github.com/nspcc-dev/neofs-node/services/public/object"
	"github.com/spf13/viper"
	"go.uber.org/dig"
	"go.uber.org/zap"
)

type (
	objectManagerParams struct {
		dig.In

		Logger     *zap.Logger
		Viper      *viper.Viper
		LocalStore localstore.Localstore

		PeersInterface peers.Interface

		Peers      peers.Store
		Placement  placement.Component
		TokenStore session.PrivateTokenStore
		Options    []string `name:"node_options"`
		Key        *ecdsa.PrivateKey

		IRStorage ir.Storage

		EpochReceiver implementations.EpochReceiver

		Placer implementations.ObjectPlacer

		ExtendedACLStore libacl.ExtendedACLSource

		ContainerStorage container.Storage
	}
)

const (
	transformersSectionPath = "object.transformers."

	aclMandatorySetBits = 0x04040444
)

const xorSalitor = "xor"

func newObjectManager(p objectManagerParams) (object.Service, error) {
	var sltr object.Salitor

	if p.Viper.GetString("object.salitor") == xorSalitor {
		sltr = hash.SaltXOR
	}

	as, err := implementations.NewAddressStore(p.Peers, p.Logger)
	if err != nil {
		return nil, err
	}

	rs := object.NewRemoteService(p.PeersInterface)

	pto := p.Viper.GetDuration("object.put.timeout")
	gto := p.Viper.GetDuration("object.get.timeout")
	hto := p.Viper.GetDuration("object.head.timeout")
	sto := p.Viper.GetDuration("object.search.timeout")
	rhto := p.Viper.GetDuration("object.range_hash.timeout")
	dto := p.Viper.GetDuration("object.dial_timeout")

	tr, err := object.NewMultiTransport(object.MultiTransportParams{
		AddressStore:     as,
		EpochReceiver:    p.EpochReceiver,
		RemoteService:    rs,
		Logger:           p.Logger,
		Key:              p.Key,
		PutTimeout:       pto,
		GetTimeout:       gto,
		HeadTimeout:      hto,
		SearchTimeout:    sto,
		RangeHashTimeout: rhto,
		DialTimeout:      dto,

		PrivateTokenStore: p.TokenStore,
	})
	if err != nil {
		return nil, err
	}

	exec, err := implementations.NewContainerTraverseExecutor(tr)
	if err != nil {
		return nil, err
	}

	selectiveExec, err := implementations.NewObjectContainerHandler(implementations.ObjectContainerHandlerParams{
		NodeLister: p.Placer,
		Executor:   exec,
		Logger:     p.Logger,
	})
	if err != nil {
		return nil, err
	}

	sgInfoRecv, err := implementations.NewStorageGroupInfoReceiver(implementations.StorageGroupInfoReceiverParams{
		SelectiveContainerExecutor: selectiveExec,
		Logger:                     p.Logger,
	})
	if err != nil {
		return nil, err
	}

	verifier, err := implementations.NewLocalIntegrityVerifier(
		core.NewNeoKeyVerifier(),
	)
	if err != nil {
		return nil, err
	}

	trans, err := transformer.NewTransformer(transformer.Params{
		SGInfoReceiver: sgInfoRecv,
		EpochReceiver:  p.EpochReceiver,
		SizeLimit:      uint64(p.Viper.GetInt64(transformersSectionPath+"payload_limiter.max_payload_size") * apiobj.UnitsKB),
		Verifier:       verifier,
	})
	if err != nil {
		return nil, err
	}

	aclChecker := libacl.NewMaskedBasicACLChecker(aclMandatorySetBits, libacl.DefaultAndFilter)

	aclHelper, err := implementations.NewACLHelper(p.ContainerStorage)
	if err != nil {
		return nil, err
	}

	verifier, err = implementations.NewLocalHeadIntegrityVerifier(
		core.NewNeoKeyVerifier(),
	)
	if err != nil {
		return nil, err
	}

	return object.New(&object.Params{
		Verifier:          verifier,
		Salitor:           sltr,
		LocalStore:        p.LocalStore,
		MaxProcessingSize: p.Viper.GetUint64("object.max_processing_size") * uint64(apiobj.UnitsMB),
		StorageCapacity:   bootstrap.NodeInfo{Options: p.Options}.Capacity() * uint64(apiobj.UnitsGB),
		PoolSize:          p.Viper.GetInt("object.workers_count"),
		Placer:            p.Placer,
		Transformer:       trans,
		ObjectRestorer: transformer.NewRestorePipeline(
			transformer.SplitRestorer(),
		),
		RemoteService:        rs,
		AddressStore:         as,
		Logger:               p.Logger,
		TokenStore:           p.TokenStore,
		EpochReceiver:        p.EpochReceiver,
		ContainerNodesLister: p.Placer,
		Key:                  p.Key,
		CheckACL:             p.Viper.GetBool("object.check_acl"),
		DialTimeout:          p.Viper.GetDuration("object.dial_timeout"),
		MaxPayloadSize:       p.Viper.GetUint64("object.transformers.payload_limiter.max_payload_size") * uint64(apiobj.UnitsKB),
		PutParams: object.OperationParams{
			Timeout:   pto,
			LogErrors: p.Viper.GetBool("object.put.log_errs"),
		},
		GetParams: object.OperationParams{
			Timeout:   gto,
			LogErrors: p.Viper.GetBool("object.get.log_errs"),
		},
		HeadParams: object.OperationParams{
			Timeout:   hto,
			LogErrors: p.Viper.GetBool("object.head.log_errs"),
		},
		DeleteParams: object.OperationParams{
			Timeout:   p.Viper.GetDuration("object.delete.timeout"),
			LogErrors: p.Viper.GetBool("object.get.log_errs"),
		},
		SearchParams: object.OperationParams{
			Timeout:   sto,
			LogErrors: p.Viper.GetBool("object.search.log_errs"),
		},
		RangeParams: object.OperationParams{
			Timeout:   p.Viper.GetDuration("object.range.timeout"),
			LogErrors: p.Viper.GetBool("object.range.log_errs"),
		},
		RangeHashParams: object.OperationParams{
			Timeout:   rhto,
			LogErrors: p.Viper.GetBool("object.range_hash.log_errs"),
		},
		Assembly: p.Viper.GetBool("object.assembly"),

		WindowSize: p.Viper.GetInt("object.window_size"),

		ACLHelper:       aclHelper,
		BasicACLChecker: aclChecker,
		IRStorage:       p.IRStorage,
		ContainerLister: p.Placer,

		SGInfoReceiver: sgInfoRecv,

		OwnerKeyVerifier: core.NewNeoKeyVerifier(),

		ExtendedACLSource: p.ExtendedACLStore,
	})
}


```

**结构分析**

```go
1.定义objectManagerParams参数结构体
	objectManagerParams struct {
		dig.In

		Logger     *zap.Logger
		Viper      *viper.Viper
		LocalStore localstore.Localstore

		PeersInterface peers.Interface

		Peers      peers.Store
		Placement  placement.Component
		TokenStore session.PrivateTokenStore
		Options    []string `name:"node_options"`
		Key        *ecdsa.PrivateKey

		IRStorage ir.Storage

		EpochReceiver implementations.EpochReceiver

		Placer implementations.ObjectPlacer

		ExtendedACLStore libacl.ExtendedACLSource

		ContainerStorage container.Storage
	}

2.定义newObjectManager函数
func newObjectManager(p objectManagerParams) (object.Service, error)
作用：利用objectManagerParams参数，构造object service服务
```



## 模块2

service.public.object.*

**源代码**

a)service/public/object/service.go

b)service/public/object/acl.go

c)service/public/object/bearer.go

d)service/public/object/capacity.go

e)service/public/object/delete.go

f)service/public/object/execution.go

g)service/public/object/filter.go

h)service/public/object/get.go

i)service/public/object/handler.go

j)service/public/object/header.go

k)service/public/object/implementations.go

l)service/public/object/listing.go

m)service/public/object/postprocessor.go

n)service/public/object/preprocessor.go

o)service/public/object/put.go

p)service/public/object/query.go

q)service/public/object/ranges.go

r)service/public/object/response.go

s)service/public/object/search.go

t)service/public/object/status.go

u)service/public/object/token.go

v)service/public/object/transport_implementations.go

w)service/public/object/travers.go

x)service/public/object/ttl.go

y)service/public/object/verb.go

z)service/public/object/verification.go



**结构分析**

```go
a.service.go
1) 定义别名
CID = refs.CID
Object = object.Object
ID = refs.ObjectID
OwnerID = refs.OwnerID
Address = refs.Address
Hash = hash.Hash
Meta = localstore.ObjectMeta
Filter = localstore.FilterPipeline
Header = object.Header
UserHeader = object.UserHeader
SystemHeader = object.SystemHeader
CreationPoint = object.CreationPoint

2)定义常量(默认配置和错误类型)
const (
	defaultDialTimeout      = 5 * time.Second
	defaultPutTimeout       = time.Second
	defaultGetTimeout       = time.Second
	defaultDeleteTimeout    = time.Second
	defaultHeadTimeout      = time.Second
	defaultSearchTimeout    = time.Second
	defaultRangeTimeout     = time.Second
	defaultRangeHashTimeout = time.Second
	defaultPoolSize = 10
	readyObjectsCheckpointFilterName = "READY_OBJECTS_PUT_CHECKPOINT"
	allObjectsCheckpointFilterName   = "ALL_OBJECTS_PUT_CHECKPOINT"
	errEmptyTokenStore      = internal.Error("objectService.New failed: key store not provided")
	errEmptyPlacer          = internal.Error("objectService.New failed: placer not provided")
	errEmptyTransformer     = internal.Error("objectService.New failed: transformer pipeline not provided")
	errEmptyGRPC            = internal.Error("objectService.New failed: gRPC connector not provided")
	errEmptyAddress         = internal.Error("objectService.New failed: address store not provided")
	errEmptyLogger          = internal.Error("objectService.New failed: logger not provided")
	errEmptyEpochReceiver   = internal.Error("objectService.New failed: epoch receiver not provided")
	errEmptyLocalStore      = internal.Error("new local client failed: <nil> localstore passed")
	errEmptyPrivateKey      = internal.Error("objectService.New failed: private key not provided")
	errEmptyVerifier        = internal.Error("objectService.New failed: object verifier not provided")
	errEmptyACLHelper       = internal.Error("objectService.New failed: ACL helper not provided")
	errEmptyBasicACLChecker = internal.Error("objectService.New failed: basic ACL checker not provided")
	errEmptyCnrLister       = internal.Error("objectService.New failed: container lister not provided")
	errEmptySGInfoRecv      = internal.Error("objectService.New failed: SG info receiver not provided")
	errInvalidCIDFilter = internal.Error("invalid CID filter")
	errTokenRetrieval = internal.Error("objectService.Put failed on token retrieval")
	errHeaderExpected = internal.Error("expected header as a first message in stream")
)

2) 定义Service相关结构体和服务
   //service参数结构体
   	Params struct {
		CheckACL bool

		Assembly bool

		WindowSize int

		MaxProcessingSize uint64
		StorageCapacity   uint64
		PoolSize          int
		Salitor           Salitor
		LocalStore        localstore.Localstore
		Placer            Placer
		ObjectRestorer    transformer.ObjectRestorer
		RemoteService     RemoteService
		AddressStore      implementations.AddressStoreComponent
		Logger            *zap.Logger
		TokenStore        session.PrivateTokenStore
		EpochReceiver     EpochReceiver

		implementations.ContainerNodesLister

		DialTimeout time.Duration

		Key *ecdsa.PrivateKey

		PutParams       OperationParams
		GetParams       OperationParams
		DeleteParams    OperationParams
		HeadParams      OperationParams
		SearchParams    OperationParams
		RangeParams     OperationParams
		RangeHashParams OperationParams

		headRecv objectReceiver

		Verifier objutil.Verifier

		Transformer transformer.Transformer

		MaxPayloadSize uint64

		// ACL pre-processor params
		ACLHelper       implementations.ACLHelper
		BasicACLChecker libacl.BasicChecker
		IRStorage       ir.Storage
		ContainerLister implementations.ContainerNodesLister

		SGInfoReceiver storagegroup.InfoReceiver

		OwnerKeyVerifier core.OwnerKeyVerifier

		ExtendedACLSource libacl.ExtendedACLSource

		requestActionCalculator

		targetFinder RequestTargeter

		aclInfoReceiver aclInfoReceiver
	}
    //参数结构体
	OperationParams struct {
		Timeout   time.Duration
		LogErrors bool
	}

   //service结构体
   objectService struct {
		ls         localstore.Localstore
		storageCap uint64

		executor implementations.SelectiveContainerExecutor

		pPut     OperationParams
		pGet     OperationParams
		pDel     OperationParams
		pHead    OperationParams
		pSrch    OperationParams
		pRng     OperationParams
		pRngHash OperationParams

		log *zap.Logger

		requestHandler requestHandler

		objSearcher objectSearcher
		objRecv     objectReceiver
		objStorer   objectStorer
		objRemover  objectRemover
		rngRecv     objectRangeReceiver

		payloadRngRecv payloadRangeReceiver

		respPreparer responsePreparer

		getChunkPreparer   responsePreparer
		rangeChunkPreparer responsePreparer

		statusCalculator *statusCalculator
	}

    //接口
	Service interface {
		grpc.Service
		CapacityMeter
		object.ServiceServer
	}
	CapacityMeter interface {
		RelativeAvailableCap() float64
		AbsoluteAvailableCap() uint64
	}

func New(p *Params) (Service, error)
作用：构造函数,根据参数生成接口对象

func (s *objectService) Name()
作用：声明service名称，Object Service
func (s *objectService) Register(g *grpc.Server)
作用：注册grpc服务

3)其他接口：时隙接口\Placer接口\WorkerPool接口
	EpochReceiver interface {
		Epoch() uint64
	}
	RemoteService interface {
		Remote(context.Context, multiaddr.Multiaddr) (object.ServiceClient, error)
	}
	Placer interface {
		IsContainerNode(ctx context.Context, addr multiaddr.Multiaddr, cid CID, previousNetMap bool) (bool, error)
		GetNodes(ctx context.Context, addr Address, usePreviousNetMap bool, excl ...multiaddr.Multiaddr) ([]multiaddr.Multiaddr, error)
	}
	WorkerPool interface {
		Submit(func()) error
	}

2.




```



## 杂项







