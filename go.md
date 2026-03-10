# Go Language — Technical Interview Preparation Guide

> Work through each chapter sequentially. Check off topics as you complete them.

---

## Chapter 1 — [Go Basics & Setup](go/GoBasicsAndSetup.md)

- [ ] Go history, philosophy, and design goals (simplicity, concurrency, fast compilation)
- [ ] Installation, `GOPATH`, `GOROOT`, Go modules (`go mod init`, `go.sum`)
- [ ] CLI tools: `go build`, `go run`, `go install`, `go fmt`, `go vet`, `go doc`
- [ ] Package structure, `main` package, entry point `func main()`
- [ ] `init()` function (runs before `main`, multiple per file, execution order)
- [ ] Import paths, blank imports (`_ "pkg"`), dot imports, aliased imports
- [ ] Comments & documentation — `godoc`, `//go:` directives (`//go:build`, `//go:generate`, `//go:embed`)
- [ ] Workspace layout conventions — `cmd/`, `internal/`, `pkg/`

---

## Chapter 2 — [Data Types & Variables](go/DataTypesAndVariables.md)

- [ ] Basic types: `int`, `int8/16/32/64`, `uint`, `uint8/16/32/64`, `uintptr`
- [ ] Floating point: `float32`, `float64`
- [ ] Complex: `complex64`, `complex128`
- [ ] Other: `byte` (alias for `uint8`), `rune` (alias for `int32`), `string`, `bool`
- [ ] Zero values (every type has a default: `0`, `""`, `false`, `nil`)
- [ ] Variable declaration: `var x int`, short declaration `:=`, multiple assignment
- [ ] Constants (`const`, `iota` for enum-like auto-incrementing)
- [ ] Untyped constants and their flexibility
- [ ] Type conversion (`int(x)`, `float64(y)`) — no implicit conversions in Go
- [ ] Type alias (`type MyInt = int`) vs type definition (`type MyInt int`)

---

## Chapter 3 — [Control Flow](go/ControlFlow.md)

- [ ] `if` / `else if` / `else`
- [ ] `if` with init statement (`if err := fn(); err != nil { }`)
- [ ] `for` loop — classic, while-style (`for condition {}`), infinite (`for {}`), range-based
- [ ] `for range` over slices, maps, strings, channels, integers (Go 1.22+)
- [ ] `switch` — expression switch, tagless switch, `fallthrough`
- [ ] Type switch (`switch v := x.(type) { }`)
- [ ] `select` statement (channel multiplexing)
- [ ] `goto`, labels
- [ ] `break` / `continue` with labels (exiting nested loops)
- [ ] `defer` — LIFO execution order, arguments evaluated immediately at defer site

---

## Chapter 4 — [Functions](go/Functions.md)

- [ ] Function declarations, parameters, return values
- [ ] Multiple return values (`func f() (int, error)`)
- [ ] Named return values and naked return
- [ ] Variadic functions (`func f(nums ...int)`)
- [ ] First-class functions — functions as values and types
- [ ] Anonymous functions (closures) and variable capture
- [ ] Recursion
- [ ] `defer` — deferred function calls, cleanup pattern
- [ ] `panic(value)` — triggers a runtime panic
- [ ] `recover()` — catches panic in deferred function
- [ ] `defer`/`panic`/`recover` error recovery pattern
- [ ] Method values (`obj.Method`) and method expressions (`Type.Method`)

---

## Chapter 5 — [Composite Types](go/CompositeTypes.md)

### Arrays
- [ ] Fixed-size, value semantics (copied on assignment)
- [ ] Declaration: `[3]int{1, 2, 3}`, `[...]int{1, 2, 3}`
- [ ] Array comparison (`==`)

### Slices
- [ ] Slice internals — header: pointer, length, capacity
- [ ] Creating slices: literal, `make([]T, len, cap)`, slicing an array/slice
- [ ] `append()` — growth strategy, when underlying array is replaced
- [ ] `copy()` — copies min(len(dst), len(src)) elements
- [ ] Nil slice vs empty slice (`var s []int` vs `s := []int{}`)
- [ ] Slice tricks: delete, insert, filter, reverse
- [ ] Full slice expression (`a[low:high:max]`) to control capacity
- [ ] `slices` package (Go 1.21+): `slices.Sort`, `slices.Contains`, `slices.Delete`

### Maps
- [ ] Creation: `make(map[K]V)`, map literal
- [ ] Access, insertion, deletion (`delete(m, key)`)
- [ ] Comma-ok idiom (`val, ok := m[key]`)
- [ ] Iteration order is **not** guaranteed (randomized)
- [ ] Nil map behavior (reads return zero value, writes panic)
- [ ] `maps` package (Go 1.21+): `maps.Keys`, `maps.Values`, `maps.Clone`

### Structs
- [ ] Struct definition and initialization (named fields, positional)
- [ ] Anonymous / embedded fields (struct embedding)
- [ ] Struct tags (`json:"name"`, `db:"col"`)
- [ ] Struct comparison (comparable if all fields are comparable)

---

## Chapter 6 — [Strings & Unicode](go/StringsAndUnicode.md)

