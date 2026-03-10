# Chapter 17 — Testing

Go has built-in testing support — no external frameworks needed. Test files end with `_test.go` and use the `testing` package. Go supports unit tests, benchmarks, fuzzing, and examples out of the box.

```
┌──────────────────────────────────────────────────────────┐
│               Go Testing Overview                       │
│                                                          │
│  File naming:  foo_test.go   (next to foo.go)           │
│                                                          │
│  Test Types:                                            │
│  ┌────────────────┬──────────────────┬──────────┐        │
│  │ Type           │ Signature        │ Run with │        │
│  ├────────────────┼──────────────────┼──────────┤        │
│  │ Unit test      │ TestXxx(t *T)    │ go test  │        │
│  │ Benchmark      │ BenchmarkXxx(b *B)│ -bench= │       │
│  │ Fuzz test      │ FuzzXxx(f *F)    │ -fuzz=   │        │
│  │ Example        │ ExampleXxx()     │ go test  │        │
│  └────────────────┴──────────────────┴──────────┘        │
│                                                          │
│  Testing Methods:                                       │
│  t.Error()   → log error, continue running              │
│  t.Fatal()   → log error, STOP this test immediately    │
│  t.Skip()    → skip this test                           │
│  t.Parallel()→ run test in parallel with others         │
│  t.Run()     → create subtest                           │
│                                                          │
│  Common commands:                                       │
│  go test                 → test current package         │
│  go test ./...           → test all packages            │
│  go test -v              → verbose output               │
│  go test -run TestName   → specific test                │
│  go test -count=1        → disable test caching         │
│  go test -cover          → show coverage %              │
│  go test -race           → enable race detector         │
└──────────────────────────────────────────────────────────┘
```

## Test Functions Basics

**Tutorial: Defining Functions to Be Tested**

Before writing tests, you define the production code in a regular `.go` file. Each function you want to test should have clear inputs and outputs. Here we define `Add` and `Divide`—simple functions that will be tested in a companion `_test.go` file. Notice `Divide` returns an error for invalid input, which is idiomatic Go.

```
┌─────────────────────────────────────────┐
│       Source + Test File Convention       │
│                                           │
│  math.go          math_test.go            │
│  ┌──────────┐     ┌───────────────┐       │
│  │ func Add  │◄────│ func TestAdd  │       │
│  │ func Div  │◄────│ func TestDiv  │       │
│  └──────────┘     └───────────────┘       │
│                                           │
│  Same package: package math               │
│  Test discovers functions automatically   │
└─────────────────────────────────────────┘
```

```go
// File: math.go
package math

func Add(a, b int) int {
    return a + b
}

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}
```

**Tutorial: Writing Your First Test Function**

Test files must end with `_test.go` and live in the same package. Test functions must start with `Test` followed by an uppercase letter, and accept `*testing.T`. Use `t.Errorf` to report failures while continuing the test, or `t.Fatal` to stop immediately. Go discovers and runs all `Test*` functions automatically—no registration needed.

```
┌──────────────────────────────────────────────┐
│       t.Error vs t.Fatal Flow                │
│                                              │
│  TestDivide(t *testing.T)                    │
│       │                                      │
│       ├── err != nil?                        │
│       │     YES ──► t.Fatal("...")            │
│       │              └── STOP test ■          │
│       │     NO  ──► continue                 │
│       │                                      │
│       ├── result != 5.0?                     │
│       │     YES ──► t.Errorf("...")           │
│       │              └── mark FAIL, continue │
│       │     NO  ──► continue                 │
│       │                                      │
│       └── test ends ──► report PASS/FAIL     │
└──────────────────────────────────────────────┘
```

```go
// File: math_test.go — test files end with _test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

// t.Error vs t.Fatal:
// t.Error/t.Errorf — logs error but continues test
// t.Fatal/t.Fatalf — logs error and STOPS test immediately
func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatal("unexpected error:", err) // stops here if error
    }
    if result != 5.0 {
        t.Errorf("Divide(10, 2) = %f; want 5.0", result)
    }
}
```

Run tests:
```bash
go test                # test current package
go test ./...          # test all packages
go test -v             # verbose output
go test -run TestAdd   # run specific test
```

---

