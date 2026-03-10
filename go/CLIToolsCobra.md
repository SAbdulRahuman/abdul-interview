# Chapter 36 — CLI Tools in Go (Cobra)

Go is the dominant language for modern CLI tools — Docker, Kubernetes (`kubectl`), Hugo, GitHub CLI (`gh`), Terraform, and Helm are all built with Go. The **Cobra** library (by spf13, the creator of Hugo and Viper) is the de facto standard for building production-grade CLI applications in Go.

## Why Go for CLI Tools?

```
┌────────────────────────────────────────────────────────────────┐
│  Why Go Dominates CLI Tooling                                  │
│                                                                │
│  1. Single binary deployment — no runtime, no dependencies     │
│  2. Cross-compilation — GOOS/GOARCH builds for any platform    │
│  3. Fast startup — no VM/interpreter warm-up                   │
│  4. Static linking — ship one binary, works everywhere         │
│  5. Concurrency — goroutines for parallel operations           │
│  6. Rich standard library — os, flag, io, net/http built-in    │
│                                                                │
│  Popular Go CLIs:                                              │
│  ┌──────────────┬────────────────────────────────────┐         │
│  │ Tool         │ Library Used                       │         │
│  ├──────────────┼────────────────────────────────────┤         │
│  │ kubectl      │ Cobra                              │         │
│  │ docker       │ Cobra                              │         │
│  │ gh (GitHub)  │ Cobra                              │         │
│  │ hugo         │ Cobra                              │         │
│  │ terraform    │ Custom (mitchellh/cli)              │         │
│  │ helm         │ Cobra                              │         │
│  │ etcdctl      │ Cobra                              │         │
│  └──────────────┴────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────────┘
```

---

## Standard Library: `flag` Package

Before Cobra, understand Go's built-in `flag` package — interviewers may ask you to build a simple CLI without external libraries.

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    // Define flags
    name := flag.String("name", "world", "name to greet")
    count := flag.Int("count", 1, "number of greetings")
    verbose := flag.Bool("verbose", false, "enable verbose output")

    // Custom usage message
    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage: greet [options]\n\nOptions:\n")
        flag.PrintDefaults()
    }

    flag.Parse()

    // Positional arguments (after flags)
    args := flag.Args()

    if *verbose {
        fmt.Printf("Name: %s, Count: %d, Extra args: %v\n", *name, *count, args)
    }

    for i := 0; i < *count; i++ {
        fmt.Printf("Hello, %s!\n", *name)
    }
}
```

```
┌──────────────────────────────────────────────────────────┐
│  flag Package Behavior                                   │
│                                                          │
│  $ greet -name=Alice -count=3                            │
│  Hello, Alice!                                           │
│  Hello, Alice!                                           │
│  Hello, Alice!                                           │
│                                                          │
│  $ greet -help                                           │
│  Usage: greet [options]                                  │
│                                                          │
│  Options:                                                │
│    -count int                                            │
│          number of greetings (default 1)                 │
│    -name string                                          │
│          name to greet (default "world")                 │
│    -verbose                                              │
│          enable verbose output                           │
│                                                          │
│  Limitations of flag:                                    │
│  • No subcommands (flag only handles flat flags)         │
│  • No POSIX-style flags (--long, -s)                     │
│  • No flag grouping (-abc ≠ -a -b -c)                    │
│  • No automatic shell completion                         │
│  • No built-in help formatting                           │
│  → This is why Cobra exists.                             │
└──────────────────────────────────────────────────────────┘
```

---

## Standard Library: `os.Args` (Manual Parsing)

For very simple CLIs or interview problems, direct `os.Args` parsing:

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: calc <add|mul> <a> <b>")
        os.Exit(1)
    }

    cmd := os.Args[1]
    if len(os.Args) < 4 {
        fmt.Fprintln(os.Stderr, "Need two numbers")
        os.Exit(1)
    }

    a, _ := strconv.Atoi(os.Args[2])
    b, _ := strconv.Atoi(os.Args[3])

    switch cmd {
    case "add":
        fmt.Println(a + b)
    case "mul":
        fmt.Println(a * b)
    default:
        fmt.Fprintf(os.Stderr, "Unknown command: %s\n", cmd)
        os.Exit(1)
    }
}
```

```
┌──────────────────────────────────────────────────────────┐
│  os.Args Layout                                          │
│                                                          │
│  $ calc add 3 5                                          │
│                                                          │
│  os.Args[0] = "calc"    ← program name                  │
│  os.Args[1] = "add"     ← subcommand / first argument   │
│  os.Args[2] = "3"       ← argument (always string)      │
│  os.Args[3] = "5"       ← argument (always string)      │
│                                                          │
│  os.Args is []string — everything is a string.           │
│  You must convert types manually.                        │
└──────────────────────────────────────────────────────────┘
```

---

