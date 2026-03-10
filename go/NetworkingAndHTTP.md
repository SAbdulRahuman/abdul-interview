# Chapter 21 — Networking & HTTP

Go's `net/http` package is production-ready out of the box — no external framework needed. Go 1.22 added method-based routing with path parameters, making it competitive with third-party routers.

```
┌──────────────────────────────────────────────────────────┐
│             HTTP Request Lifecycle                      │
│                                                          │
│  Client                Server (net/http)                 │
│  ┌──────┐              ┌────────────────────────┐        │
│  │ GET  │──────────────►│ ServeMux (router)      │       │
│  │ /api │              │   │                    │        │
│  │/users│              │   ├── "GET /api/users" │        │
│  └──────┘              │   │   └► handler func  │        │
│                        │   │       │            │        │
│  ┌──────┐              │   │  ┌────▼─────────┐  │        │
│  │ JSON │◄─────────────│   │  │ w.Write()    │  │        │
│  │ resp │              │   │  │ w.Header()   │  │        │
│  └──────┘              │   │  └──────────────┘  │        │
│                        └───┴────────────────────┘        │
│                                                          │
│  Middleware pattern (chaining handlers):                 │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐       │
│  │Logging │──►│ Auth   │──►│ CORS   │──►│ Handler │      │
│  └────────┘  └────────┘  └────────┘  └──────────┘       │
│  Each middleware wraps the next http.Handler             │
│                                                          │
│  Go 1.22+ routing: "GET /api/users/{id}"                │
│  Path params: r.PathValue("id")                         │
└──────────────────────────────────────────────────────────┘
```

## HTTP Server

**Tutorial: Building a RESTful HTTP Server with Go 1.22+ Routing**

This example demonstrates how to set up an HTTP server with both the classic `http.HandleFunc` and the new Go 1.22+ method-based routing. The new `ServeMux` supports patterns like `"GET /api/users/{id}"`, eliminating the need for third-party routers in most cases. Notice how `r.PathValue("id")` extracts path parameters, and how each handler receives the standard `http.ResponseWriter` and `*http.Request` pair.

```
┌─────────────────────────────────────────────────┐
│          Go 1.22+ ServeMux Routing              │
│                                                 │
│  mux.HandleFunc("GET /api/users", listUsers)    │
│  mux.HandleFunc("POST /api/users", createUser)  │
│  mux.HandleFunc("GET /api/users/{id}", getUser) │
│                                                 │
│  Incoming: GET /api/users/42                    │
│       │                                         │
│       ▼                                         │
│  ┌──────────┐   pattern match   ┌────────────┐  │
│  │ ServeMux │──────────────────►│ getUser()  │  │
│  └──────────┘                   └─────┬──────┘  │
│                                       │         │
│                          r.PathValue("id")="42" │
│                                       │         │
│                                       ▼         │
│                               ┌──────────────┐  │
│                               │ JSON response│  │
│                               │ {"id":"42"} │  │
│                               └──────────────┘  │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

func main() {
    // Simple handlers with HandleFunc
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, World!")
    })

    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })

    // Enhanced routing (Go 1.22+) — method + path patterns
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/users", listUsers)
    mux.HandleFunc("POST /api/users", createUser)
    mux.HandleFunc("GET /api/users/{id}", getUser)        // path parameter
    mux.HandleFunc("DELETE /api/users/{id}", deleteUser)

    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}

func listUsers(w http.ResponseWriter, r *http.Request) {
    users := []map[string]string{
        {"id": "1", "name": "Alice"},
        {"id": "2", "name": "Bob"},
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user map[string]string
    json.NewDecoder(r.Body).Decode(&user)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+ path parameter
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"id": id, "name": "User " + id})
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    w.WriteHeader(http.StatusNoContent)
    _ = id
}
```

---

## http.Handler Interface and Middleware

**Tutorial: Implementing the http.Handler Interface and Middleware Chaining**

This example shows the core abstraction of Go's HTTP ecosystem: the `http.Handler` interface with its single `ServeHTTP` method. You'll see how to build a struct-based handler, create middleware functions that wrap handlers (logging, auth), and chain them together. The key insight is that middleware is just a function that takes a handler and returns a new handler, enabling clean composition.

