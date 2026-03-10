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