## Cobra — Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Cobra Architecture                                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                      cobra.Command                          │ │
│  │                                                             │ │
│  │  Use:    "app"              ← command name                  │ │
│  │  Short:  "brief desc"      ← one-line description          │ │
│  │  Long:   "detailed desc"   ← full help text                │ │
│  │  Run:    func(cmd, args)   ← execution logic               │ │
│  │  Args:   cobra.ExactArgs() ← argument validation           │ │
│  │                                                             │ │
│  │  Flags (persistent / local)                                 │ │
│  │  ┌───────────────────┬───────────────────────────────────┐  │ │
│  │  │ Persistent Flags  │ Inherited by all subcommands      │  │ │
│  │  │ Local Flags       │ Only for this specific command    │  │ │
│  │  └───────────────────┴───────────────────────────────────┘  │ │
│  │                                                             │ │
│  │  Subcommands (tree structure)                               │ │
│  │  ┌─────────────────────────────────────────────────────┐    │ │
│  │  │  rootCmd                                            │    │ │
│  │  │  ├── getCmd                                         │    │ │
│  │  │  │   ├── getPodsCmd                                 │    │ │
│  │  │  │   └── getServicesCmd                             │    │ │
│  │  │  ├── createCmd                                      │    │ │
│  │  │  │   └── createDeploymentCmd                        │    │ │
│  │  │  └── deleteCmd                                      │    │ │
│  │  └─────────────────────────────────────────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

**Installation:**

```bash
go get github.com/spf13/cobra@latest
```

---

## Tutorial: Building a Complete CLI App with Cobra

### Step 1: Project Structure

```
┌──────────────────────────────────────────────────────────┐
│  Recommended Cobra Project Layout                        │
│                                                          │
│  myapp/                                                  │
│  ├── main.go              ← entry point (minimal)        │
│  ├── cmd/                                                │
│  │   ├── root.go          ← root command + global flags  │
│  │   ├── serve.go         ← "serve" subcommand           │
│  │   ├── migrate.go       ← "migrate" subcommand         │
│  │   └── version.go       ← "version" subcommand         │
│  ├── internal/                                           │
│  │   ├── server/          ← business logic               │
│  │   └── db/              ← database logic               │
│  ├── go.mod                                              │
│  └── go.sum                                              │
│                                                          │
│  Key principle: cmd/ holds CLI wiring only.               │
│  Business logic goes in internal/ or pkg/.               │
└──────────────────────────────────────────────────────────┘
```

### Step 2: Root Command

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var (
    cfgFile string
    verbose bool
)

// rootCmd is the base command — called when no subcommand is given.
var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "MyApp is a sample CLI application",
    Long: `MyApp demonstrates Cobra patterns for building
production-grade CLI tools in Go. It supports serving
HTTP, running database migrations, and more.`,
    // PersistentPreRun runs before any subcommand
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
        if verbose {
            fmt.Println("[verbose mode enabled]")
        }
    },
}

// Execute is called by main.main(). It only needs to happen once.
func Execute() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}

func init() {
    // Persistent flags — available to this command AND all subcommands
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
        "config file (default is $HOME/.myapp.yaml)")
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false,
        "enable verbose output")

    // Disable default completion command if you don't need it
    rootCmd.CompletionOptions.DisableDefaultCmd = true
}
```

### Step 3: Main Entry Point

```go
// main.go
package main

import "myapp/cmd"

func main() {
    cmd.Execute()
}
```

```
┌──────────────────────────────────────────────────────────┐
│  Why main.go is minimal                                  │
│                                                          │
│  main.go only calls cmd.Execute().                       │
│  All command logic lives in cmd/ package.                │
│                                                          │
│  This separation enables:                                │
│  • Testing commands without running main()               │
│  • Clean initialization order via init() functions       │
│  • Easy addition of new subcommands                      │
└──────────────────────────────────────────────────────────┘
```

### Step 4: Subcommands

```go
// cmd/serve.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var (
    port int
    host string
)

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the HTTP server",
    Long:  `Start the HTTP server with configurable host and port.`,
    RunE: func(cmd *cobra.Command, args []string) error {
        // Use RunE (not Run) to return errors — Cobra prints them
        fmt.Printf("Starting server on %s:%d\n", host, port)
        // ... actual server logic would go here
        return nil
    },
}

func init() {
    // Local flags — only for this command
    serveCmd.Flags().IntVarP(&port, "port", "p", 8080, "port to listen on")
    serveCmd.Flags().StringVar(&host, "host", "0.0.0.0", "host to bind to")

    // Mark a flag as required
    serveCmd.MarkFlagRequired("port") // will error if --port not provided

    // Register as subcommand of root
    rootCmd.AddCommand(serveCmd)
}
```

```go
// cmd/version.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

// Injected at build time via ldflags
var (
    Version   = "dev"
    GitCommit = "unknown"
    BuildDate = "unknown"
)

var versionCmd = &cobra.Command{
    Use:   "version",
    Short: "Print version information",
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Printf("myapp %s (commit: %s, built: %s)\n",
            Version, GitCommit, BuildDate)
    },
}