- [ ] String internals — immutable byte slice (`[]byte` underneath)
- [ ] `rune` vs `byte` — rune is a Unicode code point
- [ ] Iterating: `for i, b := range []byte(s)` vs `for i, r := range s` (rune iteration)
- [ ] `strings` package: `Builder`, `Split`, `Join`, `Contains`, `HasPrefix`, `HasSuffix`, `Replace`, `Trim`, `ToUpper`, `ToLower`, `NewReader`, `Map`
- [ ] `strconv` package: `Atoi`, `Itoa`, `ParseInt`, `ParseFloat`, `FormatInt`
- [ ] `fmt` package verbs: `%v`, `%+v`, `%#v`, `%T`, `%s`, `%d`, `%f`, `%p`, `%q`, `%w`
- [ ] `unicode/utf8` package: `RuneCountInString`, `ValidString`, `DecodeRuneInString`
- [ ] `len(s)` returns **byte count**, not rune count — use `utf8.RuneCountInString` for character count
- [ ] String concatenation performance: `+` operator vs `strings.Builder` vs `bytes.Buffer`

---

## Chapter 7 — [Pointers](go/Pointers.md)

- [ ] Pointer basics: `*T` (pointer type), `&x` (address of), `*p` (dereference)
- [ ] No pointer arithmetic in Go (safety by design)
- [ ] `new(T)` — allocates zeroed memory, returns `*T`
- [ ] `make(T, ...)` — creates slices, maps, channels (returns `T`, not `*T`)
- [ ] `new` vs `make` — when to use which
- [ ] Nil pointers — dereferencing causes panic
- [ ] Passing pointers to functions — for efficiency and mutation
- [ ] Pointer vs value semantics — consistency rule
- [ ] Pointers to interfaces — almost never needed (interface already holds a pointer internally)
- [ ] Stack vs heap — compiler decides allocation via escape analysis

---

## Chapter 8 — [Structs & Methods](go/StructsAndMethods.md)

- [ ] Defining methods: `func (s MyStruct) Method() {}`
- [ ] Value receivers vs pointer receivers
- [ ] When to use pointer receivers (mutating state, large structs, consistency)
- [ ] Method sets — value type has value-receiver methods; pointer type has both
- [ ] Exported (`Upper`) vs unexported (`lower`) fields and methods
- [ ] Struct embedding — promoted methods, composition over inheritance
- [ ] Constructor pattern: `func NewXxx(args) *Xxx { ... }`
- [ ] Methods on non-struct types (`type MySlice []int`)

---

## Chapter 9 — [Interfaces](go/Interfaces.md)

- [ ] Implicit interface satisfaction — no `implements` keyword
- [ ] Interface definition: `type Reader interface { Read(p []byte) (n int, err error) }`
- [ ] Empty interface: `interface{}` / `any` (Go 1.18+)
- [ ] Interface composition — embedding interfaces inside interfaces
- [ ] Type assertion: `val, ok := i.(ConcreteType)`
- [ ] Type switch: `switch v := i.(type) { case int: ... }`
- [ ] Interface internals — (type, value) pair, itab
- [ ] **Nil interface vs interface holding nil pointer** — critical gotcha
- [ ] Common standard library interfaces:
  - `fmt.Stringer` (`String() string`)
  - `error` (`Error() string`)
  - `io.Reader`, `io.Writer`, `io.Closer`, `io.ReadWriter`
  - `sort.Interface` (`Len`, `Less`, `Swap`)
  - `encoding.BinaryMarshaler`, `json.Marshaler`
  - `http.Handler` (`ServeHTTP`)
  - `context.Context`
- [ ] Best practice: accept interfaces, return concrete types
- [ ] Small interfaces — Go idiom (1–2 methods)

---

## Chapter 10 — [Error Handling](go/ErrorHandling.md)

- [ ] `error` interface: `type error interface { Error() string }`
- [ ] Creating errors: `errors.New("msg")`, `fmt.Errorf("msg: %v", err)`
- [ ] Error wrapping (Go 1.13+): `fmt.Errorf("context: %w", err)`
- [ ] `errors.Unwrap(err)` — unwrap one layer
- [ ] `errors.Is(err, target)` — check error chain for match
- [ ] `errors.As(err, &target)` — find specific error type in chain
- [ ] Custom error types: implement `Error() string`
- [ ] Sentinel errors (`var ErrNotFound = errors.New("not found")`)
- [ ] `panic` — when to use (truly unrecoverable situations)
- [ ] `recover` — catching panics in deferred functions
- [ ] Error handling patterns:
  - Early return on error
  - Wrapping with context at each layer
  - Error type hierarchies
- [ ] `errors.Join` (Go 1.20+) — combining multiple errors

---

## Chapter 11 — [Goroutines & Concurrency](go/GoroutinesAndConcurrency.md)

- [ ] Goroutine basics: `go func() { ... }()`
- [ ] Goroutines vs OS threads — lightweight, ~2KB initial stack
- [ ] Go scheduler: M:N model (M OS threads, N goroutines, P processors)
- [ ] `GOMAXPROCS` — number of OS threads for goroutines
- [ ] `sync.WaitGroup` — `Add`, `Done`, `Wait`
- [ ] Race conditions — shared state without synchronization
- [ ] Race detector: `go run -race .`
- [ ] Goroutine leaks — causes and prevention (always ensure goroutines can exit)
- [ ] `runtime.Gosched()` — yield processor
- [ ] `runtime.NumGoroutine()` — count active goroutines
- [ ] `runtime.GOMAXPROCS(n)` — set/get processor count

---

## Chapter 12 — [Channels](go/Channels.md)

