# Chapter 25 — Design Patterns in Go

Go's design philosophy rejects complex OOP hierarchies in favor of composition, interfaces, and simple functions. These patterns show idiomatic Go approaches to classic software design problems.

```
┌──────────────────────────────────────────────────────────┐
│            Design Patterns — Go Idioms                  │
│                                                          │
│  Creational:                                             │
│  ├── Singleton   → sync.Once + package-level var        │
│  ├── Factory     → NewXxx() constructor functions       │
│  ├── Builder     → method chaining with pointer receiver│
│  └── Functional  → Options pattern (variadic funcs)     │
│       Options                                            │
│                                                          │
│  Behavioral:                                             │
│  ├── Strategy    → interface + swappable implementations│
│  ├── Observer    → callbacks / channels for events      │
│  └── Middleware  → func(Handler) Handler chaining       │
│                                                          │
│  Structural:                                             │
│  ├── Decorator   → wrapping types (io.Reader chains)    │
│  ├── Repository  → interface for data access layer      │
│  └── Circuit     → state machine                        │
│       Breaker     (closed → open → half-open)           │
│                                                          │
│  Go-specific idioms:                                    │
│  • Accept interfaces, return structs                    │
│  • Small interfaces (1-2 methods)                       │
│  • Composition over inheritance (embedding)             │
│  • Functional options for configurable constructors     │
└──────────────────────────────────────────────────────────┘
```

## Singleton — sync.Once

**Tutorial: Thread-Safe Singleton with sync.Once**

The singleton pattern ensures only one instance of a type exists throughout the program's lifetime. Go's `sync.Once` guarantees the initialization function executes exactly once, even when called from multiple goroutines. The `Do` method is thread-safe and blocks concurrent callers until initialization completes. Note that all subsequent calls to `GetDB()` return the same pointer — verified by pointer equality.

```
┌────────────────────────────────────────────────────┐
│  GetDB()  call 1          call 2         call 3   │
│     │                      │              │       │
│     ▼                      ▼              ▼       │
│  dbOnce.Do(func() {       (skipped)      (skipped)│
│    dbInstance = &Database{}                        │
│  })                                                │
│     │                      │              │       │
│     ▼                      ▼              ▼       │
│  return dbInstance ────► same pointer ──► same ptr │
│                                                    │
│  db1 == db2 == db3 ──► true (one instance)         │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

type Database struct {
    ConnectionString string
}

var (
    dbInstance *Database
    dbOnce     sync.Once
)

func GetDB() *Database {
    dbOnce.Do(func() {
        fmt.Println("Creating database instance (only once)")
        dbInstance = &Database{ConnectionString: "postgres://localhost:5432/mydb"}
    })
    return dbInstance
}

func main() {
    // All calls return the SAME instance
    db1 := GetDB() // prints "Creating database instance (only once)"
    db2 := GetDB() // no print — already created
    db3 := GetDB()

    fmt.Println(db1 == db2, db2 == db3) // true true
}
```

---

## Factory — Constructor Functions Returning Interface

**Tutorial: Decoupling Object Creation with Factory Functions**

The factory pattern uses a function to create objects without exposing the concrete type to the caller. Here, `NewNotifier` returns the `Notifier` interface, letting the caller work polymorphically with `EmailNotifier` or `SMSNotifier` without knowing the concrete type. This follows Go's idiom of "accept interfaces, return structs" — the factory is the one place where concrete types are chosen based on runtime input.

```
┌────────────────────────────────────────────────────┐
│  NewNotifier(type, target) ──► Notifier interface  │
│                                                    │
│  "email" ──► &EmailNotifier{Address: target}        │
│                    │                                │
│                    └─► Notify(msg): "Email to ..."  │
│                                                    │
│  "sms"   ──► &SMSNotifier{Phone: target}           │
│                    │                                │
│                    └─► Notify(msg): "SMS to ..."    │
│                                                    │
│  Caller only knows Notifier interface:             │
│  n1.Notify("Hello!")  ← polymorphic dispatch       │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Interface
type Notifier interface {
    Notify(message string) error
}

// Concrete implementations
type EmailNotifier struct{ Address string }
type SMSNotifier struct{ Phone string }

func (e *EmailNotifier) Notify(msg string) error {
    fmt.Printf("Email to %s: %s\n", e.Address, msg)
    return nil
}

func (s *SMSNotifier) Notify(msg string) error {
    fmt.Printf("SMS to %s: %s\n", s.Phone, msg)
    return nil
}

// Factory function
func NewNotifier(notifType, target string) Notifier {
    switch notifType {
    case "email":
        return &EmailNotifier{Address: target}
    case "sms":
        return &SMSNotifier{Phone: target}
    default:
        return nil
    }
}

func main() {
    n1 := NewNotifier("email", "alice@test.com")
    n2 := NewNotifier("sms", "+1234567890")

    n1.Notify("Hello via email!")
    n2.Notify("Hello via SMS!")
}
```

---

## Builder — Method Chaining

**Tutorial: Constructing Complex Objects Step-by-Step**

The builder pattern constructs a complex object through a sequence of method calls, each returning the builder itself for chaining. This avoids constructors with many parameters and makes the code self-documenting. Each `With*` method sets one field and returns `*ServerBuilder`, enabling fluent syntax like `NewServerBuilder(...).WithTLS(true).WithMaxConns(100).Build()`. The final `Build()` returns the completed struct value.

```
┌────────────────────────────────────────────────────┐
│  NewServerBuilder("localhost", 8080)                │
│       │  returns *ServerBuilder                    │
│       ▼                                            │
│  .WithTLS(true)       ─► sets TLS=true, returns b │
│       ▼                                            │
│  .WithMaxConns(100)   ─► sets MaxConns=100         │
│       ▼                                            │
│  .WithTimeout(30)     ─► sets Timeout=30           │
│       ▼                                            │
│  .Build()             ─► returns Server{...}       │
│                                                    │
│  Result: Server{                                   │
│    Host:"localhost", Port:8080,                    │
│    TLS:true, MaxConns:100, Timeout:30              │
│  }                                                 │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Server struct {
    Host     string
    Port     int
    TLS      bool
    MaxConns int
    Timeout  int // seconds
}

type ServerBuilder struct {
    server Server
}

func NewServerBuilder(host string, port int) *ServerBuilder {
    return &ServerBuilder{
        server: Server{Host: host, Port: port},
    }
}

func (b *ServerBuilder) WithTLS(enabled bool) *ServerBuilder {
    b.server.TLS = enabled
    return b
}

func (b *ServerBuilder) WithMaxConns(n int) *ServerBuilder {
    b.server.MaxConns = n
    return b
}

func (b *ServerBuilder) WithTimeout(seconds int) *ServerBuilder {
    b.server.Timeout = seconds
    return b
}

func (b *ServerBuilder) Build() Server {
    return b.server
}

func main() {
    srv := NewServerBuilder("localhost", 8080).
        WithTLS(true).
        WithMaxConns(100).
        WithTimeout(30).
        Build()

    fmt.Printf("%+v\n", srv)
    // {Host:localhost Port:8080 TLS:true MaxConns:100 Timeout:30}
}
```

---

## Strategy — Swap Behavior via Interface

**Tutorial: Runtime Algorithm Swapping with Strategy Interfaces**

The strategy pattern defines a family of algorithms as an interface, letting you swap implementations at runtime. Here, `CompressionStrategy` declares a `Compress` method, and three concrete strategies implement it differently. The `FileProcessor` context holds a strategy reference and delegates compression to whatever implementation is currently set. Call `SetStrategy` to change behavior dynamically without modifying the processor.

```
┌────────────────────────────────────────────────────┐
│  CompressionStrategy interface                     │
│  ┌──────────────────────────────────────────┐    │
│  │  Compress(data []byte) []byte                │    │
│  └──────────────────────────────────────────┘    │
│       ▲          ▲          ▲                      │
│  ┌────┼───┐ ┌────┼───┐ ┌────┼─────┐              │
│  │  Gzip   │ │  Zip    │ │  NoneComp │              │
│  └────────┘ └────────┘ └──────────┘              │
│                                                    │
│  FileProcessor.SetStrategy(&GzipCompression{})     │
│  FileProcessor.Process(data)                       │
│       │                                            │
│       └─► strategy.Compress(data)  ← delegated     │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Strategy interface
type CompressionStrategy interface {
    Compress(data []byte) []byte
}

// Concrete strategies
type GzipCompression struct{}
type ZipCompression struct{}
type NoCompression struct{}

func (g *GzipCompression) Compress(data []byte) []byte {
    fmt.Println("Compressing with Gzip")
    return data // simplified
}

func (z *ZipCompression) Compress(data []byte) []byte {
    fmt.Println("Compressing with Zip")
    return data
}

func (n *NoCompression) Compress(data []byte) []byte {
    fmt.Println("No compression")
    return data
}

// Context that uses a strategy
type FileProcessor struct {
    strategy CompressionStrategy
}

func (fp *FileProcessor) SetStrategy(s CompressionStrategy) {
    fp.strategy = s
}

func (fp *FileProcessor) Process(data []byte) []byte {
    return fp.strategy.Compress(data)
}

func main() {
    processor := &FileProcessor{}

    processor.SetStrategy(&GzipCompression{})
    processor.Process([]byte("data"))

    processor.SetStrategy(&ZipCompression{})
    processor.Process([]byte("data"))

    processor.SetStrategy(&NoCompression{})
    processor.Process([]byte("data"))
}
```