func init() {
    rootCmd.AddCommand(versionCmd)
}
```

```
┌──────────────────────────────────────────────────────────┐
│  Build with version injection:                           │
│                                                          │
│  go build -ldflags "\                                    │
│    -X myapp/cmd.Version=1.2.3 \                         │
│    -X myapp/cmd.GitCommit=$(git rev-parse --short HEAD) \│
│    -X myapp/cmd.BuildDate=$(date -u +%Y-%m-%dT%H:%M:%S) │
│  " -o myapp                                             │
│                                                          │
│  $ ./myapp version                                       │
│  myapp 1.2.3 (commit: a1b2c3d, built: 2026-03-10)       │
└──────────────────────────────────────────────────────────┘
```

### Step 5: Nested Subcommands

```go
// cmd/migrate.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var migrateCmd = &cobra.Command{
    Use:   "migrate",
    Short: "Database migration commands",
    // No Run — this is a "group" command. Running it shows help.
}

var migrateUpCmd = &cobra.Command{
    Use:   "up [steps]",
    Short: "Run pending migrations",
    Args:  cobra.MaximumNArgs(1), // 0 or 1 argument
    RunE: func(cmd *cobra.Command, args []string) error {
        steps := "all"
        if len(args) > 0 {
            steps = args[0]
        }
        fmt.Printf("Running migrations up: %s\n", steps)
        return nil
    },
}

var migrateDownCmd = &cobra.Command{
    Use:   "down [steps]",
    Short: "Rollback migrations",
    Args:  cobra.ExactArgs(1), // exactly 1 argument required
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Rolling back %s migrations\n", args[0])
        return nil
    },
}

var migrateStatusCmd = &cobra.Command{
    Use:   "status",
    Short: "Show migration status",
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Println("Migration status:")
        fmt.Println("  001_create_users    ✓ applied")
        fmt.Println("  002_add_email       ✓ applied")
        fmt.Println("  003_create_orders   ✗ pending")
    },
}

func init() {
    migrateCmd.AddCommand(migrateUpCmd)
    migrateCmd.AddCommand(migrateDownCmd)
    migrateCmd.AddCommand(migrateStatusCmd)
    rootCmd.AddCommand(migrateCmd)
}
```

```
┌──────────────────────────────────────────────────────────┐
│  Command Tree Result                                     │
│                                                          │
│  $ myapp --help                                          │
│  MyApp is a sample CLI application                       │
│                                                          │
│  Usage:                                                  │
│    myapp [command]                                        │
│                                                          │
│  Available Commands:                                     │
│    migrate     Database migration commands                │
│    serve       Start the HTTP server                      │
│    version     Print version information                  │
│                                                          │
│  Flags:                                                  │
│    --config string   config file                          │
│    -h, --help        help for myapp                       │
│    -v, --verbose     enable verbose output                │
│                                                          │
│  $ myapp migrate --help                                   │
│  Database migration commands                              │
│                                                          │
│  Available Commands:                                     │
│    down        Rollback migrations                        │
│    status      Show migration status                      │
│    up          Run pending migrations                     │
└──────────────────────────────────────────────────────────┘
```

---

## Cobra: Argument Validation

```
┌──────────────────────────────────────────────────────────────┐
│  Built-in Argument Validators                                │
│                                                              │
│  cobra.NoArgs          — command accepts no arguments         │
│  cobra.ArbitraryArgs   — any number (default)                │
│  cobra.MinimumNArgs(n) — at least n arguments                │
│  cobra.MaximumNArgs(n) — at most n arguments                 │
│  cobra.ExactArgs(n)    — exactly n arguments                 │
│  cobra.RangeArgs(m, n) — between m and n arguments           │
│                                                              │
│  Custom validator:                                           │
│  cobra.MatchAll(cobra.ExactArgs(1), func(cmd, args) error {  │
│      if !isValidName(args[0]) {                              │
│          return fmt.Errorf("invalid name: %s", args[0])      │
│      }                                                       │
│      return nil                                              │
│  })                                                          │
└──────────────────────────────────────────────────────────────┘
```

```go
var deleteCmd = &cobra.Command{
    Use:   "delete <resource-name>",
    Short: "Delete a resource by name",
    Args: func(cmd *cobra.Command, args []string) error {
        if len(args) != 1 {
            return fmt.Errorf("requires exactly 1 argument, got %d", len(args))
        }
        if len(args[0]) < 3 {
            return fmt.Errorf("resource name must be at least 3 characters")
        }
        return nil
    },
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Deleting resource: %s\n", args[0])
        return nil
    },
}
```

---

## Cobra: Flags Deep Dive

```go
// cmd/deploy.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var deployCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deploy the application",
    RunE: func(cmd *cobra.Command, args []string) error {
        // Get flag values from the command (alternative to package-level vars)
        env, _ := cmd.Flags().GetString("env")
        replicas, _ := cmd.Flags().GetInt("replicas")
        tags, _ := cmd.Flags().GetStringSlice("tag")
        dryRun, _ := cmd.Flags().GetBool("dry-run")

        fmt.Printf("Deploying to %s with %d replicas\n", env, replicas)
        fmt.Printf("Tags: %v, Dry-run: %v\n", tags, dryRun)
        return nil
    },
}

