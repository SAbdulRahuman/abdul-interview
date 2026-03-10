# Phase 1: Algorithm Complexity — Big O Notation

## What is Big O Notation?

Big O notation describes the **upper bound** of an algorithm's growth rate. It tells you the **worst-case** scenario — the maximum number of operations an algorithm will perform as input size grows.

Formally: **f(n) = O(g(n))** means there exist constants `c > 0` and `n₀` such that `f(n) ≤ c · g(n)` for all `n ≥ n₀`.

In plain English: **"As n gets very large, f(n) grows no faster than g(n)."**

---

## Rules of Big O

1. **Drop constants**: O(2n) → O(n), O(100) → O(1)
2. **Drop lower-order terms**: O(n² + n + 1) → O(n²)
3. **Different inputs = different variables**: O(a + b), O(a × b)
4. **Sequential = add**: O(n) + O(m) = O(n + m)
5. **Nested = multiply**: O(n) × O(m) = O(n × m)

---

## Example 1: Dropping Constants — O(2n) → O(n)

```go
package main

import "fmt"

// Even though we loop twice, it's still O(n)
func twoPassSum(nums []int) (int, int) {
    sum := 0
    for _, num := range nums { // first pass: O(n)
        sum += num
    }

    max := nums[0]
    for _, num := range nums { // second pass: O(n)
        if num > max {
            max = num
        }
    }
    // Total: O(n) + O(n) = O(2n) = O(n)
    return sum, max
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}
    sum, max := twoPassSum(nums)
    fmt.Printf("Sum: %d, Max: %d\n", sum, max) // Sum: 31, Max: 9
}
```

**Why drop constants?** Big O measures **growth rate**. Whether you loop 1× or 2×, the growth is still linear. O(2n) and O(n) have the same shape on a graph.

```
  O(2n) vs O(n) — Same growth shape:

  Operations
  ▲
  │                           ╱ O(2n)
  │                         ╱
  │                       ╱
  │                     ╱   ╱ O(n)
  │                   ╱   ╱
  │                 ╱   ╱
  │               ╱   ╱
  │             ╱   ╱
  │           ╱   ╱       Both are straight lines!
  │         ╱   ╱         Only steepness differs
  │       ╱   ╱           (constant factor).
  │     ╱   ╱
  │   ╱   ╱              In Big O, they're SAME: O(n)
  │ ╱   ╱
  │╱  ╱
  └──────────────────────────────▶ n
```

---

## Example 2: Dropping Lower-Order Terms — O(n² + n) → O(n²)

```go
package main

import "fmt"

// The n² term dominates as n grows large
func dominantTerm(nums []int) int {
    n := len(nums)
    count := 0

    // O(n²) — this dominates
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            count++
        }
    }

    // O(n) — this is negligible compared to n²
    for i := 0; i < n; i++ {
        count++
    }

    // Total: O(n² + n) = O(n²)
    return count
}

func main() {
    nums := make([]int, 10)
    fmt.Println(dominantTerm(nums)) // 110 → 100 (n²) + 10 (n)

    nums2 := make([]int, 100)
    fmt.Println(dominantTerm(nums2)) // 10100 → 10000 (n²) + 100 (n)
    // As n grows, n becomes insignificant vs n²
}
```

```
  Why n² dominates n:

     n   │   n²     │   n   │  n as % of n²
  ───────┼─────────┼───────┼───────────────
     10  │     100  │   10  │  10%  ← noticeable
    100  │  10,000  │  100  │   1%  ← small
   1000  │ 1,000,000│ 1000  │  0.1% ← negligible
  10000  │ 10⁸     │ 10000 │  0.01%← irrelevant!

  As n grows, n becomes invisible next to n².
  So O(n² + n) = O(n²)
```

---

## Example 3: O(1) — Constant Time Operations

```go
package main

import "fmt"

// O(1) — operations don't depend on input size
func getMiddle(nums []int) int {
    return nums[len(nums)/2] // direct index access
}

func swapFirstLast(nums []int) {
    n := len(nums)
    nums[0], nums[n-1] = nums[n-1], nums[0] // one swap
}

func isEven(n int) bool {
    return n%2 == 0 // one operation
}

func mapLookup(m map[string]int, key string) (int, bool) {
    val, ok := m[key] // hash lookup is O(1) average
    return val, ok
}

func main() {
    nums := []int{10, 20, 30, 40, 50}
    fmt.Println(getMiddle(nums)) // 30

    swapFirstLast(nums)
    fmt.Println(nums) // [50 20 30 40 10]

    fmt.Println(isEven(42))  // true
    fmt.Println(isEven(17))  // false

    m := map[string]int{"go": 1, "rust": 2}
    val, ok := mapLookup(m, "go")
    fmt.Println(val, ok) // 1 true
}
```

