# Chapter 22 — Standard Library Highlights

Go's standard library is remarkably comprehensive — you can build production web servers, CLI tools, and system utilities without any third-party packages. Here are the most important packages beyond what's covered in dedicated chapters.

```
┌──────────────────────────────────────────────────────────┐
│          Key Standard Library Packages                  │
│                                                          │
│  time      → time operations, formatting, timers        │
│  sort      → sorting slices, custom sort                │
│  strings   → string manipulation, Builder               │
│  strconv   → string ↔ number conversion                 │
│  regexp    → regular expressions                        │
│  sync      → mutexes, WaitGroup, Pool, Once             │
│  log/slog  → structured logging (Go 1.21+)              │
│  slices    → generic slice operations (Go 1.21+)        │
│  maps      → generic map operations (Go 1.21+)          │
│  cmp       → comparison utilities (Go 1.21+)            │
│  math/rand → pseudo-random numbers                      │
│  crypto/*  → SHA, AES, TLS, etc.                        │
│                                                          │
│  Go's time format reference date:                       │
│  Mon Jan 2 15:04:05 MST 2006                            │
│   ↑   ↑  ↑  ↑  ↑  ↑   ↑   ↑                            │
│   1   1  2  3  4  5   -7  2006                          │
│  (month=1, day=2, hour=3, min=4, sec=5, year=2006)     │
└──────────────────────────────────────────────────────────┘
```

## time Package

**Tutorial: Working with Time, Durations, Timers, and Formatting**

This example covers the `time` package's essential operations: getting the current time, performing duration arithmetic, using `Timer` (fires once) and `Ticker` (fires repeatedly), and formatting/parsing with Go's unique reference time layout. Go does NOT use `%Y-%m-%d` style format strings — instead, you rearrange the reference date `Mon Jan 2 15:04:05 MST 2006` to describe your desired format. This is the most common stumbling point for newcomers.

```
┌───────────────────────────────────────────────────┐
│       Go Time Reference Layout                   │
│                                                   │
│   Mon Jan  2 15:04:05 MST 2006                    │
│    │   │  │  │  │  │   │   │                     │
│    1   1  2  3  4  5  -7  2006                    │
│   day mon d  h  m  s  tz  year                    │
│                                                   │
│   Timer vs Ticker:                                │
│   ┌─────────┐           ┌──────────┐             │
│   │  Timer  │           │  Ticker   │             │
│   │ (once)  │           │ (repeat)  │             │
│   └────┬────┘           └────┬─────┘             │
│        │  200ms           │  100ms  100ms  ...   │
│        ▼                  ▼        ▼              │
│   <-timer.C          <-ticker.C (loop)            │
│   (fire & done)      (tick, tick, Stop())         │
└───────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Current time
    now := time.Now()
    fmt.Println("Now:", now)

    // Duration
    fmt.Println("Since epoch:", time.Since(time.Time{}))
    future := now.Add(2 * time.Hour)
    fmt.Println("Until 2h from now:", time.Until(future))

    // Sleep
    fmt.Println("Sleeping 100ms...")
    time.Sleep(100 * time.Millisecond)

    // Timer — fires once
    timer := time.NewTimer(200 * time.Millisecond)
    <-timer.C
    fmt.Println("Timer fired!")

    // Ticker — fires repeatedly
    ticker := time.NewTicker(100 * time.Millisecond)
    count := 0
    for range ticker.C {
        count++
        fmt.Println("Tick", count)
        if count >= 3 {
            ticker.Stop()
            break
        }
    }

    // Formatting — Go uses a REFERENCE TIME: Mon Jan 2 15:04:05 MST 2006
    // (1/2 3:04:05 PM 2006, timezone MST = -0700)
    fmt.Println(now.Format("2006-01-02"))           // 2026-03-09
    fmt.Println(now.Format("2006-01-02 15:04:05"))  // 2026-03-09 14:30:00
    fmt.Println(now.Format(time.RFC3339))            // 2026-03-09T14:30:00Z
    fmt.Println(now.Format("Mon, 02 Jan 2006"))      // Mon, 09 Mar 2026

    // Parsing
    t, err := time.Parse("2006-01-02", "2026-03-09")
    if err != nil {
        fmt.Println("Parse error:", err)
    }
    fmt.Println("Parsed:", t)

    // Duration arithmetic
    d := 2*time.Hour + 30*time.Minute
    fmt.Println("Duration:", d)           // 2h30m0s
    fmt.Println("In minutes:", d.Minutes()) // 150
}
```

---

## sort Package

**Tutorial: Sorting with sort and slices Packages**

