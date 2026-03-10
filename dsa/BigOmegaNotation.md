# Phase 1: Algorithm Complexity — Big Omega Notation

## What is Big Omega (Ω) Notation?

Big Omega (Ω) describes the **lower bound** — the **minimum** number of operations an algorithm will perform.

**f(n) = Ω(g(n))** means there exist constants `c > 0` and `n₀` such that:
```
f(n) ≥ c · g(n)   for all n ≥ n₀
```

In plain English: **"The algorithm takes AT LEAST this much time."**

---

## The Three Asymptotic Notations

| Notation | Bound | Meaning | Analogy |
|----------|-------|---------|---------|
| O(g(n)) | Upper | At most g(n) | ≤ ceiling |
| Ω(g(n)) | Lower | At least g(n) | ≥ floor |
| Θ(g(n)) | Tight | Exactly g(n) | = both |

If **f(n) = O(g(n))** AND **f(n) = Ω(g(n))**, then **f(n) = Θ(g(n))**.

---

## Example 1: Ω(1) — Every Algorithm Has This

```go
package main

import "fmt"

// Ω(1) — must do at least one operation
func doSomething(n int) string {
    if n < 0 {
        return "negative" // even this return is Ω(1)
    }
    total := 0
    for i := 0; i < n; i++ {
        total += i
    }
    return fmt.Sprintf("sum: %d", total)
}

func main() {
    // Best case: n < 0, returns immediately → O(1), Ω(1)
    fmt.Println(doSomething(-5)) // negative

    // Worst case: loops n times → O(n), Ω(1) still holds
    fmt.Println(doSomething(10)) // sum: 45
}
```

**Why Ω(1)?** — Every algorithm must execute at least one instruction. Ω(1) is trivially true for all algorithms.

```
  Ω(1) — the absolute floor:

  Operations
  ▲
  │               ╱  actual work (varies)
  │             ╱
  │           ╱
  │         ╱
  │       ╱
  │     ╱
  │   ╱
  │ ╱
  │●────●────●────●   Ω(1) floor ← at least 1 operation
  └────────────────────▶ n
  Every algorithm does at least 1 thing.
```

---

## Example 2: Ω(n) — Must Read All Input

```go
package main

import "fmt"

// Ω(n) — finding the minimum REQUIRES looking at every element
// You can't find min without checking all elements at least once
func findMin(nums []int) int {
    min := nums[0]
    for _, num := range nums { // must check all n elements
        if num < min {
            min = num
        }
    }
    return min
}

func main() {
    nums := []int{45, 23, 67, 12, 89, 34, 56}
    fmt.Println(findMin(nums)) // 12
}
```

**Why Ω(n)?** — The minimum could be ANY element. You cannot guarantee finding it without checking all `n` elements. This is both O(n) and Ω(n), so it's Θ(n).

```
  Finding minimum — must check EVERY element:

  [45, 23, 67, 12, 89, 34, 56]
   ✓   ✓   ✓   ✓   ✓   ✓   ✓   ← all checked
                  ↑
               min=12

  Can you skip element 3? NO!
  What if you skip it and 12 was the minimum? You'd miss it.

  ┌─────────────────────────────────────────┐
  │ Lower bound: Ω(n) — must look at all n   │
  │ Upper bound: O(n) — single pass suffices  │
  │ Tight bound: Θ(n) — both match!           │
  └─────────────────────────────────────────┘
```

---

## Example 3: Ω(n log n) — Comparison Sort Lower Bound

```go
package main

import (
    "fmt"
    "sort"
)

// Any comparison-based sorting algorithm is Ω(n log n)
// This is a proven lower bound — you CAN'T sort faster with comparisons
func comparisonSort(nums []int) {
    sort.Ints(nums) // Go's sort: O(n log n), Ω(n log n) → Θ(n log n)
}

// Decision tree argument:
// n! possible permutations → binary tree needs height ≥ log₂(n!)
// By Stirling's approximation: log₂(n!) ≈ n log₂(n)
// Therefore: minimum comparisons = Ω(n log n)

func main() {
    nums := []int{38, 27, 43, 3, 9, 82, 10}
    comparisonSort(nums)
    fmt.Println(nums) // [3 9 10 27 38 43 82]

    // Even if input is almost sorted, merge sort still does Ω(n log n)
    almost := []int{1, 2, 3, 5, 4}
    comparisonSort(almost)
    fmt.Println(almost) // [1 2 3 4 5]
}
```

