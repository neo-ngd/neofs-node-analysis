# Object相关代码调查分析

## 源代码范围

```go
module.node.objectmanager
service.public.object.*
```

## 服务实例创建

module.node.objectmanager

service.public.object.service

![](C:\Users\Shinelon\Desktop\neo-document\NeoFS\代码分析\neofs-node-analysis\object.service\创建.jpg)



**源代码**

```go
1.构建ObjectService
func newObjectManager(p objectManagerParams) (object.Service, error) {
	var sltr object.Salitor

	if p.Viper.GetString("object.salitor") == xorSalitor {
		sltr = hash.SaltXOR
	}

	as, err := implementations.NewAddressStore(p.Peers, p.Logger)
	if err != nil {
		return nil, err
	}
    //构建RemoteService服务实例
	rs := object.NewRemoteService(p.PeersInterface)
    //从配置文件中获取特定操作参数
	pto := p.Viper.GetDuration("object.put.timeout")
	gto := p.Viper.GetDuration("object.get.timeout")
	hto := p.Viper.GetDuration("object.head.timeout")
	sto := p.Viper.GetDuration("object.search.timeout")
	rhto := p.Viper.GetDuration("object.range_hash.timeout")
	dto := p.Viper.GetDuration("object.dial_timeout")
    //构建对象运输组件
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
    //构建容器平移执行器
	exec, err := implementations.NewContainerTraverseExecutor(tr)
	if err != nil {
		return nil, err
	}
    //创建对象容器处理器
	selectiveExec, err := implementations.NewObjectContainerHandler(implementations.ObjectContainerHandlerParams{
		NodeLister: p.Placer,
		Executor:   exec,
		Logger:     p.Logger,
	})
	if err != nil {
		return nil, err
	}
    //分组
	sgInfoRecv, err := implementations.NewStorageGroupInfoReceiver(implementations.StorageGroupInfoReceiverParams{
		SelectiveContainerExecutor: selectiveExec,
		Logger:                     p.Logger,
	})
	if err != nil {
		return nil, err
	}
    //创建本地完整性验证器
	verifier, err := implementations.NewLocalIntegrityVerifier(
		core.NewNeoKeyVerifier(),
	)
	if err != nil {
		return nil, err
	}
    //创建适配器
	trans, err := transformer.NewTransformer(transformer.Params{
		SGInfoReceiver: sgInfoRecv,
		EpochReceiver:  p.EpochReceiver,
		SizeLimit:      uint64(p.Viper.GetInt64(transformersSectionPath+"payload_limiter.max_payload_size") * apiobj.UnitsKB),
		Verifier:       verifier,
	})
	if err != nil {
		return nil, err
	}
    //创建acl检测器
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


2.构建ObjectService
func New(p *Params) (Service, error) {
    //特定操作参数检查
	if p.PutParams.Timeout <= 0 {
		p.PutParams.Timeout = defaultPutTimeout
	}

	if p.GetParams.Timeout <= 0 {
		p.GetParams.Timeout = defaultGetTimeout
	}

	if p.DeleteParams.Timeout <= 0 {
		p.DeleteParams.Timeout = defaultDeleteTimeout
	}

	if p.HeadParams.Timeout <= 0 {
		p.HeadParams.Timeout = defaultHeadTimeout
	}

	if p.SearchParams.Timeout <= 0 {
		p.SearchParams.Timeout = defaultSearchTimeout
	}

	if p.RangeParams.Timeout <= 0 {
		p.RangeParams.Timeout = defaultRangeTimeout
	}

	if p.RangeHashParams.Timeout <= 0 {
		p.RangeHashParams.Timeout = defaultRangeHashTimeout
	}

	if p.DialTimeout <= 0 {
		p.DialTimeout = defaultDialTimeout
	}

	if p.PoolSize <= 0 {
		p.PoolSize = defaultPoolSize
	}
    //参数检查
	switch {
	case p.TokenStore == nil:
		return nil, errEmptyTokenStore
	case p.Placer == nil:
		return nil, errEmptyPlacer
	case p.LocalStore == nil:
		return nil, errEmptyLocalStore
	case (p.ObjectRestorer == nil || p.Transformer == nil) && p.Assembly:
		return nil, errEmptyTransformer
	case p.RemoteService == nil:
		return nil, errEmptyGRPC
	case p.AddressStore == nil:
		return nil, errEmptyAddress
	case p.Logger == nil:
		return nil, errEmptyLogger
	case p.EpochReceiver == nil:
		return nil, errEmptyEpochReceiver
	case p.Key == nil:
		return nil, errEmptyPrivateKey
	case p.Verifier == nil:
		return nil, errEmptyVerifier
	case p.IRStorage == nil:
		return nil, ir.ErrNilStorage
	case p.ContainerLister == nil:
		return nil, errEmptyCnrLister
	case p.ACLHelper == nil:
		return nil, errEmptyACLHelper
	case p.BasicACLChecker == nil:
		return nil, errEmptyBasicACLChecker
	case p.SGInfoReceiver == nil:
		return nil, errEmptySGInfoRecv
	case p.OwnerKeyVerifier == nil:
		return nil, core.ErrNilOwnerKeyVerifier
	case p.ExtendedACLSource == nil:
		return nil, libacl.ErrNilBinaryExtendedACLStore
	}
    //工作池，todo
	pool, err := ants.NewPool(p.PoolSize)
	if err != nil {
		return nil, errors.Wrap(err, "objectService.New failed: could not create worker pool")
	}

	if p.MaxProcessingSize <= 0 {
		p.MaxProcessingSize = math.MaxUint64
	}
    //容量检查
	if p.StorageCapacity <= 0 {
		p.StorageCapacity = math.MaxUint64
	}

	epochRespPreparer := &epochResponsePreparer{
		epochRecv: p.EpochReceiver,
	}

	p.targetFinder = &targetFinder{
		log:             p.Logger,
		irStorage:       p.IRStorage,
		cnrLister:       p.ContainerLister,
		cnrOwnerChecker: p.ACLHelper,
	}

	p.requestActionCalculator = &reqActionCalc{
		extACLChecker: libacl.NewExtendedACLChecker(),

		log: p.Logger,
	}

	p.aclInfoReceiver = aclInfoReceiver{
		basicACLGetter: p.ACLHelper,

		basicChecker: p.BasicACLChecker,

		targetFinder: p.targetFinder,
	}

	srv := &objectService{
		ls:         p.LocalStore,
		log:        p.Logger,
		pPut:       p.PutParams,
		pGet:       p.GetParams,
		pDel:       p.DeleteParams,
		pHead:      p.HeadParams,
		pSrch:      p.SearchParams,
		pRng:       p.RangeParams,
		pRngHash:   p.RangeHashParams,
		storageCap: p.StorageCapacity,

		requestHandler: &coreRequestHandler{
			preProc:  newPreProcessor(p),
			postProc: newPostProcessor(),
		},

		respPreparer: &complexResponsePreparer{
			items: []responsePreparer{
				epochRespPreparer,
				&aclResponsePreparer{
					aclInfoReceiver: p.aclInfoReceiver,

					reqActCalc: p.requestActionCalculator,

					eaclSrc: p.ExtendedACLSource,
				},
			},
		},

		getChunkPreparer: epochRespPreparer,

		rangeChunkPreparer: epochRespPreparer,

		statusCalculator: serviceStatusCalculator(),
	}

	tr, err := NewMultiTransport(MultiTransportParams{
		AddressStore:     p.AddressStore,
		EpochReceiver:    p.EpochReceiver,
		RemoteService:    p.RemoteService,
		Logger:           p.Logger,
		Key:              p.Key,
		PutTimeout:       p.PutParams.Timeout,
		GetTimeout:       p.GetParams.Timeout,
		HeadTimeout:      p.HeadParams.Timeout,
		SearchTimeout:    p.SearchParams.Timeout,
		RangeHashTimeout: p.RangeHashParams.Timeout,
		DialTimeout:      p.DialTimeout,

		PrivateTokenStore: p.TokenStore,
	})
	if err != nil {
		return nil, err
	}

	exec, err := implementations.NewContainerTraverseExecutor(tr)
	if err != nil {
		return nil, err
	}

	srv.executor, err = implementations.NewObjectContainerHandler(implementations.ObjectContainerHandlerParams{
		NodeLister: p.ContainerNodesLister,
		Executor:   exec,
		Logger:     p.Logger,
	})
	if err != nil {
		return nil, err
	}

	local := &localStoreExecutor{
		salitor:    p.Salitor,
		epochRecv:  p.EpochReceiver,
		localStore: p.LocalStore,
	}

	qvc := &queryVersionController{
		m: make(map[int]localQueryImposer),
	}

	qvc.m[1] = &coreQueryImposer{
		fCreator: new(coreFilterCreator),
		lsLister: p.LocalStore,
		log:      p.Logger,
	}

	localExec := &localOperationExecutor{
		objRecv:   local,
		headRecv:  local,
		objStore:  local,
		queryImp:  qvc,
		rngReader: local,
		rngHasher: local,
	}

	opExec := &coreOperationExecutor{
		pre: new(coreExecParamsComp),
		fin: &coreOperationFinalizer{
			curPlacementBuilder: &corePlacementUtil{
				prevNetMap:       false,
				placementBuilder: p.Placer,
				log:              p.Logger,
			},
			prevPlacementBuilder: &corePlacementUtil{
				prevNetMap:       true,
				placementBuilder: p.Placer,
				log:              p.Logger,
			},
			interceptorPreparer: &coreInterceptorPreparer{
				localExec:    localExec,
				addressStore: p.AddressStore,
			},
			workerPool:   pool,
			traverseExec: exec,
			resLogger: &coreResultLogger{
				mLog: requestLogMap(p),
				log:  p.Logger,
			},
			log: p.Logger,
		},
		loc: localExec,
	}

	srv.objSearcher = &coreObjectSearcher{
		executor: opExec,
	}

	childLister := &coreChildrenLister{
		queryFn:     coreChildrenQueryFunc,
		objSearcher: srv.objSearcher,
		log:         p.Logger,
		timeout:     p.SearchParams.Timeout,
	}

	childrenRecv := &coreChildrenReceiver{
		timeout: p.HeadParams.Timeout,
	}

	chopperTable := objio.NewChopperTable()

	relRecv := &neighborReceiver{
		firstChildQueryFn:    firstChildQueryFunc,
		leftNeighborQueryFn:  leftNeighborQueryFunc,
		rightNeighborQueryFn: rightNeighborQueryFunc,
		rangeDescRecv:        &selectiveRangeRecv{executor: srv.executor},
	}

	straightObjRecv := &straightObjectReceiver{
		executor: opExec,
	}

	rngRecv := &corePayloadRangeReceiver{
		chopTable: chopperTable,
		relRecv:   relRecv,
		payloadRecv: &corePayloadPartReceiver{
			rDataRecv: &straightRangeDataReceiver{
				executor: opExec,
			},
			windowController: &simpleWindowController{
				windowSize: p.WindowSize,
			},
		},
		mErr: map[error]struct{}{
			localstore.ErrOutOfRange: {},
		},
		log: p.Logger,
	}

	coreObjRecv := &coreObjectReceiver{
		straightObjRecv: straightObjRecv,
		childLister:     childLister,
		ancestralRecv: &coreAncestralReceiver{
			childrenRecv: childrenRecv,
			objRewinder: &coreObjectRewinder{
				transformer: p.ObjectRestorer,
			},
			pRangeRecv: rngRecv,
		},
		log: p.Logger,
	}
	childrenRecv.coreObjRecv = coreObjRecv
	srv.objRecv = coreObjRecv
	srv.payloadRngRecv = rngRecv

	if !p.Assembly {
		coreObjRecv.ancestralRecv, coreObjRecv.childLister = nil, nil
	}

	p.headRecv = srv.objRecv

	filter, err := newIncomingObjectFilter(p)
	if err != nil {
		return nil, err
	}

	straightStorer := &straightObjectStorer{
		executor: opExec,
	}

	bf, err := basicFilter(p)
	if err != nil {
		return nil, err
	}

	transformerObjStorer := &transformingObjectStorer{
		transformer: p.Transformer,
		objStorer:   straightStorer,
		mErr: map[error]struct{}{
			transformer.ErrInvalidSGLinking: {},

			implementations.ErrIncompleteSGInfo: {},
		},
	}

	srv.objStorer = &filteringObjectStorer{
		filter: bf,
		objStorer: &bifurcatingObjectStorer{
			straightStorer: &filteringObjectStorer{
				filter: filter,
				objStorer: &receivingObjectStorer{
					straightStorer: straightStorer,
					vPayload:       implementations.NewPayloadVerifier(),
				},
			},
			tokenStorer: &tokenObjectStorer{
				tokenStore: p.TokenStore,
				objStorer:  transformerObjStorer,
			},
		},
	}

	srv.objRemover = &coreObjRemover{
		delPrep: &coreDelPreparer{
			childLister: childLister,
		},
		straightRem: &straightObjRemover{
			tombCreator: new(coreTombCreator),
			objStorer:   transformerObjStorer,
		},
		tokenStore: p.TokenStore,
		mErr:       map[error]struct{}{},
		log:        p.Logger,
	}

	srv.rngRecv = &coreRangeReceiver{
		rngRevealer: &coreRngRevealer{
			relativeRecv: relRecv,
			chopTable:    chopperTable,
		},
		straightRngRecv: &straightRangeReceiver{
			executor: opExec,
		},
		mErr: map[error]struct{}{
			localstore.ErrOutOfRange: {},
		},
		log: p.Logger,
	}

	return srv, nil
}

```