This example demonstrates Go's sorting capabilities: `sort.Slice` for custom sort-by-field, `sort.SliceStable` to preserve order of equal elements, and `sort.Search` for binary search on sorted data. It also shows the newer `slices.Sort` and `slices.SortFunc` (Go 1.21+) which use generics for cleaner, type-safe APIs. Watch for the difference between `sort.Slice` (may reorder equal elements) and `sort.SliceStable` (preserves original order).

```
┌─────────────────────────────────────────────────┐
│       Sorting Options in Go                      │
│                                                   │
│  sort.Slice(s, less)     ─► unstable, func-based  │
│  sort.SliceStable(s, less) ─► stable, func-based  │
│  sort.Search(n, f)       ─► binary search         │
│                                                   │
│  slices.Sort(s)          ─► generic, ordered      │
│  slices.SortFunc(s, cmp) ─► generic, custom cmp   │
│                                                   │
│  Stable vs Unstable:                              │
│  Input:  [{Bob,20} {Eve,20} {Ann,20}]             │
│  Stable:  order of equal elements preserved       │
│  Unstable: [{Eve,20} {Bob,20} {Ann,20}] possible  │
│                                                   │
│  sort.Search ─► finds first index where f(i)=true  │
│  [1, 3, 5, 7, 9, 11]  f: nums[i] >= 7            │
│         │              ▲                           │
│         └──────────────┘ idx=3                     │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "cmp"
    "fmt"
    "slices"
    "sort"
)

type Person struct {
    Name string
    Age  int
}

func main() {
    // sort.Slice — sort with custom less function
    people := []Person{
        {"Charlie", 25},
        {"Alice", 30},
        {"Bob", 20},
    }

    sort.Slice(people, func(i, j int) bool {
        return people[i].Age < people[j].Age
    })
    fmt.Println("By age:", people)
    // [{Bob 20} {Charlie 25} {Alice 30}]

    // sort.SliceStable — preserves original order of equal elements
    sort.SliceStable(people, func(i, j int) bool {
        return people[i].Name < people[j].Name
    })
    fmt.Println("By name (stable):", people)

    // sort.Search — binary search (slice must be sorted)
    nums := []int{1, 3, 5, 7, 9, 11}
    idx := sort.Search(len(nums), func(i int) bool {
        return nums[i] >= 7
    })
    fmt.Println("Index of 7:", idx) // 3

    // slices.Sort (Go 1.21+) — generic, simpler API
    ints := []int{5, 3, 1, 4, 2}
    slices.Sort(ints)
    fmt.Println("Sorted:", ints) // [1 2 3 4 5]

    // slices.SortFunc — custom comparison
    slices.SortFunc(people, func(a, b Person) int {
        return cmp.Compare(a.Age, b.Age)
    })
    fmt.Println("SortFunc by age:", people)
}
```

---

## regexp Package

**Tutorial: Regular Expressions with the regexp Package**

This example shows Go's `regexp` package which uses the RE2 engine — guaranteed linear-time matching with no catastrophic backtracking. Use `regexp.Compile` for user-supplied patterns (returns error) or `regexp.MustCompile` for known-valid patterns (panics on error). Key methods include `FindString` (first match), `FindAllString` (all matches), `MatchString` (boolean test), `ReplaceAllString`, and `FindStringSubmatch` for capture groups.

```
┌─────────────────────────────────────────────────┐
│         regexp Method Overview                   │
│                                                   │
│  re := regexp.MustCompile(`\d+`)                  │
│                                                   │
│  Input: "abc 123 def 456"                         │
│                                                   │
│  FindString      ─► "123"        (first match)    │
│  FindAllString   ─► ["123","456"] (all matches)  │
│  ReplaceAllString─► "abc X def X" (replace)      │
│  MatchString     ─► true/false   (test)          │
│                                                   │
│  Capture groups with FindStringSubmatch:          │
│  Pattern: (\d{4})-(\d{2})-(\d{2})                 │
│  Input:   "Date: 2026-03-09"                      │
│  Result:  ["2026-03-09", "2026", "03", "09"]      │
│            matches[0]  [1]   [2]   [3]            │
│            full match  groups──────►               │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    // Compile pattern (returns error if invalid)
    re, err := regexp.Compile(`\d+`)
    if err != nil {
        fmt.Println("Bad regex:", err)
        return
    }

    // MustCompile — panics on error (use for known-good patterns)
    emailRe := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

    // FindString — first match
    fmt.Println(re.FindString("abc 123 def 456")) // 123

    // FindAllString — all matches
    fmt.Println(re.FindAllString("abc 123 def 456", -1)) // [123 456]

    // MatchString — check if matches
    fmt.Println(emailRe.MatchString("alice@test.com"))  // true
    fmt.Println(emailRe.MatchString("not-an-email"))    // false

    // ReplaceAllString
    result := re.ReplaceAllString("Call 123-456-7890", "X")
    fmt.Println(result) // Call X-X-X

    // FindStringSubmatch — capture groups
    dateRe := regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
    matches := dateRe.FindStringSubmatch("Date: 2026-03-09")
    fmt.Println("Full:", matches[0])  // 2026-03-09
    fmt.Println("Year:", matches[1])  // 2026
    fmt.Println("Month:", matches[2]) // 03
    fmt.Println("Day:", matches[3])   // 09
}
```

