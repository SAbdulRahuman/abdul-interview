# Chapter 28 — Go Toolchain & Build

Go ships with a comprehensive toolchain — formatting, linting, building, testing, and profiling are all included. No external build system (Make, Maven, Gradle) is required.

```
┌──────────────────────────────────────────────────────────┐
│              Go Toolchain Overview                      │
│                                                          │
│  Development:                                            │
│  go fmt        → format code (enforced style)           │
│  go vet        → static analysis (catch bugs)           │
│  go test       → run tests & benchmarks                 │
│  go doc        → view documentation                     │
│  go generate   → run code generators                    │
│                                                          │
│  Build & Run:                                            │
│  go build      → compile to binary                      │
│  go run        → compile + execute (no binary saved)    │
│  go install    → compile + install to $GOBIN             │
│                                                          │
│  Dependencies:                                           │
│  go mod init   → create new module                      │
│  go mod tidy   → sync dependencies                      │
│  go get        → add/update dependency                  │
│  go mod vendor → copy deps into vendor/                 │
│                                                          │
│  Cross-Compilation:                                     │
│  GOOS=linux GOARCH=amd64 go build  → Linux binary      │
│  GOOS=darwin GOARCH=arm64 go build → macOS ARM binary   │
│  GOOS=windows GOARCH=amd64 go build → Windows exe      │
│                                                          │
│  Build Flags:                                            │
│  -o name       → output file name                       │
│  -ldflags      → pass flags to linker                   │
│  -race         → enable race detector                   │
│  -tags         → set build tags                         │
│  CGO_ENABLED=0 → disable C interop (pure Go binary)    │
└──────────────────────────────────────────────────────────┘
```

## go build, go run, go install

```bash
# go build — compile packages and dependencies
go build                    # build current package
go build -o myapp           # specify output binary name
go build ./...              # build all packages recursively

# go run — compile and execute (does not produce binary)
go run main.go              # run a single file
go run .                    # run current package

# go install — compile and install binary to $GOPATH/bin (or $GOBIN)
go install                  # install current package
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

---

## go fmt / gofmt / goimports

```bash
# go fmt — format Go source files (wraps gofmt)
go fmt ./...                # format all files

# gofmt — direct formatter with more options
gofmt -w main.go            # write formatted output back to file
gofmt -d main.go            # show diff instead
gofmt -s main.go            # simplify code

# goimports — gofmt + auto-manage imports (third-party)
goimports -w main.go         # add missing imports, remove unused, format
```

---

## go vet — Static Analysis

```bash
# go vet — reports suspicious constructs
go vet ./...

# Common issues detected:
# - fmt.Printf format string mismatches
# - Unreachable code
# - Copying sync.Mutex values
# - Invalid struct tags
# - Lost cancel function from context
```

**Tutorial: Detecting Bugs with go vet**

`go vet` performs static analysis beyond what the compiler catches. It detects bugs like printf format mismatches, copying sync primitives, unreachable code, and invalid struct tags. This example demonstrates a common `go vet` finding: using the `%s` format verb with an `int` argument, and copying a `sync.Mutex` by value.

```
┌──────────────────────────────────────────┐
│        go vet Static Analysis           │
│                                          │
│  Source Code                             │
│       │                                  │
│       ▼                                  │
│  ┌─────────────────────────────┐         │
│  │  go vet checks:             │         │
│  │  ✗ Printf("%s", int)       │         │
│  │  ✗ mu2 := mu (copies lock) │         │
│  │  ✗ unreachable code        │         │
│  │  ✗ invalid struct tags     │         │
│  └─────────────────────────────┘         │
│       │                                  │
│       ▼                                  │
│  Reports warnings (not compile errors)   │
└──────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    x := 42
    fmt.Printf("%s\n", x) // go vet: arg x for directive %s has wrong type int

    // Copying a Mutex is a bug
    // var mu sync.Mutex
    // mu2 := mu  // go vet: assignment copies lock value
}
```

---

## go generate — Code Generation

**Tutorial: Automating Code Generation with go generate**

`//go:generate` directives embed shell commands in Go source files that run when you execute `go generate`. They're commonly used for creating string methods for enums, generating mocks, or compiling protobuf definitions. The directives are NOT run automatically during `go build` — you must invoke `go generate` explicitly.