func init() {
    // StringVarP: variable binding, flag name, shorthand, default, description
    deployCmd.Flags().StringP("env", "e", "staging", "target environment")
    deployCmd.Flags().IntP("replicas", "r", 2, "number of replicas")
    deployCmd.Flags().Bool("dry-run", false, "simulate without executing")

    // Slice flags — can be repeated: --tag=v1 --tag=v2
    deployCmd.Flags().StringSlice("tag", nil, "tags for the deployment")

    // Flag groups — mutually exclusive
    deployCmd.Flags().String("docker-image", "", "deploy with Docker")
    deployCmd.Flags().String("binary-path", "", "deploy binary directly")
    deployCmd.MarkFlagsMutuallyExclusive("docker-image", "binary-path")

    // Flag groups — required together
    deployCmd.Flags().String("tls-cert", "", "TLS certificate path")
    deployCmd.Flags().String("tls-key", "", "TLS key path")
    deployCmd.MarkFlagsRequiredTogether("tls-cert", "tls-key")

    rootCmd.AddCommand(deployCmd)
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Flag Types & Patterns                                       │
│                                                              │
│  Type      │ Definition              │ Usage                 │
│  ──────────┼─────────────────────────┼─────────────────────  │
│  String    │ StringP("n", "s", ...)  │ --name=value          │
│  Int       │ IntP("n", "s", ...)     │ --count=5             │
│  Bool      │ BoolP("n", "s", ...)    │ --verbose / -v        │
│  Float64   │ Float64P(...)           │ --rate=0.5            │
│  Duration  │ DurationP(...)          │ --timeout=5s          │
│  StringS.. │ StringSlice(...)        │ --tag=a --tag=b       │
│  Count     │ CountP("v", "v", ...)   │ -vvv → verbose=3     │
│  IP        │ IP(...)                 │ --ip=192.168.1.1      │
│                                                              │
│  Persistent vs Local:                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ rootCmd.PersistentFlags() → inherited by all subs   │    │
│  │ rootCmd.Flags()           → only for this command    │    │
│  │                                                      │    │
│  │ Example:                                             │    │
│  │ root --verbose serve --port=8080                     │    │
│  │      ↑ persistent      ↑ local to serve              │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## Cobra: Run Hooks (Lifecycle)

```
┌──────────────────────────────────────────────────────────────┐
│  Cobra Command Execution Lifecycle                           │
│                                                              │
│  Order of execution when running: myapp serve --port=8080    │
│                                                              │
│  1. PersistentPreRun  (rootCmd)    ← runs first (inherited)  │
│  2. PreRun            (serveCmd)   ← runs before main logic  │
│  3. Run / RunE        (serveCmd)   ← MAIN EXECUTION          │
│  4. PostRun           (serveCmd)   ← runs after main logic   │
│  5. PersistentPostRun (rootCmd)    ← runs last (inherited)   │
│                                                              │
│  PersistentPre/Post runs from the nearest parent that        │
│  defines it — NOT from every ancestor.                       │
│                                                              │
│  RunE vs Run:                                                │
│  • Run:  func(cmd, args)           ← no error return         │
│  • RunE: func(cmd, args) error     ← returns error           │
│  Prefer RunE — Cobra handles error printing + exit code.     │
└──────────────────────────────────────────────────────────────┘
```

```go
var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the server",
    PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
        // Initialize logger, load config, connect to DB
        fmt.Println("→ Loading configuration...")
        return nil
    },
    PreRunE: func(cmd *cobra.Command, args []string) error {
        // Validate that port is within range
        port, _ := cmd.Flags().GetInt("port")
        if port < 1 || port > 65535 {
            return fmt.Errorf("port must be between 1 and 65535, got %d", port)
        }
        return nil
    },
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Println("→ Starting server...")
        return nil
    },
    PostRun: func(cmd *cobra.Command, args []string) {
        fmt.Println("→ Cleanup complete.")
    },
}
```

---

## Cobra + Viper Integration (Configuration)

Viper (also by spf13) handles configuration from multiple sources. Cobra + Viper is the standard combination for configurable CLI tools.

```
┌────────────────────────────────────────────────────────────────┐
│  Viper Configuration Precedence (highest → lowest)            │
│                                                                │
│  1. Command-line flags       (--port=8080)                     │
│  2. Environment variables    (MYAPP_PORT=8080)                 │
│  3. Config file              (.myapp.yaml)                     │
│  4. Key/value store          (etcd, Consul)                    │
│  5. Default values           (flag defaults)                   │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  flags override env vars override config file          │    │
│  │  override defaults                                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                │
│  Supported config formats:                                     │
│  JSON, YAML, TOML, HCL, envfile, Java properties              │
└────────────────────────────────────────────────────────────────┘
```

```go
// cmd/root.go (with Viper)
package cmd

import (
    "fmt"
    "os"
    "strings"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "MyApp with Viper configuration",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}

func init() {
    cobra.OnInitialize(initConfig)

    rootCmd.PersistentFlags().String("config", "", "config file path")
    rootCmd.PersistentFlags().String("db-host", "localhost", "database host")
    rootCmd.PersistentFlags().Int("db-port", 5432, "database port")

    // Bind flags to Viper — allows env vars + config file to set flag values
    viper.BindPFlag("db.host", rootCmd.PersistentFlags().Lookup("db-host"))
    viper.BindPFlag("db.port", rootCmd.PersistentFlags().Lookup("db-port"))
}

func initConfig() {
    cfgFile, _ := rootCmd.Flags().GetString("config")

    if cfgFile != "" {
        viper.SetConfigFile(cfgFile)
    } else {
        home, _ := os.UserHomeDir()
        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.SetConfigName(".myapp")
        viper.SetConfigType("yaml")
    }

    // Environment variables: MYAPP_DB_HOST, MYAPP_DB_PORT
    viper.SetEnvPrefix("MYAPP")
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err == nil {
        fmt.Println("Using config file:", viper.ConfigFileUsed())
    }
}
```

```yaml
# .myapp.yaml
db:
  host: "prod-db.example.com"
  port: 5432
  name: "myapp_prod"

server:
  port: 8080
  timeout: 30s

log:
  level: "info"
  format: "json"
```

```
┌──────────────────────────────────────────────────────────────┐
│  How Viper Resolves "db.host" value:                         │
│                                                              │
│  1. Check if --db-host flag was explicitly set → use it      │
│  2. Check MYAPP_DB_HOST env var → use it                     │
│  3. Check .myapp.yaml → db.host field → use it              │
│  4. Use default: "localhost"                                 │
│                                                              │
│  Access in code:                                             │
│  viper.GetString("db.host")   → "prod-db.example.com"       │
│  viper.GetInt("db.port")      → 5432                         │
│  viper.GetDuration("server.timeout") → 30s                   │
│                                                              │
│  Unmarshal to struct:                                        │
│  type Config struct {                                        │
│      DB     DBConfig     `mapstructure:"db"`                 │
│      Server ServerConfig `mapstructure:"server"`             │
│  }                                                           │
│  var cfg Config                                              │
│  viper.Unmarshal(&cfg)                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## Shell Completion

Cobra has built-in shell completion support. This is a key feature for production CLIs.

```go
// Cobra can auto-generate completion scripts for bash, zsh, fish, PowerShell.
// Built-in: rootCmd already has a "completion" subcommand.

// Custom completions for flag values:
var envCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deploy to environment",
    RunE: func(cmd *cobra.Command, args []string) error {
        env, _ := cmd.Flags().GetString("env")
        fmt.Printf("Deploying to %s\n", env)
        return nil
    },
}

