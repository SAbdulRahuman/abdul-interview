# Chapter 4 — Functions

Functions are the building blocks of Go programs. Go functions are **first-class values** — they can be assigned to variables, passed as arguments, and returned from other functions. Go has no concept of function overloading, default parameters, or optional parameters.

```
┌──────────────────────────────────────────────────────────┐
│                Go Function Features                      │
│                                                          │
│  ✓ Multiple return values    (result, err := f())        │
│  ✓ Named return values       (func f() (n int, e error)) │
│  ✓ Variadic parameters       (func f(args ...int))       │
│  ✓ First-class functions     (fn := func() { })          │
│  ✓ Closures                  (capture outer variables)   │
│  ✓ Deferred calls            (defer cleanup())           │
│  ✓ Panic/Recover             (exception-like recovery)   │
│  ✓ Method values/expressions (bound & unbound methods)   │
│                                                          │
│  ✗ No function overloading                               │
│  ✗ No default parameters                                 │
│  ✗ No named arguments at call site                       │
│  ✗ No tail-call optimization                             │
└──────────────────────────────────────────────────────────┘
```

## Function Declarations, Parameters, Return Values

**Tutorial: Basic Function Anatomy**

Every Go function starts with `func`, followed by the name, parameters in parentheses, and the return type. Parameters of the same type can be grouped: `(a, b int)` instead of `(a int, b int)`. Functions with no return type are used for side effects (printing, writing files). The function is called by name and arguments are passed by value (copies are made).

```
┌──────────────────────────────────────────────────────────┐
│        Function Signature Anatomy                        │
│                                                          │
│  func add(a int, b int) int {                            │
│  │    │   │              │                               │
│  │    │   │              └── return type                  │
│  │    │   └── parameters (pass-by-value)                 │
│  │    └── function name (exported if Uppercase)          │
│  └── keyword                                             │
│                                                          │
│  Call: add(3, 5)                                         │
│  ┌───────┐     ┌───────┐                                 │
│  │ a = 3 │     │ b = 5 │  ← copies of arguments         │
│  └───┬───┘     └───┬───┘                                 │
│      └──────┬──────┘                                     │
│         return 8                                         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Simple function with parameters and return value
func add(a int, b int) int {
    return a + b
}

// Parameters of the same type can be grouped
func multiply(a, b int) int {
    return a * b
}

// Function with no return value
func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    fmt.Println(add(3, 5))      // 8
    fmt.Println(multiply(4, 6)) // 24
    greet("Alice")              // Hello, Alice
}
```

---

## Multiple Return Values

**Tutorial: Idiomatic Error Handling**

Go's multiple return values enable the `(result, error)` pattern — the core of Go's error handling philosophy. The `divide` function returns both a result and an `error`. The caller checks `err != nil` to handle the failure case. The `_` (blank identifier) discards an unwanted return value — the compiler doesn't complain about unused values when you use `_`.

```
┌──────────────────────────────────────────────────────────┐
│       Multiple Return Values — Flow                      │
│                                                          │
│  result, err := divide(10, 3)                            │
│                   │                                      │
│                   ▼                                      │
│            ┌─────────────┐                               │
│            │ b == 0?     │                               │
│            │ NO          │                               │
│            └──────┬──────┘                               │
│                   ▼                                      │
│         return (3.33, nil)                                │
│         │            │                                   │
│         ▼            ▼                                   │
│  result = 3.33   err = nil                               │
│                                                          │
│  _, err = divide(10, 0)                                  │
│  │               │                                       │
│  discard         return (0, error("division by zero"))   │
│  result                                                  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "errors"
    "fmt"
)

// Multiple return values — idiomatic Go for returning result + error
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Returning multiple values of same type
func swap(a, b string) (string, string) {
    return b, a
}

func main() {
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("10 / 3 = %.2f\n", result) // 10 / 3 = 3.33
    }

    _, err = divide(10, 0) // Ignore first return with _
    fmt.Println("Error:", err) // Error: division by zero

    a, b := swap("hello", "world")
    fmt.Println(a, b) // world hello
}
```