- [ ] Unbuffered channels: `make(chan T)` — synchronous send/receive
- [ ] Buffered channels: `make(chan T, cap)` — async up to capacity
- [ ] Directional channels: `chan<- T` (send-only), `<-chan T` (receive-only)
- [ ] Blocking behavior: send blocks when buffer full, receive blocks when empty
- [ ] Closing channels: `close(ch)` — signals no more values
- [ ] Receiving from closed channel returns zero value
- [ ] Range over channel: `for v := range ch { }` — reads until closed
- [ ] Nil channel behavior: send/receive on nil channel blocks forever
- [ ] `select` statement:
  - Multi-way channel operations
  - `default` case (non-blocking)
  - Timeout: `case <-time.After(d)`
  - Random selection when multiple cases ready
- [ ] Channel patterns:
  - Done channel / cancellation
  - Fan-in (merge multiple channels into one)
  - Fan-out (distribute work to multiple goroutines)
  - Pipeline (chain of stages connected by channels)
  - Generator pattern
  - Semaphore (buffered channel as counting semaphore)

---

## Chapter 13 — [Synchronization Primitives](go/SynchronizationPrimitives.md)

- [ ] `sync.Mutex` — `Lock()`, `Unlock()`
- [ ] `sync.RWMutex` — `RLock()`, `RUnlock()`, `Lock()`, `Unlock()` (multiple readers, single writer)
- [ ] `sync.Once` — `Do(func())` — execute exactly once (safe for singleton)
- [ ] `sync.WaitGroup` — coordinate goroutine completion
- [ ] `sync.Map` — concurrent-safe map (use when keys are stable or disjoint)
- [ ] `sync.Pool` — reuse temporary objects, reduce GC pressure
- [ ] `sync.Cond` — condition variable (`Wait`, `Signal`, `Broadcast`)
- [ ] `atomic` package:
  - `atomic.AddInt64`, `atomic.LoadInt64`, `atomic.StoreInt64`
  - `atomic.CompareAndSwapInt64`
  - `atomic.Value` — `Load`, `Store` (for any type)
- [ ] Mutex vs channel — when to use which

---

## Chapter 14 — [Context Package](go/ContextPackage.md)

- [ ] `context.Background()` — root context, top-level calls
- [ ] `context.TODO()` — placeholder when unsure which context to use
- [ ] `context.WithCancel(parent)` — returns `ctx, cancel`; call `cancel()` to propagate
- [ ] `context.WithTimeout(parent, duration)` — auto-cancels after duration
- [ ] `context.WithDeadline(parent, time)` — auto-cancels at specific time
- [ ] `context.WithValue(parent, key, val)` — attach request-scoped values
- [ ] `ctx.Done()` — returns channel closed on cancellation
- [ ] `ctx.Err()` — `context.Canceled` or `context.DeadlineExceeded`
- [ ] Context propagation — pass as first argument `func Foo(ctx context.Context, ...)`
- [ ] Best practices:
  - Never store context in a struct
  - Always pass as first parameter
  - Use `WithValue` sparingly, prefer explicit parameters
  - Always call `cancel()` (use `defer cancel()`)
- [ ] `context.WithoutCancel` (Go 1.21+), `context.AfterFunc` (Go 1.21+)

---

## Chapter 15 — [Generics (Go 1.18+)](go/Generics.md)

- [ ] Type parameters: `func Print[T any](val T) { }`
- [ ] Type constraints — interfaces that restrict type parameters
- [ ] Built-in constraints: `any`, `comparable`
- [ ] `constraints` package → moved to `cmp` package (Go 1.21+): `cmp.Ordered`
- [ ] Generic functions: `func Max[T cmp.Ordered](a, b T) T { }`
- [ ] Generic types: `type Stack[T any] struct { items []T }`
- [ ] Type inference — compiler infers type arguments from function arguments
- [ ] Interface type sets — union elements: `type Number interface { int | float64 }`
- [ ] Tilde `~` — underlying type approximation: `~int` matches `type MyInt int`
- [ ] `cmp.Compare`, `cmp.Or` (Go 1.21+)
- [ ] Limitations: no generic methods (only generic functions and types), no specialization, no operator methods

---

## Chapter 16 — [Reflection](go/Reflection.md)

- [ ] `reflect.TypeOf(x)` — returns `reflect.Type`
- [ ] `reflect.ValueOf(x)` — returns `reflect.Value`
- [ ] Kind vs Type — `Kind()` returns the underlying kind (e.g., `reflect.Struct`)
- [ ] Settability — `CanSet()`, must pass pointer: `reflect.ValueOf(&x).Elem()`
- [ ] Iterating struct fields: `NumField()`, `Field(i)`, `FieldByName("Name")`
- [ ] Reading struct tags: `field.Tag.Get("json")`
- [ ] Creating values: `reflect.New(t)`, `reflect.MakeSlice`, `reflect.MakeMap`
- [ ] Calling functions: `reflect.Value.Call([]reflect.Value{...})`
- [ ] Laws of reflection:
  1. Reflection goes from interface value to reflection object
  2. Reflection goes from reflection object to interface value
  3. To modify a reflection object, the value must be settable
- [ ] `reflect.DeepEqual` — recursive comparison (useful in tests)
- [ ] Performance implications — reflection is slow, avoid in hot paths
- [ ] Use cases — serialization, ORM, dependency injection, test assertions

---

## Chapter 17 — [Testing](go/Testing.md)