---

## Observer — Event Listeners / Callbacks

**Tutorial: Decoupled Event Handling with Callbacks**

The observer pattern lets objects subscribe to events without the emitter knowing about them. This implementation uses a map of event types to handler function slices. `On` registers a callback, and `Emit` invokes all callbacks for that event type. This keeps the emitter decoupled from the observers — it doesn't import or know about email senders, loggers, or cleanup handlers. Multiple observers can respond to the same event independently.

```
┌────────────────────────────────────────────────────┐
│  EventEmitter                                      │
│  handlers["user.created"] = [handler1, handler2]   │
│  handlers["user.deleted"] = [handler3]             │
│                                                    │
│  Emit("user.created", "alice")                     │
│       │                                            │
│       ├─► handler1: "Send welcome email to alice"  │
│       └─► handler2: "Log user creation: alice"     │
│                                                    │
│  Emit("user.deleted", "bob")                       │
│       │                                            │
│       └─► handler3: "Cleanup data for: bob"       │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Event struct {
    Type string
    Data string
}

type EventHandler func(Event)

type EventEmitter struct {
    handlers map[string][]EventHandler
}

func NewEventEmitter() *EventEmitter {
    return &EventEmitter{handlers: make(map[string][]EventHandler)}
}

func (e *EventEmitter) On(eventType string, handler EventHandler) {
    e.handlers[eventType] = append(e.handlers[eventType], handler)
}

func (e *EventEmitter) Emit(event Event) {
    for _, handler := range e.handlers[event.Type] {
        handler(event)
    }
}

func main() {
    emitter := NewEventEmitter()

    // Register observers
    emitter.On("user.created", func(e Event) {
        fmt.Println("Send welcome email to:", e.Data)
    })

    emitter.On("user.created", func(e Event) {
        fmt.Println("Log user creation:", e.Data)
    })

    emitter.On("user.deleted", func(e Event) {
        fmt.Println("Cleanup data for:", e.Data)
    })

    // Emit events
    emitter.Emit(Event{Type: "user.created", Data: "alice@test.com"})
    emitter.Emit(Event{Type: "user.deleted", Data: "bob@test.com"})
}
```

---

## Decorator — HTTP Middleware

**Tutorial: Wrapping HTTP Handlers with Middleware Functions**

HTTP middleware in Go follows the decorator pattern: each middleware is a function that takes an `http.Handler` and returns a new `http.Handler` that wraps the original with additional behavior. The logging middleware times the request, and the auth middleware checks for an Authorization header. Decorators are chained by nesting: `authMiddleware(loggingMiddleware(handler))` creates a pipeline where auth runs first, then logging, then the actual handler.

```
┌────────────────────────────────────────────────────┐
│  Request Flow Through Middleware Chain              │
│                                                    │
│  HTTP Request                                      │
│       │                                            │
│       ▼                                            │
│  ┌──────────────────┐                              │
│  │ authMiddleware  │  Check Authorization header    │
│  │ (outer wrapper) │  ✗ 401 if missing               │
│  └────────┬─────────┘                              │
│           ▼                                        │
│  ┌──────────────────┐                              │
│  │loggingMiddleware│  Record start time              │
│  │ (inner wrapper) │  Log method + path + duration   │
│  └────────┬─────────┘                              │
│           ▼                                        │
│  ┌──────────────────┐                              │
│  │  helloHandler   │  "Hello, World!"               │
│  └──────────────────┘                              │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// Decorator: logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Decorator: auth middleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World!")
}

func main() {
    // Chain decorators: auth → logging → handler
    handler := authMiddleware(loggingMiddleware(http.HandlerFunc(helloHandler)))

    http.Handle("/hello", handler)
    fmt.Println("Server on :8080")
    // http.ListenAndServe(":8080", nil)
}
```

---

## Adapter — Convert One Interface to Another

**Tutorial: Bridging Incompatible APIs with Adapter Structs**

The adapter pattern wraps an existing type with a different interface to make it compatible with code that expects a specific API. Here, `LegacyLogger` has `WriteLog(level, msg)` but our code expects the `Logger` interface with `Log(message)`. The `LegacyLoggerAdapter` embeds the legacy logger and translates `Log()` calls into `WriteLog()` calls. This lets you integrate third-party or legacy code without modifying it.

```
┌────────────────────────────────────────────────────┐
│  Our Code expects:        Legacy code has:         │
│  ┌──────────────┐         ┌────────────────┐       │
│  │ Logger       │         │ LegacyLogger   │       │
│  │ Log(msg)     │         │ WriteLog(l,m)  │       │
│  └──────────────┘         └────────────────┘       │
│       ▲                         ▲                  │
│       │  implements            │  wraps              │
│  ┌────┼────────────────────┼─────────────┐       │
│  │ LegacyLoggerAdapter                      │       │
│  │ Log(msg) ─► legacy.WriteLog(level, msg)  │       │
│  └─────────────────────────────────────────┘       │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Target interface our code expects
type Logger interface {
    Log(message string)
}

// Existing third-party "legacy" logger with different API
type LegacyLogger struct{}

func (l *LegacyLogger) WriteLog(level string, msg string) {
    fmt.Printf("[%s] %s\n", level, msg)
}

// Adapter — makes LegacyLogger satisfy our Logger interface
type LegacyLoggerAdapter struct {
    legacy *LegacyLogger
    level  string
}

func (a *LegacyLoggerAdapter) Log(message string) {
    a.legacy.WriteLog(a.level, message) // adapt the call
}

func processWithLogger(l Logger) {
    l.Log("Processing started")
    l.Log("Processing finished")
}

func main() {
    legacy := &LegacyLogger{}
    adapter := &LegacyLoggerAdapter{legacy: legacy, level: "INFO"}

    processWithLogger(adapter) // works via adapter!
}
```

---

## Dependency Injection

**Tutorial: Injecting Dependencies Through Constructor Functions**

Dependency injection (DI) in Go is done by passing interface values through constructor functions rather than hardcoding concrete types. `UserService` depends on the `UserStore` interface, so in production you inject `PostgresUserStore` and in tests you inject `MockUserStore`. This makes code testable without mocking frameworks — any type implementing the interface can be injected. The `NewUserService(store)` constructor is the injection point.

```
┌────────────────────────────────────────────────────┐
│  UserStore interface                               │
│  ┌──────────────────────────┐                        │
│  │ GetUser(id int) (string, error) │                │
│  └──────────────────────────┘                        │
│       ▲                  ▲                          │
│  ┌────┼────────┐   ┌────┼────────┐                  │
│  │ PostgresStore│   │ MockStore   │                  │
│  └─────────────┘   └────────────┘                  │
│  (production)         (testing)                     │
│       │                  │                          │
│       ▼                  ▼                          │
│  NewUserService(store)  ← inject at construction    │
│       │                                            │
│  UserService.GetUserName(id)                       │
│       └─► store.GetUser(id)  ← delegates            │
└────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Interface for storage
type UserStore interface {
    GetUser(id int) (string, error)
}

// Concrete implementation
type PostgresUserStore struct{}

func (p *PostgresUserStore) GetUser(id int) (string, error) {
    return fmt.Sprintf("User-%d-from-postgres", id), nil
}

// Mock implementation for testing
type MockUserStore struct {
    Users map[int]string
}

func (m *MockUserStore) GetUser(id int) (string, error) {
    if name, ok := m.Users[id]; ok {
        return name, nil
    }
    return "", fmt.Errorf("user %d not found", id)
}

// Service depends on interface, not concrete implementation
type UserService struct {
    store UserStore // injected dependency
}

func NewUserService(store UserStore) *UserService {
    return &UserService{store: store}
}

func (s *UserService) GetUserName(id int) (string, error) {
    return s.store.GetUser(id)
}

func main() {
    // Production — inject real store
    realStore := &PostgresUserStore{}
    svc := NewUserService(realStore)
    name, _ := svc.GetUserName(1)
    fmt.Println(name) // User-1-from-postgres

    // Test — inject mock store
    mockStore := &MockUserStore{
        Users: map[int]string{1: "Alice", 2: "Bob"},
    }
    testSvc := NewUserService(mockStore)
    name, _ = testSvc.GetUserName(2)
    fmt.Println(name) // Bob
}
```

---

## Functional Options Pattern

**Tutorial: Extensible Constructors with Variadic Option Functions**

The functional options pattern provides clean, extensible constructors without breaking changes. Each option is a closure (`func(*ClientConfig)`) that modifies one field. The constructor `NewClient(opts ...Option)` applies defaults first, then applies each option. This pattern avoids long parameter lists, makes defaults explicit, and allows adding new options without changing the function signature. It's the idiomatic Go alternative to the builder pattern.