## Table-Driven Tests

**Tutorial: The Idiomatic Go Testing Pattern**

Table-driven tests define test cases as a slice of structs, then iterate with `t.Run` to create named subtests. This is the most common testing pattern in Go—it eliminates code duplication, makes adding cases trivial, and gives each case a descriptive name in test output. The `t.Run` subtests can be individually selected with `go test -run TestName/subtest_name`.

```
┌───────────────────────────────────────────────┐
│       Table-Driven Test Flow                  │
│                                               │
│  tests := []struct{ name, a, b, want }        │
│  ┌──────┬───┬───┬──────┐                      │
│  │ name │ a │ b │ want │                      │
│  ├──────┼───┼───┼──────┤                      │
│  │ pos  │ 2 │ 3 │  5   │ ──► t.Run("pos")    │
│  │ neg  │-2 │-3 │ -5   │ ──► t.Run("neg")    │
│  │ mix  │-2 │ 3 │  1   │ ──► t.Run("mix")    │
│  │ zero │ 0 │ 0 │  0   │ ──► t.Run("zero")   │
│  └──────┴───┴───┴──────┘                      │
│                     │                         │
│                     ▼                         │
│  Each subtest: result := Add(a, b)            │
│                if result != want → t.Errorf   │
└───────────────────────────────────────────────┘
```

```go
// File: calculator_test.go
package math

import "testing"

func TestAdd_TableDriven(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed", -2, 3, 1},
        {"zeros", 0, 0, 0},
        {"large numbers", 1000000, 2000000, 3000000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}

func TestDivide_TableDriven(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        expected  float64
        expectErr bool
    }{
        {"normal division", 10, 2, 5.0, false},
        {"decimal result", 7, 2, 3.5, false},
        {"divide by zero", 10, 0, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Divide(tt.a, tt.b)
            if tt.expectErr {
                if err == nil {
                    t.Error("expected error but got nil")
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if result != tt.expected {
                t.Errorf("Divide(%.1f, %.1f) = %.1f; want %.1f",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

---

## t.Parallel(), t.Helper(), t.Cleanup()

**Tutorial: Advanced Test Control Helpers**

`t.Parallel()` marks a subtest to run concurrently with other parallel tests—great for independent, slow tests. `t.Helper()` marks a function as a test helper so error messages point to the caller, not the helper. `t.Cleanup()` registers a teardown function that runs after the test completes (like `defer` but for test lifecycle). `t.TempDir()` creates and auto-cleans a temporary directory.

```
┌─────────────────────────────────────────────────┐
│       Test Lifecycle & Helpers                  │
│                                                 │
│  TestParallel                                   │
│  ├── t.Run("case1", func(t) {                   │
│  │       t.Parallel() ──► runs concurrently ─┐  │
│  │   })                                      │  │
│  ├── t.Run("case2", func(t) {                │  │
│  │       t.Parallel() ──► runs concurrently ─┤  │
│  │   })                                      │  │
│  └── waits for all parallel subtests ◄───────┘  │
│                                                 │
│  t.Helper() effect:                             │
│  ┌────────────────────────────┐                 │
│  │ WITHOUT: error at helper.go:5  ✗            │
│  │ WITH:    error at test.go:12   ✓            │
│  └────────────────────────────┘                 │
│                                                 │
│  t.Cleanup() order: LIFO (like defer)           │
│  t.TempDir() ──► auto-removed after test        │
└─────────────────────────────────────────────────┘
```

```go
package math

import (
    "os"
    "testing"
)

func TestParallel(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"case1", 1, 2, 3},
        {"case2", 3, 4, 7},
        {"case3", 5, 6, 11},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // runs this subtest in parallel with others
            result := Add(tt.a, tt.b)
            if result != tt.want {
                t.Errorf("got %d, want %d", result, tt.want)
            }
        })
    }
}

// t.Helper marks a function as a test helper
// Error messages show the caller's line, not the helper's line
func assertEqual(t *testing.T, got, want int) {
    t.Helper() // without this, error points to this function instead of caller
    if got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}

func TestWithHelper(t *testing.T) {
    assertEqual(t, Add(2, 3), 5) // error shows THIS line, not assertEqual's line
}