- [ ] `testing` package, `go test`, `go test ./...`
- [ ] Test functions: `func TestXxx(t *testing.T)`
- [ ] `t.Error`, `t.Errorf`, `t.Fatal`, `t.Fatalf`, `t.Log`, `t.Logf`
- [ ] Table-driven tests — slice of test cases with `t.Run`
- [ ] Sub-tests: `t.Run("name", func(t *testing.T) { })`
- [ ] `t.Parallel()` — run tests in parallel
- [ ] `t.Helper()` — mark function as test helper (cleaner stack traces)
- [ ] `t.Cleanup(func())` — register cleanup function
- [ ] `t.Skip()`, `t.Skipf()` — skip tests conditionally
- [ ] `t.TempDir()` — creates a temporary directory, cleaned up automatically
- [ ] Benchmarks: `func BenchmarkXxx(b *testing.B)` — loop `b.N` times
- [ ] Sub-benchmarks: `b.Run("name", func(b *testing.B) { })`
- [ ] `b.ResetTimer()`, `b.StopTimer()`, `b.StartTimer()`
- [ ] Test coverage: `go test -cover`, `go test -coverprofile=cover.out`
- [ ] Fuzzing (Go 1.18+): `func FuzzXxx(f *testing.F)` — `f.Add()`, `f.Fuzz()`
- [ ] `testdata` directory — test fixtures
- [ ] `_test.go` suffix — files only compiled during tests
- [ ] `httptest` package — `httptest.NewServer`, `httptest.NewRecorder`
- [ ] Common third-party: `testify` (assert, require, mock, suite)
- [ ] Example tests: `func ExampleXxx()` — verified by `go test`

---

## Chapter 18 — [Packages & Modules](go/PackagesAndModules.md)

- [ ] Modules: `go.mod` (module path, Go version, dependencies), `go.sum` (checksums)
- [ ] `go mod init`, `go mod tidy`, `go mod download`, `go mod vendor`, `go mod graph`
- [ ] Semantic versioning: `v1.2.3`, major version paths (`v2+`)
- [ ] `go get` — add/update dependencies
- [ ] Package naming conventions — short, lowercase, no underscores, singular
- [ ] Internal packages — `internal/` restricts import scope
- [ ] Exported (`UpperCase`) vs unexported (`lowerCase`) identifiers
- [ ] `go install` — build and install binary
- [ ] Module proxies: `GOPROXY`, `GONOSUMDB`, `GOPRIVATE`
- [ ] Workspaces (Go 1.18+): `go.work`, `go work init`, `go work use`
- [ ] Vendor directory: `go mod vendor`, `go build -mod=vendor`
- [ ] Build tags / build constraints: `//go:build linux && amd64`

---

## Chapter 19 — [I/O & File Handling](go/IOAndFileHandling.md)

- [ ] `io.Reader` interface: `Read(p []byte) (n int, err error)`
- [ ] `io.Writer` interface: `Write(p []byte) (n int, err error)`
- [ ] `io.Copy`, `io.CopyN`, `io.CopyBuffer`
- [ ] `io.TeeReader`, `io.MultiReader`, `io.MultiWriter`
- [ ] `io.ReadAll` (Go 1.16+), `io.NopCloser`
- [ ] `io.LimitReader` — limits bytes read from a reader
- [ ] `os` package:
  - `os.Open`, `os.Create`, `os.OpenFile`
  - `os.ReadFile`, `os.WriteFile` (Go 1.16+)
  - `os.Stat`, `os.IsNotExist`
  - `os.Getenv`, `os.Setenv`, `os.Args`
  - `os.Exit`
- [ ] `bufio` package: `bufio.Scanner`, `bufio.Reader`, `bufio.Writer`
- [ ] `io/fs` package (Go 1.16+): `fs.FS`, `fs.WalkDir`
- [ ] `embed` package (Go 1.16+): `//go:embed` directive — embed files in binary
- [ ] `os/exec` — running external commands: `exec.Command`, `cmd.Run`, `cmd.Output`

---

## Chapter 20 — [Encoding & Serialization](go/EncodingAndSerialization.md)

- [ ] `encoding/json`:
  - `json.Marshal` / `json.Unmarshal`
  - Struct tags: `json:"name,omitempty"`, `json:"-"`
  - Custom marshaling: `MarshalJSON()`, `UnmarshalJSON()`
  - Streaming: `json.NewEncoder`, `json.NewDecoder`
  - `json.RawMessage` — delay parsing
  - `json.Number` — preserve number precision
  - `map[string]interface{}` for dynamic JSON
- [ ] `encoding/xml`: `xml.Marshal`, `xml.Unmarshal`, struct tags
- [ ] `encoding/csv`: `csv.NewReader`, `csv.NewWriter`
- [ ] `encoding/gob`: Go-specific binary encoding (for Go-to-Go communication)
- [ ] `encoding/binary`: `binary.Read`, `binary.Write`, byte order (`BigEndian`, `LittleEndian`)
- [ ] Protocol Buffers & gRPC — overview (`.proto` files, code generation, service definitions)

---

## Chapter 21 — [Networking & HTTP](go/NetworkingAndHTTP.md)

- [ ] `net/http` — HTTP server:
  - `http.HandleFunc`, `http.Handle`
  - `http.ServeMux` (default mux), custom mux
  - Enhanced routing (Go 1.22+): method + path patterns (`GET /api/users/{id}`)
  - `http.ListenAndServe`
  - Middleware pattern (handler wrapping)
- [ ] `http.Handler` interface: `ServeHTTP(w http.ResponseWriter, r *http.Request)`
- [ ] `http.ResponseWriter`, `http.Request` — key fields and methods
- [ ] `http.Client` — custom clients, timeouts, `Transport`
- [ ] `http.Server` — graceful shutdown with `Shutdown(ctx)`
- [ ] `net` package — TCP/UDP:
  - `net.Listen`, `net.Dial`
  - `net.Conn` interface