```
┌────────────────────────────────────────────────────┐
│  NewClient(opts ...Option)                         │
│       │                                            │
│       ▼  defaults:                                  │
│  cfg = { Timeout:30s, MaxRetries:3,                │
│          BaseURL:"api.example.com", Debug:false }  │
│       │                                            │
│       ▼  apply each option:                         │
│  WithTimeout(10s)    ─► cfg.Timeout = 10s          │
│  WithBaseURL("...")  ─► cfg.BaseURL = "..."        │
│  WithDebug(true)     ─► cfg.Debug = true           │
│       │                                            │
│       ▼  return cfg                                 │
│                                                    │
│  Adding new option: just add WithNewField()         │
│  No signature change needed!                       │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

type ClientConfig struct {
    Timeout    time.Duration
    MaxRetries int
    BaseURL    string
    Debug      bool
}

type Option func(*ClientConfig)

func WithTimeout(d time.Duration) Option {
    return func(c *ClientConfig) {
        c.Timeout = d
    }
}

func WithMaxRetries(n int) Option {
    return func(c *ClientConfig) {
        c.MaxRetries = n
    }
}

func WithBaseURL(url string) Option {
    return func(c *ClientConfig) {
        c.BaseURL = url
    }
}

func WithDebug(enabled bool) Option {
    return func(c *ClientConfig) {
        c.Debug = enabled
    }
}

// Constructor with functional options
func NewClient(opts ...Option) *ClientConfig {
    // Defaults
    cfg := &ClientConfig{
        Timeout:    30 * time.Second,
        MaxRetries: 3,
        BaseURL:    "https://api.example.com",
        Debug:      false,
    }

    for _, opt := range opts {
        opt(cfg)
    }

    return cfg
}

func main() {
    // Uses all defaults
    client1 := NewClient()
    fmt.Printf("%+v\n", client1)

    // Override some defaults
    client2 := NewClient(
        WithTimeout(10*time.Second),
        WithBaseURL("https://staging.example.com"),
        WithDebug(true),
    )
    fmt.Printf("%+v\n", client2)
}
```

---

## Repository Pattern

**Tutorial: Abstracting Data Storage Behind an Interface**

The repository pattern abstracts data access behind an interface with CRUD methods (`FindByID`, `FindAll`, `Create`, `Update`, `Delete`). This example uses an in-memory map as the backing store, but the interface could be implemented with PostgreSQL, MongoDB, or any storage backend. Application logic depends on the interface, not the implementation, making it easy to swap storage engines or mock the data layer in tests.

```
┌────────────────────────────────────────────────────┐
│  UserRepository interface                          │
│  ┌──────────────────────────────────────────┐    │
│  │ FindByID(id) │ FindAll() │ Create()      │    │
│  │ Update()     │ Delete(id)                 │    │
│  └──────────────────────────────────────────┘    │
│              ▲                                     │
│  ┌───────────┼───────────────┐                      │
│  │ InMemoryUserRepo              │                      │
│  │ users: map[int]*User          │                      │
│  │ nextID: auto-increment        │                      │
│  └────────────────────────────┘                      │
│                                                    │
│  Swap to PostgresUserRepo, MongoUserRepo, etc.     │
│  without changing application logic.                │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "errors"
    "fmt"
)

type User struct {
    ID    int
    Name  string
    Email string
}

// Repository interface — abstracts data store
type UserRepository interface {
    FindByID(id int) (*User, error)
    FindAll() ([]*User, error)
    Create(user *User) error
    Update(user *User) error
    Delete(id int) error
}

// In-memory implementation
type InMemoryUserRepo struct {
    users  map[int]*User
    nextID int
}

func NewInMemoryUserRepo() *InMemoryUserRepo {
    return &InMemoryUserRepo{users: make(map[int]*User), nextID: 1}
}

func (r *InMemoryUserRepo) FindByID(id int) (*User, error) {
    u, ok := r.users[id]
    if !ok {
        return nil, errors.New("user not found")
    }
    return u, nil
}

func (r *InMemoryUserRepo) FindAll() ([]*User, error) {
    result := make([]*User, 0, len(r.users))
    for _, u := range r.users {
        result = append(result, u)
    }
    return result, nil
}

func (r *InMemoryUserRepo) Create(user *User) error {
    user.ID = r.nextID
    r.nextID++
    r.users[user.ID] = user
    return nil
}

func (r *InMemoryUserRepo) Update(user *User) error {
    if _, ok := r.users[user.ID]; !ok {
        return errors.New("user not found")
    }
    r.users[user.ID] = user
    return nil
}

func (r *InMemoryUserRepo) Delete(id int) error {
    if _, ok := r.users[id]; !ok {
        return errors.New("user not found")
    }
    delete(r.users, id)
    return nil
}

func main() {
    repo := NewInMemoryUserRepo()

    repo.Create(&User{Name: "Alice", Email: "alice@test.com"})
    repo.Create(&User{Name: "Bob", Email: "bob@test.com"})

    users, _ := repo.FindAll()
    for _, u := range users {
        fmt.Printf("%+v\n", *u)
    }

    user, _ := repo.FindByID(1)
    fmt.Println("Found:", user.Name)
}
```

---

## Table-Driven Logic

**Tutorial: Replacing Long Switch Chains with Map-Based Dispatch**

Instead of long `if/else` or `switch` chains, table-driven logic uses a map of command names to handler functions. This makes adding new commands a one-line addition to the map, and the dispatch logic stays constant. The pattern is data-driven: the behavior is defined by data (the map) rather than control flow. Unknown commands are handled with a simple map lookup check. This pattern scales well and keeps each command handler isolated.

```
┌────────────────────────────────────────────────────┐
│  commands map[string]CommandHandler                 │
│  ┌──────────────────────────────────────────┐    │
│  │ "greet" ─► func(args) → "Hello, Alice!"    │    │
│  │ "upper" ─► func(args) → "HELLO WORLD"     │    │
│  │ "count" ─► func(args) → "3 arguments"      │    │
│  └──────────────────────────────────────────┘    │
│                                                    │
│  Dispatch:                                         │
│  handler, ok := commands[input.cmd]                 │
│   ├─ ok:  handler(args) ─► execute                │
│   └─ !ok: "unknown command"                        │
│                                                    │
│  Add new command: just add one map entry           │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "strings"
)

// Instead of long if/switch chains, use maps for dispatch
type CommandHandler func(args []string) string

func main() {
    commands := map[string]CommandHandler{
        "greet": func(args []string) string {
            if len(args) > 0 {
                return "Hello, " + args[0] + "!"
            }
            return "Hello!"
        },
        "upper": func(args []string) string {
            return strings.ToUpper(strings.Join(args, " "))
        },
        "count": func(args []string) string {
            return fmt.Sprintf("You provided %d arguments", len(args))
        },
    }

    // Dispatch
    inputs := []struct {
        cmd  string
        args []string
    }{
        {"greet", []string{"Alice"}},
        {"upper", []string{"hello", "world"}},
        {"count", []string{"a", "b", "c"}},
        {"unknown", nil},
    }

    for _, input := range inputs {
        handler, ok := commands[input.cmd]
        if !ok {
            fmt.Printf("%s: unknown command\n", input.cmd)
            continue
        }
        fmt.Printf("%s: %s\n", input.cmd, handler(input.args))
    }
}
```

---

## Interview Questions

1. **How is the Singleton pattern implemented in Go?**
   - Use `sync.Once`: `var once sync.Once; once.Do(func() { instance = &Singleton{} })`. Thread-safe, lazy initialization. Package-level variables with `init()` work for eager singletons.

2. **How does Go implement the Strategy pattern?**
   - Use function types or interfaces. Pass different implementations as function parameters or interface values. Example: `sort.Slice(s, lessFn)` uses a strategy function for comparison.

3. **What is the Functional Options pattern?**
   - `func WithTimeout(d time.Duration) Option` returns closures that modify a config struct. Constructor: `NewServer(opts ...Option)`. Provides clean, extensible API without breaking changes.

4. **How does Go implement the Observer pattern?**
   - Use channels for event notification. Observers register channels with a subject. Subject sends events to all registered channels. More idiomatic than callback-based observers.

5. **What is the Builder pattern in Go?**
   - Method chaining: `builder.SetName("x").SetPort(8080).Build()`. Alternative: functional options pattern (more idiomatic). Used when constructing complex objects with many optional parameters.

6. **How is the Factory pattern used in Go?**
   - Constructor functions: `func NewReader(r io.Reader) *Reader`. Return interfaces for flexibility: `func NewStore(typ string) Store`. Go doesn't need abstract factory classes—functions suffice.

7. **What is the Decorator pattern in Go?**
   - Wrap interfaces to add behavior: `func LoggingMiddleware(next http.Handler) http.Handler`. HTTP middleware is the classic example. Also used with `io.Reader`/`io.Writer` wrappers.

8. **How does Go handle the Repository pattern?**
   - Define an interface: `type UserRepo interface { FindByID(id int) (*User, error) }`. Implement with concrete types (PostgresRepo, MockRepo). Inject via constructor. Enables testing with mocks.