```
┌───────────────────────────────────────────────────┐
│         Middleware Chain Execution Order           │
│                                                   │
│  Request arrives                                  │
│       │                                           │
│       ▼                                           │
│  ┌──────────────────┐                             │
│  │ loggingMiddleware │  log "→ GET /api/"         │
│  │   next.ServeHTTP ─┼──┐                         │
│  └──────────────────┘  │                          │
│                        ▼                          │
│  ┌──────────────────┐                             │
│  │ authMiddleware    │  check Authorization hdr   │
│  │   next.ServeHTTP ─┼──┐                         │
│  └──────────────────┘  │                          │
│                        ▼                          │
│  ┌──────────────────┐                             │
│  │ APIHandler       │  ServeHTTP writes response  │
│  │   (core handler) │                             │
│  └──────────────────┘                             │
│       │                                           │
│       ▼                                           │
│  Response unwinds back through middleware          │
│  loggingMiddleware logs "← GET /api/ (12ms)"      │
└───────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// http.Handler interface:
// type Handler interface {
//     ServeHTTP(w http.ResponseWriter, r *http.Request)
// }

// Custom handler implementing http.Handler
type APIHandler struct {
    prefix string
}

func (h *APIHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "[%s] Handling: %s %s", h.prefix, r.Method, r.URL.Path)
}

// Middleware pattern — handler wrapping
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("→ %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        log.Printf("← %s %s (%v)", r.Method, r.URL.Path, time.Since(start))
    })
}

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

// Chain middleware
func chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func main() {
    api := &APIHandler{prefix: "v1"}

    // Apply middleware chain
    handler := chain(api, loggingMiddleware, authMiddleware)

    mux := http.NewServeMux()
    mux.Handle("/api/", handler)

    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## HTTP Client

**Tutorial: Making HTTP Requests with a Custom Client**

This example covers the three main ways to make HTTP requests in Go: simple `client.Get()`, `client.Post()` with a JSON body, and fully custom requests via `http.NewRequestWithContext`. Always create a custom `http.Client` with a `Timeout` — the default client has no timeout and can hang forever. Watch how `context.WithTimeout` provides per-request cancellation, and how `defer resp.Body.Close()` prevents resource leaks.

```
┌──────────────────────────────────────────────────┐
│          HTTP Client Request Patterns            │
│                                                  │
│  Simple GET:                                     │
│  client.Get(url) ──► resp, err                   │
│                       └► resp.Body (io.ReadAll)  │
│                                                  │
│  POST with body:                                 │
│  client.Post(url, contentType, body) ──► resp    │
│                                                  │
│  Custom request (full control):                  │
│  ┌────────────────────────────┐                  │
│  │ http.NewRequestWithContext │                  │
│  │  ctx (timeout/cancel)     │                  │
│  │  method: "GET"            │                  │
│  │  headers: X-Custom, Accept│                  │
│  └───────────┬────────────────┘                  │
│              ▼                                   │
│  client.Do(req) ──► resp ──► json.Decode         │
│                                                  │
│  ⚠ ALWAYS: defer resp.Body.Close()              │
│  ⚠ ALWAYS: set client.Timeout                   │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strings"
    "time"
)