func init() {
    envCmd.Flags().String("env", "", "target environment")
    // Register completion function for --env flag
    envCmd.RegisterFlagCompletionFunc("env",
        func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
            return []string{"development", "staging", "production"},
                cobra.ShellCompDirectiveNoFileComp
        })

    rootCmd.AddCommand(envCmd)
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Shell Completion Directives                                 │
│                                                              │
│  ShellCompDirectiveNoFileComp   — don't suggest files        │
│  ShellCompDirectiveFilterDirs   — only suggest directories   │
│  ShellCompDirectiveDefault      — default completion         │
│  ShellCompDirectiveError        — stop completion            │
│  ShellCompDirectiveNoSpace      — don't add trailing space   │
│                                                              │
│  Dynamic completions (e.g., Kubernetes resource names):      │
│  ValidArgsFunction: func(cmd, args, toComplete) ([]string,  │
│      ShellCompDirective) {                                   │
│      // Query API for resource names                         │
│      names, _ := fetchResourceNames()                        │
│      return names, cobra.ShellCompDirectiveNoFileComp        │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Testing Cobra Commands

```go
package cmd

import (
    "bytes"
    "testing"
)

func executeCommand(root *cobra.Command, args ...string) (string, error) {
    buf := new(bytes.Buffer)
    root.SetOut(buf)       // capture stdout
    root.SetErr(buf)       // capture stderr
    root.SetArgs(args)     // simulate CLI args

    err := root.Execute()
    return buf.String(), err
}

func TestServeCommand(t *testing.T) {
    output, err := executeCommand(rootCmd, "serve", "--port", "9090")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if !strings.Contains(output, "9090") {
        t.Errorf("expected port 9090 in output, got: %s", output)
    }
}

func TestServeInvalidPort(t *testing.T) {
    _, err := executeCommand(rootCmd, "serve", "--port", "99999")
    if err == nil {
        t.Fatal("expected error for invalid port")
    }
}

func TestVersionCommand(t *testing.T) {
    output, err := executeCommand(rootCmd, "version")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if !strings.Contains(output, "myapp") {
        t.Errorf("expected 'myapp' in version output, got: %s", output)
    }
}

func TestMigrateUpNoArgs(t *testing.T) {
    output, err := executeCommand(rootCmd, "migrate", "up")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if !strings.Contains(output, "all") {
        t.Errorf("expected 'all' in output, got: %s", output)
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Testing Strategy for Cobra CLIs                             │
│                                                              │
│  1. Unit test: Test each command handler function directly    │
│  2. Integration test: Use executeCommand() helper above      │
│  3. Golden file test: Compare output against saved .golden   │
│  4. Table-driven: Test flag combinations as test cases       │
│                                                              │
│  Key pattern:                                                │
│  • Create root command fresh per test (avoid state leaks)    │
│  • Use SetOut/SetErr to capture output                       │
│  • Use SetArgs to simulate CLI arguments                     │
│  • Test error cases: missing flags, invalid values           │
└──────────────────────────────────────────────────────────────┘
```

---

## Production Patterns

### Graceful Shutdown

```go
var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start server with graceful shutdown",
    RunE: func(cmd *cobra.Command, args []string) error {
        srv := &http.Server{Addr: ":8080", Handler: setupRouter()}

        // Start server in background
        go func() {
            if err := srv.ListenAndServe(); err != http.ErrServerClosed {
                log.Fatalf("server error: %v", err)
            }
        }()
        fmt.Println("Server started on :8080")

        // Wait for interrupt signal
        quit := make(chan os.Signal, 1)
        signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
        <-quit
        fmt.Println("\nShutting down...")

        // Graceful shutdown with timeout
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        return srv.Shutdown(ctx)
    },
}
```

### Interactive Confirmation

```go
func confirmAction(prompt string) bool {
    fmt.Printf("%s [y/N]: ", prompt)
    var response string
    fmt.Scanln(&response)
    return strings.ToLower(response) == "y"
}

var deleteCmd = &cobra.Command{
    Use:   "delete <name>",
    Short: "Delete a resource",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        force, _ := cmd.Flags().GetBool("force")

        if !force && !confirmAction(fmt.Sprintf("Delete %q?", args[0])) {
            fmt.Println("Aborted.")
            return nil
        }

        fmt.Printf("Deleted: %s\n", args[0])
        return nil
    },
}
```

### Output Formatting (Table, JSON, YAML)

```go
var listCmd = &cobra.Command{
    Use:   "list",
    Short: "List resources",
    RunE: func(cmd *cobra.Command, args []string) error {
        format, _ := cmd.Flags().GetString("output")

        items := []Resource{
            {Name: "web-server", Status: "running", Replicas: 3},
            {Name: "worker", Status: "stopped", Replicas: 0},
        }

        switch format {
        case "json":
            enc := json.NewEncoder(os.Stdout)
            enc.SetIndent("", "  ")
            return enc.Encode(items)
        case "yaml":
            data, err := yaml.Marshal(items)
            if err != nil {
                return err
            }
            fmt.Print(string(data))
        default: // table
            w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
            fmt.Fprintln(w, "NAME\tSTATUS\tREPLICAS")
            for _, item := range items {
                fmt.Fprintf(w, "%s\t%s\t%d\n",
                    item.Name, item.Status, item.Replicas)
            }
            w.Flush()
        }
        return nil
    },
}

func init() {
    listCmd.Flags().StringP("output", "o", "table", "output format (table|json|yaml)")
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Output format pattern (kubectl-style):                      │
│                                                              │
│  $ myapp list                                                │
│  NAME         STATUS    REPLICAS                             │
│  web-server   running   3                                    │
│  worker       stopped   0                                    │
│                                                              │
│  $ myapp list -o json                                        │
│  [                                                           │
│    {"name": "web-server", "status": "running", "replicas":3},│
│    {"name": "worker", "status": "stopped", "replicas": 0}   │
│  ]                                                           │
│                                                              │
│  $ myapp list -o yaml                                        │
│  - name: web-server                                          │
│    status: running                                           │
│    replicas: 3                                               │
│  - name: worker                                              │
│    status: stopped                                           │
│    replicas: 0                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Alternative CLI Libraries

```
┌────────────────────────────────────────────────────────────────────┐
│  Go CLI Library Comparison                                         │
│                                                                    │
│  Library        │ Style           │ Pros              │ Used By    │
│  ───────────────┼─────────────────┼───────────────────┼──────────  │
│  cobra          │ Builder pattern │ Feature-rich,     │ kubectl,   │
│  (spf13/cobra)  │ cmd tree        │ shell completion, │ docker,    │
│                 │                 │ huge ecosystem    │ gh         │
│  ───────────────┼─────────────────┼───────────────────┼──────────  │
│  urfave/cli     │ Structured      │ Simpler API,      │ Some OSS   │
│  (v2)           │ app config      │ less boilerplate  │ tools      │
│  ───────────────┼─────────────────┼───────────────────┼──────────  │
│  kong           │ Struct tags     │ Declarative,      │ Smaller    │
│  (alecthomas)   │                 │ minimal code      │ projects   │
│  ───────────────┼─────────────────┼───────────────────┼──────────  │
│  flag (stdlib)  │ Flat flags      │ No dependencies,  │ Simple     │
│                 │                 │ built-in          │ tools      │
│  ───────────────┼─────────────────┼───────────────────┼──────────  │
│  bubbletea      │ TUI framework   │ Rich terminal UI, │ gh (TUI),  │
│  (charmbracelet)│ (Elm-like)      │ interactive       │ gum        │
│                                                                    │
│  Recommendation for interviews:                                    │
│  • Know flag (stdlib) for simple cases                             │
│  • Know Cobra for real-world projects                              │
│  • Mention urfave/cli or kong to show breadth                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why is Go particularly well-suited for CLI tools?**

Single binary deployment (no runtime dependencies), fast startup (no interpreter warm-up), built-in cross-compilation (`GOOS`/`GOARCH`), static linking, concurrency support for parallel operations, and a rich standard library. Go binaries can be distributed as a single executable that works on any target OS without installing dependencies.

**Q2: What's the difference between `flag` (stdlib) and Cobra?**

`flag` provides flat flag parsing only — no subcommands, no POSIX-style long/short flags, no shell completion, no automatic help generation. Cobra provides a full command tree with nested subcommands, persistent/local flags, argument validation, shell completion, help formatting, pre/post run hooks, and Viper integration for configuration management.

**Q3: Explain `PersistentFlags` vs `Flags` in Cobra.**

`PersistentFlags()` — flags defined on a command that are also available to all its subcommands. Example: `--verbose` on root is available to `serve`, `migrate up`, etc. `Flags()` — local flags only available to the specific command they're defined on. Example: `--port` on `serve` is not available on `migrate`.

**Q4: What's the difference between `Run` and `RunE` in Cobra?**

`Run: func(cmd *cobra.Command, args []string)` — no error return, you must handle errors yourself (e.g., `os.Exit(1)`). `RunE: func(cmd *cobra.Command, args []string) error` — returns an error, and Cobra automatically prints the error and sets a non-zero exit code. **Prefer `RunE`** — it integrates properly with Cobra's error handling and makes testing easier.

**Q5: Describe Cobra's execution lifecycle hooks.**

When a command runs, Cobra executes hooks in this order: (1) `PersistentPreRun` of the nearest parent, (2) `PreRun` of the command, (3) `Run`/`RunE` (main logic), (4) `PostRun`, (5) `PersistentPostRun`. Each has an `E` variant (e.g., `PreRunE`) that returns an error. `PersistentPre/PostRun` is inherited from the closest parent that defines it — not from every ancestor. Common uses: `PersistentPreRun` for config loading, `PreRun` for flag validation, `PostRun` for cleanup.

**Q6: How does Viper's configuration precedence work with Cobra?**

Viper resolves values in priority order (highest to lowest): (1) explicit command-line flags, (2) environment variables, (3) config file values, (4) key/value store (etcd, Consul), (5) default values. You bind Cobra flags to Viper with `viper.BindPFlag()`, set an env prefix with `viper.SetEnvPrefix()`, and read config files with `viper.ReadInConfig()`. This means a flag `--db-host` can be set via `MYAPP_DB_HOST` env var or `db.host` in `.myapp.yaml`, with the flag taking highest priority.

**Q7: How would you test Cobra commands?**

Create a test helper `executeCommand()` that sets `cmd.SetOut()`, `cmd.SetErr()`, and `cmd.SetArgs()`, then calls `cmd.Execute()`. This captures output and simulates CLI arguments without running a real process. Test both success and error cases. For complex CLIs, use table-driven tests for flag combinations. For output formatting, use golden file testing (compare against saved `.golden` files). Avoid testing Cobra framework behavior — focus on your handler logic.

**Q8: How do you inject build-time information (version, commit) into a Go CLI?**

Use `ldflags` during `go build`: `go build -ldflags "-X main.Version=1.0.0 -X main.GitCommit=$(git rev-parse --short HEAD)"`. Declare package-level `var Version = "dev"` — `ldflags -X` replaces the value at link time. This is the standard pattern used by Kubernetes, Docker, and most Go CLI tools.

**Q9: What argument validators does Cobra provide, and when would you write a custom one?**

Built-in: `cobra.NoArgs`, `cobra.ExactArgs(n)`, `cobra.MinimumNArgs(n)`, `cobra.MaximumNArgs(n)`, `cobra.RangeArgs(min, max)`, `cobra.ArbitraryArgs`. Write a custom `Args` function when you need semantic validation — e.g., checking that an argument is a valid resource name, matches a regex, or exists in the system. Custom validators receive `(cmd, args)` and return an error.

**Q10: What is graceful shutdown and how do you implement it in a Cobra `serve` command?**

Graceful shutdown means stopping the server without dropping in-flight requests. Implementation: (1) start `http.Server` in a goroutine, (2) listen for `SIGINT`/`SIGTERM` with `signal.Notify()`, (3) on signal, call `srv.Shutdown(ctx)` with a timeout context. `Shutdown` stops accepting new connections, waits for active requests to complete (up to the timeout), then returns. This is critical for production services behind load balancers — it prevents 5xx errors during deploys.

---

## Interview Problems

**Problem 1: Implement a Simple CLI Calculator**

Design a CLI tool `calc` with subcommands `add`, `sub`, `mul`, `div` that take two number arguments.

```go
// Solution using Cobra
package main

import (
    "fmt"
    "os"
    "strconv"

    "github.com/spf13/cobra"
)

func main() {
    rootCmd := &cobra.Command{
        Use:   "calc",
        Short: "A simple calculator CLI",
    }

    makeCmd := func(name string, op func(a, b float64) float64) *cobra.Command {
        return &cobra.Command{
            Use:   fmt.Sprintf("%s <a> <b>", name),
            Short: fmt.Sprintf("%s two numbers", name),
            Args:  cobra.ExactArgs(2),
            RunE: func(cmd *cobra.Command, args []string) error {
                a, err := strconv.ParseFloat(args[0], 64)
                if err != nil {
                    return fmt.Errorf("invalid number: %s", args[0])
                }
                b, err := strconv.ParseFloat(args[1], 64)
                if err != nil {
                    return fmt.Errorf("invalid number: %s", args[1])
                }
                fmt.Printf("%.2f\n", op(a, b))
                return nil
            },
        }
    }

    rootCmd.AddCommand(
        makeCmd("add", func(a, b float64) float64 { return a + b }),
        makeCmd("sub", func(a, b float64) float64 { return a - b }),
        makeCmd("mul", func(a, b float64) float64 { return a * b }),
        makeCmd("div", func(a, b float64) float64 {
            if b == 0 {
                fmt.Fprintln(os.Stderr, "error: division by zero")
                os.Exit(1)
            }
            return a / b
        }),
    )

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

**Problem 2: Design a CLI for a Task Manager**

Design the command structure (not full implementation) for a `task` CLI tool that manages todos. Show command tree, flag design, and output format handling.

```
// Answer:

Command tree:
  task
  ├── add "Buy groceries" --priority=high --due=2026-03-15
  ├── list --status=pending --output=table|json
  ├── done <task-id>
  ├── delete <task-id> --force
  ├── edit <task-id> --title="..." --priority=...
  └── export --format=csv --file=tasks.csv

Flag design decisions:
  • --output/-o on root as PersistentFlag (all commands support it)
  • --force/-f on delete as local flag (skip confirmation)
  • --priority as StringP with custom completion (low/medium/high)
  • --due as StringP with date validation in PreRunE
  • --status as StringSlice (--status=pending --status=done)

Argument validation:
  • add: MinimumNArgs(1) — title required
  • done/delete/edit: ExactArgs(1) — task ID required
  • list/export: NoArgs — everything via flags

Persistent hooks:
  • PersistentPreRunE: load task store (file/DB)
  • PersistentPostRun: save task store if modified
```

**Problem 3: Implement the `--output` flag pattern (like `kubectl -o json`)**

Write a reusable output formatter that supports `table`, `json`, and `yaml` formats, driven by a flag.

```go
// Solution
type OutputFormat string

const (
    FormatTable OutputFormat = "table"
    FormatJSON  OutputFormat = "json"
    FormatYAML  OutputFormat = "yaml"
)

type TableColumn struct {
    Header string
    Field  func(item interface{}) string
}

func PrintOutput(format OutputFormat, items interface{}, columns []TableColumn, w io.Writer) error {
    switch format {
    case FormatJSON:
        enc := json.NewEncoder(w)
        enc.SetIndent("", "  ")
        return enc.Encode(items)
    case FormatYAML:
        data, err := yaml.Marshal(items)
        if err != nil {
            return err
        }
        _, err = w.Write(data)
        return err
    case FormatTable:
        tw := tabwriter.NewWriter(w, 0, 0, 2, ' ', 0)
        // Print header
        headers := make([]string, len(columns))
        for i, col := range columns {
            headers[i] = col.Header
        }
        fmt.Fprintln(tw, strings.Join(headers, "\t"))
        // Print rows
        v := reflect.ValueOf(items)
        for i := 0; i < v.Len(); i++ {
            vals := make([]string, len(columns))
            for j, col := range columns {
                vals[j] = col.Field(v.Index(i).Interface())
            }
            fmt.Fprintln(tw, strings.Join(vals, "\t"))
        }
        return tw.Flush()
    default:
        return fmt.Errorf("unsupported format: %s", format)
    }
}
```