9. **What is the Circuit Breaker pattern?**
   - Protects against cascading failures. States: Closed (normal), Open (failing, reject requests), Half-Open (test recovery). Use libraries like `sony/gobreaker`. Important for microservices.

10. **How does composition replace inheritance in Go design patterns?**
    - Go uses struct embedding and interface composition instead of class hierarchies. Embed types for code reuse, implement interfaces for polymorphism. This keeps designs flat and flexible.

---

## Interview Problems & Solutions

### Problem 1 — Singleton: Thread-Safe Configuration Manager

**Problem:** Design a thread-safe configuration manager that reads settings from a map. It should be a singleton (only one instance across the entire application). Multiple goroutines will call `GetConfig()` concurrently. Implement `Get(key)` and `Set(key, value)` with proper synchronization.

```
┌────────────────────────────────────────────────────────┐
│  goroutine 1 ──► GetConfig() ──┐                       │
│  goroutine 2 ──► GetConfig() ──┼──► same *ConfigManager│
│  goroutine 3 ──► GetConfig() ──┘                       │
│                                                        │
│  ConfigManager                                         │
│  ┌──────────────────────────────┐                      │
│  │  mu      sync.RWMutex       │                       │
│  │  settings map[string]string  │                       │
│  │                              │                       │
│  │  Get(key) ──► RLock (read)   │                       │
│  │  Set(k,v) ──► Lock  (write)  │                       │
│  └──────────────────────────────┘                      │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
)

type ConfigManager struct {
	mu       sync.RWMutex
	settings map[string]string
}

var (
	cfgInstance *ConfigManager
	cfgOnce    sync.Once
)

func GetConfig() *ConfigManager {
	cfgOnce.Do(func() {
		cfgInstance = &ConfigManager{
			settings: map[string]string{
				"db_host": "localhost",
				"db_port": "5432",
			},
		}
	})
	return cfgInstance
}

func (c *ConfigManager) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	val, ok := c.settings[key]
	return val, ok
}

func (c *ConfigManager) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.settings[key] = value
}

func main() {
	var wg sync.WaitGroup

	// 10 goroutines reading/writing concurrently
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cfg := GetConfig()
			if id%2 == 0 {
				cfg.Set(fmt.Sprintf("key_%d", id), fmt.Sprintf("val_%d", id))
			} else {
				host, _ := cfg.Get("db_host")
				fmt.Printf("goroutine %d read db_host=%s\n", id, host)
			}
		}(i)
	}
	wg.Wait()

	// All point to the same instance
	cfg := GetConfig()
	fmt.Println(cfg == GetConfig()) // true
}
```

**Key Points:**
- `sync.Once` ensures only one `ConfigManager` is created even under concurrent access
- `sync.RWMutex` allows multiple concurrent readers (`RLock`) but exclusive writers (`Lock`)
- This combines two patterns: Singleton (single instance) + concurrent safety (RWMutex)

---

### Problem 2 — Factory: Multi-Channel Payment Processor

**Problem:** Build a payment processing system that supports credit card, PayPal, and crypto payments. Use the factory pattern to create processors based on a string type. Each processor should validate the payment and process it. Return an error for unsupported payment types.

```
┌────────────────────────────────────────────────────────┐
│  NewPaymentProcessor(type) ──► PaymentProcessor iface  │
│                                                        │
│  "credit_card" ──► CreditCardProcessor                 │
│     Process(100.0) ──► "CC charge $100.00, fee $2.00"  │
│                                                        │
│  "paypal"      ──► PayPalProcessor                     │
│     Process(50.0) ──► "PayPal transfer $50.00"         │
│                                                        │
│  "crypto"      ──► CryptoProcessor                     │
│     Process(200.0) ──► "Crypto payment 200.00 USD"     │
│                                                        │
│  "unknown"     ──► error: unsupported payment type     │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
)

type PaymentProcessor interface {
	Process(amount float64) (string, error)
	Validate() error
}

type CreditCardProcessor struct {
	CardNumber string
	FeePercent float64
}

func (c *CreditCardProcessor) Validate() error {
	if len(c.CardNumber) != 16 {
		return fmt.Errorf("invalid card number length: %d", len(c.CardNumber))
	}
	return nil
}

func (c *CreditCardProcessor) Process(amount float64) (string, error) {
	if err := c.Validate(); err != nil {
		return "", err
	}
	fee := amount * c.FeePercent / 100
	return fmt.Sprintf("CC charge $%.2f, fee $%.2f", amount, fee), nil
}

type PayPalProcessor struct {
	Email string
}

func (p *PayPalProcessor) Validate() error {
	if p.Email == "" {
		return fmt.Errorf("paypal email required")
	}
	return nil
}

func (p *PayPalProcessor) Process(amount float64) (string, error) {
	if err := p.Validate(); err != nil {
		return "", err
	}
	return fmt.Sprintf("PayPal transfer $%.2f to %s", amount, p.Email), nil
}

type CryptoProcessor struct {
	WalletAddr string
	Currency   string
}

func (cr *CryptoProcessor) Validate() error {
	if cr.WalletAddr == "" {
		return fmt.Errorf("wallet address required")
	}
	return nil
}

func (cr *CryptoProcessor) Process(amount float64) (string, error) {
	if err := cr.Validate(); err != nil {
		return "", err
	}
	return fmt.Sprintf("Crypto payment %.2f %s to %s", amount, cr.Currency, cr.WalletAddr), nil
}

// Factory function
func NewPaymentProcessor(pType string, details map[string]string) (PaymentProcessor, error) {
	switch pType {
	case "credit_card":
		return &CreditCardProcessor{
			CardNumber: details["card_number"],
			FeePercent: 2.0,
		}, nil
	case "paypal":
		return &PayPalProcessor{Email: details["email"]}, nil
	case "crypto":
		return &CryptoProcessor{
			WalletAddr: details["wallet"],
			Currency:   details["currency"],
		}, nil
	default:
		return nil, fmt.Errorf("unsupported payment type: %s", pType)
	}
}

func main() {
	payments := []struct {
		pType   string
		details map[string]string
		amount  float64
	}{
		{"credit_card", map[string]string{"card_number": "1234567890123456"}, 100.0},
		{"paypal", map[string]string{"email": "user@example.com"}, 50.0},
		{"crypto", map[string]string{"wallet": "0xABC123", "currency": "ETH"}, 200.0},
		{"unknown", nil, 10.0},
	}

	for _, p := range payments {
		processor, err := NewPaymentProcessor(p.pType, p.details)
		if err != nil {
			fmt.Println("Factory error:", err)
			continue
		}
		result, err := processor.Process(p.amount)
		if err != nil {
			fmt.Println("Process error:", err)
			continue
		}
		fmt.Println(result)
	}
}
```

**Key Points:**
- Factory returns interface type — caller doesn't know concrete type
- Each processor encapsulates its own validation logic
- Adding a new payment type = add struct + add one case to factory

---

### Problem 3 — Builder: SQL Query Builder

**Problem:** Implement a SQL query builder that constructs SELECT queries using method chaining. Support `Select(columns...)`, `From(table)`, `Where(condition)`, `OrderBy(column)`, `Limit(n)`, and `Build()` which returns the final SQL string.

```
┌────────────────────────────────────────────────────────┐
│  NewQueryBuilder()                                     │
│    .Select("name", "email")                            │
│    .From("users")                                      │
│    .Where("age > 18")                                  │
│    .Where("active = true")     ← multiple wheres ANDed│
│    .OrderBy("name ASC")                                │
│    .Limit(10)                                          │
│    .Build()                                            │
│                                                        │
│  ──► "SELECT name, email FROM users                    │
│       WHERE age > 18 AND active = true                 │
│       ORDER BY name ASC LIMIT 10"                      │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"strings"
)

type QueryBuilder struct {
	columns  []string
	table    string
	wheres   []string
	orderBy  string
	limit    int
	hasLimit bool
}

func NewQueryBuilder() *QueryBuilder {
	return &QueryBuilder{}
}

func (q *QueryBuilder) Select(cols ...string) *QueryBuilder {
	q.columns = cols
	return q
}

func (q *QueryBuilder) From(table string) *QueryBuilder {
	q.table = table
	return q
}

func (q *QueryBuilder) Where(condition string) *QueryBuilder {
	q.wheres = append(q.wheres, condition)
	return q
}

func (q *QueryBuilder) OrderBy(clause string) *QueryBuilder {
	q.orderBy = clause
	return q
}

func (q *QueryBuilder) Limit(n int) *QueryBuilder {
	q.limit = n
	q.hasLimit = true
	return q
}

func (q *QueryBuilder) Build() string {
	var sb strings.Builder

	// SELECT
	if len(q.columns) == 0 {
		sb.WriteString("SELECT *")
	} else {
		sb.WriteString("SELECT ")
		sb.WriteString(strings.Join(q.columns, ", "))
	}

	// FROM
	if q.table != "" {
		sb.WriteString(" FROM ")
		sb.WriteString(q.table)
	}

	// WHERE
	if len(q.wheres) > 0 {
		sb.WriteString(" WHERE ")
		sb.WriteString(strings.Join(q.wheres, " AND "))
	}

	// ORDER BY
	if q.orderBy != "" {
		sb.WriteString(" ORDER BY ")
		sb.WriteString(q.orderBy)
	}

	// LIMIT
	if q.hasLimit {
		sb.WriteString(fmt.Sprintf(" LIMIT %d", q.limit))
	}

	return sb.String()
}

func main() {
	// Complex query
	query := NewQueryBuilder().
		Select("name", "email", "age").
		From("users").
		Where("age > 18").
		Where("active = true").
		OrderBy("name ASC").
		Limit(10).
		Build()
	fmt.Println(query)
	// SELECT name, email, age FROM users WHERE age > 18 AND active = true ORDER BY name ASC LIMIT 10

	// Simple query — select all
	simple := NewQueryBuilder().
		From("products").
		Build()
	fmt.Println(simple)
	// SELECT * FROM products

	// Query with only WHERE
	filtered := NewQueryBuilder().
		Select("id", "title").
		From("posts").
		Where("published = true").
		Limit(5).
		Build()
	fmt.Println(filtered)
	// SELECT id, title FROM posts WHERE published = true LIMIT 5
}
```