```
  O(1) operations — instant regardless of size:

  ┌─────────────────────────────────────────┐
  │ Array index access    nums[i]     O(1) │
  │ Hash map lookup       m["key"]    O(1) │
  │ Arithmetic            a + b       O(1) │
  │ Comparison            a == b      O(1) │
  │ Stack push/pop        s.push(x)   O(1) │
  │ Variable assignment   x := 5      O(1) │
  └─────────────────────────────────────────┘
  No loops, no recursion → constant time
```

```go
package main

import "fmt"

// O(n) — visit each element exactly once
func contains(nums []int, target int) bool {
    for _, num := range nums {
        if num == target {
            return true // best case: O(1), but Big O = worst case
        }
    }
    return false
}

// O(n) — counting occurrences
func countOccurrences(s string, ch byte) int {
    count := 0
    for i := 0; i < len(s); i++ {
        if s[i] == ch {
            count++
        }
    }
    return count
}

// O(n) — copying a slice
func copySlice(nums []int) []int {
    result := make([]int, len(nums))
    copy(result, nums) // copies n elements
    return result
}

func main() {
    nums := []int{5, 3, 8, 1, 9}
    fmt.Println(contains(nums, 8))  // true
    fmt.Println(contains(nums, 99)) // false

    fmt.Println(countOccurrences("hello world", 'l')) // 3

    copied := copySlice(nums)
    fmt.Println(copied) // [5 3 8 1 9]
}
```

```
  Linear algorithms — visit each element once:

  Input:  [5, 3, 8, 1, 9]
           ↓  ↓  ↓  ↓  ↓     ← every element checked
  Check:   1  2  3  4  5     ← n operations total

  Double the input, double the work:
  n=5    → 5 ops     █████
  n=10   → 10 ops    ██████████
  n=20   → 20 ops    ████████████████████
```

```go
package main

import "fmt"

// O(log n) — number is halved each iteration
func countHalves(n int) int {
    count := 0
    for n > 1 {
        n /= 2
        count++
    }
    return count
}

// O(log n) — number of digits
func digitCount(n int) int {
    if n == 0 {
        return 1
    }
    count := 0
    for n > 0 {
        n /= 10 // dividing by 10 each time = O(log₁₀ n)
        count++
    }
    return count
}

// O(log n) — exponentiation by squaring
func power(base, exp int) int {
    result := 1
    for exp > 0 {
        if exp%2 == 1 {
            result *= base
        }
        base *= base
        exp /= 2 // halving each step
    }
    return result
}

func main() {
    fmt.Println(countHalves(16))    // 4 (16→8→4→2→1)
    fmt.Println(countHalves(1024))  // 10

    fmt.Println(digitCount(12345))  // 5
    fmt.Println(digitCount(0))      // 1

    fmt.Println(power(2, 10))       // 1024
    fmt.Println(power(3, 5))        // 243
}
```

**Key pattern:** Whenever you see **dividing by a constant** (÷2, ÷10, etc.), think O(log n).

```
  Halving pattern — countHalves(16):

  n=16 → n=8 → n=4 → n=2 → n=1    (4 steps)

  Each step: n = n / 2
  Steps until n=1: log₂(n)

  ┌────────────────────────────────┐  step 0: n=16
  └────────────────────────────────┘
  ┌────────────────┐                  step 1: n=8
  └────────────────┘
  ┌────────┐                          step 2: n=4
  └────────┘
  ┌────┐                              step 3: n=2
  └────┘
  ┌─┐                                  step 4: n=1 DONE
  └─┘

  n=1,000,000 → only ~20 steps!
```

---

## Example 6: O(n log n) — Efficient Sorting

```go
package main

import (
    "fmt"
    "sort"
)

// O(n log n) — Go's sort uses introsort (quicksort + heapsort)
func sortAndDedup(nums []int) []int {
    sort.Ints(nums) // O(n log n)

    // Dedup in O(n)
    result := []int{nums[0]}
    for i := 1; i < len(nums); i++ {
        if nums[i] != nums[i-1] {
            result = append(result, nums[i])
        }
    }
    // Total: O(n log n) + O(n) = O(n log n)
    return result
}

func main() {
    nums := []int{5, 3, 8, 3, 1, 5, 9, 1, 2}
    fmt.Println(sortAndDedup(nums)) // [1 2 3 5 8 9]
}
```