---

## Named Return Values and Naked Return

**Tutorial: Named Returns as Pre-Declared Variables**

Named return values (`area, perimeter float64`) are automatically declared as local variables initialized to their zero values. You can assign to them directly and use a bare `return` (naked return) to return their current values. This is useful for short functions and for documenting what each return value means. Avoid naked returns in long functions — they make it hard to track what's being returned.

```
┌──────────────────────────────────────────────────────────┐
│     Named Returns — Variable Lifecycle                   │
│                                                          │
│  func rectangleProps(l, w float64) (area, perimeter float64)│
│                                      │         │         │
│     auto-declared:  area = 0.0   perimeter = 0.0         │
│                                                          │
│     area = l * w          → area = 15.0                  │
│     perimeter = 2*(l+w)   → perimeter = 16.0             │
│     return                → returns (15.0, 16.0)         │
│     └── naked return = return area, perimeter            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Named return values act as local variables
func rectangleProps(length, width float64) (area, perimeter float64) {
    area = length * width
    perimeter = 2 * (length + width)
    return // naked return — returns named values (avoid in long functions)
}

// Named returns are useful for documenting what the function returns
func minMax(nums []int) (min, max int) {
    min = nums[0]
    max = nums[0]
    for _, n := range nums {
        if n < min {
            min = n
        }
        if n > max {
            max = n
        }
    }
    return
}

func main() {
    area, perim := rectangleProps(5, 3)
    fmt.Printf("Area: %.1f, Perimeter: %.1f\n", area, perim)
    // Area: 15.0, Perimeter: 16.0

    min, max := minMax([]int{3, 1, 4, 1, 5, 9, 2, 6})
    fmt.Printf("Min: %d, Max: %d\n", min, max)
    // Min: 1, Max: 9
}
```

---

## Variadic Functions

**Tutorial: Variable Argument Lists**

The `...` syntax makes a function accept zero or more arguments of the same type. Inside the function, the variadic parameter is a slice. You can pass an existing slice with `numbers...` (spread operator). The variadic parameter must be the last in the parameter list.

```
┌──────────────────────────────────────────────────────────┐
│       Variadic — What Happens Under the Hood             │
│                                                          │
│  sum(1, 2, 3)                                            │
│       │                                                  │
│       ▼                                                  │
│  nums = []int{1, 2, 3}   ← compiler wraps args in slice │
│                                                          │
│  sum()                                                   │
│       │                                                  │
│       ▼                                                  │
│  nums = []int{}           ← empty slice (not nil)        │
│                                                          │
│  numbers := []int{5, 10, 15}                             │
│  sum(numbers...)          ← spread: passes slice directly│
│       │                                                  │
│       ▼                                                  │
│  nums = numbers           ← no new slice created         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Variadic function — accepts zero or more arguments
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// Variadic must be the last parameter
func logMessage(level string, messages ...string) {
    for _, msg := range messages {
        fmt.Printf("[%s] %s\n", level, msg)
    }
}

func main() {
    fmt.Println(sum())           // 0
    fmt.Println(sum(1, 2, 3))   // 6
    fmt.Println(sum(10, 20, 30, 40)) // 100

    // Pass a slice to a variadic function with ...
    numbers := []int{5, 10, 15}
    fmt.Println(sum(numbers...)) // 30

    logMessage("INFO", "Starting server", "Port: 8080")
    // [INFO] Starting server
    // [INFO] Port: 8080
}
```

---

## First-Class Functions

Functions in Go are first-class citizens — they can be assigned to variables, passed as arguments, and returned from other functions.

**Tutorial: Functions as Values**