**Key Points:**
- Each method returns `*QueryBuilder` for chaining
- Multiple `Where()` calls accumulate conditions joined with AND
- `Build()` assembles parts in correct SQL order
- Builder resets for each new `NewQueryBuilder()` call

---

### Problem 4 — Strategy: Pluggable Pricing Calculator

**Problem:** An e-commerce system needs different pricing strategies: regular pricing, percentage discount, buy-one-get-one, and seasonal markup. Implement the strategy pattern so the pricing algorithm can be swapped at runtime. Calculate total price for a cart of items.

```
┌────────────────────────────────────────────────────────┐
│  PricingStrategy interface                             │
│  CalculateTotal(items) ──► float64                     │
│       ▲          ▲          ▲          ▲               │
│  ┌────┴───┐ ┌────┴───┐ ┌────┴───┐ ┌────┴──────┐      │
│  │Regular │ │Discount│ │  BOGO  │ │ Seasonal   │      │
│  │ sum    │ │ sum*0.8│ │pay half│ │ sum * 1.15 │      │
│  └────────┘ └────────┘ └────────┘ └────────────┘      │
│                                                        │
│  Cart items: [{Shirt $25}, {Pants $40}, {Hat $15}]     │
│  Regular total:  $80.00                                │
│  20% discount:   $64.00                                │
│  BOGO:           $55.00  (cheapest free)               │
│  Seasonal +15%:  $92.00                                │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct {
	Name  string
	Price float64
}

// Strategy interface
type PricingStrategy interface {
	CalculateTotal(items []Item) float64
	Name() string
}

// Regular pricing — just sum everything
type RegularPricing struct{}

func (r *RegularPricing) Name() string { return "Regular" }
func (r *RegularPricing) CalculateTotal(items []Item) float64 {
	total := 0.0
	for _, item := range items {
		total += item.Price
	}
	return total
}

// Percentage discount
type DiscountPricing struct {
	Percent float64
}

func (d *DiscountPricing) Name() string {
	return fmt.Sprintf("%.0f%% Discount", d.Percent)
}
func (d *DiscountPricing) CalculateTotal(items []Item) float64 {
	total := 0.0
	for _, item := range items {
		total += item.Price
	}
	return total * (1 - d.Percent/100)
}

// Buy-One-Get-One: cheapest item free
type BOGOPricing struct{}

func (b *BOGOPricing) Name() string { return "BOGO (cheapest free)" }
func (b *BOGOPricing) CalculateTotal(items []Item) float64 {
	if len(items) < 2 {
		total := 0.0
		for _, item := range items {
			total += item.Price
		}
		return total
	}
	// Find cheapest item
	prices := make([]float64, len(items))
	for i, item := range items {
		prices[i] = item.Price
	}
	sort.Float64s(prices)

	total := 0.0
	for _, p := range prices[1:] { // skip cheapest
		total += p
	}
	return total
}

// Seasonal markup
type SeasonalPricing struct {
	MarkupPercent float64
}

func (s *SeasonalPricing) Name() string {
	return fmt.Sprintf("Seasonal +%.0f%%", s.MarkupPercent)
}
func (s *SeasonalPricing) CalculateTotal(items []Item) float64 {
	total := 0.0
	for _, item := range items {
		total += item.Price
	}
	return total * (1 + s.MarkupPercent/100)
}

// Context
type Cart struct {
	items    []Item
	strategy PricingStrategy
}

func (c *Cart) SetStrategy(s PricingStrategy) {
	c.strategy = s
}

func (c *Cart) Checkout() float64 {
	return c.strategy.CalculateTotal(c.items)
}

func main() {
	cart := &Cart{
		items: []Item{
			{"Shirt", 25.00},
			{"Pants", 40.00},
			{"Hat", 15.00},
		},
	}

	strategies := []PricingStrategy{
		&RegularPricing{},
		&DiscountPricing{Percent: 20},
		&BOGOPricing{},
		&SeasonalPricing{MarkupPercent: 15},
	}

	for _, s := range strategies {
		cart.SetStrategy(s)
		fmt.Printf("%-25s $%.2f\n", s.Name()+":", cart.Checkout())
	}
	// Regular:                  $80.00
	// 20% Discount:             $64.00
	// BOGO (cheapest free):     $65.00
	// Seasonal +15%:            $92.00
}
```

**Key Points:**
- Adding a new strategy requires zero changes to `Cart`
- Each strategy is independently testable
- `SetStrategy` allows runtime algorithm switching (e.g., apply promo code mid-session)

---

### Problem 5 — Observer: Real-Time Stock Price Alert System

**Problem:** Build a stock price monitoring system. When a stock price changes, notify all registered observers (email alerts, SMS alerts, dashboard updates). Observers can subscribe/unsubscribe dynamically. Use channels for goroutine-safe notification.

```
┌────────────────────────────────────────────────────────┐
│  StockTicker                                           │
│  ┌─────────────────────────────────────┐               │
│  │  symbol: "GOOG"                     │               │
│  │  observers: [email, sms, dashboard] │               │
│  └─────────────────────────────────────┘               │
│                                                        │
│  UpdatePrice(2850.50)                                  │
│       │                                                │
│       ├─► EmailAlert:  "GOOG price: $2850.50"          │
│       ├─► SMSAlert:    "GOOG now $2850.50"             │
│       └─► Dashboard:   "Updating GOOG: $2850.50"      │
│                                                        │
│  Unsubscribe(sms)                                      │
│  UpdatePrice(2855.00)                                  │
│       ├─► EmailAlert:  "GOOG price: $2855.00"          │
│       └─► Dashboard:   "Updating GOOG: $2855.00"      │
│           (SMS not notified — unsubscribed)             │
└────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type PriceUpdate struct {
	Symbol string
	Price  float64
}

// Observer interface
type StockObserver interface {
	OnPriceUpdate(update PriceUpdate)
	ID() string
}

// Subject
type StockTicker struct {
	symbol    string
	price     float64
	observers map[string]StockObserver
}

func NewStockTicker(symbol string) *StockTicker {
	return &StockTicker{
		symbol:    symbol,
		observers: make(map[string]StockObserver),
	}
}

func (s *StockTicker) Subscribe(observer StockObserver) {
	s.observers[observer.ID()] = observer
}

func (s *StockTicker) Unsubscribe(id string) {
	delete(s.observers, id)
}

func (s *StockTicker) UpdatePrice(price float64) {
	s.price = price
	update := PriceUpdate{Symbol: s.symbol, Price: price}
	for _, obs := range s.observers {
		obs.OnPriceUpdate(update)
	}
}

// Concrete observers
type EmailAlert struct {
	Email string
}

func (e *EmailAlert) ID() string { return "email-" + e.Email }
func (e *EmailAlert) OnPriceUpdate(u PriceUpdate) {
	fmt.Printf("[EMAIL to %s] %s price: $%.2f\n", e.Email, u.Symbol, u.Price)
}

type SMSAlert struct {
	Phone string
}

func (s *SMSAlert) ID() string { return "sms-" + s.Phone }
func (s *SMSAlert) OnPriceUpdate(u PriceUpdate) {
	fmt.Printf("[SMS to %s] %s now $%.2f\n", s.Phone, u.Symbol, u.Price)
}

type DashboardWidget struct {
	WidgetID string
}

func (d *DashboardWidget) ID() string { return "dash-" + d.WidgetID }
func (d *DashboardWidget) OnPriceUpdate(u PriceUpdate) {
	fmt.Printf("[DASHBOARD %s] Updating %s: $%.2f\n", d.WidgetID, u.Symbol, u.Price)
}

func main() {
	ticker := NewStockTicker("GOOG")

	email := &EmailAlert{Email: "trader@example.com"}
	sms := &SMSAlert{Phone: "+1234567890"}
	dash := &DashboardWidget{WidgetID: "main-panel"}

	ticker.Subscribe(email)
	ticker.Subscribe(sms)
	ticker.Subscribe(dash)

	fmt.Println("=== Price update: all observers ===")
	ticker.UpdatePrice(2850.50)

	// Unsubscribe SMS
	ticker.Unsubscribe(sms.ID())

	fmt.Println("\n=== Price update: SMS unsubscribed ===")
	ticker.UpdatePrice(2855.00)
}
```