func main() {
    // Custom client with timeout (always set timeouts!)
    client := &http.Client{
        Timeout: 10 * time.Second,
    }

    // GET request
    resp, err := client.Get("https://httpbin.org/get")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Status:", resp.StatusCode)
    fmt.Println("Body:", string(body[:100]), "...")

    // POST with JSON body
    payload := strings.NewReader(`{"name":"Alice","age":30}`)
    resp2, err := client.Post("https://httpbin.org/post",
        "application/json", payload)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp2.Body.Close()
    fmt.Println("POST Status:", resp2.StatusCode)

    // Custom request with context and headers
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", "https://httpbin.org/headers", nil)
    req.Header.Set("X-Custom-Header", "my-value")
    req.Header.Set("Accept", "application/json")

    resp3, err := client.Do(req)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp3.Body.Close()

    var result map[string]any
    json.NewDecoder(resp3.Body).Decode(&result)
    fmt.Println("Headers response:", result)
}
```

---

## Graceful Shutdown

**Tutorial: Gracefully Shutting Down an HTTP Server**

This example demonstrates the production pattern for server shutdown: run the server in a goroutine, listen for OS signals (SIGINT/SIGTERM) on a channel, then call `server.Shutdown(ctx)`. The `Shutdown` method stops accepting new connections while letting in-flight requests finish within the context's deadline. This prevents dropped requests during deployments and is essential for containerized environments like Kubernetes.

```
┌──────────────────────────────────────────────────┐
│          Graceful Shutdown Flow                  │
│                                                  │
│  main goroutine          server goroutine        │
│  ┌─────────────┐         ┌─────────────────┐     │
│  │ signal.Notify│         │ ListenAndServe  │    │
│  │ (SIGINT,    │         │  :8080          │     │
│  │  SIGTERM)   │         │  ┌────────┐     │     │
│  └──────┬──────┘         │  │handling│     │     │
│         │                │  │requests│     │     │
│  <-quit │                │  └────────┘     │     │
│  ───────┤ Ctrl+C         └────────┬────────┘     │
│         ▼                         │              │
│  server.Shutdown(ctx)             │              │
│    │ stop new connections ────────┤              │
│    │ wait for in-flight ──────────┤              │
│    │ (up to 30s timeout)          │              │
│    ▼                              ▼              │
│  "Server stopped"         returns ErrServerClosed│
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(2 * time.Second) // simulate slow request
        fmt.Fprintln(w, "Hello!")
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // Start server in goroutine
    go func() {
        fmt.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    fmt.Println("\nShutting down gracefully...")

    // Give in-flight requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    fmt.Println("Server stopped")
}
```

---

## TCP/UDP with net Package

**Tutorial: Building a TCP Echo Server and Client**

This example shows low-level networking with Go's `net` package. The server calls `net.Listen("tcp", ":9090")` to bind a port, then loops over `listener.Accept()` to handle incoming connections — each in its own goroutine for concurrency. The client uses `net.Dial` to connect. Notice the `bufio.Scanner` for line-delimited message reading, which is a common pattern in text-based TCP protocols.

```
┌──────────────────────────────────────────────────┐
│          TCP Server-Client Architecture          │
│                                                  │
│  Server                       Client             │
│  ┌──────────────────┐         ┌──────────────┐   │
│  │ net.Listen(":9090")│        │ net.Dial()   │  │
│  └────────┬─────────┘         └──────┬───────┘   │
│           ▼                          │           │
│  ┌──────────────────┐                │           │
│  │ listener.Accept()│◄───────────────┘           │
│  └────────┬─────────┘    TCP handshake           │
│           ▼                                      │
│  ┌──────────────────┐                            │
│  │ go handleConn()  │         "Hello, Server!\n"│
│  │  scanner.Scan()  │◄──────────────────────────│
│  │  fmt.Fprintf()   │──────────────────────────►│
│  │  "Echo: Hello.."│         read response      │
│  └──────────────────┘                            │
│                                                  │
│  Each connection is handled in a separate         │
│  goroutine for concurrent clients                │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "bufio"
    "fmt"
    "net"
)

// TCP Server
func startServer() {
    listener, err := net.Listen("tcp", ":9090")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer listener.Close()
    fmt.Println("TCP server listening on :9090")

    for {
        conn, err := listener.Accept()
        if err != nil {
            continue
        }
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()

    scanner := bufio.NewScanner(conn)
    for scanner.Scan() {
        msg := scanner.Text()
        fmt.Println("Received:", msg)
        fmt.Fprintf(conn, "Echo: %s\n", msg)
    }
}

// TCP Client
func startClient() {
    conn, err := net.Dial("tcp", "localhost:9090")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer conn.Close()

    fmt.Fprintf(conn, "Hello, Server!\n")

    response, _ := bufio.NewReader(conn).ReadString('\n')
    fmt.Println("Response:", response)
}

func main() {
    go startServer()
    // startClient()
    select {} // block forever (for demo)
}
```

---

## net/url Package

**Tutorial: Parsing and Building URLs with net/url**

This example demonstrates Go's `net/url` package for safely parsing URLs into their components (scheme, host, path, query, fragment) and building URLs with properly encoded query parameters. Use `url.Parse` to decompose a URL string, `u.Query()` to access parameters as a map, and `q.Encode()` to safely build query strings. This is essential for constructing API calls without manual string concatenation.

```
┌──────────────────────────────────────────────────────┐
│  URL Structure (parsed by url.Parse)                 │
│                                                      │
│  https://example.com:8080/path/to/page?name=Alice#s1 │
│  ├─────┤ ├──────────────┤├────────────┤├─────────┤├─┤│
│  Scheme      Host           Path        RawQuery Frag│
│                                                      │
│  u.Query() returns url.Values (map[string][]string): │
│  ┌──────────────────────────┐                        │
│  │ "name" ──► ["Alice"]     │                        │
│  │ "age"  ──► ["30"]        │                        │
│  └──────────────────────────┘                        │
│                                                      │
│  Building URL:                                       │
│  q.Set("q", "golang tutorial")                       │
│  base.RawQuery = q.Encode()                          │
│  ──► "q=golang+tutorial&page=1"  (auto-encoded)      │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "net/url"
)