```
  Sort then dedup — complexity breakdown:

  Step 1: Sort O(n log n)
  [5, 3, 8, 3, 1, 5, 9, 1, 2] → [1, 1, 2, 3, 3, 5, 5, 8, 9]

  Step 2: Dedup O(n)
  [1, 1, 2, 3, 3, 5, 5, 8, 9] → [1, 2, 3, 5, 8, 9]
   ✓  ✗  ✓  ✓  ✗  ✓  ✗  ✓  ✓

  Total: O(n log n) + O(n) = O(n log n)
         └────────┘   └─┤
         dominates    dropped
```

```go
package main

import "fmt"

// O(n²) — checking all pairs
func hasPairWithSum(nums []int, target int) bool {
    n := len(nums)
    for i := 0; i < n; i++ {          // O(n)
        for j := i + 1; j < n; j++ {  // O(n)
            if nums[i]+nums[j] == target {
                return true
            }
        }
    }
    return false
}

// O(n²) — selection sort
func selectionSort(nums []int) {
    n := len(nums)
    for i := 0; i < n; i++ {
        minIdx := i
        for j := i + 1; j < n; j++ {
            if nums[j] < nums[minIdx] {
                minIdx = j
            }
        }
        nums[i], nums[minIdx] = nums[minIdx], nums[i]
    }
}

func main() {
    nums := []int{1, 3, 5, 7, 9}
    fmt.Println(hasPairWithSum(nums, 10)) // true (1+9, 3+7)
    fmt.Println(hasPairWithSum(nums, 2))  // false

    arr := []int{64, 25, 12, 22, 11}
    selectionSort(arr)
    fmt.Println(arr) // [11 12 22 25 64]
}
```

```
  Pair checking with nested loops:

  nums = [1, 3, 5, 7, 9]    target = 10

  i=0: (1,3) (1,5) (1,7) (1,9✓)
  i=1:       (3,5) (3,7✓)(3,9)
  i=2:             (5,7) (5,9)
  i=3:                   (7,9)

  Pairs checked:
  ┌───┬───┬───┬───┬───┐
  │   │ 1 │ 3 │ 5 │ 7 │ 9 │
  ├───┼───┼───┼───┼───┤
  │ 1 │   │ ✗ │ ✗ │ ✗ │ ✓ │
  │ 3 │   │   │ ✗ │ ✓ │ ✗ │
  │ 5 │   │   │   │ ✗ │ ✗ │
  │ 7 │   │   │   │   │ ✗ │
  └───┴───┴───┴───┴───┘
  Total pairs = n(n-1)/2 ≈ n²/2 → O(n²)
```

```go
package main

import "fmt"

// O(a + b) — sequential processing of different inputs
func printBothArrays(a, b []int) {
    for _, val := range a { // O(a)
        fmt.Print(val, " ")
    }
    fmt.Println()
    for _, val := range b { // O(b)
        fmt.Print(val, " ")
    }
    fmt.Println()
}

// O(a × b) — nested processing of different inputs
func commonElements(a, b []int) []int {
    var common []int
    for _, x := range a {        // O(a)
        for _, y := range b {    // O(b)
            if x == y {
                common = append(common, x)
            }
        }
    }
    return common // Total: O(a × b)
}

func main() {
    a := []int{1, 2, 3}
    b := []int{2, 3, 4, 5}

    printBothArrays(a, b) // sequential → O(a + b)
    fmt.Println(commonElements(a, b)) // nested → O(a × b), [2 3]
}
```

**Rule:** Sequential loops = add. Nested loops = multiply.

```
  Sequential (ADD):            Nested (MULTIPLY):

  for a { ... }   O(a)         for a {
  for b { ... }   O(b)           for b { ... }  O(a × b)
  ───────────────────         }
  Total: O(a + b)              ───────────────────
                               Total: O(a × b)

  a=100, b=200:                a=100, b=200:
  100 + 200 = 300              100 × 200 = 20,000
```

---

## Example 9: O(2ⁿ) — Exponential Growth

```go
package main

import "fmt"

// O(2ⁿ) — each call branches into 2
func climbStairs(n int) int {
    if n <= 1 {
        return 1
    }
    return climbStairs(n-1) + climbStairs(n-2)
}

// Counting the actual number of calls to demonstrate growth
func climbStairsCount(n int, count *int) int {
    *count++
    if n <= 1 {
        return 1
    }
    return climbStairsCount(n-1, count) + climbStairsCount(n-2, count)
}

func main() {
    for n := 5; n <= 30; n += 5 {
        count := 0
        result := climbStairsCount(n, &count)
        fmt.Printf("n=%2d → result=%d, calls=%d\n", n, result, count)
    }
    // n= 5 → result=8, calls=15
    // n=10 → result=89, calls=177
    // n=15 → result=987, calls=1973
    // n=20 → result=10946, calls=21891
    // n=25 → result=121393, calls=242785
    // n=30 → result=1346269, calls=2692537
}
```

