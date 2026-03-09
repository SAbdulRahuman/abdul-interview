# Chapter 1 — Go Basics & Setup

## Go History, Philosophy, and Design Goals

Go (Golang) was created at Google in 2007 by Robert Griesemer, Rob Pike, and Ken Thompson. It was open-sourced in 2009. Go was designed to solve real problems faced at Google: slow C++ builds, complex dependency management, and difficulty writing concurrent software.

**Key Design Principles:**

- **Simplicity** — minimal syntax, easy to read and learn. No classes, no inheritance, no generics (until 1.18), no exceptions.
- **Concurrency** — first-class support via goroutines and channels. Built on Tony Hoare's CSP (Communicating Sequential Processes) model.
- **Fast compilation** — compiles to native machine code in seconds. The entire Go standard library compiles in under 10 seconds.
- **Garbage collected** — automatic memory management, no manual `malloc`/`free`.
- **Statically linked** — produces a single binary with no external dependencies.

```
┌─────────────────────────────────────────────────────────┐
│              Go Compilation Pipeline                    │
│                                                         │
│  .go source ──► Parser ──► AST ──► SSA ──► Machine Code│
│                                                         │
│  main.go  ─────────────────────────────►  ./myapp       │
│  utils.go ─┘          Single Binary        (statically  │
│  go.mod   ─┘          No runtime deps       linked)     │
└─────────────────────────────────────────────────────────┘
```

**How Go differs from other languages:**

```
┌──────────────┬────────────┬────────────┬────────────────┐
│   Feature    │    Go      │   Java     │    Python      │
├──────────────┼────────────┼────────────┼────────────────┤
│ Typing       │ Static     │ Static     │ Dynamic        │
│ Compilation  │ Native bin │ Bytecode   │ Interpreted    │
│ Concurrency  │ Goroutines │ Threads    │ GIL + asyncio  │
│ Memory       │ GC         │ GC         │ GC             │
│ Inheritance  │ None       │ Classes    │ Classes        │
│ Error Model  │ Values     │ Exceptions │ Exceptions     │
│ Deploy       │ Single bin │ JVM + JAR  │ Runtime + deps │
└──────────────┴────────────┴────────────┴────────────────┘
```

**Tutorial: Your First Go Program**

The program below is the simplest valid Go program. Every Go executable starts with `package main` — this tells the compiler to produce a binary. The `import "fmt"` brings in the standard formatting/printing package. The `func main()` is the entry point — when you run the binary, execution begins here.

```
┌──────────────────────────────────────────────────────────┐
│        "Hello, Go!" — Execution Walkthrough              │
│                                                          │
│  Step 1: Compiler reads package main → executable mode   │
│  Step 2: Compiler resolves import "fmt" → stdlib fmt pkg │
│  Step 3: Runtime calls func main()                       │
│  Step 4: fmt.Println writes "Hello, Go!\n" to stdout     │
│  Step 5: main returns → program exits with code 0        │
│                                                          │
│  ┌────────────┐     ┌──────────┐     ┌──────────┐       │
│  │ package    │────►│ import   │────►│ func     │       │
│  │ main       │     │ "fmt"    │     │ main()   │       │
│  └────────────┘     └──────────┘     └────┬─────┘       │
│                                           │              │
│                                      fmt.Println()       │
│                                           │              │
│                                      ┌────▼─────┐       │
│                                      │  stdout   │       │
│                                      │ Hello, Go!│       │
│                                      └──────────┘       │
└──────────────────────────────────────────────────────────┘
```

```go
// The simplest Go program
package main          // Required — declares this as an executable package

import "fmt"          // Import the "format" package for I/O functions

func main() {         // Entry point — called by the Go runtime
    fmt.Println("Hello, Go!")  // Print string + newline to stdout
}
```

---

## Installation, GOPATH, GOROOT, Go Modules

### GOROOT and GOPATH