// t.Cleanup — register cleanup function (runs after test completes)
func TestWithCleanup(t *testing.T) {
    // Create temp file
    f, _ := os.CreateTemp("", "test-*")
    t.Cleanup(func() {
        os.Remove(f.Name()) // automatically cleaned up
    })

    // Use temp file in test...
    f.WriteString("test data")
}

// t.TempDir — creates temp directory, auto-cleaned
func TestWithTempDir(t *testing.T) {
    dir := t.TempDir() // automatically removed after test
    _ = dir
    // Use dir for test files...
}

// t.Skip — skip test conditionally
func TestSkipExample(t *testing.T) {
    if os.Getenv("INTEGRATION") == "" {
        t.Skip("Skipping integration test; set INTEGRATION=1 to run")
    }
    // Integration test code...
}
```

---

## Benchmarks

**Tutorial: Measuring Performance with Go Benchmarks**

Benchmark functions start with `Benchmark` and accept `*testing.B`. The key is the `b.N` loop—the framework automatically adjusts `b.N` to run enough iterations for statistically reliable timing. Use `b.Run` for sub-benchmarks, `b.ResetTimer()` to exclude setup time, and `-benchmem` flag to track allocations. Results show ns/op, bytes/op, and allocs/op.

```
┌───────────────────────────────────────────────┐
│       Benchmark Execution Flow                │
│                                               │
│  go test -bench=BenchmarkAdd                  │
│       │                                       │
│       ▼                                       │
│  Framework calibrates b.N:                    │
│  ┌─────────────────────────────────┐          │
│  │ Run 1:   b.N = 1        → fast │          │
│  │ Run 2:   b.N = 100      → fast │          │
│  │ Run 3:   b.N = 10000    → fast │          │
│  │ Run 4:   b.N = 1000000  → ~1s  │ ◄─ done │
│  └─────────────────────────────────┘          │
│       │                                       │
│       ▼                                       │
│  Output: BenchmarkAdd-8  1000000  2.5 ns/op   │
│                                               │
│  b.ResetTimer() ──► excludes setup from time  │
└───────────────────────────────────────────────┘
```

```go
package math

import "testing"

func BenchmarkAdd(b *testing.B) {
    // b.N is set by the framework — runs enough iterations for accuracy
    for i := 0; i < b.N; i++ {
        Add(42, 58)
    }
}

// Sub-benchmarks
func BenchmarkDivide(b *testing.B) {
    b.Run("small", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Divide(10, 3)
        }
    })
    b.Run("large", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Divide(1e18, 7)
        }
    })
}

