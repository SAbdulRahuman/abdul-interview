# Chapter 18 — Packages & Modules

Go modules (introduced in Go 1.11, default since 1.16) are the official dependency management system. A module is defined by `go.mod` at the repository root and uses semantic versioning.

```
┌──────────────────────────────────────────────────────────┐
│              Module Dependency Resolution               │
│                                                          │
│  go.mod (your module)                                    │
│  ┌──────────────────────────────┐                        │
│  │ module github.com/you/app   │                        │
│  │ go 1.23                     │                        │
│  │ require (                    │                        │
│  │   gin v1.9.1                │                        │
│  │   sync v0.5.0               │                        │
│  │ )                           │                        │
│  └─────────┬───────────────────┘                        │
│            │ go mod tidy                                │
│            ▼                                             │
│  go.sum (checksums)    $GOMODCACHE (downloaded modules)  │
│  ┌───────────────┐     ┌─────────────────────┐           │
│  │ gin v1.9.1 h1:│     │ pkg/mod/            │           │
│  │ sha256=xxxxx  │     │   gin@v1.9.1/       │           │
│  │ sync v0.5.0   │     │   sync@v0.5.0/      │           │
│  │ sha256=yyyyy  │     │   net@v0.17.0/      │           │
│  └───────────────┘     └─────────────────────┘           │
│                                                          │
│  MVS (Minimum Version Selection):                       │
│  Go picks the MINIMUM version that satisfies all        │
│  constraints — NOT the latest. This ensures builds      │
│  are reproducible.                                      │
└──────────────────────────────────────────────────────────┘
```

## Modules: go.mod and go.sum

```
# go.mod — defines the module and its dependencies
module github.com/yourname/myproject

go 1.23

require (
    github.com/gin-gonic/gin v1.9.1
    golang.org/x/sync v0.5.0
)

require (
    // Indirect dependencies (auto-managed)
    golang.org/x/net v0.17.0 // indirect
)
```

```bash
# Initialize a new module
go mod init github.com/yourname/myproject

# Add missing / remove unused dependencies
go mod tidy

# Download dependencies to local cache
go mod download

# Copy dependencies into vendor/ directory
go mod vendor

# Show module dependency graph
go mod graph
```

---

## Semantic Versioning

```bash
# Format: vMAJOR.MINOR.PATCH
# v1.2.3 → Major=1, Minor=2, Patch=3

# Major version: breaking changes
# Minor version: new features, backward compatible
# Patch version: bug fixes

# Add a dependency
go get github.com/gin-gonic/gin@v1.9.1

# Update to latest minor/patch
go get -u github.com/gin-gonic/gin

# Update all dependencies
go get -u ./...

# Major version 2+ requires path suffix
go get github.com/user/pkg/v2
```

```go
// Importing v2 of a module
import "github.com/user/pkg/v2"
```

---

## Package Naming Conventions

```go
// ✅ Good package names — short, lowercase, singular
package user        // not "users"
package http        // not "httpHelper"
package auth        // not "authentication"
package db          // short and clear

// ❌ Bad package names
// package httpUtils     // no camelCase
// package string_utils  // no underscores
// package mypackage     // not meaningful
// package common        // too vague

// Package name becomes the qualifier:
// user.New()    ← reads naturally
// http.Get()    ← reads naturally
```

---

## Internal Packages

```
myproject/
├── cmd/server/main.go
├── internal/
│   ├── auth/          ← can only be imported by myproject/...
│   │   └── auth.go
│   └── database/
│       └── db.go
├── pkg/
│   └── models/        ← can be imported by anyone
│       └── user.go
└── go.mod
```

```go
// cmd/server/main.go — CAN import internal
package main

import (
    "github.com/yourname/myproject/internal/auth"
    "github.com/yourname/myproject/pkg/models"
)

// Another project cannot import:
// import "github.com/yourname/myproject/internal/auth"
// ^^^ COMPILE ERROR: use of internal package not allowed
```

---

## Exported vs Unexported Identifiers