```
  Comparison Sort Lower Bound Proof — Decision Tree:

  For n=3 elements, there are 3! = 6 possible orderings:
  (a<b<c, a<c<b, b<a<c, b<c<a, c<a<b, c<b<a)

  Each comparison is a YES/NO branch:

                   a < b ?
                 /         \
              yes            no
            b < c ?         a < c ?
           /     \         /     \
        [a,b,c]  a<c?   [b,a,c]  b<c?
                 / \             / \
          [a,c,b] [c,a,b] [b,c,a] [c,b,a]

  6 leaf nodes (outcomes) requires
  at least ⌈log₂(6)⌉ = 3 levels → 3 comparisons minimum

  For n elements: ⌈log₂(n!)⌉ ≈ n log₂ n comparisons → Ω(n log n)
```

```go
package main

import "fmt"

// O(n) — upper bound (worst case: element not found)
// Ω(1) — lower bound (best case: element at index 0)
func linearSearch(nums []int, target int) int {
    for i, num := range nums {
        if num == target {
            return i
        }
    }
    return -1
}

func main() {
    nums := []int{10, 20, 30, 40, 50}

    // Best case: Ω(1) — found immediately
    fmt.Println(linearSearch(nums, 10)) // 0

    // Worst case: O(n) — must scan entire array
    fmt.Println(linearSearch(nums, 50)) // 4
    fmt.Println(linearSearch(nums, 99)) // -1

    // Ω(1) and O(n), but NOT Θ(anything) — bounds don't match
}
```

```
  Linear Search — Ω(1), O(n):

  [10, 20, 30, 40, 50]

  Search for 10:  ✓  (1 check)   → best case Ω(1)
                  ↑
  Search for 50:  ✗  ✗  ✗  ✗  ✓  (5 checks)  → worst case O(n)
                  ↑  ↑  ↑  ↑  ↑

  Operations
  ▲
  │            ╱ O(n) upper bound (worst case)
  │          ╱
  │        ╱
  │  •   ╱       possible actual runtime
  │    ╱       is anywhere in this range
  │  ╱  •
  │╱     •
  │●────●───●  Ω(1) lower bound (best case)
  └────────────────▶ n
```

```go
package main

import "fmt"

// Best case: Ω(1) — target is at the middle
// Worst case: O(log n) — target is not present
func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid // could return on FIRST check → Ω(1)
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}

func main() {
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15}

    // Best case: target at middle → Ω(1)
    fmt.Println(binarySearch(nums, 7)) // 3 (middle of 0..7)

    // Worst case: O(log n) steps
    fmt.Println(binarySearch(nums, 15)) // 7
    fmt.Println(binarySearch(nums, 6))  // -1
}
```

---

## Example 6: Ω(n²) — Matrix Traversal

```go
package main

import "fmt"

// Ω(n²) — must visit every cell in an n×n matrix
func sumMatrix(matrix [][]int) int {
    total := 0
    for i := range matrix {           // n rows
        for j := range matrix[i] {    // n cols
            total += matrix[i][j]     // every cell
        }
    }
    return total
}

func main() {
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    fmt.Println(sumMatrix(matrix)) // 45
    // Must look at all n² cells → Ω(n²)
    // Also O(n²) → so it's Θ(n²)
}
```

```
  Matrix traversal — every cell must be visited:

  ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │  ← row 0: 3 cells
  ├───┼───┼───┤
  │ 4 │ 5 │ 6 │  ← row 1: 3 cells
  ├───┼───┼───┤
  │ 7 │ 8 │ 9 │  ← row 2: 3 cells
  └───┴───┴───┘
  Total: 9 = n² cells, all must be read for sum.

  Could you skip a cell? NO!
  Every cell contributes to the sum.
  → Must visit all n² cells → Ω(n²)
```

```go
package main

import "fmt"

// Quick sort:
// Best case:    Θ(n log n) — balanced partitions
// Average case: Θ(n log n) — statistically likely
// Worst case:   Θ(n²) — already sorted + bad pivot
// Lower bound:  Ω(n log n) — must do at least n log n comparisons on average
func quickSort(nums []int, low, high int) {
    if low < high {
        pivot := partition(nums, low, high)
        quickSort(nums, low, pivot-1)
        quickSort(nums, pivot+1, high)
    }
}

func partition(nums []int, low, high int) int {
    pivot := nums[high]
    i := low - 1
    for j := low; j < high; j++ {
        if nums[j] <= pivot {
            i++
            nums[i], nums[j] = nums[j], nums[i]
        }
    }
    nums[i+1], nums[high] = nums[high], nums[i+1]
    return i + 1
}

func main() {
    // Average case: Ω(n log n) — balanced-ish partitions
    nums := []int{10, 7, 8, 9, 1, 5}
    quickSort(nums, 0, len(nums)-1)
    fmt.Println(nums) // [1 5 7 8 9 10]

    // Worst case: O(n²) — already sorted, last element as pivot
    sorted := []int{1, 2, 3, 4, 5, 6, 7, 8}
    quickSort(sorted, 0, len(sorted)-1)
    fmt.Println(sorted) // [1 2 3 4 5 6 7 8]
}
```