```
┌──────────────────────────────────────────┐
│     go generate Workflow                │
│                                          │
│  Source file:                            │
│  //go:generate stringer -type=Color     │
│       │                                  │
│       ▼  (go generate ./...)              │
│  ┌────────────────────────────┐         │
│  │  Runs: stringer -type=Color │         │
│  └────────────┬───────────────┘         │
│               │                          │
│               ▼                          │
│  Generated file: color_string.go        │
│  func (c Color) String() string { ... } │
│                                          │
│  ⚠ Not run by go build automatically    │
└──────────────────────────────────────────┘
```

```go
// main.go
package main

//go:generate stringer -type=Color

type Color int

const (
    Red Color = iota
    Green
    Blue
)

func main() {
    // After running "go generate", a String() method is auto-generated
    // fmt.Println(Red)   // "Red"
    // fmt.Println(Green) // "Green"
}
```

```bash
# Run all //go:generate directives in current package
go generate ./...
```

---

## go doc / godoc

```bash
# go doc — show documentation for a symbol
go doc fmt.Println
go doc net/http.Handler
go doc sync.Mutex

# Show package doc
go doc fmt

# godoc — start a documentation server (install separately)
# go install golang.org/x/tools/cmd/godoc@latest
# godoc -http=:6060
# Then visit http://localhost:6060
```

---

## go list — List Packages

```bash
# List all packages in module
go list ./...

# JSON output with details
go list -json ./...

# List module dependencies
go list -m all

# List specific package info
go list -f '{{.Dir}}' fmt
```

---

## go env — Environment Variables

```bash
# Show all Go environment variables
go env

# Show specific variable
go env GOPATH
go env GOROOT
go env GOOS
go env GOARCH

# Set a variable permanently
go env -w GOPROXY=https://proxy.golang.org,direct
go env -w GOPRIVATE=github.com/mycompany/*
```

---

## golangci-lint — Linter Aggregator

```bash
# Install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run all enabled linters
golangci-lint run

# Run specific linters
golangci-lint run --enable=errcheck,govet,staticcheck

# Run on specific files/directories
golangci-lint run ./pkg/...

# Configuration file: .golangci.yml
```

```yaml
# .golangci.yml example
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - gosimple
    - ineffassign
    - unused
    - misspell

linters-settings:
  govet:
    check-shadowing: true

run:
  timeout: 5m
```

---

## Cross-Compilation

```bash
# Set GOOS and GOARCH to target platform
GOOS=linux   GOARCH=amd64  go build -o myapp-linux
GOOS=darwin  GOARCH=arm64  go build -o myapp-mac
GOOS=windows GOARCH=amd64  go build -o myapp.exe

# Common GOOS values: linux, darwin, windows, freebsd
# Common GOARCH values: amd64, arm64, 386, arm

# List all supported platforms
go tool dist list
```

---

## Build Constraints / Tags

**Tutorial: Platform-Specific Compilation with Build Constraints**

Build constraints (build tags) control which files are included during compilation based on the target OS, architecture, or custom tags. The `//go:build` directive at the top of a file tells the compiler when to include it. Files named with OS/arch suffixes (e.g., `_linux.go`, `_darwin.go`) also use implicit build constraints.

```
┌──────────────────────────────────────────────┐
│      Build Constraints File Selection       │
│                                              │
│  GOOS=linux go build                        │
│       │                                      │
│       ▼                                      │
│  ┌────────────────────┬──────────┐           │
│  │ server_linux.go    │ ✓ BUILD  │           │
│  │ //go:build linux   │          │           │
│  ├────────────────────┼──────────┤           │
│  │ server_darwin.go   │ ✗ SKIP   │           │
│  │ //go:build darwin  │          │           │
│  ├────────────────────┼──────────┤           │
│  │ common.go          │ ✓ BUILD  │           │
│  │ (no constraint)    │          │           │
│  └────────────────────┴──────────┘           │
└──────────────────────────────────────────────┘
```