```go
package user

import "fmt"

// Exported (uppercase) — accessible from other packages
type User struct {
    Name  string    // exported field
    Email string    // exported field
    age   int       // unexported — only accessible within 'user' package
}

// Exported function
func NewUser(name, email string, age int) *User {
    return &User{Name: name, Email: email, age: age}
}

// Exported method
func (u *User) String() string {
    return fmt.Sprintf("%s <%s>", u.Name, u.Email)
}

// Unexported — helper function, not visible to other packages
func validateEmail(email string) bool {
    return len(email) > 0
}
```

---

## Workspaces (Go 1.18+)

Workspaces let you work with multiple modules simultaneously.

```bash
# Create a workspace
go work init ./module-a ./module-b

# Add a module to the workspace
go work use ./module-c
```

```
# go.work file
go 1.23

use (
    ./module-a
    ./module-b
    ./module-c
)
```

```
myworkspace/
├── go.work
├── module-a/
│   ├── go.mod
│   └── main.go
├── module-b/
│   ├── go.mod
│   └── lib.go
```

---

## Build Tags / Build Constraints

```go
//go:build linux && amd64

package platform

// This file only compiles on linux/amd64
func GetPlatform() string {
    return "linux-amd64"
}
```

```go
//go:build windows

package platform

func GetPlatform() string {
    return "windows"
}
```

```go
//go:build !production

package config

// This file is excluded when building with -tags=production
func getDefaultPort() int {
    return 8080 // development default
}
```

```bash
# Build with tags
go build -tags=production .
```

---

## Module Proxies

```bash
# GOPROXY — where Go fetches modules from
# Default: https://proxy.golang.org,direct
go env GOPROXY

# GOPRIVATE — modules that should bypass the proxy
go env -w GOPRIVATE="github.com/mycompany/*"

# GONOSUMDB — modules that skip checksum verification
go env -w GONOSUMDB="github.com/mycompany/*"
```

---

## Vendor Directory

```bash
# Create vendor directory with all dependencies
go mod vendor

# Build using vendor instead of module cache
go build -mod=vendor .

# Useful for:
# - Reproducible builds
# - Air-gapped environments
# - Dependency auditing
```

---

## Interview Questions

1. **What is a Go module?**
   - A module is a collection of packages versioned together, defined by `go.mod`. It specifies the module path, Go version, and dependencies. Introduced in Go 1.11; default since Go 1.16.

2. **What is the difference between `go.mod` and `go.sum`?**
   - `go.mod` declares the module path, Go version, and direct/indirect dependencies. `go.sum` contains cryptographic hashes of all dependency versions for integrity verification. Both should be committed to version control.

3. **What is the difference between a package and a module?**
   - A package is a directory of `.go` files sharing the same `package` declaration—Go's unit of compilation. A module is a collection of packages released together with a `go.mod` file—Go's unit of versioning.

4. **What does `go mod tidy` do?**
   - Adds missing dependencies and removes unused ones from `go.mod` and `go.sum`. It ensures the dependency graph is clean and complete.

5. **How does Go handle package visibility/access control?**
   - Exported identifiers start with an uppercase letter and are accessible outside the package. Unexported (lowercase) identifiers are package-private. The `internal` directory restricts imports to the parent tree.

6. **What is semantic import versioning?**
   - Major versions v2+ must include the version in the import path: `github.com/foo/bar/v2`. This allows different major versions to coexist. v0/v1 have no suffix.

7. **What are build tags/constraints?**
   - `//go:build` directives control which files are included in a build. Used for OS/arch-specific code, feature flags, or integration tests. Example: `//go:build linux && amd64`.

8. **What is the `init()` function?**
   - A special function called automatically before `main()`. Each package can have multiple `init()` functions. Executed in dependency order. Used for initialization, but overuse is discouraged.

9. **What is a `replace` directive in `go.mod`?**
   - Replaces a module dependency with another version or local path. Useful for local development, forks, or workarounds. Example: `replace github.com/old/pkg => ../local/pkg`.

10. **How does Go resolve diamond dependency conflicts?**
    - Go uses Minimum Version Selection (MVS): it picks the minimum version that satisfies all requirements from all dependents. This is deterministic and avoids "newest version" surprises.