- [ ] `net/url` — `url.Parse`, `url.Values`
- [ ] REST API patterns in Go
- [ ] Popular routers/frameworks: `chi`, `gorilla/mux`, `gin`, `echo` (overview)

---

## Chapter 22 — [Standard Library Highlights](go/StandardLibraryHighlights.md)

- [ ] `time` package:
  - `time.Now()`, `time.Since()`, `time.Until()`
  - `time.Duration`, `time.Sleep`
  - `time.Ticker`, `time.Timer`, `time.After`, `time.NewTicker`
  - Time formatting — reference time: `2006-01-02 15:04:05` (Mon Jan 2 15:04:05 MST 2006)
  - `time.Parse`, `time.Format`
- [ ] `sort` package:
  - `sort.Slice`, `sort.SliceStable`
  - `sort.Interface` (`Len`, `Less`, `Swap`)
  - `sort.Search` — binary search
  - `slices.Sort`, `slices.SortFunc` (Go 1.21+)
- [ ] `math` / `math/rand` / `crypto/rand`
- [ ] `regexp` package: `regexp.Compile`, `regexp.MustCompile`, `FindString`, `ReplaceAll`
- [ ] `log` package: `log.Println`, `log.Fatal`, `log.SetFlags`
- [ ] `log/slog` (Go 1.21+): structured logging — `slog.Info`, `slog.Error`, handlers (JSON, Text)
- [ ] `flag` package: `flag.String`, `flag.Int`, `flag.Parse`
- [ ] `path/filepath`: `Join`, `Ext`, `Base`, `Dir`, `Walk`, `Glob`
- [ ] `container/heap` — heap / priority queue interface
- [ ] `container/list` — doubly linked list
- [ ] `container/ring` — circular list

---

## Chapter 23 — [Memory Management & Internals](go/MemoryManagementAndInternals.md)

- [ ] Stack vs heap allocation — Go compiler decides (escape analysis)
- [ ] Escape analysis: `go build -gcflags="-m"` — see what escapes to heap
- [ ] Garbage collector:
  - Tri-color mark-and-sweep (concurrent, low-latency)
  - GC pauses and tuning with `GOGC` (default 100 = GC when heap doubles)
  - `GOMEMLIMIT` (Go 1.19+) — soft memory limit
  - `runtime.GC()` — force GC
  - `debug.SetGCPercent()`, `debug.SetMemoryLimit()`
- [ ] Memory alignment and struct padding — field ordering matters for size
- [ ] `unsafe` package:
  - `unsafe.Pointer` — bypass type safety
  - `unsafe.Sizeof`, `unsafe.Alignof`, `unsafe.Offsetof`
  - `unsafe.Slice`, `unsafe.String` (Go 1.17+)
- [ ] `cgo` basics — calling C from Go (`import "C"`, `// #include`, overhead)
- [ ] Go runtime — goroutine stack growth (segmented → contiguous, starts ~2-8KB)
- [ ] Memory profiling: `runtime.MemStats`, `runtime.ReadMemStats()`

---

## Chapter 24 — [Concurrency Patterns](go/ConcurrencyPatterns.md)

- [ ] **Worker Pool** — fixed goroutines reading from shared job channel
- [ ] **Fan-In** — merge results from multiple goroutines into one channel
- [ ] **Fan-Out** — distribute tasks to multiple goroutines
- [ ] **Pipeline** — chain of stages: each stage is a goroutine reading from input channel, writing to output channel
- [ ] **Rate Limiting** — `time.Ticker` for steady rate, buffered channel for burst
- [ ] **Semaphore** — buffered channel limits concurrent access
- [ ] **errgroup** — `golang.org/x/sync/errgroup` — run goroutines, collect first error, cancel on failure
- [ ] **singleflight** — `golang.org/x/sync/singleflight` — deduplicate concurrent calls
- [ ] **Pub/Sub** pattern
- [ ] **Circuit Breaker** pattern (concept for resilience)
- [ ] **Graceful Shutdown** — `os.Signal`, `signal.Notify`, context cancellation, `http.Server.Shutdown`
- [ ] **Producer-Consumer** — goroutine with channel as buffer
- [ ] **Timeout & Cancellation** — `select` with `context.Done()` or `time.After`

---

## Chapter 25 — [Design Patterns in Go](go/DesignPatternsInGo.md)

- [ ] **Singleton** — `sync.Once` + package-level variable
- [ ] **Factory** — constructor functions returning interface
- [ ] **Builder** — method chaining or functional options
- [ ] **Strategy** — swap behavior via interface implementations
- [ ] **Observer** — event listeners / callbacks
- [ ] **Decorator** — HTTP middleware, wrapping functions
- [ ] **Adapter** — convert one interface to another
- [ ] **Dependency Injection** — pass dependencies via constructor, not globals
- [ ] **Functional Options** — `func WithTimeout(d time.Duration) Option` pattern
- [ ] **Repository** — abstract data store behind interface
- [ ] **Table-driven logic** — use maps/slices for dispatch instead of long if/switch

---

## Chapter 26 — [Performance & Profiling](go/PerformanceAndProfiling.md)

- [ ] `pprof` — CPU, memory, goroutine, block, mutex profiles
  - `import _ "net/http/pprof"` for HTTP server profiling
  - `runtime/pprof` for non-server programs
  - `go tool pprof` — interactive analysis
- [ ] `go tool trace` — execution tracer, visualize goroutine scheduling
- [ ] Benchmarking:
  - `testing.B`, `b.N`, `b.ReportAllocs()`
  - `benchstat` — compare benchmark results
  - `go test -bench=. -benchmem`