This example shows three patterns: (1) defining a function type `MathFunc` for type safety, (2) passing functions as arguments to `apply()`, and (3) returning functions from `makeMultiplier()` — creating a function factory. The `double` and `triple` variables each hold a closure that remembers the `factor` value from when it was created.

```
┌──────────────────────────────────────────────────────────┐
│       Function Factory — makeMultiplier(2)               │
│                                                          │
│  makeMultiplier(2) returns:                              │
│  ┌───────────────────────────────────┐                   │
│  │ func(x int) int {                │                   │
│  │    return x * factor              │                   │
│  │ }                                 │                   │
│  │         │                         │                   │
│  │         └── factor = 2 (captured) │                   │
│  └───────────────────────────────────┘                   │
│                                                          │
│  double := makeMultiplier(2)  │  triple := makeMultiplier(3)│
│  double(5) → 5 * 2 = 10      │  triple(5) → 5 * 3 = 15 │
│  (factor=2 captured)          │  (factor=3 captured)     │
└──────────────────────────────────────────────────────────┘
```

```go
    return fn(a, b)
}

// Function that returns a function
func makeMultiplier(factor int) func(int) int {
    return func(x int) int {
        return x * factor
    }
}

func main() {
    // Assign function to variable
    add := func(a, b int) int { return a + b }
    sub := func(a, b int) int { return a - b }

    fmt.Println(apply(10, 5, add)) // 15
    fmt.Println(apply(10, 5, sub)) // 5

    // Function factory
    double := makeMultiplier(2)
    triple := makeMultiplier(3)

    fmt.Println(double(5))  // 10
    fmt.Println(triple(5))  // 15
}
```

---

## Anonymous Functions (Closures) and Variable Capture

A closure is an anonymous function that "closes over" (captures) variables from its enclosing scope. The closure holds a **reference** to the variable, not a copy — so changes to the variable are visible inside the closure and vice versa.

```
┌──────────────────────────────────────────────────────────┐
│             Closure Variable Capture                     │
│                                                          │
│  func makeCounter() func() int {                         │
│      count := 0           ← lives on heap (escaped)     │
│      return func() int {                                 │
│          count++           ← captures reference to count │
│          return count                                    │
│      }                                                   │
│  }                                                       │
│                                                          │
│  counter := makeCounter()                                │
│                                                          │
│  ┌──────────────┐     ┌─────────────────┐               │
│  │   closure     │────►│ count: 0 (heap) │               │
│  │  (func value) │     └─────────────────┘               │
│  └──────────────┘                                        │
│                                                          │
│  counter() → count becomes 1, returns 1                  │
│  counter() → count becomes 2, returns 2                  │
│  counter() → count becomes 3, returns 3                  │
│                                                          │
│  The closure keeps "count" alive even after              │
│  makeCounter() has returned!                             │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Closures and the Loop Variable Gotcha**

A closure captures variables by **reference**, not by value. When `increment()` modifies `counter`, the outer `counter` variable changes too — they share the same memory. The classic gotcha: in a loop, all closures captured the same loop variable `i`. In Go 1.22+, each iteration gets its own copy of `i`, fixing this issue. Pre-1.22, you had to shadow the variable with `i := i` inside the loop.

```go

    // Closure — captures variables from surrounding scope
    message := "Hello"
    greet := func(name string) {
        fmt.Printf("%s, %s!\n", message, name) // captures 'message'
    }
    greet("Alice") // Hello, Alice!

    // Closure captures by REFERENCE, not by value
    counter := 0
    increment := func() int {
        counter++ // modifies the outer variable
        return counter
    }

    fmt.Println(increment()) // 1
    fmt.Println(increment()) // 2
    fmt.Println(increment()) // 3
    fmt.Println(counter)     // 3 — outer variable was modified

    // Classic gotcha: loop variable capture (pre-Go 1.22)
    // In Go 1.22+, each iteration gets its own variable — this is fixed!
    funcs := make([]func(), 3)
    for i := 0; i < 3; i++ {
        funcs[i] = func() {
            fmt.Println(i) // In Go 1.22+: prints 0, 1, 2 correctly
        }
    }
    for _, f := range funcs {
        f()
    }

    // Pre-Go 1.22 fix: capture via parameter
    funcs2 := make([]func(), 3)
    for i := 0; i < 3; i++ {
        i := i // shadowing — creates a new variable for each iteration
        funcs2[i] = func() {
            fmt.Println(i)
        }
    }
}
```

---

## Recursion

**Tutorial: Recursive Functions**

Recursion in Go works like any other language — a function calls itself with smaller input until reaching a base case. `factorial(5)` calls `factorial(4)` which calls `factorial(3)` and so on until `factorial(1)` returns `1`. The tree example shows recursive traversal of a tree structure — each call processes one node and recurses into its children with increased indentation.

```
┌──────────────────────────────────────────────────────────┐
│       factorial(5) — Call Stack Unwinding                │
│                                                          │
│  factorial(5) = 5 * factorial(4)                         │
│                     4 * factorial(3)                      │
│                         3 * factorial(2)                  │
│                             2 * factorial(1)             │
│                                 return 1  ← base case   │
│                             return 2 * 1 = 2            │
│                         return 3 * 2 = 6                 │
│                     return 4 * 6 = 24                    │
│                 return 5 * 24 = 120                      │
│                                                          │
│  Stack depth: 5 frames (Go goroutine stacks grow        │
│  dynamically, so deep recursion is safer than in C)     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Classic recursion — factorial
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

