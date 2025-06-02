## mockflight.0 - Specification

### Table of Contents

- [mockflight.0 - Specification](#mockflight0---specification)
  - [Table of Contents](#table-of-contents)
- [1. Goals \& Non‑Goals](#1-goalsnongoals)
- [2. Feature Matrix](#2-feature-matrix)
- [3. High‑Level Runtime Architecture](#3-highlevel-runtime-architecture)
- [4. Package Layout](#4-package-layout)
- [5. Public API](#5-public-api)
  - [5.1 Server Construction](#51server-construction)
  - [5.2 Fluent Builder DSL](#52fluent-builder-dsl)
  - [5.3 Matchers](#53matchers)
  - [5.4 Behaviors](#54behaviors)
  - [5.5 Middleware](#55middleware)
- [6. Internal Components](#6-internal-components)
  - [6.1 BehaviorRegistry](#61behaviorregistry)
  - [6.2 Server Lifecycle](#62server-lifecycle)
  - [6.3 NetConditions](#63netconditions)
- [7. Concurrency \& Data Flow](#7-concurrencydata-flow)
- [8. Error Handling \& Defaults](#8-error-handlingdefaults)
- [9. Observability](#9-observability)
  - [Tracing](#tracing)
  - [Metrics](#metrics)
  - [Capture](#capture)
- [10. Performance Considerations](#10-performance-considerations)
- [11. Testing Strategy](#11-testing-strategy)
- [12. Example Scenarios](#12-example-scenarios)
  - [12.1 Simple Happy‑Path Query](#121simple-happypath-query)
  - [12.2 Replay Real Server](#122replay-real-server)
  - [12.3 Retry Assertion](#123retry-assertion)
- [13. Roadmap](#13-roadmap)

---

## 1. Goals & Non‑Goals

| Category          | Statement                                                                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Primary Goal**  | Provide a lightweight, fully‑featured Arrow Flight SQL server implemented in Go for deterministic testing.                                     |
| **Zero‑friction** | Start an in‑process server with a single function call; no external binaries, containers, or datasets.                                         |
| **Full Protocol** | Implement **all** Flight SQL RPCs: `Handshake`, `GetFlightInfo`, `DoGet`, `DoPut`, `Execute`, `Prepare`, `BeginTransaction`, `EndTransaction`. |
| **Configurable**  | Allow users to specify exact responses, network conditions, retry behaviors, and error injections.                                             |
| **Extensible**    | Expose well‑defined hooks for custom behaviors, middleware, and future protocol extensions.                                                    |
| **Non‑Goal**      | Persistent storage or a production‑grade database. MockFlight is strictly for testing and simulation.                                          |

---

## 2. Feature Matrix

| Feature                                                          | Supported? | Notes                                                 |
| ---------------------------------------------------------------- | ---------- | ----------------------------------------------------- |
| Flight SQL RPC coverage                                          | ✅          | All RPCs as of Arrow 14 release are covered.          |
| Fluent builder DSL                                               | ✅          | Declarative test setup.                               |
| Static, scripted, proxy, and error behaviors                     | ✅          | Four built‑ins with custom behavior interface.        |
| Predicate‑based matchers (`equals`, `contains`, `regex`, `func`) | ✅          | Composable with logical `And`, `Or`, `Not`.           |
| Default/fallback behavior                                        | ✅          | User‑configured; returns `NotFound` by default.       |
| Network chaos (latency, jitter, bandwidth, drop %)               | ✅          | Injection at per‑connection level.                    |
| TLS / mTLS                                                       | ✅          | `WithTLS(cert, key)` and `WithClientCAs(pool)`.       |
| Middleware chain                                                 | ✅          | Similar to `net/http` middleware pattern.             |
| Retry tracking middleware                                        | ✅          | Counts attempts per matcher key.                      |
| OpenTelemetry tracing & metrics                                  | ✅          | Provided middleware exports spans and metrics.        |
| Golden record capture & replay                                   | ✅          | `capture.Middleware` and `behavior.Replay`.           |
| Schema validation utility                                        | ✅          | Compile‑time schema check in DSL builder.             |
| Thread‑safety across test goroutines                             | ✅          | All mutable structures protected by locks or atomics. |
| Graceful shutdown with deadline                                  | ✅          | `Shutdown(ctx)` flushes streams then stops gRPC.      |

---

## 3. High‑Level Runtime Architecture

```text
                               ┌─────────────────────────────────────────┐
                               │        mockflight.Server (gRPC)        │
                               │                                         │
 Test Code ───► DSL Builder ───► BehaviorRegistry ──► Dispatcher ──► Behavior
                               │      ▲   ▲   ▲                       │
                               │      │   │   │                       │
                               │ NetConditions Middleware             │
                               │ User Middleware Chain                │
                               │ Observability Hooks                  │
                               └─────────────────────────────────────────┘
```

* **Dispatcher** selects the first behavior whose matcher returns `true` for the incoming RPC payload.
* **Behavior** executes the RPC‑specific logic (`QueryBehavior`, `PutBehavior`, *etc.*).
* **Middleware chain** wraps every RPC in the order: network conditions ➝ user middlewares ➝ dispatcher ➝ response hook.

---

## 4. Package Layout

```text
mockflight/
├── server.go               # Server struct, lifecycle, TLS
├── options.go              # Functional options for New()
├── registry.go             # BehaviorRegistry, matcher chains
├── builder.go              # Fluent DSL
├── matcher/
│   ├── predicate.go        # Matcher type & logical combinators
│   ├── sql_equals.go
│   ├── sql_contains.go
│   ├── sql_regex.go
│   ├── ticket.go
│   └── examples_test.go
├── behavior/
│   ├── interfaces.go       # QueryBehavior, PutBehavior, etc.
│   ├── static.go           # StaticData behavior
│   ├── error.go            # Error behavior
│   ├── proxy.go            # Forward to real server & record
│   ├── scripted.go         # User function behavior
│   └── replay.go           # Golden replay behavior
├── middleware/
│   ├── netcond.go          # Latency, jitter, drops, bandwidth
│   ├── tracing.go          # OpenTelemetry spans
│   ├── retrytrack.go       # Counts attempts
│   ├── capture.go          # Golden capture
│   └── examples_test.go
├── fixtures/
│   ├── schema.go           # Helpers to build Arrow schemas
│   ├── batch.go            # StaticBatch, RandomBatch
│   └── parquet.go          # Load from Parquet file
└── examples/
    ├── integration_test.go
    ├── chaos_test.go
    └── replay_test.go
```

All packages are internal to the module; users import only `github.com/yourorg/mockflight`.

---

## 5. Public API

### 5.1 Server Construction

```go
srv := mockflight.New(
    mockflight.WithPort(0),                            // pick free port
    mockflight.WithTLS(certPEM, keyPEM),               // optional TLS
    mockflight.WithDefaultBehavior(
        behavior.Error(codes.NotFound),                // fallback
    ),
    mockflight.WithLatency(50 * time.Millisecond),     // baseline RTT
    mockflight.WithJitter(0.1),                        // ±10 % jitter
    mockflight.WithDropRate(0.03),                     // 3 % packet drop
)
```

### 5.2 Fluent Builder DSL

```go
userSchema := fixtures.Schema(`
    id:   int64,
    name: utf8
`)

userBatch := fixtures.StaticBatch(userSchema, [][]any{
    {1, "Alice"}, {2, "Bob"},
})

srv.When().
      Query(matcher.QueryContains("FROM users")).
   Then().
      ReturnWithSchema(userSchema, userBatch).
      After(1).Error(codes.DeadlineExceeded)  // fail second invocation.

srv.When().
      Query(matcher.QueryEquals("SELECT 1")).
   Then().
      Error(codes.Unavailable)

// Finally start and register test cleanup.
srv.Start(t)
```

### 5.3 Matchers

```go
// Shortcut helpers
matcher.QueryEquals("SELECT * FROM foo")
matcher.QueryContains("FROM bar")
matcher.QueryRegex(regexp.MustCompile(`(?i)^select .* from baz`))
matcher.Ticket(func(t *flight.Ticket) bool { return bytes.HasPrefix(t.Ticket, []byte("abc")) })

// Logical composition
matchFoo := matcher.QueryContains("foo").And(matcher.Not(matcher.QueryContains("bar")))
```

### 5.4 Behaviors

```go
// StaticData
staticBeh := behavior.Static(
    schema,                         // arrow.Schema
    []arrow.Record{batch1, batch2}, // records streamed via DoGet
)

// Error
errBeh := behavior.Error(codes.Internal)

// Scripted
scrBeh := behavior.Scripted(func(ctx context.Context, rpc string, payload any, resp behavior.Responder) error {
    // custom logic…
    return resp.Error(codes.Aborted, "simulated abort")
})

// Proxy (live capture)
proxyBeh := behavior.Proxy("grpc://localhost:8815", behavior.ProxyOptions{
    CaptureResponses: true,
    Timeout:          5 * time.Second,
})
```

### 5.5 Middleware

```go
srv.Use(middleware.Tracing())
srv.Use(middleware.RetryTracker())
srv.Use(middleware.Capture("/tmp/golden"))

// Access retry counts later:
attempts := middleware.GetRetryTracker(srv).Attempts()
fmt.Println(attempts["SELECT 1"]) // e.g. 3
```

---

## 6. Internal Components

### 6.1 BehaviorRegistry

```go
type entry struct {
    match matcher.Matcher
    beh   behavior.Behavior
}

type BehaviorRegistry struct {
    sync.RWMutex
    chains map[string][]entry // key = RPC name
    def    behavior.Behavior
}

func (r *BehaviorRegistry) Lookup(rpc string, cmd any) behavior.Behavior {
    r.RLock()
    chain := r.chains[rpc]
    def := r.def
    r.RUnlock()

    for _, e := range chain {
        if e.match(cmd) {
            return e.beh
        }
    }
    return def
}
```

### 6.2 Server Lifecycle

```go
type Server struct {
    grpc   *grpc.Server
    reg    *BehaviorRegistry
    netMw  middleware.NetConditions
    chain  []middleware.Func      // user middlewares
    addr   string
    tlsCfg *tls.Config
}

func (s *Server) Start(t *testing.T) {
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", s.port))
    require.NoError(t, err)

    go func() {
        if err := s.grpc.Serve(lis); err != nil && !errors.Is(err, grpc.ErrServerStopped) {
            panic(err)
        }
    }()

    t.Cleanup(func() {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = s.Shutdown(ctx)
    })
}

func (s *Server) Shutdown(ctx context.Context) error {
    stopped := make(chan struct{})
    go func() {
        s.grpc.GracefulStop()
        close(stopped)
    }()
    select {
    case <-ctx.Done():
        s.grpc.Stop() // force
        return ctx.Err()
    case <-stopped:
        return nil
    }
}
```

### 6.3 NetConditions

```go
type NetConditions struct {
    baseLatency time.Duration
    jitter      float64 // 0–1
    dropRate    float64 // 0–1
    bwLimitBps  int64
}

func (nc NetConditions) Wrap(next middleware.Func) middleware.Func {
    return func(ctx context.Context, rpc string, cmd any, resp middleware.Responder) error {
        // Simulate latency + jitter
        delay := nc.baseLatency + time.Duration(rand.NormFloat64()*nc.jitter*float64(nc.baseLatency))
        time.Sleep(delay)

        // Simulate drop
        if rand.Float64() < nc.dropRate {
            return status.Error(codes.Unavailable, "network drop")
        }

        // Bandwidth throttling handled by limited reader/writer around resp.Stream()
        return next(ctx, rpc, cmd, resp)
    }
}
```

---

## 7. Concurrency & Data Flow

1. **gRPC stream** → wrapped by NetConditions reader/writer to throttle.
2. **Dispatcher** is lock‑free once registry pointers are read.
3. **Behavior** may emit hundreds of record batches; they are sent as Arrow IPC messages on the same goroutine that handles gRPC stream to avoid cross‑goroutine synchronization.
4. **Middlewares** are executed synchronously in the order they were added.
5. **RetryTracker** uses a `sync.Map` with `atomic.Int64` counters; safe under concurrent client goroutines.

---

## 8. Error Handling & Defaults

* **Unmatched RPC**: registry returns default behavior (configurable, default is `codes.NotFound`).
* **Schema mismatch** in `ReturnWithSchema` panics during DSL build, ensuring test fails fast.
* **Fatal Behavior error**: returned status code propagated to client; middleware `OnResponse` still executes.
* **Panic inside Behavior**: recovered by gRPC interceptor, converted to `codes.Internal`.

---

## 9. Observability

### Tracing

```go
srv.Use(middleware.Tracing()) // creates span "FlightSQL.DoGet"
```

*Exports via OpenTelemetry’s global provider; user configures exporter (stdout, OTLP).*

### Metrics

* Built‑in Prometheus registry at `/metrics` (disabled by default).
* Histograms: request latency, bytes sent, bytes received.
* Counters: RPC calls, errors, drops.

### Capture

* Middleware writes JSON lines: `{"rpc":"DoGet","request":{...},"response":{...}}`.
* Works with `behavior.Replay` to stub future runs.

---

## 10. Performance Considerations

* Registry lookup is O(M) where M = matchers for a given RPC; typical values <10.
* Arrow record batches are zero‑copy; no per‑row marshaling.
* NetConditions bandwidth limiting uses `io.LimitedReader/LimitedWriter` to avoid busy‑spin sleeps.
* All allocations inside Static behavior are done at DSL build‑time; runtime is allocation‑free.
* gRPC – `grpc-go` with `grpc.WithReadBufferSize(32<<20)` to improve large batch throughput.

---

## 11. Testing Strategy

| Test Type         | Purpose                             | Implementation                 |
| ----------------- | ----------------------------------- | ------------------------------ |
| Unit tests        | Registry, matcher logic, middleware | `go test ./...`                |
| Integration tests | Full client round‑trip              | `examples/integration_test.go` |
| Chaos tests       | NetConditions correctness           | `examples/chaos_test.go`       |
| Concurrency tests | Data races under ‑race              | GitHub Actions matrix          |
| Fuzz tests        | SQL matcher fuzzing                 | `go test -fuzz=.`              |

All CI runs with `GOFLAGS=-race -count=1` to ensure race safety.

---

## 12. Example Scenarios

### 12.1 Simple Happy‑Path Query

```go
func TestSimpleQuery(t *testing.T) {
    schema := fixtures.Schema("id: int64, name: utf8")
    batch := fixtures.StaticBatch(schema, [][]any{{1, "hello"}})

    srv := mockflight.New()
    srv.When().
          Query(matcher.QueryEquals("SELECT * FROM greetings")).
       Then().
          ReturnWithSchema(schema, batch).
    srv.Start(t)

    client := flightsql.NewFlightClient(srv.Addr(), nil, nil)
    recs, err := client.Query(context.Background(), "SELECT * FROM greetings")
    require.NoError(t, err)
    require.Equal(t, int64(1), recs.NumRows())
}
```

### 12.2 Replay Real Server

```go
// Capture mode
capSrv := mockflight.New()
capSrv.When().Any().Then().Proxy(realAddr, behavior.ProxyOptions{CaptureResponses: true})
capSrv.Start(t)

// Run suite against capSrv, capture saved to /tmp/golden

// Replay mode
repSrv := mockflight.New()
repSrv.When().Any().Then().Replay("/tmp/golden")
repSrv.Start(t)
```

### 12.3 Retry Assertion

```go
rt := middleware.NewRetryTracker()
srv := mockflight.New(mockflight.WithRetryTracker(rt))
srv.When().
      Query(matcher.QueryEquals("SELECT 1")).
   Then().
      Error(codes.Unavailable)
srv.Start(t)

// client runs retry logic …

assert.Equal(t, int64(4), rt.Attempts()["SELECT 1"])
```

---

## 13. Roadmap

| Phase | Milestone | Description                                               |
| ----- | --------- | --------------------------------------------------------- |
| 1.0   | GA        | Current spec; publish on GitHub with MIT license.         |
| 1.1   | Arrow 18  | Add support for new RPCs if the spec evolves.             |
| 1.2   | Scripting | Embed Lua interpreter for stateful behaviors.             |
| 1.3   | UI        | Web dashboard to live‑inspect calls and mutate behaviors. |
| 2.0   | Cluster   | Multi‑node mock cluster to test client load‑balancing.    |