- [ ] Common optimizations:
  - Pre-allocate slices with `make([]T, 0, cap)`
  - Use `strings.Builder` for string concatenation
  - `sync.Pool` for reusing objects
  - Avoid unnecessary allocations, reduce GC pressure
  - Struct field ordering for alignment
  - Use `[]byte` instead of `string` for mutable text
- [ ] `GODEBUG` environment variable — runtime debugging flags
- [ ] Inlining — `go build -gcflags="-m"` shows inlining decisions
- [ ] Bounds check elimination (BCE)

---

## Chapter 27 — [Common Interview Gotchas & Tricky Questions](go/InterviewGotchas.md)

- [ ] **Slice append & capacity** — `append` may return a new underlying array; modifying a sub-slice can affect the original
- [ ] **Map iteration order** — randomized by design, never rely on it
- [ ] **Goroutine loop variable capture** — pre-Go 1.22: `for _, v := range list { go func() { use(v) }() }` captures same `v`; fixed in Go 1.22
- [ ] **Defer with closures** — deferred closure sees variable's final value; deferred method call evaluates args at defer site
- [ ] **Nil interface vs nil pointer in interface** — `var err error = (*MyError)(nil)` → `err != nil` is `true`!
- [ ] **String immutability** — strings are immutable; `s[0] = 'a'` won't compile; use `[]byte` for mutation
- [ ] **String concatenation in loops** — `+=` is O(n²); use `strings.Builder`
- [ ] **Channel deadlocks** — sending/receiving on unbuffered channel with no partner goroutine
- [ ] **Value vs pointer receiver method sets** — value type cannot call pointer-receiver methods via interface
- [ ] **`init()` execution order** — runs in dependency order, multiple `init` per package
- [ ] **Named return + defer** — deferred function can modify named return values
- [ ] **Map value not addressable** — `m[key].field = val` won't work; use pointer values or reassign
- [ ] **`range` copies values** — `for _, v := range slice` gives a copy of each element
- [ ] **`select` randomness** — when multiple cases ready, Go picks one randomly
- [ ] **Struct comparison** — structs with non-comparable fields (slice, map, func) cannot use `==`
- [ ] **Zero-size struct `struct{}`** — uses no memory; idiomatic for sets (`map[K]struct{}`) and signal channels (`chan struct{}`)

---

## Chapter 28 — [Go Toolchain & Build](go/GoToolchainAndBuild.md)

- [ ] `go build`, `go run`, `go install`
- [ ] `go fmt` / `gofmt` / `goimports` — formatting
- [ ] `go vet` — static analysis
- [ ] `go generate` — code generation (`//go:generate` directive)
- [ ] `go doc` / `godoc` — documentation
- [ ] `go list` — list packages
- [ ] `go env` — environment variables
- [ ] `golangci-lint` — linter aggregator (common third-party)
- [ ] Cross-compilation: `GOOS=linux GOARCH=amd64 go build`
- [ ] Build constraints / tags: `//go:build` (Go 1.17+)
- [ ] `ldflags`: `-ldflags "-X main.version=1.0.0"` — inject values at build time
- [ ] Link-time optimizations, static binary

---

## Chapter 29 — [Quick Reference: Keywords & Built-in Functions](go/KeywordsAndBuiltins.md)

### 25 Go Keywords

| Keyword | Category |
|---------|----------|
| `break`, `continue`, `goto`, `return`, `fallthrough` | Flow control |
| `if`, `else`, `switch`, `case`, `default` | Conditionals |
| `for`, `range` | Loops |
| `func`, `return` | Functions |
| `var`, `const`, `type` | Declarations |
| `package`, `import` | Organization |
| `struct`, `interface` | Types |
| `map` | Data structure |
| `chan`, `go`, `select` | Concurrency |
| `defer` | Deferred execution |

### Built-in Functions

| Function | Purpose |
|----------|---------|
| `make()` | Create slice, map, channel |
| `new()` | Allocate zeroed memory, return pointer |
| `len()` | Length of string, slice, map, channel, array |
| `cap()` | Capacity of slice, channel, array |
| `append()` | Append elements to slice |
| `copy()` | Copy slice elements |
| `delete()` | Delete map entry |
| `close()` | Close channel |
| `panic()` | Trigger runtime panic |
| `recover()` | Catch panic (in deferred function) |
| `print()`, `println()` | Basic output (use `fmt` instead) |
| `complex()`, `real()`, `imag()` | Complex number operations |
| `clear()` (Go 1.21+) | Clear map or zero-out slice |
| `min()`, `max()` (Go 1.21+) | Return min/max of arguments |

### Predeclared Types

| Category | Types |
|----------|-------|
| Boolean | `bool` |
| Integer | `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr` |
| Float | `float32`, `float64` |
| Complex | `complex64`, `complex128` |
| String | `string` |
| Aliases | `byte` (`uint8`), `rune` (`int32`), `any` (`interface{}`), `comparable` |
| Error | `error` |

---

## Chapter 30 — [Garbage Collector (GC)](go/GarbageCollector.md)

- [ ] GC overview — concurrent, tri-color mark-and-sweep, non-generational, non-compacting
- [ ] Tri-color algorithm — white (unvisited), grey (discovered), black (scanned)
- [ ] Tri-color invariant — black must never point to white; enforced by write barrier
- [ ] GC phases:
  - Mark Setup (STW — enable write barrier, identify roots)
  - Concurrent Marking (runs alongside application, ~25% CPU)
  - Mark Termination (STW — finish marking, disable write barrier)
  - Concurrent Sweeping (reclaim unreachable memory)