- **GOROOT** — where Go is installed (e.g., `/usr/local/go`). Contains the standard library and compiler.
- **GOPATH** — your workspace directory (default: `~/go`). Contains `bin/`, `pkg/`, `src/`.
- **GOMODCACHE** — cache for downloaded modules (default: `$GOPATH/pkg/mod`).

```
┌──────────────────────────────────────────────────────────┐
│                    Go Environment                        │
│                                                          │
│  GOROOT (/usr/local/go)         GOPATH (~/go)            │
│  ├── bin/                       ├── bin/                 │
│  │   ├── go         ← compiler │   └── myapp  ← your    │
│  │   └── gofmt      ← tools    │              installed  │
│  ├── src/                       │              binaries   │
│  │   └── fmt/       ← stdlib   ├── pkg/                 │
│  │       net/                   │   └── mod/  ← module   │
│  │       ...                    │       cache (GOMODCACHE)│
│  └── pkg/                       └── src/     ← legacy    │
│      └── tool/                       (pre-modules)       │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Checking Your Go Environment**

After installing Go, use these commands to verify everything is configured correctly. `go version` confirms the compiler is accessible, while `go env` reveals the environment variables that control where Go finds its tools, stdlib, and your workspace.

```
┌──────────────────────────────────────────────────────────┐
│           go env — What Each Variable Means              │
│                                                          │
│  $ go version                                            │
│  go version go1.23.0 linux/amd64                         │
│             ──────── ──────────                          │
│             version  OS/architecture                     │
│                                                          │
│  $ go env GOROOT    → /usr/local/go                      │
│    Where compiler + stdlib live (don't edit)             │
│                                                          │
│  $ go env GOPATH    → /home/user/go                      │
│    Your workspace: bin/ (installed binaries),            │
│    pkg/mod/ (cached modules)                             │
└──────────────────────────────────────────────────────────┘
```

```bash
# Check Go installation
go version

# See environment variables
go env GOROOT
go env GOPATH
```

### Go Modules

Go modules (introduced in Go 1.11) are the standard way to manage dependencies.

**Tutorial: Initializing a Go Module**

Every modern Go project starts with `go mod init`. This creates a `go.mod` file — the project's identity card and dependency manifest. The module path (e.g., `github.com/yourname/myproject`) is how other packages will import your code. After adding imports, `go mod tidy` automatically downloads dependencies and generates `go.sum` (cryptographic checksums to ensure dependency integrity).

```
┌──────────────────────────────────────────────────────────┐
│         Module Initialization Flow                       │
│                                                          │
│  $ mkdir myproject && cd myproject                       │
│                                                          │
│  $ go mod init github.com/yourname/myproject             │
│    │                                                     │
│    └──► creates go.mod:                                  │
│         ┌──────────────────────────────────────┐         │
│         │ module github.com/yourname/myproject │         │
│         │ go 1.23                              │         │
│         └──────────────────────────────────────┘         │
│                                                          │
│  $ go mod tidy                                           │
│    │                                                     │
│    ├──► scans .go files for import statements            │
│    ├──► downloads missing dependencies                   │
│    ├──► removes unused dependencies from go.mod          │
│    └──► creates/updates go.sum (checksums)               │
└──────────────────────────────────────────────────────────┘
```

```bash
# Initialize a new module
mkdir myproject && cd myproject
go mod init github.com/yourname/myproject

# This creates go.mod:
# module github.com/yourname/myproject
# go 1.23

# go.sum — auto-generated file with checksums for dependencies
# go mod tidy — add missing and remove unused dependencies
go mod tidy
```

**Tutorial: The go.mod File Anatomy**

Below is a real `go.mod` file. Line 1 declares the module path — the unique identity used by `import` statements. Line 2 specifies the minimum Go version needed. The `require` block lists direct dependencies with their exact semantic versions. Go uses Minimum Version Selection (MVS) — it always picks the oldest allowed version, not the latest.

```
┌──────────────────────────────────────────────────────────┐
│              go.mod File Breakdown                       │
│                                                          │
│  module github.com/yourname/myproject  ← module identity │
│         └──── matches your repo URL                      │
│                                                          │
│  go 1.23   ← minimum Go version required                │
│                                                          │
│  require (                                               │
│      github.com/gin-gonic/gin v1.9.1   ← dependency     │
│      └─── module path ──────┘ └version┘                 │
│  )                                                       │
│                                                          │
│  go.sum (auto-generated):                                │
│  github.com/gin-gonic/gin v1.9.1 h1:abc123...           │
│  └── SHA-256 hash ensures no tampering                   │
└──────────────────────────────────────────────────────────┘
```

**go.mod example:**
```
module github.com/yourname/myproject

go 1.23

require (
    github.com/gin-gonic/gin v1.9.1
)
```

---

## CLI Tools

**Tutorial: Essential Go Command-Line Tools**

These are the commands you'll use daily. `go build` compiles to a binary without running it — useful for production builds. `go run` combines compile + execute for quick iteration during development. `go fmt` enforces Go's canonical formatting (tabs, spacing) so all Go code looks the same. `go vet` performs static analysis to catch subtle bugs the compiler won't flag (e.g., unreachable code, incorrect format strings).

```
┌──────────────────────────────────────────────────────────┐
│              Go CLI Tools — Decision Tree                │
│                                                          │
│  Writing code?                                           │
│  ├── Quick test  → go run main.go                        │
│  ├── Build binary → go build -o myapp .                  │
│  └── Install globally → go install .  (→ $GOPATH/bin/)   │
│                                                          │
│  Code quality?                                           │
│  ├── Format code → go fmt ./...                          │
│  ├── Static analysis → go vet ./...                      │
│  └── Read docs → go doc fmt.Println                      │
│                                                          │
│  Dependencies?                                           │
│  ├── Add/remove → go mod tidy                            │
│  └── Download → go mod download                          │
│                                                          │
│  Testing?                                                │
│  └── Run tests → go test ./...                           │
└──────────────────────────────────────────────────────────┘
```

```bash
# go build — compile packages and dependencies
go build .                    # compile current package
go build -o myapp .           # compile with custom output name

# go run — compile and run a Go program
go run main.go                # run a single file
go run .                      # run current package

# go install — compile and install the binary to $GOPATH/bin
go install .

# go fmt — format Go source code (enforces standard style)
go fmt ./...                  # format all files in project

# go vet — static analysis, catches common errors
go vet ./...                  # vet all packages

# go doc — display documentation
go doc fmt.Println            # show docs for a specific function
go doc -all fmt               # show all docs for a package
```

---

## Package Structure, main Package, func main()

Every Go file starts with a `package` declaration. The `main` package is special — it defines an executable program. The `func main()` is the entry point.

```
┌──────────────────────────────────────────────────────────┐
│               Go Program Execution Flow                  │
│                                                          │
│    1. Import dependencies (recursively, depth-first)     │
│    2. Initialize package-level variables (each package)  │
│    3. Run init() functions (each package, import order)  │
│    4. Run main() in the main package                     │
│                                                          │
│  pkg "database/sql"     pkg "lib/pq"       pkg "main"   │
│  ┌─────────────┐      ┌─────────────┐    ┌──────────┐   │
│  │ var db *DB   │      │ var driver  │    │ var cfg  │   │
│  │ init()  ─────┤      │ init() ─────┤    │ init()   │   │
│  └──────┬──────┘      └──────┬──────┘    │ main() ◄─┤   │
│         │                    │            └──────────┘   │
│         └────────────────────┘                           │
│     Imported first ──────────────────► Runs last         │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: The main Package Entry Point**

This is the minimal executable Go program. The `package main` declaration combined with `func main()` forms the entry point. When Go compiles a `main` package, it produces an executable binary. Any other package name (e.g., `package mathutils`) produces a library — it cannot be run directly.

```go
// File: main.go
package main          // "main" package = executable program

import "fmt"          // fmt = formatting I/O (Println, Printf, etc.)

func main() {         // Entry point — exactly one per executable
    fmt.Println("This is the entry point")
}
```

**Tutorial: Creating a Library Package**

Library packages use any name other than `main`. The package name matches the directory name by convention. Below, `package mathutils` exports the function `Add` (uppercase first letter = exported/public). Packages in Go enforce visibility through capitalization — uppercase means public, lowercase means private to the package.

```
┌──────────────────────────────────────────────────────────┐
│          Library Package: Export Rules                    │
│                                                          │
│  package mathutils                                       │
│                                                          │
│  func Add(a, b int) int    ← Exported (uppercase A)     │
│  func subtract(a, b int)   ← unexported (lowercase s)   │
│                                                          │
│  From another package:                                   │
│    mathutils.Add(3, 5)     ✓ accessible                  │
│    mathutils.subtract(3,5) ✗ compile error               │
└──────────────────────────────────────────────────────────┘
```

```go
// File: mathutils/math.go — a library package
package mathutils

// Add returns the sum of two integers.
func Add(a, b int) int {
    return a + b
}
```

**Tutorial: Using a Library Package**

This shows how `main.go` imports and uses the `mathutils` library. The import path `github.com/yourname/myproject/mathutils` is resolved from the `go.mod` module path. The compiler verifies that `Add` is exported (uppercase). The `:=` short declaration assigns the return value to `result` with automatic type inference.

```
┌──────────────────────────────────────────────────────────┐
│        Import Resolution Flow                            │
│                                                          │
│  main.go:                                                │
│  import "github.com/yourname/myproject/mathutils"        │
│          └── module path (from go.mod) ──┘└ pkg dir ┘    │
│                                                          │
│  Compiler resolves:                                      │
│  go.mod module = github.com/yourname/myproject           │
│  package dir   = mathutils/                              │
│  function      = Add (exported, uppercase)               │
│                                                          │
│  result := mathutils.Add(3, 5)                           │
│  │         │         │                                   │
│  │         │         └── function name (must be Exported)│
│  │         └── package reference                         │
│  └── short declaration with type inference (int)         │
└──────────────────────────────────────────────────────────┘
```

```go
// File: main.go — using the library
package main

import (
    "fmt"
    "github.com/yourname/myproject/mathutils"
)

func main() {
    result := mathutils.Add(3, 5)
    fmt.Println("3 + 5 =", result) // 3 + 5 = 8
}
```

---

## init() Function

The `init()` function runs **before** `main()`. You can have multiple `init()` functions per file, and they execute in the order they appear. Across packages, they run in dependency order (deepest dependency first).

```
┌──────────────────────────────────────────────────────────┐
│                init() Execution Order                    │
│                                                          │
│  Given: main imports A, A imports B                      │
│                                                          │
│  Step 1:  Package B loaded                               │
│           → B's package-level vars initialized           │
│           → B's init() functions run                     │
│                                                          │
│  Step 2:  Package A loaded                               │
│           → A's package-level vars initialized           │
│           → A's init() functions run                     │
│                                                          │
│  Step 3:  Package main loaded                            │
│           → main's package-level vars initialized        │
│           → main's init() functions run                  │
│           → main() called                                │
│                                                          │
│  Timeline: B.init() ──► A.init() ──► main.init()──► main()│
│            (deepest dependency first)                    │
└──────────────────────────────────────────────────────────┘
```

Within a single file, if multiple `init()` exist, they run top-to-bottom. Within a package with multiple files, files are processed in **lexical (alphabetical) filename order**.

**Tutorial: Multiple init() Functions**

This example demonstrates that a single file can have multiple `init()` functions. They execute in declaration order (top-to-bottom). The package-level variable `config` is initialized first (before any `init()` runs), then each `init()` runs in sequence, and finally `main()` executes. Notice that `init()` has no parameters and no return values — it's purely for side effects.

```
┌──────────────────────────────────────────────────────────┐
│       Execution Timeline for This Example                │
│                                                          │
│  Time ──────────────────────────────────────────►        │
│                                                          │
│  1. var config = ""     ← package var gets zero value    │
│  2. init() #1           ← sets config = "production"    │
│  3. init() #2           ← prints "validating config..." │
│  4. main()              ← prints "running with config"  │
│                                                          │
│  Variable state:                                         │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ config="" │─►│config=       │─►│config=       │       │
│  │ (zero val)│  │"production"  │  │"production"  │       │
│  └──────────┘  └──────────────┘  └──────────────┘       │
│    after var      after init#1      in main()            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

var config string

func init() {
    // First init — runs before main
    config = "production"
    fmt.Println("init 1: config set to", config)
}

func init() {
    // Second init — runs after the first init
    fmt.Println("init 2: validating config...")
}

func main() {
    fmt.Println("main: running with config =", config)
}

// Output:
// init 1: config set to production
// init 2: validating config...
// main: running with config = production
```

**Common use cases for `init()`:**
- Register database drivers
- Set default configuration values
- Validate environment variables

---

## Import Paths, Blank Imports, Dot Imports, Aliased Imports

Go's import system is path-based. Each import path corresponds to a directory containing `.go` files. The compiler resolves imports at compile time — unused imports are a **compile error**.

```
┌──────────────────────────────────────────────────────────┐
│                  Import Styles in Go                     │
│                                                          │
│  import "fmt"               → Standard: fmt.Println()    │
│  import f "fmt"             → Aliased:  f.Println()      │
│  import . "fmt"             → Dot:      Println()        │
│  import _ "github.com/pq"  → Blank:    only runs init() │
│                                                          │
│  ⚠ Dot import pulls all exports into current namespace   │
│    — avoid in production (name collisions, unclear origin)│
│                                                          │
│  ⚠ Unused imports = compile error (use _ or remove them) │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Import Styles in Action**

This example shows all four import styles used together. The standard import `"fmt"` is the most common. Aliased imports (`myfmt`) resolve name conflicts when two packages have the same name. Blank imports (`_ "github.com/lib/pq"`) import only for side effects — the package's `init()` runs but no exported names are available. Dot imports pull everything into the current namespace — avoid in production because it obscures the origin of functions.

```
┌──────────────────────────────────────────────────────────┐
│      What Happens at Compile Time for Each Import        │
│                                                          │
│  "fmt"                 → fmt.Println() available         │
│   │                                                      │
│   │  "math/rand"       → rand.Intn() available           │
│   │   │                                                  │
│   │   │  myfmt "..."   → myfmt.CustomPrint() available   │
│   │   │   │               (original pkg name hidden)     │
│   │   │   │                                              │
│   │   │   │  _ "lib/pq" → init() runs, registers driver  │
│   │   │   │   │            NO exported names available    │
│   │   │   │   │                                          │
│   └───┴───┴───┘                                          │
│       All resolved at compile time.                      │
│       Unused imports = compile error!                    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"                           // Standard import
    "math/rand"                     // Standard library package

    myfmt "github.com/custom/fmt"   // Aliased import — use as myfmt.Something()

    _ "github.com/lib/pq"           // Blank import — only runs init(), registers postgres driver

    // . "fmt"                      // Dot import — imports into current namespace (avoid this!)
    //                              // Would allow: Println("hi") instead of fmt.Println("hi")
)