```go
// file: server_linux.go
//go:build linux

package main

func platformSpecific() string {
    return "Running on Linux"
}
```

**Tutorial: macOS-Specific Build Variant**

This companion file provides the macOS implementation of the same function. When building for `darwin` (macOS), the compiler selects this file instead of `server_linux.go`. Both files define the same function signature, ensuring the package compiles on both platforms with platform-specific behavior.

```
┌──────────────────────────────────────┐
│   Platform-Specific Compilation     │
│                                      │
│  GOOS=darwin go build               │
│       │                              │
│       ▼                              │
│  server_darwin.go ──► compiled       │
│  server_linux.go  ──► skipped        │
│                                      │
│  Both define:                        │
│  func platformSpecific() string      │
│       └──► only ONE is compiled      │
└──────────────────────────────────────┘
```

```go
// file: server_darwin.go
//go:build darwin

package main

func platformSpecific() string {
    return "Running on macOS"
}
```

**Tutorial: Custom Build Tags for Feature Flags**

Beyond OS and architecture, you can define custom build tags for conditional compilation. This is commonly used for separating integration tests from unit tests or for feature flags. Files with `//go:build integration` are only compiled when you explicitly pass `-tags=integration` to `go build` or `go test`.

```
┌──────────────────────────────────────────┐
│        Custom Build Tags                │
│                                          │
│  //go:build integration                  │
│                                          │
│  go test ./...                           │
│  └──► this file SKIPPED                  │
│                                          │
│  go test -tags=integration ./...         │
│  └──► this file INCLUDED                 │
│                                          │
│  Tag combinations:                       │
│  linux && amd64    → both required       │
│  linux || darwin   → either suffices     │
│  !windows          → everything except   │
└──────────────────────────────────────────┘
```

```go
// Custom build tags
//go:build integration

package mypackage

// This file only compiles when: go build -tags=integration
```

```bash
# Build with custom tags
go build -tags=integration
go test -tags=integration ./...

# Combine tags
//go:build linux && amd64
//go:build !windows
//go:build (linux || darwin) && amd64
```

---

## ldflags — Inject Values at Build Time

**Tutorial: Setting Variables at Compile Time with ldflags**

The `-ldflags` flag passes instructions to the Go linker at build time. The `-X` flag sets string variables in your code to specific values — commonly used for injecting version numbers, build timestamps, and git commit hashes. Flags `-s` and `-w` strip debug information to produce smaller binaries.

```
┌──────────────────────────────────────────────┐
│      ldflags Build-Time Injection           │
│                                              │
│  Source code:                                │
│  var version = "dev"                         │
│  var gitCommit = "none"                      │
│                                              │
│  Build command:                              │
│  go build -ldflags "-X main.version=1.2.3   │
│            -X main.gitCommit=abc123"         │
│       │                                      │
│       ▼                                      │
│  Binary contains:                            │
│  version   = "1.2.3"   (replaced)            │
│  gitCommit = "abc123"  (replaced)            │
│                                              │
│  Size flags: -s (no symbols) -w (no DWARF)  │
└──────────────────────────────────────────────┘
```

```go
// main.go
package main

import "fmt"

// These will be set at build time
var (
    version   = "dev"
    buildTime = "unknown"
    gitCommit = "none"
)

func main() {
    fmt.Printf("Version: %s\n", version)
    fmt.Printf("Build Time: %s\n", buildTime)
    fmt.Printf("Git Commit: %s\n", gitCommit)
}
```

```bash
# Inject values using -ldflags
go build -ldflags "\
  -X main.version=1.2.3 \
  -X 'main.buildTime=$(date -u)' \
  -X main.gitCommit=$(git rev-parse --short HEAD)" \
  -o myapp

# Strip debug info for smaller binary
go build -ldflags="-s -w" -o myapp
# -s: omit symbol table
# -w: omit DWARF debug info
```