// Fibonacci with recursion
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

// Recursive directory-like tree traversal
type TreeNode struct {
    Value    string
    Children []*TreeNode
}

func printTree(node *TreeNode, indent string) {
    fmt.Println(indent + node.Value)
    for _, child := range node.Children {
        printTree(child, indent+"  ")
    }
}

func main() {
    fmt.Println("5! =", factorial(5))  // 5! = 120
    fmt.Println("fib(10) =", fibonacci(10)) // fib(10) = 55

    root := &TreeNode{
        Value: "root",
        Children: []*TreeNode{
            {Value: "child1", Children: []*TreeNode{
                {Value: "grandchild1"},
            }},
            {Value: "child2"},
        },
    }
    printTree(root, "")
    // root
    //   child1
    //     grandchild1
    //   child2
}
```

---

## defer — Deferred Function Calls

**Tutorial: defer for Cleanup and Mutex Safety**

The `processFile` function defers `"Closing file..."` right after opening — guaranteeing cleanup even if a panic occurs later. The `safeIncrement` function uses `defer mu.Unlock()` immediately after `mu.Lock()` — this is the standard mutex pattern in Go. It ensures the lock is always released, even if the code between lock and unlock panics.

```
┌──────────────────────────────────────────────────────────┐
│       Mutex + defer Pattern                              │
│                                                          │
│  mu.Lock()         ← acquire lock                        │
│  defer mu.Unlock() ← schedule unlock for function exit   │
│                                                          │
│  *counter++        ← critical section (safe)             │
│                                                          │
│  } ← function returns                                    │
│    └─► mu.Unlock() runs ← lock always released          │
│                                                          │
│  Even if *counter++ panics, Unlock still runs!           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
)

// defer for cleanup
func processFile() {
    fmt.Println("Opening file...")
    defer fmt.Println("Closing file...") // runs when function exits

    fmt.Println("Processing file...")
    // Even if we return early or panic, the file will be closed
}

// defer with mutex (common pattern)
func safeIncrement(mu *sync.Mutex, counter *int) {
    mu.Lock()
    defer mu.Unlock() // ensures unlock even if panic occurs

    *counter++
}