- [ ] Write barrier — intercepts pointer writes during concurrent marking
- [ ] `GOGC` — GC trigger percentage (default 100 = GC when heap doubles)
- [ ] `GOMEMLIMIT` (Go 1.19+) — soft memory limit, `GOGC=off` + `GOMEMLIMIT` pattern
- [ ] GC pacing — how the GC decides when to start, mark assist for fast allocators
- [ ] `runtime.GC()` — force GC (rarely needed)
- [ ] `runtime.ReadMemStats()` — heap, GC, and allocation metrics
- [ ] `GODEBUG=gctrace=1` — GC trace logging, reading trace output
- [ ] Escape analysis — stack vs heap, `go build -gcflags="-m"`, common escape causes
- [ ] `runtime.SetFinalizer` — caveats, prefer explicit `Close()` methods
- [ ] GC tuning strategies:
  - Reduce allocations (sync.Pool, pre-allocate, strings.Builder)
  - Tune GOGC / GOMEMLIMIT
  - Ballast technique (pre-GOMEMLIMIT)
  - Off-heap / arena (experimental)
- [ ] GC-friendly code patterns — value receivers, `[]T` vs `[]*T`, struct of arrays
- [ ] Monitoring GC in production — expvar, Prometheus, alert thresholds

---

## Chapter 31 — [Go Scheduler Deep Dive (GMP Model)](go/GoSchedulerDeepDive.md)

- [ ] GMP model overview — Goroutines (G), OS Threads (M), Processors (P)
- [ ] G entity — goroutine state machine (Idle → Runnable → Running → Waiting → Dead)
- [ ] M entity — OS thread, bound to one P at a time, `GOMAXPROCS` controls P count
- [ ] P entity — logical processor, owns local run queue (LRQ), max 256 entries
- [ ] Scheduling cycle — `findRunnable` priority order (local → global → netpoll → steal)
- [ ] Work stealing — idle P steals half of another P's LRQ
- [ ] Preemption — cooperative (Go <1.14) vs asynchronous signal-based (Go 1.14+)
- [ ] Sysmon thread — preempts long-running goroutines, retakes Ps from syscalls
- [ ] Syscall handoff — blocking syscall detaches P, netpoller for non-blocking I/O
- [ ] `GODEBUG=schedtrace` — runtime scheduler tracing, reading trace output
- [ ] `runtime.LockOSThread` — pin goroutine to OS thread (CGo, GUI, TLS)

---

## Chapter 32 — [Iterators & Range-Over-Func (Go 1.23+)](go/Iterators.md)

- [ ] Iterator convention — `iter.Seq[V]` and `iter.Seq2[K, V]` function signatures
- [ ] Push-based iterators — yield callback pattern, range-over-func syntax
- [ ] Standard combinators — Filter, Map, Take as composable iterator functions
- [ ] Chaining pipelines — composing iterators without intermediate allocations
- [ ] Two-value iterators (`Seq2`) — key-value pairs, index-element patterns
- [ ] Standard library support — `slices.All`, `slices.Values`, `maps.Keys`, `maps.Values`
- [ ] Custom data structure iterators — BST in-order traversal, tree iterators
- [ ] Pull-based iterators — `iter.Pull`, `iter.Pull2`, manual advancement
- [ ] Merged/sorted iterators — combining multiple sorted sequences

---

## Chapter 33 — [Iota & Enum Patterns](go/IotaAndEnumPatterns.md)

- [ ] `iota` basics — auto-incrementing constant generator, resets per `const` block
- [ ] Iota expressions — bit shifts, arithmetic, skip-with-blank-identifier
- [ ] Bitmask / bitflag enums — permission flags with `1 << iota`, bitwise operations
- [ ] Type-safe enums — custom types prevent mixing enum kinds
- [ ] `go:generate` + `stringer` — auto-generate `String()` method for enum types
- [ ] String-based enums — `const` with explicit string values
- [ ] Sentinel / unknown value — placing unknown/invalid at iota 0
- [ ] Multiple constants per line — multi-dimensional iota expressions
- [ ] JSON marshal/unmarshal for enums — custom `MarshalJSON` / `UnmarshalJSON`

---

## Chapter 34 — [Database Access (database/sql)](go/DatabaseAccess.md)

- [ ] `database/sql` architecture — driver-agnostic interface, driver registration
- [ ] `sql.DB` — connection pool (not a connection), `sql.Open`, `db.Ping`
- [ ] Query methods — `QueryRow` (single row), `Query` (multi row), `Exec` (no rows)
- [ ] Parameterized queries — `$1/$2` (Postgres), `?` (MySQL), SQL injection prevention
- [ ] `*sql.Rows` — iterate with `Next()`, `Scan()`, always `defer rows.Close()`
- [ ] `sql.ErrNoRows` — sentinel error for empty `QueryRow` results
- [ ] Prepared statements — `db.Prepare`, parse-once-execute-many pattern
- [ ] Transactions — `BeginTx`, `Commit`, `Rollback`, `defer tx.Rollback()` pattern
- [ ] Isolation levels — `sql.LevelSerializable`, `sql.LevelRepeatableRead`, etc.
- [ ] NULL handling — `sql.NullString`, `sql.NullInt64`, pointer types, `sql.Null[T]`
- [ ] Connection pool tuning — `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`
- [ ] `db.Stats()` — pool health monitoring metrics
- [ ] Repository pattern — wrapping `sql.DB` for clean data access layers
- [ ] Testing — `go-sqlmock`, interface-based mocking
- [ ] Third-party libraries — `sqlx`, `pgx`, GORM, `sqlc`, migration tools