func main() {
    fmt.Println("Standard import")
    _ = rand.Intn(100)
    myfmt.CustomPrint("Aliased import")
}
```

**Tutorial: Blank Imports for Driver Registration**

The most common use of blank imports is database driver registration. The `lib/pq` package has an `init()` function that calls `sql.Register("postgres", &Driver{})` — registering the PostgreSQL driver with Go's `database/sql` package. Without the blank import, the driver would never be registered and `sql.Open("postgres", ...)` would fail with "unknown driver."

```
┌──────────────────────────────────────────────────────────┐
│        Blank Import — Driver Registration Flow           │
│                                                          │
│  import _ "github.com/lib/pq"                            │
│            │                                             │
│            └──► lib/pq's init() runs automatically:      │
│                 func init() {                            │
│                     sql.Register("postgres", &Driver{})  │
│                 }                                        │
│                          │                               │
│                          ▼                               │
│               ┌──────────────────┐                       │
│               │ sql driver registry │                    │
│               │ "postgres" → pq.Driver │                 │
│               │ "mysql"  → mysql.Driver│                 │
│               └──────────────────┘                       │
│                          │                               │
│                          ▼                               │
│            sql.Open("postgres", connStr)                 │
│            → looks up "postgres" in registry → ✓         │
└──────────────────────────────────────────────────────────┘
```

### Why blank imports?
```go
import (
    "database/sql"
    _ "github.com/lib/pq" // Registers PostgreSQL driver via its init() function
)