// ResetTimer — exclude setup time from benchmark
func BenchmarkWithSetup(b *testing.B) {
    // Expensive setup
    data := make([]int, 10000)
    for i := range data {
        data[i] = i
    }

    b.ResetTimer() // exclude setup from timing
    for i := 0; i < b.N; i++ {
        _ = data[len(data)/2]
    }
}
```

Run benchmarks:
```bash
go test -bench=.                    # run all benchmarks
go test -bench=BenchmarkAdd         # run specific
go test -bench=. -benchmem          # show memory allocations
go test -bench=. -count=5           # run 5 times for statistics
```

---

## Test Coverage and Fuzzing

**Tutorial: Measuring Code Coverage**

Go has built-in coverage analysis—no external tools needed. The `-cover` flag shows a coverage percentage, `-coverprofile` generates a detailed report, and `go tool cover -html` renders an interactive HTML view highlighting covered (green) and uncovered (red) lines. Aim for meaningful coverage of critical paths rather than chasing 100%.

```
┌────────────────────────────────────────────┐
│       Coverage Workflow                    │
│                                            │
│  go test -cover                            │
│       │                                    │
│       ▼                                    │
│  "coverage: 85.7% of statements"           │
│                                            │
│  go test -coverprofile=cover.out           │
│       │                                    │
│       ▼                                    │
│  go tool cover -html=cover.out             │
│       │                                    │
│       ▼                                    │
│  ┌─── Browser View ─────────────────┐        │
│  │  func Add(a, b int) int {      │ green  │
│  │      return a + b              │ green  │
│  │  }                             │        │
│  │  func Unused() {               │ red    │
│  │      // not tested             │ red    │
│  │  }                             │        │
│  └────────────────────────────────┘        │
└────────────────────────────────────────────┘
```

```bash
# Coverage
go test -cover                            # show coverage percentage
go test -coverprofile=cover.out           # generate profile
go tool cover -html=cover.out             # view in browser
go tool cover -func=cover.out             # function-level coverage
```

**Tutorial: Property-Based Fuzz Testing**

Fuzz testing (Go 1.18+) automatically generates random inputs to find edge cases you wouldn't think to test manually. You provide seed values via `f.Add()`, then define a fuzz function that asserts properties (invariants) rather than exact values. The fuzzer mutates inputs, tracking code coverage to explore new paths. It's excellent for finding panics, off-by-one errors, and boundary conditions.

```
┌────────────────────────────────────────────────┐
│       Fuzz Testing Flow                        │
│                                                │
│  f.Add(1, 2)      ──► Seed corpus             │
│  f.Add(0, 0)          (known test cases)       │
│  f.Add(-1, 1)                                  │
│       │                                        │
│       ▼                                        │
│  Fuzz engine generates random (a, b):          │
│  ┌────────┬────────┬──────────────────┐        │
│  │ a      │ b      │ Check property   │        │
│  ├────────┼────────┼──────────────────┤        │
│  │ 39182  │ -7721  │ Add(a,b)==a+b? ✓ │        │
│  │ 0      │ MaxInt │ Add(a,b)==a+b? ✓ │        │
│  │ ...    │ ...    │ keeps mutating.. │        │
│  └────────┴────────┴──────────────────┘        │
│       │                                        │
│       ▼                                        │
│  Failure? → saved to testdata/fuzz/            │
└────────────────────────────────────────────────┘
```

```go
// Fuzzing (Go 1.18+)
package math

import "testing"

func FuzzAdd(f *testing.F) {
    // Seed corpus — initial test cases
    f.Add(1, 2)
    f.Add(0, 0)
    f.Add(-1, 1)
    f.Add(100, -100)

    // Fuzz function — receives random inputs
    f.Fuzz(func(t *testing.T, a, b int) {
        result := Add(a, b)
        // Property-based testing
        if result != a+b {
            t.Errorf("Add(%d, %d) = %d; want %d", a, b, result, a+b)
        }
    })
}
```

```bash
go test -fuzz=FuzzAdd -fuzztime=10s    # fuzz for 10 seconds
```

---

## httptest Package

**Tutorial: Testing HTTP Handlers Without a Network**

The `net/http/httptest` package provides two key tools: `httptest.NewRecorder()` captures handler output in memory (no real HTTP needed), and `httptest.NewServer()` spins up a real test server on localhost for integration testing. Use `NewRecorder` for fast unit tests of individual handlers, and `NewServer` when you need to test the full HTTP stack including middleware and routing.

```
┌─────────────────────────────────────────────────┐
│       httptest Two Approaches                   │
│                                                 │
│  Approach 1: ResponseRecorder (unit test)       │
│  ┌──────────────┐    ┌──────────────┐           │
│  │  NewRequest   │───►│  handler()   │           │
│  │  GET /health  │    │              │           │
│  └──────────────┘    └──────┬───────┘           │
│                             │ writes to         │
│                             ▼                   │
│                     ┌──────────────┐            │
│                     │  Recorder    │            │
│                     │  .Code = 200 │            │
│                     │  .Body = {}  │            │
│                     └──────────────┘            │
│                                                 │
│  Approach 2: NewServer (integration test)       │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐  │
│  │ http.Get │───►│ localhost │───►│ handler  │  │
│  │          │◄───│ :random   │◄───│          │  │
│  └──────────┘    └───────────┘    └──────────┘  │
│  defer server.Close()                           │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func TestHealthHandler(t *testing.T) {
    // Create a test request
    req := httptest.NewRequest("GET", "/health", nil)

    // Create a ResponseRecorder to capture the response
    rr := httptest.NewRecorder()

    // Call the handler
    healthHandler(rr, req)

    // Check status code
    if rr.Code != http.StatusOK {
        t.Errorf("status = %d; want %d", rr.Code, http.StatusOK)
    }

    // Check response body
    var body map[string]string
    json.NewDecoder(rr.Body).Decode(&body)
    if body["status"] != "ok" {
        t.Errorf("status = %q; want %q", body["status"], "ok")
    }
}