---

## Chapter 35 — [Race Conditions & Data Races](go/RaceConditions.md)

- [ ] Data race vs race condition — unsynchronized access vs timing-dependent logic
- [ ] Go race detector — `go run -race`, ThreadSanitizer, runtime detection only
- [ ] Classic race: shared counter — `counter++` is not atomic, lost updates
- [ ] Classic race: concurrent map access — `fatal error: concurrent map writes`
- [ ] Classic race: loop variable capture (pre Go 1.22) — closure captures reference
- [ ] Classic race: check-then-act (TOCTOU) — gap between check and action
- [ ] Classic race: slice append — concurrent append corrupts slice header
- [ ] `sync.Mutex` vs `sync.RWMutex` — exclusive vs reader-writer locks
- [ ] `sync/atomic` — lock-free operations, `atomic.Int64`, `CompareAndSwap`
- [ ] `sync.Once` — exactly-once initialization pattern
- [ ] Goroutine leaks — unbuffered channel blocking, prevention strategies
- [ ] Channel ownership — producer creates/closes, consumer reads
- [ ] Race-free design — confinement, immutability, channels over shared memory

---

## Quick Reference: Go Versions & Key Features

| Version | Key Features |
|---------|-------------|
| Go 1.11 | Modules (experimental) |
| Go 1.13 | Error wrapping (`%w`, `errors.Is`, `errors.As`) |
| Go 1.14 | Testing cleanup (`t.Cleanup`) |
| Go 1.16 | `embed` package, `io/fs`, `os.ReadFile` / `os.WriteFile` |
| Go 1.17 | Build constraints `//go:build`, `unsafe.Slice` |
| Go 1.18 | **Generics**, Fuzzing, Workspaces |
| Go 1.19 | `GOMEMLIMIT`, atomic types |
| Go 1.20 | `errors.Join`, `context.Cause` |
| Go 1.21 | `log/slog`, `slices`/`maps`/`cmp` packages, `min`/`max` builtins |
| Go 1.22 | Range over integers, loop variable fix, enhanced `ServeMux` routing |
| Go 1.23 | Iterator functions (`iter` package), `range over func` |

---

*Last updated: March 9, 2026*

---

## Study Checklist

- [ ] Chapter 1: [Go Basics & Setup](go/GoBasicsAndSetup.md)
- [ ] Chapter 2: [Data Types & Variables](go/DataTypesAndVariables.md)
- [ ] Chapter 3: [Control Flow](go/ControlFlow.md)
- [ ] Chapter 4: [Functions](go/Functions.md)
- [ ] Chapter 5: [Composite Types](go/CompositeTypes.md)
- [ ] Chapter 6: [Strings & Unicode](go/StringsAndUnicode.md)
- [ ] Chapter 7: [Pointers](go/Pointers.md)
- [ ] Chapter 8: [Structs & Methods](go/StructsAndMethods.md)
- [ ] Chapter 9: [Interfaces](go/Interfaces.md)
- [ ] Chapter 10: [Error Handling](go/ErrorHandling.md)
- [ ] Chapter 11: [Goroutines & Concurrency](go/GoroutinesAndConcurrency.md)
- [ ] Chapter 12: [Channels](go/Channels.md)
- [ ] Chapter 13: [Synchronization Primitives](go/SynchronizationPrimitives.md)
- [ ] Chapter 14: [Context Package](go/ContextPackage.md)
- [ ] Chapter 15: [Generics](go/Generics.md)
- [ ] Chapter 16: [Reflection](go/Reflection.md)
- [ ] Chapter 17: [Testing](go/Testing.md)
- [ ] Chapter 18: [Packages & Modules](go/PackagesAndModules.md)
- [ ] Chapter 19: [I/O & File Handling](go/IOAndFileHandling.md)
- [ ] Chapter 20: [Encoding & Serialization](go/EncodingAndSerialization.md)
- [ ] Chapter 21: [Networking & HTTP](go/NetworkingAndHTTP.md)
- [ ] Chapter 22: [Standard Library Highlights](go/StandardLibraryHighlights.md)
- [ ] Chapter 23: [Memory Management & Internals](go/MemoryManagementAndInternals.md)
- [ ] Chapter 24: [Concurrency Patterns](go/ConcurrencyPatterns.md)
- [ ] Chapter 25: [Design Patterns in Go](go/DesignPatternsInGo.md)
- [ ] Chapter 26: [Performance & Profiling](go/PerformanceAndProfiling.md)
- [ ] Chapter 27: [Common Interview Gotchas](go/InterviewGotchas.md)
- [ ] Chapter 28: [Go Toolchain & Build](go/GoToolchainAndBuild.md)
- [ ] Chapter 29: [Keywords & Built-in Functions](go/KeywordsAndBuiltins.md)
- [ ] Chapter 30: [Garbage Collector (GC)](go/GarbageCollector.md)
- [ ] Chapter 31: [Go Scheduler Deep Dive (GMP Model)](go/GoSchedulerDeepDive.md)
- [ ] Chapter 32: [Iterators & Range-Over-Func (Go 1.23+)](go/Iterators.md)
- [ ] Chapter 33: [Iota & Enum Patterns](go/IotaAndEnumPatterns.md)
- [ ] Chapter 34: [Database Access (database/sql)](go/DatabaseAccess.md)
- [ ] Chapter 35: [Race Conditions & Data Races](go/RaceConditions.md)
