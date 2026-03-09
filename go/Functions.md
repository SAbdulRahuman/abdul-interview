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

```go
package main

import "fmt"

// Function type
type MathFunc func(int, int) int

// Function that takes a function as argument
func apply(a, b int, fn MathFunc) int {
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

```go
package main

import "fmt"

func main() {
    // Anonymous function — defined and called inline
    func() {
        fmt.Println("I'm an anonymous function!")
    }()

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

```go
package main

import "fmt"

// panic triggers a runtime error — the program crashes unless recovered
func mustPositive(n int) int {
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

```go
package main

import "fmt"

type Calculator struct {
    Name string
}

func (c Calculator) Add(a, b int) int {
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