func main() {
    db, err := sql.Open("postgres", "connection-string")
    // lib/pq's init() registered the "postgres" driver
    _ = db
    _ = err
}
```

---

## Comments & Documentation

**Tutorial: Comments and Documentation**

Go has two comment styles: `//` for single-line and `/* */` for multi-line. The special convention is that comments immediately preceding a package, function, type, or variable declaration become its documentation — accessible via `go doc` and pkg.go.dev. Package comments should start with `// Package <name>`.

### Comments
```go
// Single-line comment

/*
Multi-line comment
Spans multiple lines
*/

// Package mathutils provides mathematical utility functions.
// This comment appears in documentation generated by godoc.
package mathutils
```

### Go Directives

Go compiler directives are special comments that start with `//go:` (no space after `//`). They instruct the compiler to alter behavior during compilation. Note: there must be **no space** between `//` and `go:`.

```
┌──────────────────────────────────────────────────────────┐
│                Common Go Directives                      │
│                                                          │
│  //go:build     → Build constraints (replaces +build)    │
│  //go:generate  → Run code generators (stringer, etc.)   │
│  //go:embed     → Embed files into binary at compile time│
│  //go:noinline  → Prevent function inlining              │
│  //go:nosplit   → Don't insert stack growth precheck     │
│  //go:linkname  → Link to unexported symbols             │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Go Compiler Directives**

Directives are special comments that start with `//go:` (no space!) and instruct the compiler to change behavior. `//go:build` restricts which OS/architecture a file compiles on. `//go:generate` runs external code generators when you execute `go generate ./...`. `//go:embed` embeds the contents of files directly into the compiled binary at compile time — no file I/O needed at runtime.