## 请求处理流程



<img src="C:\Users\Shinelon\Desktop\neo-document\NeoFS\代码分析\neofs-node-analysis\object.service\流程.jpg" style="zoom: 200%;" />



service.public.object.*

流程：

1）用户请求时，通过GRPC服务调用ObjectService模组处理

2）ObjectService先定义异常处理逻辑

3）ObjectService根据实际情况决定是否预处理请求

4）请求交由ObjectService中的RequestHandler处理器处理

​      4.1）请求交由预处理器preProcess执行相关逻辑

​      4.2)   请求交由执行器executeProcess执行相关操作

​      4.3）请求交由后置处理器postProcess执行相关操作

5）执行结果根据实际情况，构建和发送response

​      5.1)  构建response

​      5.2)  发送response



## 预处理器类型PreProcessor

preprocessor.go

acl.go

ttl.go

token.go

verification.go



共实现了7种预处理器

基类

```gogo
requestPreProcessor interface {
   preProcess(context.Context, serviceRequest) error
}
```

实现

```go
complexPreProcessor struct {
   list []requestPreProcessor
}

func (s *complexPreProcessor) preProcess(ctx context.Context, req serviceRequest) error
执行逻辑：
遍历执行内部封装的所有预处理器的preProcess
```



```go
signingPreProcessor struct {
   preProc requestPreProcessor
   key     *ecdsa.PrivateKey
   log *zap.Logger
}

func (s *signingPreProcessor) preProcess(ctx context.Context, req serviceRequest) (err error)
执行逻辑：
1）执行内部封装的预处理器的preProcess
2）对request签名
```



