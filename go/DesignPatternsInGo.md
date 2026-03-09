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