**Key Points:**
- Map-based observer storage enables O(1) subscribe/unsubscribe
- Observers are identified by unique IDs to prevent duplicates
- Subject knows nothing about observer implementations — only the interface

---

### Problem 6 — Decorator: Composable HTTP Middleware Stack

**Problem:** Build a middleware system that supports logging, request-ID injection, panic recovery, and CORS headers. Each middleware wraps the next handler. Demonstrate composing them in different orders and show how order affects behavior.

```
┌────────────────────────────────────────────────────────┐
│  Request ──► Recovery ──► RequestID ──► Logger ──► App │
│                                                        │
│  RecoveryMiddleware:                                   │
│    defer func() { recover() } ← catches panics        │
│                                                        │
│  RequestIDMiddleware:                                  │
│    r.Header.Set("X-Request-ID", uuid)                  │
│                                                        │
│  LoggerMiddleware:                                     │
│    log start, call next, log end + duration            │
│                                                        │
│  CORSMiddleware:                                       │
│    w.Header().Set("Access-Control-Allow-Origin", "*")  │
│                                                        │
│  Chain(handler, mw1, mw2, mw3) applies right-to-left  │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"net/http/httptest"
	"time"
)

type Middleware func(http.Handler) http.Handler

// Chain applies middlewares right-to-left (last listed = innermost)
func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		h = middlewares[i](h)
	}
	return h
}

// Recovery middleware — catches panics
func RecoveryMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("[RECOVERY] panic: %v", err)
				http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

// Request ID middleware
var requestCounter int

func RequestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		requestCounter++
		reqID := fmt.Sprintf("req-%d", requestCounter)
		w.Header().Set("X-Request-ID", reqID)
		fmt.Printf("[REQUEST-ID] assigned %s\n", reqID)
		next.ServeHTTP(w, r)
	})
}

// Logger middleware
func LoggerMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		fmt.Printf("[LOG] started %s %s\n", r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
		fmt.Printf("[LOG] completed %s %s in %v\n", r.Method, r.URL.Path, time.Since(start))
	})
}

// CORS middleware
func CORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
		fmt.Println("[CORS] headers set")
		next.ServeHTTP(w, r)
	})
}

func main() {
	app := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from app!")
	})

	// Chain: Recovery → CORS → RequestID → Logger → App
	handler := Chain(app,
		RecoveryMiddleware,
		CORSMiddleware,
		RequestIDMiddleware,
		LoggerMiddleware,
	)

	// Simulate request
	req := httptest.NewRequest("GET", "/api/data", nil)
	rr := httptest.NewRecorder()

	handler.ServeHTTP(rr, req)

	fmt.Println("\nResponse headers:")
	for k, v := range rr.Header() {
		fmt.Printf("  %s: %s\n", k, v)
	}
	fmt.Println("Body:", rr.Body.String())
}
```

**Key Points:**
- `Chain()` applies middlewares so the first in the list is the outermost wrapper
- Recovery must be outermost to catch panics from any inner middleware
- Each middleware is independently composable and testable
- `httptest` enables testing without starting a real server

---

### Problem 7 — Adapter: Legacy XML API to Modern JSON Interface

**Problem:** Your system works with a `DataFetcher` interface that returns JSON responses. You need to integrate a legacy XML service that returns XML strings. Write an adapter that converts the legacy XML API into your modern JSON interface without modifying the legacy code.

```
┌────────────────────────────────────────────────────────┐
│  Your code expects:         Legacy code returns:       │
│  ┌─────────────────┐        ┌──────────────────┐      │
│  │ DataFetcher     │        │ LegacyXMLService │      │
│  │ FetchJSON(id)   │        │ GetXML(id)       │      │
│  │  → JSON string  │        │  → XML string    │      │
│  └─────────────────┘        └──────────────────┘      │
│        ▲                          ▲                    │
│        │  implements              │  wraps             │
│  ┌─────┴──────────────────────────┴──────────┐        │
│  │  XMLToJSONAdapter                          │        │
│  │  FetchJSON(id)                             │        │
│  │    1. call legacy.GetXML(id)               │        │
│  │    2. parse XML                            │        │
│  │    3. convert to JSON                      │        │
│  │    4. return JSON string                   │        │
│  └────────────────────────────────────────────┘        │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"encoding/json"
	"encoding/xml"
	"fmt"
)

// Modern interface our code expects
type DataFetcher interface {
	FetchJSON(id string) (string, error)
}

// Legacy XML service — cannot modify this
type LegacyXMLService struct{}

type XMLProduct struct {
	XMLName xml.Name `xml:"product"`
	ID      string   `xml:"id"`
	Name    string   `xml:"name"`
	Price   float64  `xml:"price"`
}

func (l *LegacyXMLService) GetXML(id string) string {
	// Simulates legacy XML response
	products := map[string]string{
		"1": `<product><id>1</id><name>Laptop</name><price>999.99</price></product>`,
		"2": `<product><id>2</id><name>Phone</name><price>699.99</price></product>`,
	}
	if xml, ok := products[id]; ok {
		return xml
	}
	return ""
}

// JSON representation
type JSONProduct struct {
	ID    string  `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

// Adapter — converts XML API to JSON interface
type XMLToJSONAdapter struct {
	legacy *LegacyXMLService
}

func (a *XMLToJSONAdapter) FetchJSON(id string) (string, error) {
	// Step 1: Get XML from legacy service
	xmlData := a.legacy.GetXML(id)
	if xmlData == "" {
		return "", fmt.Errorf("product %s not found", id)
	}

	// Step 2: Parse XML
	var xmlProd XMLProduct
	if err := xml.Unmarshal([]byte(xmlData), &xmlProd); err != nil {
		return "", fmt.Errorf("xml parse error: %w", err)
	}

	// Step 3: Convert to JSON
	jsonProd := JSONProduct{
		ID:    xmlProd.ID,
		Name:  xmlProd.Name,
		Price: xmlProd.Price,
	}

	jsonBytes, err := json.Marshal(jsonProd)
	if err != nil {
		return "", fmt.Errorf("json encode error: %w", err)
	}

	return string(jsonBytes), nil
}

// Application code — works with DataFetcher interface
func displayProduct(fetcher DataFetcher, id string) {
	result, err := fetcher.FetchJSON(id)
	if err != nil {
		fmt.Printf("Error: %s\n", err)
		return
	}
	fmt.Printf("Product JSON: %s\n", result)
}

func main() {
	legacy := &LegacyXMLService{}
	adapter := &XMLToJSONAdapter{legacy: legacy}

	displayProduct(adapter, "1") // Product JSON: {"id":"1","name":"Laptop","price":999.99}
	displayProduct(adapter, "2") // Product JSON: {"id":"2","name":"Phone","price":699.99}
	displayProduct(adapter, "9") // Error: product 9 not found
}
```

**Key Points:**
- Legacy code is never modified — adapter wraps it
- Adapter translates between incompatible APIs (XML ↔ JSON)
- `displayProduct` only depends on `DataFetcher` — doesn't know about XML

---

### Problem 8 — Dependency Injection: Testable Order Service

**Problem:** Build an order processing service with dependencies on inventory checking, payment processing, and notification sending. Use dependency injection so every dependency is an interface. Write both production implementations and mock implementations for testing.

```
┌────────────────────────────────────────────────────────┐
│  OrderService                                          │
│  ┌──────────────────────────────────────────────┐      │
│  │  inventory InventoryChecker                   │      │
│  │  payment   PaymentGateway                     │      │
│  │  notifier  OrderNotifier                      │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  PlaceOrder(itemID, qty, amount)                       │
│    1. inventory.CheckStock(itemID, qty)                 │
│    2. payment.Charge(amount)                           │
│    3. notifier.SendConfirmation(orderID)               │
│                                                        │
│  Production: RealInventory, StripePayment, EmailNotify │
│  Testing:    MockInventory, MockPayment, MockNotify    │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
)

// === Dependency interfaces ===

type InventoryChecker interface {
	CheckStock(itemID string, qty int) (bool, error)
}

type PaymentGateway interface {
	Charge(amount float64) (string, error) // returns transaction ID
}

type OrderNotifier interface {
	SendConfirmation(orderID string) error
}

// === Production implementations ===

type RealInventory struct{}

func (r *RealInventory) CheckStock(itemID string, qty int) (bool, error) {
	fmt.Printf("[Inventory] Checking stock for %s, qty=%d\n", itemID, qty)
	return true, nil // simplified: always in stock
}

type StripePayment struct{}

func (s *StripePayment) Charge(amount float64) (string, error) {
	fmt.Printf("[Stripe] Charging $%.2f\n", amount)
	return "txn_abc123", nil
}

type EmailNotifier struct{}