```go
aclPreProcessor struct {
   log *zap.Logger

   aclInfoReceiver aclInfoReceiver

   basicChecker libacl.BasicChecker

   reqActionCalc requestActionCalculator

   localStore localstore.Localstore

   extACLSource libacl.ExtendedACLSource

   bearerVerifier bearerTokenVerifier
}

func (p *aclPreProcessor) preProcess(ctx context.Context, req serviceRequest) error 
执行逻辑：
1)检查request是否为空
2)获取ACL信息
3)ACL权限检查：acl匹配规则
4)ACL权限检查:是否是内环请求以及内环请求合法性
5)ACL权限检查:是否既不包含令牌又不包含扩展检查
6)ACL权限检查:接入者令牌合法性
7)ACL权限检查:扩展检查
```



```go
ttlPreProcessor struct {
   staticCond []service.TTLCondition
   condPreps []ttlConditionPreparer
   fProc func(service.TTLSource, ...service.TTLCondition) error
}

ttlConditionPreparer interface {
	prepareTTLCondition(context.Context, object.Request) service.TTLCondition
}

containerAffiliationChecker interface {
	affiliated(context.Context, CID) containerAffiliationResult
}

func (s *ttlPreProcessor) preProcess(ctx context.Context, req serviceRequest) error 
执行逻辑：
1)检查request是否为空
2)构建动态TTL匹配函数条件组dynamicCond
3)获取request ttl，遍历dynamicCond，判断是否满足
```