func main() {
    processFile()
    // Opening file...
    // Processing file...
    // Closing file...
}
```

---

## panic and recover

`panic` triggers an immediate runtime error — the current function stops, deferred functions run, and the panic propagates up the call stack. `recover` catches a panic but **only works inside a deferred function**.

```
┌──────────────────────────────────────────────────────────┐
│           panic / recover Flow                           │
│                                                          │
│  main() ──► funcA() ──► funcB() ──► panic("boom!")       │
│                                        │                 │
│                                   ┌────▼────┐            │
│                                   │ funcB   │            │
│                                   │ defers  │ ← run now  │
│                                   │ unwind  │            │
│                              ┌────▼────┐    │            │
│                              │ funcA   │    │            │
│                              │ defers  │ ← run now       │
│                              │ unwind  │                 │
│                         ┌────▼────┐                      │
│                         │ main    │                      │
│                         │ defers  │ ← run now            │
│                         │ CRASH!  │ (if no recover)      │
│                         └─────────┘                      │
│                                                          │
│  With recover:                                           │
│  func safe() {                                           │
│      defer func() {                                      │
│          if r := recover(); r != nil {  ← catches panic  │
│              log.Println("recovered:", r)                │
│          }                                               │
│      }()                                                 │
│      riskyOperation()  // if this panics, we recover     │
│  }                                                       │
│  // Program continues normally after safe() returns      │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: panic and recover — Exception-Like Recovery**

`panic` immediately stops the current function, runs its deferred functions, and propagates up the call stack. `recover` catches a panic but **only works inside a deferred function**. In `safeDivide`, dividing by zero triggers a runtime panic. The deferred function calls `recover()` which catches the panic, converts it to an error, and allows the program to continue normally.

```go
    if n < 0 {
        panic(fmt.Sprintf("negative number: %d", n))
    }
    return n
}

// recover catches a panic in a deferred function
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()

    return a / b, nil // panics if b == 0 (integer division by zero)
}

func main() {
    // Normal case
    fmt.Println(mustPositive(5)) // 5

    // Panic case — recover in deferred function
    result, err := safeDivide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
        // Error: recovered from panic: runtime error: integer divide by zero
    } else {
        fmt.Println("Result:", result)
    }

    // Program continues after recover
    fmt.Println("Program continues normally")
}
```

---

## defer/panic/recover Error Recovery Pattern

**Tutorial: Isolating Panics in Request Handlers**

This pattern is used in web servers — each request handler is wrapped with `defer/recover` so that a panic in one request doesn't crash the entire server. The `handleRequest("/crash")` panics, but the deferred `recover()` catches it and logs the error. The program continues to handle subsequent requests normally.

```
┌──────────────────────────────────────────────────────────┐
│       Request Isolation with recover                     │
│                                                          │
│  main() calls:                                           │
│  ├── handleRequest("/home")   → prints "Handled"    ✓   │
│  ├── handleRequest("/crash")  → panic("crash!")          │
│  │     └── defer recover()    → catches panic, logs it   │
│  │         └── program survives                          │
│  └── handleRequest("/about")  → prints "Handled"    ✓   │
│                                                          │
│  Without recover: server crashes at "/crash"             │
│  With recover: only that request fails, server lives     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "log"
)

// Simulates a web server handler that shouldn't crash the whole server
func handleRequest(path string) {
    // Recover from any panic in this handler
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Handler for %s panicked: %v", path, r)
        }
    }()

    if path == "/crash" {
        panic("intentional crash!")
    }

    fmt.Printf("Handled request: %s\n", path)
}

func main() {
    handleRequest("/home")   // Handled request: /home
    handleRequest("/crash")  // Logs: Handler for /crash panicked: intentional crash!
    handleRequest("/about")  // Handled request: /about
    // Server didn't crash — each request is isolated
}
```

---

## Method Values and Method Expressions

Go has two ways to obtain a function value from a method: **method values** (bound to a specific receiver) and **method expressions** (unbound — receiver becomes the first argument).