// httptest.NewServer — creates a test HTTP server
func TestWithServer(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(healthHandler))
    defer server.Close()

    // Make real HTTP request to test server
    resp, err := http.Get(server.URL + "/health")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d; want %d", resp.StatusCode, http.StatusOK)
    }
}
```

---

## Example Tests

**Tutorial: Tests That Double as Documentation**

Example functions serve two purposes: they are runnable tests verified by `go test`, and they appear in `go doc` output as live documentation. The `// Output:` comment at the end specifies expected output—if it doesn't match, the test fails. Name them `ExampleFunctionName` for function-level docs, or `ExampleType_Method` for method-level docs.

```
┌──────────────────────────────────────────────┐
│       Example Test Dual Purpose              │
│                                              │
│  func ExampleAdd() {                         │
│      fmt.Println(Add(2, 3))                  │
│      // Output:                              │
│      // 5                                    │
│  }                                           │
│       │                    │                  │
│       ▼                    ▼                  │
│  ┌──────────┐      ┌──────────────┐          │
│  │ go test  │      │ go doc math  │          │
│  │ verifies │      │ shows as     │          │
│  │ output=5 │      │ usage example│          │
│  └──────────┘      └──────────────┘          │
│                                              │
│  Naming: ExampleAdd           → func Add     │
│          ExampleAdd_negative  → variant      │
│          ExampleUser_String   → method       │
└──────────────────────────────────────────────┘
```

```go
package math

import "fmt"

// Example tests serve as both tests AND documentation
// go test verifies the output matches the // Output: comment

func ExampleAdd() {
    fmt.Println(Add(2, 3))
    fmt.Println(Add(-1, 1))
    // Output:
    // 5
    // 0
}

func ExampleAdd_negative() {
    fmt.Println(Add(-5, -3))
    // Output:
    // -8
}
```

---

## Interview Questions

1. **How does Go's testing framework work?**
   - Test files end with `_test.go`, test functions start with `Test` and accept `*testing.T`. Run with `go test`. The framework auto-discovers tests—no registration required.

2. **What are table-driven tests?**
   - A pattern where test cases are defined as a slice of structs, then iterated with `t.Run(name, func)` for subtests. Each entry has inputs and expected outputs. This is idiomatic Go testing.

3. **How do benchmarks work in Go?**
   - Functions start with `Benchmark` and accept `*testing.B`. Run with `go test -bench=.`. The function body runs `b.N` times (auto-calibrated). Use `b.ResetTimer()` to exclude setup time.

4. **What is fuzzing in Go?**
   - Go 1.18+ supports fuzz testing with `Fuzz` functions and `*testing.F`. Seed corpus is provided via `f.Add()`. Run with `go test -fuzz=FuzzName`. The engine mutates inputs to find bugs.

5. **What is `httptest` package used for?**
   - `httptest.NewServer(handler)` creates a test HTTP server. `httptest.NewRecorder()` creates a `ResponseRecorder` to capture handler output without network. Essential for testing HTTP handlers.

6. **What is the difference between `t.Error` and `t.Fatal`?**
   - `t.Error`/`t.Errorf` marks the test as failed but continues execution. `t.Fatal`/`t.Fatalf` marks failed and stops the test immediately via `runtime.Goexit()`.

7. **How do you skip a test conditionally?**
   - Use `t.Skip("reason")` or `testing.Short()` with `t.SkipNow()`. Example: `if testing.Short() { t.Skip("skipping integration test") }`. Run with `go test -short`.

8. **What is `testdata` directory used for?**
   - A special directory ignored by `go build` but accessible in tests. Store fixture files, golden files, and test inputs there. Access via relative paths from the test file.

9. **How do you test unexported functions?**
   - Place test files in the same package (e.g., `package foo` not `package foo_test`). This gives access to unexported identifiers. Use `_test` suffix for black-box testing of the public API.

10. **What is `t.Parallel()` and when should you use it?**
    - Marks a test or subtest to run in parallel with other parallel tests. Call at the start of the test function. Be careful with shared state—use local variables or proper synchronization.