```go
decTTLPreProcessor struct {
}

func (s *decTTLPreProcessor) preProcess(_ context.Context, req serviceRequest) error 
执行逻辑：
请求有效期-1
```



```go
tokenPreProcessor struct {
   keyVerifier core.OwnerKeyVerifier

   staticVerifier sessionTokenVerifier
}

func (s tokenPreProcessor) preProcess(ctx context.Context, req serviceRequest) error 
执行逻辑：
1)获取session令牌
2)验证令牌是否包含request该类型权限
3)验证令牌密钥格式规范
4)验证令牌签名合法性
```



```go

verifyRequestFunc func(token service.RequestVerifyData) error

verifyPreProcessor struct {
	fVerify verifyRequestFunc
}

func (s *verifyPreProcessor) preProcess(_ context.Context, req serviceRequest) (err error) 
执行逻辑：
1)检查请求是否为空
2)调用验证函数验证请求合法性
```



| 名称                | 用途                                               | 类型      |
| ------------------- | -------------------------------------------------- | --------- |
| complexPreProcessor | 封装多个预处理器                                   | 封装 复合 |
| signingPreProcessor | 封装一个预处理器，并在预处理request后对request签名 | 封装 单一 |
| aclPreProcessor     | 与acl相关的预处理器                                | 单一      |
| ttlPreProcessor     | 与有效期ttl相关的处理器                            | 单一      |
| decTTLPreProcessor  | 与有效期ttl相关的处理器，降低                      | 单一      |
| tokenPreProcessor   | 与session 令牌相关的处理器                         | 单一      |
| verifyPreProcessor  | 与请求request自身验证相关的处理器                  | 单一      |