---

## Static Binary

```bash
# Build fully static binary (no external dependencies)
CGO_ENABLED=0 go build -o myapp

# With ldflags for smaller size
CGO_ENABLED=0 go build -ldflags="-s -w" -o myapp

# Verify it's static
file myapp
# myapp: ELF 64-bit LSB executable, x86-64, statically linked

ldd myapp
# not a dynamic executable
```

---

## go generate — Deep Dive

**Tutorial: Code Generation Patterns and Best Practices**

`go generate` is a tool for automating code transformations. It runs arbitrary commands specified in `//go:generate` directives. It is NOT part of the build pipeline — you must run it manually and commit the generated files. This gives full control and transparency.

```
┌──────────────────────────────────────────────────────────┐
│  go generate Best Practices                             │
│                                                          │
│  1. Always commit generated files to VCS                │
│     → CI/CD doesn't need generator tools installed       │
│     → Code review can verify generated output            │
│                                                          │
│  2. Place //go:generate near the type it generates for  │
│     → Keeps directive close to what it affects            │
│                                                          │
│  3. Add a header comment to generated files              │
│     // Code generated by <tool>; DO NOT EDIT.            │
│     → Go tools recognize this and skip linting           │
│                                                          │
│  4. Use go generate ./... in CI to verify freshness     │
│     → Run generate, then git diff — if anything changed, │
│       generated files are stale → fail CI                │
│                                                          │
│  Common generators:                                     │
│  ┌───────────────────┬──────────────────────────────┐   │
│  │ stringer          │ String() for enum types       │   │
│  │ mockgen           │ Mock interfaces for testing   │   │
│  │ protoc-gen-go     │ Protobuf Go types             │   │
│  │ protoc-gen-go-grpc│ gRPC service stubs            │   │
│  │ sqlc              │ Type-safe SQL                  │   │
│  │ wire              │ Compile-time dependency inject│   │
│  │ enumer            │ Enum utilities (String, parse)│   │
│  │ go-bindata        │ Embed binary data (pre 1.16)  │   │
│  │ oapi-codegen      │ OpenAPI → Go server/client    │   │
│  └───────────────────┴──────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Pattern: stringer — Auto-Generate String() for Enums

```go
package main

//go:generate stringer -type=Status -trimprefix=Status

type Status int

const (
	StatusPending   Status = iota // "Pending"
	StatusActive                  // "Active"
	StatusSuspended               // "Suspended"
	StatusDeleted                 // "Deleted"
)

// After `go generate`:
// produces status_string.go with:
// func (s Status) String() string { ... }
// StatusActive.String() → "Active"
```

```bash
# Install stringer
go install golang.org/x/tools/cmd/stringer@latest

# Run
go generate ./...

# Verify generated file exists
cat status_string.go
```

### Pattern: mockgen — Generate Mock Interfaces for Testing

```go
package service

//go:generate mockgen -source=service.go -destination=mock_service.go -package=service

type UserStore interface {
	GetByID(ctx context.Context, id int) (*User, error)
	Create(ctx context.Context, u *User) error
	Delete(ctx context.Context, id int) error
}

// After `go generate`, mock_service.go contains:
// type MockUserStore struct { ... }
// func (m *MockUserStore) GetByID(ctx context.Context, id int) (*User, error) { ... }
// func (m *MockUserStore) EXPECT() *MockUserStoreMockRecorder { ... }
```

```bash
# Install mockgen
go install go.uber.org/mock/mockgen@latest

# Or generate from interface name (without -source)
#go:generate mockgen -destination=mock_store.go -package=service . UserStore
```

### Pattern: protoc — Generate Protobuf + gRPC Code

```go
package api

//go:generate protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative user.proto

// After `go generate`:
// user.pb.go       — message types
// user_grpc.pb.go  — gRPC client/server stubs
```

### Pattern: Verify Generated Code Is Fresh in CI

```bash
#!/bin/bash
# ci-check-generate.sh — fail if generated files are stale

go generate ./...