```
┌──────────────────────────────────────────────────────────┐
│       How //go:embed Works at Compile Time               │
│                                                          │
│  Source:                   Binary:                       │
│  ┌──────────────┐          ┌──────────────────┐          │
│  │ main.go      │          │ compiled binary   │          │
│  │ config.json  │──embed──►│ ┌──────────────┐ │          │
│  └──────────────┘          │ │ config.json  │ │          │
│                            │ │ (embedded)   │ │          │
│  //go:embed config.json    │ └──────────────┘ │          │
│  var configData string     │ configData = ... │          │
│                            └──────────────────┘          │
│                                                          │
│  At runtime: configData contains file contents           │
│  No need to ship config.json alongside the binary!       │
└──────────────────────────────────────────────────────────┘
```

```go
//go:build linux && amd64
// ^^^ Build constraint — this file only compiles on linux/amd64

//go:generate stringer -type=Color
// ^^^ Code generation — run with `go generate ./...`

//go:embed config.json
var configData string
// ^^^ Embed — embeds file contents into the binary at compile time
```

---

## Workspace Layout Conventions

```
myproject/
├── cmd/                  # Entry points (main packages)
│   ├── server/
│   │   └── main.go       # go run ./cmd/server
│   └── cli/
│       └── main.go       # go run ./cmd/cli
├── internal/             # Private packages (can't be imported externally)
│   ├── auth/
│   │   └── auth.go
│   └── database/
│       └── db.go
├── pkg/                  # Public library code (importable by others)
│   └── models/
│       └── user.go
├── go.mod
├── go.sum
└── README.md
```