---

## log and log/slog (Go 1.21+)

**Tutorial: Logging with log and Structured Logging with slog**

This example contrasts Go's basic `log` package with the modern `log/slog` structured logger introduced in Go 1.21. The basic `log` writes plain-text messages with timestamps and optional file/line info. `slog` adds log levels (Info, Warn, Error, Debug) and key-value structured fields, making logs machine-parseable. The `slog.NewJSONHandler` outputs structured JSON — ideal for log aggregation systems like ELK or Datadog.

```
┌─────────────────────────────────────────────────┐
│      log vs slog Comparison                      │
│                                                   │
│  log.Println("msg")                                │
│    └► 2026/03/09 14:30:00 msg                     │
│       (plain text, no levels)                      │
│                                                   │
│  slog.Info("msg", "key", value)                    │
│    └► 2026/03/09 INFO msg key=value               │
│       (leveled, structured key-value pairs)        │
│                                                   │
│  slog Handlers:                                   │
│  ┌─────────────┐  ┌────────────────┐              │
│  │ TextHandler │  │  JSONHandler    │              │
│  │ (default)   │  │ {"level":..}   │              │
│  └──────┬──────┘  └───────┬────────┘              │
│         └────────────┬─┘                         │
│                      ▼                            │
│             ┌─────────────┐                      │
│             │  os.Stdout  │                      │
│             └─────────────┘                      │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "log"
    "log/slog"
    "os"
)

func main() {
    // Basic log package
    log.Println("Standard log message")
    log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
    log.Println("With file and line") // 2026/03/09 14:30:00 main.go:12: With file and line

    // log.Fatal — log and os.Exit(1)
    // log.Fatal("Critical error!") // exits program

    // slog — structured logging (Go 1.21+)
    slog.Info("User logged in",
        "user_id", 42,
        "ip", "192.168.1.1",
    )
    // 2026/03/09 14:30:00 INFO User logged in user_id=42 ip=192.168.1.1

    slog.Warn("Rate limit approaching",
        "current", 90,
        "max", 100,
    )

    slog.Error("Database connection failed",
        "host", "db.example.com",
        "port", 5432,
    )

    // JSON handler for structured output
    jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    jsonLogger.Info("Request processed",
        "method", "GET",
        "path", "/api/users",
        "status", 200,
    )
    // {"time":"2026-03-09T14:30:00Z","level":"INFO","msg":"Request processed","method":"GET","path":"/api/users","status":200}
}
```

---

## flag Package

**Tutorial: Parsing Command-Line Arguments with the flag Package**

This example shows how to define and parse command-line flags using Go's built-in `flag` package. Each `flag.Int`, `flag.String`, or `flag.Bool` returns a pointer to the value. You must call `flag.Parse()` before accessing values. Non-flag arguments (positional args) are available via `flag.Args()`. Note that flag values are accessed with `*port`, `*host` since the functions return pointers.

```
┌─────────────────────────────────────────────────┐
│     flag Package Flow                             │
│                                                   │
│  go run main.go -port=9090 -debug arg1 arg2       │
│                  │          │      │    │         │
│                  ▼          ▼      └────┤         │
│           ┌────────────────┐   flag.Args()     │
│           │  flag.Parse() │  ["arg1","arg2"]   │
│           └─────┬──────────┘                   │
│                 │                                │
│                 ▼                                │
│    *port = 9090   (flag.Int returns *int)         │
│    *host = "localhost" (default, not overridden)  │
│    *debug = true  (flag.Bool returns *bool)       │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // Define flags
    port := flag.Int("port", 8080, "server port")
    host := flag.String("host", "localhost", "server host")
    debug := flag.Bool("debug", false, "enable debug mode")

    // Parse command line arguments
    flag.Parse()

    fmt.Printf("Server: %s:%d (debug=%v)\n", *host, *port, *debug)

    // Remaining non-flag arguments
    fmt.Println("Args:", flag.Args())

    // Usage: go run main.go -port=9090 -host=0.0.0.0 -debug arg1 arg2
    // Output: Server: 0.0.0.0:9090 (debug=true)
    //         Args: [arg1 arg2]
}
```

---

## container Packages