```
  Exponential call tree for climbStairs(5):

                    cs(5)
                  /       \
              cs(4)        cs(3)
             /    \        /    \
          cs(3)  cs(2)  cs(2)  cs(1)
          /  \    / \    / \
       cs(2) cs(1) ...  ...  ...
       / \
    cs(1) cs(0)

  Each level doubles the calls:
  Level 0: 1 call
  Level 1: 2 calls
  Level 2: 4 calls
  Level 3: 8 calls
  ...
  Total ≈ 2ⁿ calls

  n=10: ~1,000     n=20: ~1,000,000     n=30: ~1,000,000,000
```

```go
package main

import (
    "fmt"
    "math"
)

// Demonstrate how different complexities scale
func main() {
    fmt.Println("n\t| O(1)\t| O(log n)\t| O(n)\t| O(n log n)\t| O(n²)\t\t| O(2ⁿ)")
    fmt.Println("--------|-------|---------------|-------|---------------|---------------|--------")

    for _, n := range []float64{1, 10, 100, 1000, 10000} {
        o1 := 1.0
        ologn := math.Log2(n)
        on := n
        onlogn := n * math.Log2(n)
        on2 := n * n
        o2n := "—"
        if n <= 25 {
            o2n = fmt.Sprintf("%.0f", math.Pow(2, n))
        }

        fmt.Printf("%.0f\t| %.0f\t| %.1f\t\t| %.0f\t| %.0f\t\t| %.0f\t\t| %s\n",
            n, o1, ologn, on, onlogn, on2, o2n)
    }
}
```

Output shows dramatic growth differences:
```
n=10:    O(1)=1,  O(log n)=3.3,  O(n)=10,     O(n²)=100
n=1000:  O(1)=1,  O(log n)=10,   O(n)=1000,   O(n²)=1,000,000
n=10000: O(1)=1,  O(log n)=13.3, O(n)=10000,  O(n²)=100,000,000
```

---

## Example 11: Multi-Part Complexity — Add vs Multiply

```go
package main

import "fmt"

// O(n + n²) = O(n²) — drop the lower term
func processData(data []int) {
    n := len(data)

    // Part 1: O(n) — find min and max
    min, max := data[0], data[0]
    for _, v := range data {
        if v < min { min = v }
        if v > max { max = v }
    }
    fmt.Printf("Range: [%d, %d]\n", min, max)

    // Part 2: O(n²) — find all pairs where sum > max
    count := 0
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            if data[i]+data[j] > max {
                count++
            }
        }
    }
    fmt.Printf("Pairs with sum > max: %d\n", count)
    // Total: O(n) + O(n²) = O(n²)
}

func main() {
    data := []int{3, 7, 1, 9, 4, 6}
    processData(data)
}
```

---

## Example 12: Logarithmic Bases Don't Matter

```go
package main

import (
    "fmt"
    "math"
)

// All logarithmic bases are equivalent in Big O
// because log_a(n) = log_b(n) / log_b(a) — constant factor!
func main() {
    n := 1000000.0

    log2 := math.Log2(n)
    log10 := math.Log10(n)
    ln := math.Log(n)

    fmt.Printf("log₂(%v)  = %.2f\n", n, log2)   // 19.93
    fmt.Printf("log₁₀(%v) = %.2f\n", n, log10)   // 6.00
    fmt.Printf("ln(%v)    = %.2f\n", n, ln)       // 13.82

    // They differ by a constant factor:
    fmt.Printf("\nlog₂/log₁₀ = %.2f (constant)\n", log2/log10) // ~3.32
    fmt.Printf("log₂/ln    = %.2f (constant)\n", log2/ln)      // ~1.44

    // So O(log₂ n) = O(log₁₀ n) = O(ln n) = O(log n)
    fmt.Println("\nAll log bases are O(log n) — base is a constant factor!")
}
```

---

## Common Big O Complexity Hierarchy

```
O(1) < O(log n) < O(√n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!)
 ▲        ▲         ▲       ▲        ▲           ▲       ▲       ▲       ▲
Best  Excellent   Good    Fair   Acceptable    Bad    Terrible Awful  Worst
```

---

## Key Takeaways

1. **Big O = upper bound** on growth rate (worst case)
2. **Drop constants** and **lower-order terms**
3. **Different inputs → different variables** (a, b not n)
4. **Sequential = add**, **Nested = multiply**
5. **Log base doesn't matter** (all O(log n))
6. **Common target**: reduce O(n²) → O(n log n) or O(n)

> **Next up:** Big Theta Notation →