```
┌──────────────────────────────────────────────────────────┐
│        Method Value vs Method Expression                 │
│                                                          │
│  type Calc struct{ Name string }                         │
│  func (c Calc) Add(a, b int) int { return a + b }       │
│                                                          │
│  calc := Calc{Name: "mine"}                              │
│                                                          │
│  Method Value (bound):                                   │
│  ┌────────────────────────────────────┐                  │
│  │ fn := calc.Add  ← receiver bound  │                  │
│  │ fn(3, 5)        → 8               │                  │
│  │ Signature: func(int, int) int     │                  │
│  └────────────────────────────────────┘                  │
│                                                          │
│  Method Expression (unbound):                            │
│  ┌────────────────────────────────────┐                  │
│  │ fn := Calc.Add  ← no receiver yet │                  │
│  │ fn(calc, 3, 5)  → 8 (pass recv)  │                  │
│  │ Signature: func(Calc, int, int) int│                 │
│  └────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Method Value (Bound) vs Method Expression (Unbound)**

A method value (`calc.Add`) binds the receiver `calc` into the function — you call it as `addFn(3, 5)` without specifying the receiver. A method expression (`Calculator.Add`) is unbound — the receiver becomes the first argument: `addExpr(calc, 10, 20)`. Method values are commonly used when passing methods as callback functions.

```go
    return a + b
}

func (c Calculator) Greet() string {
    return "Hi from " + c.Name
}

func main() {
    calc := Calculator{Name: "MyCalc"}

    // Method value — bound to a specific receiver
    addFn := calc.Add            // bound method: calc is captured
    fmt.Println(addFn(3, 5))     // 8

    greetFn := calc.Greet
    fmt.Println(greetFn())       // Hi from MyCalc

    // Method expression — unbound, receiver becomes first argument
    addExpr := Calculator.Add
    fmt.Println(addExpr(calc, 10, 20)) // 30

    // Useful for passing methods as function values
    applyOp := func(fn func(int, int) int, a, b int) int {
        return fn(a, b)
    }
    fmt.Println(applyOp(calc.Add, 7, 3)) // 10
}
```

---

## Interview Questions

1. **Can Go functions return multiple values? Give an example.**
   - Yes. `func divide(a, b float64) (float64, error)`. This is idiomatic for returning a result and an error.

2. **What are named return values in Go?**
   - Named returns give names to return values: `func f() (result int, err error)`. They act as local variables and enable naked returns (`return` without arguments). They can be modified by deferred functions.

3. **What is a variadic function?**
   - A function accepting a variable number of arguments of the same type: `func sum(nums ...int) int`. The parameter is received as a slice. You can spread a slice with `slice...`.

4. **Are functions first-class citizens in Go?**
   - Yes. Functions can be assigned to variables, passed as arguments, returned from other functions, and stored in data structures.

5. **What is a closure in Go?**
   - A closure is an anonymous function that captures and retains access to variables from its enclosing scope. The closure and the outer function share the same variable — changes are reflected in both.

6. **Explain the `defer`, `panic`, `recover` pattern.**
   - `panic` triggers a runtime error and begins unwinding the stack. `defer` ensures cleanup functions run during unwinding. `recover` (called inside a deferred function) catches the panic and returns the panic value, allowing the program to continue.

7. **When are `defer` arguments evaluated?**
   - Arguments to a deferred function call are evaluated immediately at the `defer` statement, not when the deferred function executes. However, deferred closures capture variable references and see final values.

8. **What is the difference between a method value and a method expression?**
   - Method value: `obj.Method` — binds the receiver, can be called as `fn(args)`. Method expression: `Type.Method` — unbound, requires the receiver as first argument: `fn(obj, args)`.

9. **Can Go functions be recursive?**
   - Yes. Go supports recursion. There's no tail-call optimization, so deep recursion can cause stack overflow (though goroutine stacks grow dynamically).

10. **What happens if you call `recover()` outside of a deferred function?**
    - It returns `nil` and has no effect. `recover` only works when called directly inside a deferred function during a panic.