if ! git diff --quiet; then
    echo "ERROR: Generated files are out of date!"
    echo "Run 'go generate ./...' and commit the results."
    git diff --stat
    exit 1
fi
```

### Advanced: Custom Code Generator

```go
// gen.go — a simple custom generator
//go:build ignore

// This file is run by go generate, not built normally
package main

import (
	"fmt"
	"os"
	"text/template"
)

var tmpl = `// Code generated by gen.go; DO NOT EDIT.
package {{.Package}}

var Routes = map[string]string{
{{- range $path, $handler := .Routes}}
	"{{$path}}": "{{$handler}}",
{{- end}}
}
`

func main() {
	data := struct {
		Package string
		Routes  map[string]string
	}{
		Package: "main",
		Routes: map[string]string{
			"/api/users":    "handleUsers",
			"/api/products": "handleProducts",
			"/health":       "handleHealth",
		},
	}

	f, _ := os.Create("routes_gen.go")
	defer f.Close()

	t := template.Must(template.New("routes").Parse(tmpl))
	if err := t.Execute(f, data); err != nil {
		fmt.Fprintf(os.Stderr, "generate: %v\n", err)
		os.Exit(1)
	}
}
```

```go
// In your main package:
//go:generate go run gen.go
package main
```

---

## CGO — Deep Dive

**Tutorial: Calling C Code from Go and Its Complexities**

CGO enables Go programs to call C libraries. Import the pseudo-package `"C"` with C declarations in a comment block above the import. CGO introduces significant complexity: slower builds, cross-compilation challenges, no static binaries by default, and potential memory safety issues.

```
┌──────────────────────────────────────────────────────────┐
│  CGO Architecture                                       │
│                                                          │
│  Go code                                                │
│  ┌──────────────────────────────────┐                    │
│  │  // #include <stdio.h>          │ ← C preamble       │
│  │  // #cgo LDFLAGS: -lmylib       │ ← linker flags     │
│  │  import "C"                      │ ← triggers CGO     │
│  │                                  │                    │
│  │  C.puts(C.CString("hello"))     │ ← call C function  │
│  └──────────────────┬───────────────┘                    │
│                     │ CGO bridge                         │
│                     ▼                                    │
│  C code  (compiled by system C compiler: gcc/clang)     │
│  ┌──────────────────────────────────┐                    │
│  │  puts("hello")  ← standard C    │                    │
│  └──────────────────────────────────┘                    │
│                                                          │
│  Build: CGO_ENABLED=1 go build (default on native OS)   │
│                                                          │
│  ⚠ Costs of CGO:                                        │
│  ┌──────────────────────────────────────────────────┐    │
│  │ 1. Build time: ~2-5x slower (invokes C compiler) │    │
│  │ 2. Cross-compile: needs C cross-compiler toolchain│   │
│  │ 3. Binary size: larger, dynamically linked        │    │
│  │ 4. Goroutine overhead: CGO calls pin goroutine    │    │
│  │    to OS thread (like runtime.LockOSThread)       │    │
│  │ 5. Memory: C.CString allocates C memory — YOU     │    │
│  │    must free it with C.free (Go GC won't)         │    │
│  │ 6. No race detector coverage for C code           │    │
│  │ 7. Portability: binary needs matching libc         │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Basic CGO Example

```go
package main

/*
#include <stdlib.h>
#include <string.h>

// Custom C function
int add(int a, int b) {
    return a + b;
}

// String function
const char* greeting(const char* name) {
    static char buf[256];
    snprintf(buf, sizeof(buf), "Hello, %s!", name);
    return buf;
}
*/
import "C"

import (
	"fmt"
	"unsafe"
)

func main() {
	// Call C function
	result := C.add(3, 5)
	fmt.Println("C add(3, 5):", result) // 8

	// String conversion — Go string ↔ C string
	name := C.CString("Go Developer") // allocates C memory!
	defer C.free(unsafe.Pointer(name)) // MUST free C memory manually

	greeting := C.greeting(name)
	goGreeting := C.GoString(greeting) // C string → Go string
	fmt.Println(goGreeting)            // Hello, Go Developer!
}
```

### CGO Memory Rules

```
┌──────────────────────────────────────────────────────────┐
│  CGO Memory Safety Rules                                │
│                                                          │
│  Go memory and C memory are managed separately.         │
│                                                          │
│  ┌───────────────────┬───────────────────────────────┐   │
│  │ Operation         │ Rule                          │   │
│  ├───────────────────┼───────────────────────────────┤   │
│  │ C.CString(s)      │ Allocates C memory.           │   │
│  │                   │ YOU must C.free() it.          │   │
│  │                   │ Go GC will NOT collect it.    │   │
│  ├───────────────────┼───────────────────────────────┤   │
│  │ C.GoString(cs)    │ Copies C string to Go memory. │   │
│  │                   │ Safe — Go GC manages the copy.│   │
│  ├───────────────────┼───────────────────────────────┤   │
│  │ C.GoBytes(p, n)   │ Copies C bytes to Go []byte.  │   │
│  │                   │ Safe — Go GC manages the copy.│   │
│  ├───────────────────┼───────────────────────────────┤   │
│  │ Passing Go ptr    │ Can pass Go pointer to C, but │   │
│  │ to C              │ the Go pointer must NOT       │   │
│  │                   │ contain other Go pointers.    │   │
│  │                   │ C must NOT store the pointer. │   │
│  └───────────────────┴───────────────────────────────┘   │
│                                                          │
│  Common leak:                                           │
│  cs := C.CString("hello")                               │
│  // forgot C.free(unsafe.Pointer(cs)) ← memory leak!   │
│                                                          │
│  Always: defer C.free(unsafe.Pointer(cs))               │
└──────────────────────────────────────────────────────────┘
```

### CGO Build Flags

```go
package main

/*
// Link flags — link against external C libraries
#cgo LDFLAGS: -L/usr/local/lib -lmylib -lm

// Compiler flags — include paths, defines
#cgo CFLAGS: -I/usr/local/include -DDEBUG

// Platform-specific flags
#cgo linux LDFLAGS: -lpthread
#cgo darwin LDFLAGS: -framework CoreFoundation

// pkg-config integration
#cgo pkg-config: sqlite3
*/
import "C"
```

```bash
# CGO environment variables
CGO_ENABLED=1          # enable CGO (default for native builds)
CGO_ENABLED=0          # disable CGO → pure Go, static binary
CC=gcc                 # C compiler to use
CXX=g++                # C++ compiler to use
CGO_CFLAGS="-O2"       # C compiler flags
CGO_LDFLAGS="-L/lib"   # Linker flags

# Cross-compile with CGO (requires cross-compiler)
CC=x86_64-linux-musl-gcc \
CGO_ENABLED=1 \
GOOS=linux GOARCH=amd64 \
go build -o myapp

# Alternatives to CGO:
# • modernc.org/sqlite — pure Go SQLite (no CGO)
# • jackc/pgx           — pure Go PostgreSQL
# • Use CGO_ENABLED=0 and find pure-Go alternatives
```

### When to Avoid CGO

```
┌──────────────────────────────────────────────────────────┐
│  Should You Use CGO?                                    │
│                                                          │
│  ❌ AVOID CGO when:                                      │
│  ├── A pure-Go alternative exists                        │
│  │   • sqlite → modernc.org/sqlite                      │
│  │   • image processing → pure Go libs                  │
│  ├── You need easy cross-compilation                    │
│  ├── You want static binaries (Docker scratch)          │
│  ├── Your team doesn't know C                           │
│  └── Performance difference is negligible               │
│                                                          │
│  ✅ USE CGO when:                                        │
│  ├── Wrapping an existing C library (OpenSSL, FFmpeg)    │
│  ├── Hardware/OS APIs only available in C               │
│  ├── Performance-critical code already in C             │
│  └── No pure-Go alternative exists                      │
│                                                          │
│  "cgo is not Go" — Dave Cheney                          │
│  When you enable CGO, you lose many Go advantages:      │
│  simplicity, portability, fast builds, easy deployment   │
└──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **What are the most important `go` commands?**
   - `go build`, `go run`, `go test`, `go mod tidy`, `go vet`, `go fmt`, `go get`, `go install`, `go generate`, `go tool pprof`. Each serves a specific purpose in the development workflow.

2. **How does cross-compilation work in Go?**
   - Set `GOOS` and `GOARCH` environment variables: `GOOS=linux GOARCH=amd64 go build`. Go compiles natively without external toolchains. Supports ~20 OS/arch combinations.

3. **What is `go vet` and how does it differ from `go build`?**
   - `go vet` performs static analysis to find suspicious constructs (printf format mismatches, unreachable code, mutex copy, etc.). `go build` only checks compilation errors. Use both in CI.

4. **What is `go generate` used for?**
   - Runs commands specified in `//go:generate` comments. Used for code generation: protobuf, mocks, enums, string methods. Run `go generate ./...` before building. Not run automatically.

5. **What are build tags and how do you use them?**
   - `//go:build` constraints control file inclusion. Examples: `//go:build linux`, `//go:build !windows`, `//go:build integration`. Custom tags: `go build -tags=integration`.

6. **How does `go install` differ from `go build`?**
   - `go build` compiles and places binary in current directory. `go install` compiles and places binary in `$GOPATH/bin` (or `$GOBIN`). Since Go 1.17, `go install pkg@version` installs from any module.

7. **What is `ldflags` and how do you use it?**
   - Linker flags passed at build time: `go build -ldflags="-X main.version=1.0.0 -s -w"`. `-X` sets string variables. `-s -w` strips debug info for smaller binaries.

8. **What is `CGO_ENABLED` and when would you disable it?**
   - Controls whether C code can be used. `CGO_ENABLED=0` produces a fully static binary (no libc dependency). Required for scratch Docker images. Some packages (sqlite) need CGO enabled.

9. **What is `go work` (workspaces)?**
   - Go 1.18+ multi-module workspaces. `go.work` file lists modules: `go work init ./mod1 ./mod2`. Allows developing multiple modules together without `replace` directives.

10. **How do you create reproducible builds in Go?**
    - `go.sum` ensures dependency integrity. `go mod vendor` vendors dependencies. Use `-trimpath` to remove local paths. Pin Go version in `go.mod`. Use `govulncheck` for vulnerability scanning.

11. **What are the common `go generate` tools and patterns?**
    - `stringer` (String() for enums), `mockgen` (mock interfaces), `protoc-gen-go` (protobuf types), `sqlc` (type-safe SQL), `wire` (compile-time DI), `oapi-codegen` (OpenAPI). Place `//go:generate` near the type it affects. Always commit generated files. Verify freshness in CI with `go generate && git diff`.

12. **How do you verify generated code is up-to-date in CI?**
    - Run `go generate ./...` in CI, then check `git diff --quiet`. If any files changed, the committed generated code is stale. Fail the build and require the developer to regenerate and commit.

13. **What are the costs of using CGO?**
    - 2-5x slower builds (invokes C compiler). Cross-compilation requires C toolchain. No static binaries by default. CGO calls pin goroutine to OS thread (overhead). C memory (e.g., `C.CString`) must be manually freed — Go GC won't collect it. Race detector doesn't cover C code. Binary is dynamically linked.

14. **Why does Dave Cheney say "cgo is not Go"?**
    - CGO breaks many Go guarantees: easy cross-compilation, static binaries, fast builds, goroutine efficiency, memory safety (C memory leaks), and race detection. When CGO is enabled, you're writing a hybrid C/Go program with the complexity of both.

15. **What is the CGO pointer passing rule?**
    - Go can pass a pointer to C, but that pointer must not contain other Go pointers (no Go pointers of pointers). C must not store the Go pointer beyond the call. Always copy data if C needs to keep a reference. Use `C.CString`/`C.GoString` for string conversion, and always `C.free` C-allocated memory.
