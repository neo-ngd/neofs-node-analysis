# Worker相关代码调查分析

## 源代码范围

```go
modules.workers
lib.fix.worker
```

worker.go

```go
package worker

import (
   "context"
   "sync"
   "sync/atomic"
   "time"
)

type (
   // Workers is an interface of worker tool.
   Workers interface {
      Start(context.Context)
      Stop()

      Add(Job Handler)
   }

   workers struct {
      cancel  context.CancelFunc
      started *int32
      wg      *sync.WaitGroup
      jobs    []Handler
   }

   // Handler is a worker's handling function.
   Handler func(ctx context.Context)

   // Jobs is a map of worker names to handlers.
   Jobs map[string]Handler

   // Job groups the parameters of worker's job.
   Job struct {
      Disabled    bool
      Immediately bool
      Timer       time.Duration
      Ticker      time.Duration
      Handler     Handler
   }
)

// New is a constructor of workers.
func New() Workers {
   return &workers{
      started: new(int32),
      wg:      new(sync.WaitGroup),
   }
}

func (w *workers) Add(job Handler) {
   w.jobs = append(w.jobs, job)
}

func (w *workers) Stop() {
   if !atomic.CompareAndSwapInt32(w.started, 1, 0) {
      // already stopped
      return
   }

   w.cancel()
   w.wg.Wait()
}

func (w *workers) Start(ctx context.Context) {
   if !atomic.CompareAndSwapInt32(w.started, 0, 1) {
      // already started
      return
   }

   ctx, w.cancel = context.WithCancel(ctx)
   for _, job := range w.jobs {
      w.wg.Add(1)

      go func(handler Handler) {
         defer w.wg.Done()
         handler(ctx)
      }(job)
   }
}
```

源代码分析

```go
一个线程池模型

启动---》每个任务单独线程工作----》停止
支持添加任务
支持总线停止
带有线程安全锁
```





prepare.go

```go
package workers

import (
   "context"
   "time"

   "github.com/nspcc-dev/neofs-node/lib/fix/worker"
   "github.com/spf13/viper"
   "go.uber.org/dig"
   "go.uber.org/zap"
)

type (
   // Result returns wrapped workers group for DI.
   Result struct {
      dig.Out

      Workers []*worker.Job
   }

   // Params is dependencies for create workers slice.
   Params struct {
      dig.In

      Jobs   worker.Jobs
      Viper  *viper.Viper
      Logger *zap.Logger
   }
)

func prepare(p Params) worker.Workers {
   w := worker.New()

   for name, handler := range p.Jobs {
      if job := byConfig(name, handler, p.Logger, p.Viper); job != nil {
         p.Logger.Debug("worker: add new job",
            zap.String("name", name))

         w.Add(job)
      }
   }

   return w
}

func byTicker(d time.Duration, h worker.Handler) worker.Handler {
   return func(ctx context.Context) {
      ticker := time.NewTicker(d)
      defer ticker.Stop()

      for {
         select {
         case <-ctx.Done():
            return
         default:
            select {
            case <-ctx.Done():
               return
            case <-ticker.C:
               h(ctx)
            }
         }
      }
   }
}

func byTimer(d time.Duration, h worker.Handler) worker.Handler {
   return func(ctx context.Context) {
      timer := time.NewTimer(d)
      defer timer.Stop()

      for {
         select {
         case <-ctx.Done():
            return
         default:
            select {
            case <-ctx.Done():
               return
            case <-timer.C:
               h(ctx)
               timer.Reset(d)
            }
         }
      }
   }
}

func byConfig(name string, h worker.Handler, l *zap.Logger, v *viper.Viper) worker.Handler {
   var job worker.Handler

   if !v.IsSet("workers." + name) {
      l.Info("worker: has no configuration",
         zap.String("worker", name))
      return nil
   }

   if v.GetBool("workers." + name + ".disabled") {
      l.Info("worker: disabled",
         zap.String("worker", name))
      return nil
   }

   if ticker := v.GetDuration("workers." + name + ".ticker"); ticker > 0 {
      job = byTicker(ticker, h)
   }

   if timer := v.GetDuration("workers." + name + ".timer"); timer > 0 {
      job = byTimer(timer, h)
   }

    //无限循环任务
   if v.GetBool("workers." + name + ".immediately") {
      return func(ctx context.Context) {
         h(ctx)

         if job == nil {
            return
         }

         // check context before run immediately job again
         select {
         case <-ctx.Done():
            return
         default:
         }

         job(ctx)
      }
   }

   return job
}

```



源代码分析

```go
工作线程辅助类

func prepare(p Params) worker.Workers 
作用：按照配置文件，生成一个带有任务的工作线程池

func byConfig(name string, h worker.Handler, l *zap.Logger, v *viper.Viper) worker.Handler
作用：按照配置文件，生成指定执行模式的任务
执行逻辑：
1.检查配置文件中是否配置workers.name，没有配置返回null
2.检查配置文件中是否disabled，没有配置返回null
3.检查是否是ticker定时任务,是生成定时模式的任务
4.检查是否是timer定时任务,是生成定时模式的任务
5.检查是否是立即执行的任务，,是生成立即执行但是执行后会如果未完成会定时尝试的任务


func byTimer(d time.Duration, h worker.Handler) worker.Handler
func byTicker(d time.Duration, h worker.Handler) worker.Handler
作用：定时任务
执行逻辑：
死循环{
    设置定时
    检查任务是否完成
    检查是否超时
    执行handler任务
}

```