```
  Quick Sort — Best vs Worst Partition:

  Best case (balanced split):
           [1,5,7,8,9,10]
           /            \
       [1,5,7]       [9,10]
       /    \          \
     [1]  [5,7]       [10]
  → log n levels, n work per level → Ω(n log n)

  Worst case (already sorted, pivot=last):
  [1,2,3,4,5,6,7,8]
  [1,2,3,4,5,6,7] | [8]      pivot=8
  [1,2,3,4,5,6] | [7]        pivot=7
  [1,2,3,4,5] | [6]          pivot=6
  ...                         → n levels!
  → n levels, n work per level → O(n²)

  Summary:  Ω(n log n) ≤ actual ≤ O(n²)
```

```go
package main

import "fmt"

// Problem: Does an unsorted array contain a duplicate?
// Lower bound: Ω(n) — you must look at every element at least once
//
// Proof: If you skip element at index k, and it was the only duplicate,
// you'd miss it. Therefore, Ω(n) is a lower bound.

// O(n) solution using hash set — this is optimal!
func containsDuplicate(nums []int) bool {
    seen := make(map[int]bool)
    for _, num := range nums {
        if seen[num] {
            return true
        }
        seen[num] = true
    }
    return false
}

// O(n²) solution — correct but NOT optimal (above lower bound)
func containsDuplicateSlow(nums []int) bool {
    for i := 0; i < len(nums); i++ {
        for j := i + 1; j < len(nums); j++ {
            if nums[i] == nums[j] {
                return true
            }
        }
    }
    return false
}

func main() {
    nums1 := []int{1, 2, 3, 1}
    nums2 := []int{1, 2, 3, 4}

    fmt.Println(containsDuplicate(nums1))     // true  — O(n) ✓ optimal
    fmt.Println(containsDuplicate(nums2))     // false
    fmt.Println(containsDuplicateSlow(nums1)) // true  — O(n²) ✗ not optimal
}
```

---

## Example 9: Ω in Recursive Algorithms

```go
package main

import "fmt"

// Fibonacci: two recursive branches
// Each call does at least Ω(1) work
// There are at least Ω(2^(n/2)) calls → Ω(2^(n/2))
// Upper bound: O(2ⁿ)
func fib(n int, calls *int) int {
    *calls++
    if n <= 1 {
        return n
    }
    return fib(n-1, calls) + fib(n-2, calls)
}

func main() {
    for _, n := range []int{5, 10, 15, 20, 25} {
        calls := 0
        result := fib(n, &calls)
        fmt.Printf("fib(%2d) = %-10d calls = %d\n", n, result, calls)
    }
    // The calls grow exponentially — Ω(2^(n/2))
}
```

---

## Example 10: Practical Significance of Lower Bounds

```go
package main

import "fmt"

// Lower bounds tell us when our algorithm IS already optimal
// and we can't do better

func main() {
    problems := []struct {
        Problem    string
        LowerBound string
        BestKnown  string
        Optimal    string
    }{
        {"Find min in unsorted array", "Ω(n)", "O(n)", "✓ Yes"},
        {"Sort with comparisons", "Ω(n log n)", "O(n log n)", "✓ Yes"},
        {"Find element in sorted array", "Ω(log n)", "O(log n)", "✓ Yes"},
        {"Matrix multiplication", "Ω(n²)", "O(n^2.37)", "✗ Gap exists"},
        {"Substring search", "Ω(n + m)", "O(n + m)", "✓ Yes (KMP)"},
        {"All pairs shortest path", "Ω(n²)", "O(n³)", "✗ Gap exists"},
    }

    fmt.Printf("%-35s %-15s %-15s %-10s\n",
        "Problem", "Lower Bound", "Best Known", "Optimal?")
    fmt.Println("-----------------------------------------------------------------------")
    for _, p := range problems {
        fmt.Printf("%-35s %-15s %-15s %-10s\n",
            p.Problem, p.LowerBound, p.BestKnown, p.Optimal)
    }
}
```

---

## Key Takeaways

1. **Ω(g(n)) = lower bound**: algorithm takes AT LEAST g(n) operations
2. **Ω tells you what's IMPOSSIBLE to beat** for a given problem
3. **If your algorithm matches the lower bound**, it's **optimal**
4. **Comparison sorting is Ω(n log n)** — proven, can't do better with comparisons
5. **Finding an element is Ω(n) in unsorted** arrays — must check everything
6. **O(n) = Ω(1) for search** means there's a gap → no tight bound
7. **When O = Ω, you have Θ** (tight bound)

> **Next up:** Amortized Complexity →