func (e *EmailNotifier) SendConfirmation(orderID string) error {
	fmt.Printf("[Email] Confirmation sent for order %s\n", orderID)
	return nil
}

// === Mock implementations for testing ===

type MockInventory struct {
	Available bool
}

func (m *MockInventory) CheckStock(itemID string, qty int) (bool, error) {
	return m.Available, nil
}

type MockPayment struct {
	ShouldFail bool
}

func (m *MockPayment) Charge(amount float64) (string, error) {
	if m.ShouldFail {
		return "", fmt.Errorf("payment declined")
	}
	return "mock_txn_001", nil
}

type MockNotifier struct {
	Sent []string // track what was sent
}

func (m *MockNotifier) SendConfirmation(orderID string) error {
	m.Sent = append(m.Sent, orderID)
	return nil
}

// === Service with injected dependencies ===

type OrderService struct {
	inventory InventoryChecker
	payment   PaymentGateway
	notifier  OrderNotifier
	nextID    int
}

func NewOrderService(inv InventoryChecker, pay PaymentGateway, notif OrderNotifier) *OrderService {
	return &OrderService{
		inventory: inv,
		payment:   pay,
		notifier:  notif,
	}
}

func (o *OrderService) PlaceOrder(itemID string, qty int, amount float64) (string, error) {
	// Step 1: Check inventory
	inStock, err := o.inventory.CheckStock(itemID, qty)
	if err != nil {
		return "", fmt.Errorf("inventory check failed: %w", err)
	}
	if !inStock {
		return "", fmt.Errorf("item %s out of stock", itemID)
	}

	// Step 2: Process payment
	txnID, err := o.payment.Charge(amount)
	if err != nil {
		return "", fmt.Errorf("payment failed: %w", err)
	}

	// Step 3: Create order
	o.nextID++
	orderID := fmt.Sprintf("ORD-%d-%s", o.nextID, txnID)

	// Step 4: Send notification
	if err := o.notifier.SendConfirmation(orderID); err != nil {
		fmt.Printf("Warning: notification failed for %s: %v\n", orderID, err)
	}

	return orderID, nil
}

func main() {
	fmt.Println("=== PRODUCTION ===")
	prodSvc := NewOrderService(&RealInventory{}, &StripePayment{}, &EmailNotifier{})
	orderID, err := prodSvc.PlaceOrder("LAPTOP-01", 1, 999.99)
	if err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Println("Order placed:", orderID)
	}

	fmt.Println("\n=== TEST: Happy path ===")
	mockNotif := &MockNotifier{}
	testSvc := NewOrderService(
		&MockInventory{Available: true},
		&MockPayment{ShouldFail: false},
		mockNotif,
	)
	orderID, err = testSvc.PlaceOrder("TEST-01", 2, 50.00)
	fmt.Printf("Order: %s, Error: %v, Notifications: %v\n", orderID, err, mockNotif.Sent)

	fmt.Println("\n=== TEST: Out of stock ===")
	testSvc2 := NewOrderService(
		&MockInventory{Available: false},
		&MockPayment{},
		&MockNotifier{},
	)
	_, err = testSvc2.PlaceOrder("TEST-02", 1, 25.00)
	fmt.Println("Expected error:", err)

	fmt.Println("\n=== TEST: Payment failure ===")
	testSvc3 := NewOrderService(
		&MockInventory{Available: true},
		&MockPayment{ShouldFail: true},
		&MockNotifier{},
	)
	_, err = testSvc3.PlaceOrder("TEST-03", 1, 75.00)
	fmt.Println("Expected error:", err)
}
```

**Key Points:**
- Every external dependency is an interface — fully testable
- Mock implementations let you test edge cases (out of stock, payment failure)
- `MockNotifier.Sent` slice enables assertion on side effects
- Constructor injection makes dependencies explicit and visible

---

### Problem 9 — Functional Options: Configurable Worker Pool

**Problem:** Build a worker pool that processes jobs concurrently. Use the functional options pattern to configure: number of workers, buffer size, job timeout, retry count, and a custom logger. All options should have sensible defaults.

```
┌────────────────────────────────────────────────────────┐
│  NewWorkerPool(opts ...PoolOption)                     │
│       │                                                │
│       ▼  defaults:                                     │
│  cfg = { Workers:4, BufSize:100, Timeout:30s,          │
│          Retries:0, Logger: defaultLogger }            │
│       │                                                │
│       ▼  apply opts:                                   │
│  WithWorkers(8)     ──► Workers = 8                    │
│  WithTimeout(10s)   ──► Timeout = 10s                  │
│  WithRetries(3)     ──► Retries = 3                    │
│       │                                                │
│       ▼  Start() → spawns 8 goroutines                 │
│  Submit(job) ──► buffered chan ──► workers process      │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Job func() error

type LogFunc func(format string, args ...any)

type PoolConfig struct {
	Workers int
	BufSize int
	Timeout time.Duration
	Retries int
	Logger  LogFunc
}

type PoolOption func(*PoolConfig)

func WithWorkers(n int) PoolOption {
	return func(c *PoolConfig) { c.Workers = n }
}

func WithBufferSize(n int) PoolOption {
	return func(c *PoolConfig) { c.BufSize = n }
}

func WithTimeout(d time.Duration) PoolOption {
	return func(c *PoolConfig) { c.Timeout = d }
}

func WithRetries(n int) PoolOption {
	return func(c *PoolConfig) { c.Retries = n }
}

func WithLogger(l LogFunc) PoolOption {
	return func(c *PoolConfig) { c.Logger = l }
}

type WorkerPool struct {
	cfg  PoolConfig
	jobs chan Job
	wg   sync.WaitGroup
}

func NewWorkerPool(opts ...PoolOption) *WorkerPool {
	cfg := PoolConfig{
		Workers: 4,
		BufSize: 100,
		Timeout: 30 * time.Second,
		Retries: 0,
		Logger:  func(f string, a ...any) { fmt.Printf(f+"\n", a...) },
	}
	for _, opt := range opts {
		opt(&cfg)
	}

	return &WorkerPool{
		cfg:  cfg,
		jobs: make(chan Job, cfg.BufSize),
	}
}

func (p *WorkerPool) Start() {
	for i := 0; i < p.cfg.Workers; i++ {
		p.wg.Add(1)
		go p.worker(i)
	}
	p.cfg.Logger("Pool started with %d workers (buf=%d, timeout=%v, retries=%d)",
		p.cfg.Workers, p.cfg.BufSize, p.cfg.Timeout, p.cfg.Retries)
}

func (p *WorkerPool) worker(id int) {
	defer p.wg.Done()
	for job := range p.jobs {
		p.executeWithRetry(id, job)
	}
}

func (p *WorkerPool) executeWithRetry(workerID int, job Job) {
	var err error
	for attempt := 0; attempt <= p.cfg.Retries; attempt++ {
		done := make(chan error, 1)
		go func() { done <- job() }()

		select {
		case err = <-done:
			if err == nil {
				p.cfg.Logger("[worker %d] job succeeded (attempt %d)", workerID, attempt+1)
				return
			}
			p.cfg.Logger("[worker %d] job failed (attempt %d): %v", workerID, attempt+1, err)
		case <-time.After(p.cfg.Timeout):
			p.cfg.Logger("[worker %d] job timed out (attempt %d)", workerID, attempt+1)
			err = fmt.Errorf("timeout")
		}
	}
	p.cfg.Logger("[worker %d] job exhausted retries: %v", workerID, err)
}

func (p *WorkerPool) Submit(job Job) {
	p.jobs <- job
}

func (p *WorkerPool) Shutdown() {
	close(p.jobs)
	p.wg.Wait()
	p.cfg.Logger("Pool shut down")
}

func main() {
	pool := NewWorkerPool(
		WithWorkers(3),
		WithBufferSize(10),
		WithTimeout(2*time.Second),
		WithRetries(2),
	)

	pool.Start()

	// Submit jobs
	for i := 0; i < 5; i++ {
		id := i
		pool.Submit(func() error {
			time.Sleep(100 * time.Millisecond)
			if id == 2 {
				return fmt.Errorf("job %d failed", id)
			}
			fmt.Printf("  → job %d completed\n", id)
			return nil
		})
	}

	pool.Shutdown()
}
```

**Key Points:**
- All options have defaults — `NewWorkerPool()` with no args is valid
- Adding a new option = add one `WithXxx` function, no signature change
- Custom logger option allows injecting structured logging in production
- Pattern composes well: `WithRetries(3)` + `WithTimeout(2s)` interact cleanly

---

### Problem 10 — Repository: Generic CRUD with Type Constraints

**Problem:** Implement a generic in-memory repository that works with any entity type having an `ID` field. Support `Create`, `FindByID`, `FindAll`, `Update`, `Delete`, and `FindWhere(predicate)` for filtered queries. Demonstrate with both `Product` and `Order` entities.

```
┌────────────────────────────────────────────────────────┐
│  Repository[T Entity] interface                        │
│  ┌──────────────────────────────────────────────┐      │
│  │ Create(entity) │ FindByID(id) │ FindAll()    │      │
│  │ Update(entity) │ Delete(id)   │ FindWhere(fn)│      │
│  └──────────────────────────────────────────────┘      │
│              ▲                                         │
│  ┌───────────┴──────────────────┐                      │
│  │ InMemoryRepo[T]              │                      │
│  │  data: map[int]T             │                      │
│  └──────────────────────────────┘                      │
│                                                        │
│  repo := NewInMemoryRepo[Product]()                    │
│  repo.Create(Product{Name:"Laptop", Price: 999})       │
│  repo.FindWhere(func(p Product) bool {                 │
│      return p.Price > 500                              │
│  })                                                    │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
)