**Tutorial: Workspace Layout — Putting It All Together**

This code shows how the three conventional directories work together. The `cmd/server/main.go` is the entry point that imports from both `internal/` (private packages) and `pkg/` (public library). Go enforces the `internal/` restriction at the compiler level — any code outside the module that tries to import from `internal/` will fail to compile.

```
┌──────────────────────────────────────────────────────────┐
│       Import Visibility & Restriction                    │
│                                                          │
│  External packages         Your module                   │
│  ┌─────────────┐           ┌──────────────────────┐      │
│  │ other/repo  │           │ cmd/server/main.go   │      │
│  │             │           │  ├─ import internal ✓│      │
│  │ import pkg/ ✓│          │  └─ import pkg/    ✓│      │
│  │ import      │           ├──────────────────────┤      │
│  │ internal/ ✗ │           │ internal/auth/       │      │
│  │ (compiler   │           │  (only your module   │      │
│  │  blocks it) │           │   can import this)   │      │
│  └─────────────┘           ├──────────────────────┤      │
│                            │ pkg/models/          │      │
│                            │  (anyone can import)  │      │
│                            └──────────────────────┘      │
└──────────────────────────────────────────────────────────┘
```

```go
// cmd/server/main.go
package main

import (
    "fmt"
    "github.com/yourname/myproject/internal/auth"
    "github.com/yourname/myproject/pkg/models"
)

func main() {
    user := models.NewUser("Alice")
    token := auth.GenerateToken(user)
    fmt.Println("Token:", token)
}
```

