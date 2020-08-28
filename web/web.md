# Web相关代码调查分析

## 源代码范围

```go
modules.network.module
lib.fix.web
```



http.go

server.go

```go
package web

import (
   "context"
   "net/http"
   "sync/atomic"
   "time"

   "github.com/spf13/viper"
   "go.uber.org/zap"
)

type (
   httpParams struct {
      Key     string
      Viper   *viper.Viper
      Logger  *zap.Logger
      Handler http.Handler
   }

   httpServer struct {
      name        string
      started     *int32
      logger      *zap.Logger
      shutdownTTL time.Duration
      server      server
   }
)

func (h *httpServer) Start(ctx context.Context) {
   if h == nil {
      return
   }

   if !atomic.CompareAndSwapInt32(h.started, 0, 1) {
      h.logger.Info("http: already started",
         zap.String("server", h.name))
      return
   }

   go func() {
      if err := h.server.serve(ctx); err != nil {
         if err != http.ErrServerClosed {
            h.logger.Error("http: could not start server",
               zap.Error(err))
         }
      }
   }()
}

func (h *httpServer) Stop() {
   if h == nil {
      return
   }

   if !atomic.CompareAndSwapInt32(h.started, 1, 0) {
      h.logger.Info("http: already stopped",
         zap.String("server", h.name))
      return
   }

   ctx, cancel := context.WithTimeout(context.Background(), h.shutdownTTL)
   defer cancel()

   h.logger.Debug("http: try to stop server",
      zap.String("server", h.name))

   if err := h.server.shutdown(ctx); err != nil {
      h.logger.Error("http: could not stop server",
         zap.Error(err))
   }
}

const defaultShutdownTTL = 30 * time.Second

func newHTTPServer(p httpParams) *httpServer {
   var (
      address  string
      shutdown time.Duration
   )

   if address = p.Viper.GetString(p.Key + ".address"); address == "" {
      p.Logger.Info("Empty bind address, skip",
         zap.String("server", p.Key))
      return nil
   }
   if p.Handler == nil {
      p.Logger.Info("Empty handler, skip",
         zap.String("server", p.Key))
      return nil
   }

   p.Logger.Info("Create http.Server",
      zap.String("server", p.Key),
      zap.String("address", address))

   if shutdown = p.Viper.GetDuration(p.Key + ".shutdown_ttl"); shutdown <= 0 {
      shutdown = defaultShutdownTTL
   }

   return &httpServer{
      name:        p.Key,
      started:     new(int32),
      logger:      p.Logger,
      shutdownTTL: shutdown,
      server: newServer(params{
         Address: address,
         Name:    p.Key,
         Config:  p.Viper,
         Logger:  p.Logger,
         Handler: p.Handler,
      }),
   }
}
```

```go
package web

import (
   "context"
   "net/http"

   "github.com/spf13/viper"
   "go.uber.org/zap"
)

type (
   // Server is an interface of server.
   server interface {
      serve(ctx context.Context) error
      shutdown(ctx context.Context) error
   }

   contextServer struct {
      logger *zap.Logger
      server *http.Server
   }

   params struct {
      Address string
      Name    string
      Config  *viper.Viper
      Logger  *zap.Logger
      Handler http.Handler
   }
)

func newServer(p params) server {
   return &contextServer{
      logger: p.Logger,
      server: &http.Server{
         Addr:              p.Address,
         Handler:           p.Handler,
         ReadTimeout:       p.Config.GetDuration(p.Name + ".read_timeout"),
         ReadHeaderTimeout: p.Config.GetDuration(p.Name + ".read_header_timeout"),
         WriteTimeout:      p.Config.GetDuration(p.Name + ".write_timeout"),
         IdleTimeout:       p.Config.GetDuration(p.Name + ".idle_timeout"),
         MaxHeaderBytes:    p.Config.GetInt(p.Name + ".max_header_bytes"),
      },
   }
}

func (cs *contextServer) serve(ctx context.Context) error {
   go func() {
      <-ctx.Done()

      if err := cs.server.Close(); err != nil {
         cs.logger.Info("something went wrong",
            zap.Error(err))
      }
   }()

   return cs.server.ListenAndServe()
}

func (cs *contextServer) shutdown(ctx context.Context) error {
   return cs.server.Shutdown(ctx)
}
```



源代码分析

```go
创建server实例---》启动---》httpHandler--->停止
```



metrics.go

```go
package web

import (
   "context"

   "github.com/prometheus/client_golang/prometheus/promhttp"
   "github.com/spf13/viper"
   "go.uber.org/zap"
)

// Metrics is an interface of metric tool.
type Metrics interface {
   Start(ctx context.Context)
   Stop()
}

const metricsKey = "metrics"

// NewMetrics is a metric tool's constructor.
func NewMetrics(l *zap.Logger, v *viper.Viper) Metrics {
   if !v.GetBool(metricsKey + ".enabled") {
      l.Debug("metrics server disabled")
      return nil
   }

   return newHTTPServer(httpParams{
      Key:     metricsKey,
      Viper:   v,
      Logger:  l,
      Handler: promhttp.Handler(),
   })
}
```

源代码分析

```go
性能监控服务器
```





```go
package web

import (
   "context"
   "expvar"
   "net/http"
   "net/http/pprof"

   "github.com/spf13/viper"
   "go.uber.org/zap"
)

// Profiler is an interface of profiler.
type Profiler interface {
   Start(ctx context.Context)
   Stop()
}

const profilerKey = "pprof"

// NewProfiler is a profiler's constructor.
func NewProfiler(l *zap.Logger, v *viper.Viper) Profiler {
   if !v.GetBool(profilerKey + ".enabled") {
      l.Debug("pprof server disabled")
      return nil
   }

   mux := http.NewServeMux()

   mux.Handle("/debug/vars", expvar.Handler())

   mux.HandleFunc("/debug/pprof/", pprof.Index)
   mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
   mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
   mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
   mux.HandleFunc("/debug/pprof/trace", pprof.Trace)

   return newHTTPServer(httpParams{
      Key:     profilerKey,
      Viper:   v,
      Logger:  l,
      Handler: mux,
   })
}
```

源代码分析

```go
性能分析工具
```