// Entity constraint — must have GetID/SetID
type Entity interface {
	GetID() int
	SetID(int)
}

// Generic repository interface
type Repository[T Entity] interface {
	Create(entity T) T
	FindByID(id int) (T, bool)
	FindAll() []T
	Update(entity T) error
	Delete(id int) error
	FindWhere(predicate func(T) bool) []T
}

// Generic in-memory implementation
type InMemoryRepo[T Entity] struct {
	data   map[int]T
	nextID int
}

func NewInMemoryRepo[T Entity]() *InMemoryRepo[T] {
	return &InMemoryRepo[T]{data: make(map[int]T), nextID: 1}
}

func (r *InMemoryRepo[T]) Create(entity T) T {
	entity.SetID(r.nextID)
	r.data[r.nextID] = entity
	r.nextID++
	return entity
}

func (r *InMemoryRepo[T]) FindByID(id int) (T, bool) {
	e, ok := r.data[id]
	return e, ok
}

func (r *InMemoryRepo[T]) FindAll() []T {
	result := make([]T, 0, len(r.data))
	for _, e := range r.data {
		result = append(result, e)
	}
	return result
}

func (r *InMemoryRepo[T]) Update(entity T) error {
	id := entity.GetID()
	if _, ok := r.data[id]; !ok {
		return fmt.Errorf("entity with id %d not found", id)
	}
	r.data[id] = entity
	return nil
}

func (r *InMemoryRepo[T]) Delete(id int) error {
	if _, ok := r.data[id]; !ok {
		return fmt.Errorf("entity with id %d not found", id)
	}
	delete(r.data, id)
	return nil
}

func (r *InMemoryRepo[T]) FindWhere(predicate func(T) bool) []T {
	var result []T
	for _, e := range r.data {
		if predicate(e) {
			result = append(result, e)
		}
	}
	return result
}

// === Concrete entities ===

type Product struct {
	ID       int
	Name     string
	Price    float64
	Category string
}

func (p *Product) GetID() int  { return p.ID }
func (p *Product) SetID(id int) { p.ID = id }

type Order struct {
	ID       int
	Customer string
	Total    float64
	Status   string
}

func (o *Order) GetID() int  { return o.ID }
func (o *Order) SetID(id int) { o.ID = id }

func main() {
	// Product repository
	productRepo := NewInMemoryRepo[*Product]()

	productRepo.Create(&Product{Name: "Laptop", Price: 999.99, Category: "Electronics"})
	productRepo.Create(&Product{Name: "Shirt", Price: 29.99, Category: "Clothing"})
	productRepo.Create(&Product{Name: "Monitor", Price: 549.99, Category: "Electronics"})
	productRepo.Create(&Product{Name: "Socks", Price: 9.99, Category: "Clothing"})

	// FindWhere: electronics over $500
	expensive := productRepo.FindWhere(func(p *Product) bool {
		return p.Category == "Electronics" && p.Price > 500
	})
	fmt.Println("Expensive electronics:")
	for _, p := range expensive {
		fmt.Printf("  %s ($%.2f)\n", p.Name, p.Price)
	}

	// Order repository — same generic code
	orderRepo := NewInMemoryRepo[*Order]()

	orderRepo.Create(&Order{Customer: "Alice", Total: 150.00, Status: "pending"})
	orderRepo.Create(&Order{Customer: "Bob", Total: 89.50, Status: "shipped"})
	orderRepo.Create(&Order{Customer: "Charlie", Total: 220.00, Status: "pending"})

	// FindWhere: pending orders
	pending := orderRepo.FindWhere(func(o *Order) bool {
		return o.Status == "pending"
	})
	fmt.Println("\nPending orders:")
	for _, o := range pending {
		fmt.Printf("  %s: $%.2f\n", o.Customer, o.Total)
	}

	// Update + Delete
	if prod, ok := productRepo.FindByID(2); ok {
		prod.Price = 24.99
		productRepo.Update(prod)
		fmt.Printf("\nUpdated product 2: %s now $%.2f\n", prod.Name, prod.Price)
	}

	productRepo.Delete(4)
	fmt.Printf("After delete: %d products remain\n", len(productRepo.FindAll()))
}
```

**Key Points:**
- Go generics eliminate duplicated repository code for each entity type
- `FindWhere` accepts any predicate — flexible querying without query language
- Entity interface constraint ensures all types have ID access
- Same `InMemoryRepo` works for `Product`, `Order`, or any future entity

---

### Problem 11 — Table-Driven Logic: REST API Router

**Problem:** Build a mini HTTP router using table-driven logic. Support method+path matching, path parameters (`:id`), and grouped route registration. The route table maps `"METHOD /path"` to handler functions, replacing nested switch statements.

```
┌────────────────────────────────────────────────────────┐
│  routes map[string]HandlerFunc                         │
│  ┌──────────────────────────────────────────────┐      │
│  │ "GET /users"        ──► listUsers()           │      │
│  │ "POST /users"       ──► createUser()          │      │
│  │ "GET /users/:id"    ──► getUser(id)           │      │
│  │ "DELETE /users/:id" ──► deleteUser(id)        │      │
│  │ "GET /health"       ──► healthCheck()         │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  Match("GET", "/users/42")                             │
│    1. try exact: "GET /users/42" → miss                │
│    2. try pattern: "GET /users/:id" → hit              │
│    3. extract params: {id: "42"}                       │
│    4. call handler with params                         │
└────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"strings"
)

type Params map[string]string
type HandlerFunc func(params Params) string

type Router struct {
	routes map[string]HandlerFunc // "METHOD /path" → handler
}

func NewRouter() *Router {
	return &Router{routes: make(map[string]HandlerFunc)}
}

func (r *Router) Handle(method, path string, handler HandlerFunc) {
	key := method + " " + path
	r.routes[key] = handler
}

// matchPattern checks if a path matches a route pattern with :param segments
func matchPattern(pattern, path string) (Params, bool) {
	patternParts := strings.Split(strings.Trim(pattern, "/"), "/")
	pathParts := strings.Split(strings.Trim(path, "/"), "/")

	if len(patternParts) != len(pathParts) {
		return nil, false
	}

	params := make(Params)
	for i, pp := range patternParts {
		if strings.HasPrefix(pp, ":") {
			params[pp[1:]] = pathParts[i] // extract parameter
		} else if pp != pathParts[i] {
			return nil, false
		}
	}
	return params, true
}

func (r *Router) Dispatch(method, path string) string {
	// 1. Try exact match
	exactKey := method + " " + path
	if handler, ok := r.routes[exactKey]; ok {
		return handler(Params{})
	}

	// 2. Try pattern matching
	for routeKey, handler := range r.routes {
		parts := strings.SplitN(routeKey, " ", 2)
		if parts[0] != method {
			continue
		}
		if params, ok := matchPattern(parts[1], path); ok {
			return handler(params)
		}
	}

	return fmt.Sprintf("404: %s %s not found", method, path)
}

func main() {
	router := NewRouter()

	// Register routes — table-driven
	router.Handle("GET", "/users", func(p Params) string {
		return "List of all users: [Alice, Bob, Charlie]"
	})

	router.Handle("POST", "/users", func(p Params) string {
		return "User created successfully"
	})

	router.Handle("GET", "/users/:id", func(p Params) string {
		return fmt.Sprintf("User details for ID=%s", p["id"])
	})

	router.Handle("DELETE", "/users/:id", func(p Params) string {
		return fmt.Sprintf("Deleted user ID=%s", p["id"])
	})

	router.Handle("GET", "/products/:category/:id", func(p Params) string {
		return fmt.Sprintf("Product %s in category %s", p["id"], p["category"])
	})

	router.Handle("GET", "/health", func(p Params) string {
		return "OK"
	})

	// Dispatch requests
	requests := []struct {
		method string
		path   string
	}{
		{"GET", "/users"},
		{"POST", "/users"},
		{"GET", "/users/42"},
		{"DELETE", "/users/7"},
		{"GET", "/products/electronics/99"},
		{"GET", "/health"},
		{"PUT", "/unknown"},
	}

	for _, req := range requests {
		result := router.Dispatch(req.method, req.path)
		fmt.Printf("%-7s %-30s → %s\n", req.method, req.path, result)
	}
}
```

**Key Points:**
- Route table replaces large `switch` statements — adding routes is one-line each
- Pattern matching with `:param` extracts path parameters automatically
- Multi-segment patterns (`/products/:category/:id`) work naturally
- Exact match is tried first for performance, then pattern fallback