## 后置处理器类型PostProcessor

postprocess.go

暂时无效

```go
requestPostProcessor interface {
   postProcess(context.Context, serviceRequest, error)
}
```



```go
complexPostProcessor struct {
   list []requestPostProcessor
}
```





## 执行器requestHandleExecutor

handler.go

```go
requestHandleExecutor interface {
   // Executes actions parameter-bound logic and returns execution result.
   executeRequest(context.Context, serviceRequest) (interface{}, error)
}

func (s *objectService) executeRequest(ctx context.Context, req serviceRequest) (interface{}, error) 
执行逻辑：
根据请求类型返回不同的执行器，并由执行器调用相应的操作执行器operationExecutor
switch  requestType
case SearchRequest:
case putRequest:
case DeleteRequest:
case GetRequest:
case HeadRequest:
case GetRangeRequest:
case GetRangeHashRequest:
```



Executor内部操作执行器

execution.go

```go
operationExecutor interface {
   executeOperation(context.Context, transport.MetaInfo, responseItemHandler) error
}

//需要远程完成的操作
coreOperationExecutor struct {
	pre executionParamsComputer
	fin operationFinalizer
	loc operationExecutor
}

//本地就可以完成的操作
localOperationExecutor struct {
	objRecv   localFullObjectReceiver
	headRecv  localHeadReceiver
	objStore  localObjectStorer
	queryImp  localQueryImposer
	rngReader localRangeReader
	rngHasher localRangeHasher
}


//本地存储
localStoreExecutor struct {
	salitor    Salitor
	epochRecv  EpochReceiver
	localStore localstore.Localstore
}

operationFinalizer interface {
	completeExecution(context.Context, operationParams) error
}

coreOperationFinalizer struct {
	curPlacementBuilder  placementBuilder
	prevPlacementBuilder placementBuilder
	interceptorPreparer  interceptorPreparer
	workerPool           WorkerPool
	traverseExec         implementations.ContainerTraverseExecutor
	resLogger            resultLogger
	log                  *zap.Logger
}


func (s *coreOperationExecutor) executeOperation(ctx context.Context, req transport.MetaInfo, h responseItemHandler) error 
执行逻辑：
1.检查request 的TTL是否为0，
2.为0，则通过本地执行器localOperationExecutor
3.不为0，则准备相关参数并执行completeExecution


func (s *localOperationExecutor) executeOperation(ctx context.Context, req transport.MetaInfo, h responseItemHandler) error 
执行逻辑：
根据不同的请求类型，执行不同的操作：
switch requestType:
case object.RequestPut：
case object.RequestGet:
case object.RequestHead:
case object.RequestSearch:
case object.RequestRange:
case object.RequestRangeHash:


func (s *coreOperationFinalizer) completeExecution(ctx context.Context, p operationParams) error 
执行逻辑：
1)
```



**远程服务**

implementation.go