**Tutorial: Heap, Linked List, and Ring from container/**

This example covers Go's three `container/` packages: `container/heap` for priority queues (you implement the `heap.Interface`), `container/list` for doubly-linked lists with O(1) insert/remove, and `container/ring` for fixed-size circular buffers. The heap requires implementing five methods (`Len`, `Less`, `Swap`, `Push`, `Pop`) on your type, then using `heap.Init`, `heap.Push`, and `heap.Pop` to operate on it.

```
┌─────────────────────────────────────────────────┐
│       container/ Data Structures                 │
│                                                   │
│  container/heap (Min-Heap / Priority Queue):      │
│            1                                      │
│          ┌─┴─┐                                     │
│        ┌─┤   ├─┐                                   │
│        2   3                                      │
│      ┌─┴─┐                                         │
│      4   5   heap.Pop() ─► returns 1 (min)        │
│                                                   │
│  container/list (Doubly Linked List):             │
│  nil ◄── [z] ◄─► [a] ◄─► [b] ──► nil               │
│        Front()              Back()               │
│                                                   │
│  container/ring (Circular Buffer):                │
│  ┌─► [0] ─► [1] ─► [2] ─► [3] ─► [4] ─┐            │
│  └────────────────────────────────┘            │
│  r.Next() always wraps around                     │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "container/heap"
    "container/list"
    "container/ring"
    "fmt"
)

// Priority Queue using container/heap
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min-heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func main() {
    // Heap (Priority Queue)
    h := &IntHeap{5, 3, 1, 4, 2}
    heap.Init(h)
    heap.Push(h, 0)

    fmt.Print("Heap (min first): ")
    for h.Len() > 0 {
        fmt.Print(heap.Pop(h), " ") // 0 1 2 3 4 5
    }
    fmt.Println()

    // Doubly Linked List
    l := list.New()
    l.PushBack("a")
    l.PushBack("b")
    l.PushFront("z")

    fmt.Print("List: ")
    for e := l.Front(); e != nil; e = e.Next() {
        fmt.Print(e.Value, " ") // z a b
    }
    fmt.Println()

    // Ring (Circular List)
    r := ring.New(5)
    for i := 0; i < r.Len(); i++ {
        r.Value = i
        r = r.Next()
    }

    fmt.Print("Ring: ")
    r.Do(func(v any) {
        fmt.Print(v, " ") // 0 1 2 3 4
    })
    fmt.Println()
}
```

---

## Interview Questions

1. **What are the most important packages in Go's standard library?**
   - `fmt`, `io`, `os`, `net/http`, `encoding/json`, `sync`, `context`, `testing`, `strings`, `strconv`, `sort`, `time`, `log`, `errors`, `crypto`. Go's stdlib is production-grade—often no third-party libraries needed.

2. **How does the `time` package work?**
   - `time.Now()`, `time.Since()`, `time.Duration`, `time.After()`, `time.Ticker`. Go uses a reference time `Mon Jan 2 15:04:05 MST 2006` for formatting. Use `time.Parse` / `time.Format` with this layout.

3. **What is the `sort` package and how do you use it?**
   - `sort.Ints()`, `sort.Strings()`, `sort.Slice(s, less)` for custom sorting. For stable sort: `sort.SliceStable()`. Implement `sort.Interface` (Len, Less, Swap) for complex types.

4. **How does `strings.Builder` differ from string concatenation?**
   - `strings.Builder` uses an internal `[]byte` buffer—O(n) for building strings. Concatenation with `+` creates new strings each time—O(n²). Always use `Builder` in loops.

5. **What is `sync.Pool` and when should you use it?**
   - A cache of temporary objects to reduce GC pressure. Objects may be evicted at any GC cycle. Use for frequently allocated/deallocated objects like buffers. Not for connection pools.

6. **How does `log` package work and what are its limitations?**
   - `log.Println()`, `log.Fatalf()` (logs + os.Exit), `log.Panicf()` (logs + panic). Limited: no log levels, no structured logging. Production apps use `slog` (Go 1.21+), `zap`, or `zerolog`.

7. **What is `crypto/rand` vs `math/rand`?**
   - `crypto/rand` provides cryptographically secure random bytes (for keys, tokens). `math/rand` is a pseudo-random generator for non-security use (simulations, shuffling). Since Go 1.20, `math/rand` auto-seeds.

8. **How do you use `regexp` in Go?**
   - `regexp.MustCompile(pattern)` compiles a regex (panics on error). `re.FindString()`, `re.FindAllString()`, `re.ReplaceAllString()`. Go uses RE2 syntax—no backreferences, guaranteed linear time.

9. **What is the `flag` package?**
   - Parses command-line flags: `flag.String("name", "default", "usage")`. Call `flag.Parse()` in main. Supports `-flag=value`, `-flag value`. For complex CLI, use third-party like `cobra` or `cli`.

10. **What is `slog` (structured logging) in Go 1.21+?**
    - `slog.Info("msg", "key", value)` provides structured, leveled logging. Supports JSON and text handlers. Replaceable backend. Context-aware via `slog.InfoContext(ctx, ...)`.