- **`cmd/`** — each subdirectory is a `main` package that produces an executable
- **`internal/`** — Go enforces that these packages can only be imported by code within the parent of `internal/`
- **`pkg/`** — public library code, safe for external consumption

---

## Interview Questions

1. **What is Go, and what are its key design goals?**
   - Go is a statically typed, compiled language created at Google. Its design goals are simplicity, built-in concurrency, fast compilation, and garbage collection.

2. **What is the difference between `GOPATH` and `GOROOT`?**
   - `GOROOT` is where Go is installed (the standard library). `GOPATH` is the workspace directory for your Go code, dependencies, and binaries. With modules (`go mod`), `GOPATH` is less relevant for dependency management.

3. **What is `go mod init` and what files does it create?**
   - `go mod init <module-path>` initializes a new Go module. It creates `go.mod` (module path + Go version + dependencies) and after `go mod tidy`, a `go.sum` (checksums for dependency integrity).

4. **What is the purpose of the `init()` function in Go?**
   - `init()` runs automatically before `main()`. It is used for initialization tasks like setting up configuration, registering drivers, or validating preconditions. Multiple `init()` functions can exist per file, and they execute in source order.

5. **Can a Go program have multiple `init()` functions? What is their execution order?**
   - Yes. Multiple `init()` functions can exist per file and per package. They execute in the order of dependency imports (depth-first), then lexical file order within a package, then declaration order within a file.

6. **What does a blank import (`_ "pkg"`) do?**
   - It imports a package only for its side effects (i.e., its `init()` function runs), without making the package's exported identifiers available.

7. **Explain the conventional Go project layout (`cmd/`, `internal/`, `pkg/`).**
   - `cmd/` contains entry points — each subdirectory is a `main` package producing an executable. `internal/` restricts imports to the parent module (enforced by Go). `pkg/` holds public library code reusable by external projects.

8. **What is the difference between `go build`, `go run`, and `go install`?**
   - `go build` compiles packages into a binary. `go run` compiles and executes without saving a binary. `go install` compiles and places the binary in `$GOPATH/bin` (or `$GOBIN`).

9. **What are `//go:` directives? Name a few.**
   - Compiler directives: `//go:build` (build constraints), `//go:generate` (code generation), `//go:embed` (embed files into binary), `//go:noinline`, `//go:nosplit`.

10. **How does Go's package system differ from other languages like Java or Python?**
    - Go uses directory-based packages (one package per directory). There's no class system — packages are the primary unit of encapsulation. Exported identifiers start with uppercase. There's no circular import allowed.