```go
1)定义remoteService结构体
	remoteService struct {
		ps peers.Interface
	}

2)声明remoteService构造函数
func NewRemoteService(ps peers.Interface) RemoteService
作用：构造函数，创建RemoteService实例

3)声明remoteService的远端连接方法
func (rs remoteService) Remote(ctx context.Context, addr multiaddr.Multiaddr) (object.ServiceClient, error)
作用：获取GRPC的远端连接
```



transport_implementation.go

连接组件实现类

```go
func NewMultiTransport(p MultiTransportParams) (transport.ObjectTransport, error)
构造方法

func (s *transportComponent) Transport(ctx context.Context, p transport.ObjectTransportParams)
根据参数，调用远程处理器，发送请求

remoteProcessCaller interface {
	call(context.Context, serviceRequest, *clientInfo) (interface{}, error)
}
远程调用器，定义了5种
getCaller/putCaller/headCaller/rangeCaller/rangeHashCaller

transport---->Caller----(send request)---->tracker监听请求

```






## 其他

**Verb映射**

session的令牌所代表的操作和请求类型的别名,  注意位计算

verb.go

状态表

| 名称               | 值   | 备注                                                         |
| ------------------ | ---- | ------------------------------------------------------------ |
| undefinedVerbDesc  | 1    | 单                                                           |
| putVerbDesc        | 2    | 单                                                           |
| getVerbDesc        | 4    | 单                                                           |
| headVerbDesc       | 8    | 单                                                           |
| deleteVerbDesc     | 16   | 单                                                           |
| searchVerbDesc     | 32   | 单                                                           |
| rangeVerbDesc      | 64   | 单                                                           |
| rangeHashVerbDesc  | 128  | 单                                                           |
| headSpawnMask      | 206  | 复合     **headVerbDesc** or **getVerbDesc** or  **putVerbDesc** or  **rangeVerbDesc** or  **rangeHashVerbDesc** |
| rangeHashSpawnMask | 128  | 复合    **rangeHashVerbDesc**                                |
| rangeSpawnMask     | 68   | 复合    **rangeVerbDesc** or  **getVerbDesc**                |
| getSpawnMask       | 4    | 复合    **getVerbDesc**                                      |
| putSpawnMask       | 18   | 复合   **putVerbDesc** or  **deleteVerbDesc**                |
| deleteSpawnMask    | 16   | 复合   **deleteVerbDesc**                                    |
| searchSpawnMask    | 254  | 复合   **searchVerbDesc** or  **getVerbDesc** or  **putVerbDesc** or  **headVerbDesc** or  **rangeVerbDesc** or  **rangeHashVerbDesc** or  **deleteVerbDesc** |

令牌所操作映射

| 名称                 | 动作              |
| -------------------- | ----------------- |
| Token_Info_Put       | putVerbDesc       |
| Token_Info_Get       | getVerbDesc       |
| Token_Info_Head      | headVerbDesc      |
| Token_Info_Delete    | deleteVerbDesc    |
| Token_Info_Search    | searchVerbDesc    |
| Token_Info_Range     | rangeVerbDesc     |
| Token_Info_RangeHash | rangeHashVerbDesc |

请求类型映射

| RequestPut       | putSpawnMask       |
| ---------------- | ------------------ |
| RequestGet       | getSpawnMask       |
| RequestHead      | headSpawnMask      |
| RequestDelete    | deleteSpawnMask    |
| RequestSearch    | searchSpawnMask    |
| RequestRange     | rangeSpawnMask     |
| RequestRangeHash | rangeHashSpawnMask |



**容量**

capacity.go

```go
实现objectService的CapacityMeter接口
1) 声明函数RelativeAvailableCap() float64
作用：返回相对可用容量=(存储容量-本地容量)/存储容量  百分比

2) 声明函数AbsoluteAvailableCap() uint64
作用：返回觉对可用容量=存储容量-本地容量           实际大小

```



**错误常量**

status.go



**容器封装**

Traverse.go

构建容器的封装类

```go
containerTraverser interface {
   implementations.Traverser
   add(multiaddr.Multiaddr, bool)
   done(multiaddr.Multiaddr) bool
   finished() bool
   close()
   Err() error
}

func (s *coreTraverser) next(ctx context.Context) (nodes []multiaddr.Multiaddr)
每一轮涉及的节点
```