func main() {
    // Parse URL
    u, _ := url.Parse("https://example.com:8080/path/to/page?name=Alice&age=30#section")
    fmt.Println("Scheme:", u.Scheme)   // https
    fmt.Println("Host:", u.Host)       // example.com:8080
    fmt.Println("Path:", u.Path)       // /path/to/page
    fmt.Println("Query:", u.RawQuery)  // name=Alice&age=30
    fmt.Println("Fragment:", u.Fragment) // section

    // Parse query parameters
    params := u.Query()
    fmt.Println("Name:", params.Get("name")) // Alice
    fmt.Println("Age:", params.Get("age"))   // 30

    // Build URL with query parameters
    base, _ := url.Parse("https://api.example.com/search")
    q := base.Query()
    q.Set("q", "golang tutorial")
    q.Set("page", "1")
    base.RawQuery = q.Encode()
    fmt.Println("Built URL:", base.String())
    // https://api.example.com/search?page=1&q=golang+tutorial
}
```

---

## Interview Questions

1. **How do you create an HTTP server in Go?**
   - Use `http.ListenAndServe(":8080", handler)` or create `http.Server{}` for more control (timeouts, TLS). Handlers implement `http.Handler` interface or use `http.HandlerFunc` adapter.

2. **What is the difference between `http.Handle` and `http.HandleFunc`?**
   - `Handle` takes an `http.Handler` interface. `HandleFunc` takes a function `func(w http.ResponseWriter, r *http.Request)`. `HandleFunc` is a convenience wrapper that adapts the function to `Handler`.

3. **What is middleware in Go HTTP?**
   - A function that wraps an `http.Handler` to add cross-cutting concerns: `func(next http.Handler) http.Handler`. Used for logging, auth, CORS, rate limiting. Chainable via composition.

4. **How do you make HTTP requests in Go?**
   - `http.Get(url)`, `http.Post(url, contentType, body)`, or `http.NewRequest` + `client.Do(req)` for full control. Always close `resp.Body`: `defer resp.Body.Close()`.

5. **Why should you not use the default `http.Client` in production?**
   - The default client has no timeout—requests can hang forever. Create a custom client: `&http.Client{Timeout: 10 * time.Second}`. Also configure transport for connection pooling.

6. **How does Go's HTTP/2 support work?**
   - Go supports HTTP/2 automatically for TLS connections in `net/http`. No code changes needed. Use `golang.org/x/net/http2` for h2c (HTTP/2 cleartext) or advanced configuration.

7. **What is `context` used for in HTTP handlers?**
   - `r.Context()` carries request-scoped values, cancellation, and deadlines. When the client disconnects, the context is canceled. Use `r.WithContext(ctx)` to derive new requests with context.

8. **How do you handle graceful shutdown of an HTTP server?**
   - Call `server.Shutdown(ctx)` which stops accepting new connections and waits for active ones to complete. Listen for OS signals (`SIGINT`/`SIGTERM`) to trigger shutdown.

9. **What is `net.Listener` and when would you use it?**
   - `net.Listen("tcp", ":8080")` creates a listener. Useful when you need the actual port (`:0` for random), or to wrap with TLS, or to pass to `server.Serve(listener)`.

10. **How do you implement WebSocket connections in Go?**
    - The standard library doesn't include WebSocket support. Use `gorilla/websocket` or `nhooyr.io/websocket`. Upgrade HTTP connections via `websocket.Upgrade()`, then read/write messages.
