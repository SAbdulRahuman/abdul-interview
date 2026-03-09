# Phase 1: Algorithm Complexity — Big Theta Notation

## What is Big Theta (Θ) Notation?

Big Theta (Θ) describes the **tight bound** — an algorithm's growth rate is **both upper-bounded AND lower-bounded** by the same function.

**f(n) = Θ(g(n))** means there exist constants `c₁, c₂ > 0` and `n₀` such that:
```
c₁ · g(n) ≤ f(n) ≤ c₂ · g(n)   for all n ≥ n₀
```

In plain English: **"f(n) grows at exactly the same rate as g(n)"** — not faster, not slower.

---

## Big O vs Big Θ vs Big Ω — Quick Comparison

| Notation | Meaning | Analogy |
|----------|---------|---------|
| O(g(n)) | Upper bound (at most) | ≤ |
| Θ(g(n)) | Tight bound (exactly) | = |
| Ω(g(n)) | Lower bound (at least) | ≥ |

---

## Example 1: Θ(n) — Always Linear

```go
package main

import "fmt"

// Θ(n) — always visits every element exactly once
// Best case = Worst case = Average case = n operations
func sumAll(nums []int) int {
    total := 0
    for _, num := range nums { // always n iterations
        total += num
    }
    return total
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    fmt.Println(sumAll(nums)) // 55
}
```

**Why Θ(n)?** — The loop ALWAYS runs exactly `n` times regardless of input values. There's no best/worst case variation. The lower bound is n, the upper bound is n → tight bound is Θ(n).

---

## Example 2: O(n) but NOT Θ(n) — Linear Search

```go
package main

import "fmt"

// O(n) but NOT Θ(n)
// Best case: O(1) — found at first position
// Worst case: O(n) — found at last position or not found
// Since best ≠ worst, cannot say Θ(n)
func linearSearch(nums []int, target int) int {
    for i, num := range nums {
        if num == target {
            return i // could return on first iteration!
        }
    }
    return -1
}

func main() {
    nums := []int{10, 20, 30, 40, 50}

    // Best case: target at index 0 → O(1) operations
    fmt.Println(linearSearch(nums, 10)) // 0

    // Worst case: target at end → O(n) operations
    fmt.Println(linearSearch(nums, 50)) // 4

    // Worst case: target not found → O(n) operations
    fmt.Println(linearSearch(nums, 99)) // -1
}
```

**Key insight:** Linear search is **O(n)** (upper bound) and **Ω(1)** (lower bound). Since the bounds don't match, we **cannot** say it's Θ(n).

---

## Example 3: Θ(n²) — Bubble Sort Always Does n² Work

```go
package main

import "fmt"

// Θ(n²) — this unoptimized version always does n² comparisons
func bubbleSortUnoptimized(nums []int) {
    n := len(nums)
    for i := 0; i < n; i++ {
        for j := 0; j < n-1; j++ { // always runs fully
            if nums[j] > nums[j+1] {
                nums[j], nums[j+1] = nums[j+1], nums[j]
            }
        }
    }
}

func main() {
    // Already sorted — still does n² comparisons
    sorted := []int{1, 2, 3, 4, 5}
    bubbleSortUnoptimized(sorted)
    fmt.Println(sorted)

    // Reverse sorted — still does n² comparisons
    reverse := []int{5, 4, 3, 2, 1}
    bubbleSortUnoptimized(reverse)
    fmt.Println(reverse)
}
```

---

## Example 4: O(n²) but Θ varies — Optimized Bubble Sort

```go
package main

import "fmt"

// Best case: Θ(n) — already sorted, swapped flag exits early
// Worst case: Θ(n²) — reverse sorted
// Overall: O(n²), Ω(n)
func bubbleSortOptimized(nums []int) int {
    n := len(nums)
    comparisons := 0

    for i := 0; i < n; i++ {
        swapped := false
        for j := 0; j < n-1-i; j++ {
            comparisons++
            if nums[j] > nums[j+1] {
                nums[j], nums[j+1] = nums[j+1], nums[j]
                swapped = true
            }
        }
        if !swapped { // no swaps → array is sorted
            break
        }
    }
    return comparisons
}

func main() {
    // Best case: already sorted → ~n comparisons
    sorted := []int{1, 2, 3, 4, 5}
    c1 := bubbleSortOptimized(sorted)
    fmt.Printf("Sorted input: %d comparisons\n", c1) // 4

    // Worst case: reverse sorted → ~n² comparisons
    reverse := []int{5, 4, 3, 2, 1}
    c2 := bubbleSortOptimized(reverse)
    fmt.Printf("Reverse input: %d comparisons\n", c2) // 10
}
```

---

## Example 5: Θ(n log n) — Merge Sort (Always)

```go
package main

import "fmt"

// Θ(n log n) — merge sort ALWAYS divides and merges
// Best, average, worst are all n log n
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSort(nums[:mid])
    right := mergeSort(nums[mid:])
    return merge(left, right)
}

func merge(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i])
            i++
        } else {
            result = append(result, b[j])
            j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}

func main() {
    // Regardless of input order, merge sort always does n log n work
    nums1 := []int{1, 2, 3, 4, 5}         // already sorted
    nums2 := []int{5, 4, 3, 2, 1}         // reverse sorted
    nums3 := []int{3, 1, 4, 1, 5, 9, 2}   // random

    fmt.Println(mergeSort(nums1)) // [1 2 3 4 5]
    fmt.Println(mergeSort(nums2)) // [1 2 3 4 5]
    fmt.Println(mergeSort(nums3)) // [1 1 2 3 4 5 9]
}
```

**Why Θ(n log n)?** — Merge sort always divides (log n levels) and always merges all elements (n per level), regardless of input order.

---

## Example 6: Θ(1) — Direct Computation

```go
package main

import "fmt"

// Θ(1) — always exactly the same number of operations
func triangleArea(base, height float64) float64 {
    return 0.5 * base * height // always 2 operations
}

// Θ(1) — arithmetic sum formula
func sumFirstN(n int) int {
    return n * (n + 1) / 2 // always 3 operations, regardless of n
}

// Θ(1) — array access by index
func getElement(nums []int, i int) int {
    return nums[i] // direct memory access
}

func main() {
    fmt.Println(triangleArea(10, 5)) // 25

    fmt.Println(sumFirstN(100))    // 5050
    fmt.Println(sumFirstN(1000000)) // 500000500000

    nums := []int{10, 20, 30, 40, 50}
    fmt.Println(getElement(nums, 3)) // 40
}
```

---

## Example 7: Θ(log n) — Binary GCD (Always Logarithmic)

```go
package main

import "fmt"

// Θ(log(min(a,b))) — Euclidean GCD
// Always takes logarithmic steps (proven by Fibonacci worst case)
func gcd(a, b int) int {
    steps := 0
    for b != 0 {
        a, b = b, a%b
        steps++
    }
    fmt.Printf("  gcd steps: %d\n", steps)
    return a
}

func main() {
    fmt.Println(gcd(48, 18))       // steps ~3, result 6
    fmt.Println(gcd(100, 75))      // result 25
    fmt.Println(gcd(1000000, 999)) // result 999
}
```

---

## Example 8: Demonstrating Tight Bounds Empirically

```go
package main

import "fmt"

// Count exact operations for different algorithms
func countOpsLinear(n int) int {
    ops := 0
    for i := 0; i < n; i++ {
        ops++
    }
    return ops // always exactly n → Θ(n)
}

func countOpsQuadratic(n int) int {
    ops := 0
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            ops++
        }
    }
    return ops // always exactly n² → Θ(n²)
}

func countOpsTriangular(n int) int {
    ops := 0
    for i := 0; i < n; i++ {
        for j := 0; j <= i; j++ {
            ops++
        }
    }
    return ops // always n(n+1)/2 → Θ(n²) (drop constant ½)
}

func main() {
    for _, n := range []int{10, 100, 1000} {
        linear := countOpsLinear(n)
        quad := countOpsQuadratic(n)
        tri := countOpsTriangular(n)

        fmt.Printf("n=%4d | linear=%d | quad=%d | triangular=%d | tri/n²=%.2f\n",
            n, linear, quad, tri, float64(tri)/float64(n*n))
    }
    // tri/n² converges to 0.50 → confirming it's Θ(n²) with constant ~0.5
}
```

---

## Example 9: Θ(n) — Hash Table Iteration

```go
package main

import "fmt"

// Θ(n) — must visit every key-value pair
func sumMapValues(m map[string]int) int {
    total := 0
    for _, v := range m { // always iterates all n entries
        total += v
    }
    return total
}

// Θ(n) — building frequency map always processes all elements
func buildFrequencyMap(words []string) map[string]int {
    freq := make(map[string]int)
    for _, word := range words { // always n iterations
        freq[word]++
    }
    return freq
}

func main() {
    m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4}
    fmt.Println(sumMapValues(m)) // 10

    words := []string{"go", "is", "go", "fun", "go", "is", "great"}
    fmt.Println(buildFrequencyMap(words)) // map[fun:1 go:3 great:1 is:2]
}
```

---

## Example 10: When Θ Notation is Appropriate

```go
package main

import "fmt"

// Algorithms where best = worst = average → use Θ
type Example struct {
    Name      string
    BestCase  string
    WorstCase string
    Theta     string
}

func main() {
    examples := []Example{
        {"Array sum", "Θ(n)", "Θ(n)", "Θ(n) ✓"},
        {"Merge sort", "Θ(n log n)", "Θ(n log n)", "Θ(n log n) ✓"},
        {"Matrix multiply", "Θ(n³)", "Θ(n³)", "Θ(n³) ✓"},
        {"Linear search", "O(1)", "O(n)", "NO Θ — bounds differ"},
        {"Quick sort", "O(n log n)", "O(n²)", "NO Θ — bounds differ"},
        {"Binary search", "O(1)", "O(log n)", "NO Θ — bounds differ"},
        {"Hash lookup", "O(1)", "O(n)", "NO Θ — bounds differ"},
    }

    fmt.Printf("%-20s %-15s %-15s %-25s\n", "Algorithm", "Best", "Worst", "Theta?")
    fmt.Println("----------------------------------------------------------------------")
    for _, e := range examples {
        fmt.Printf("%-20s %-15s %-15s %-25s\n", e.Name, e.BestCase, e.WorstCase, e.Theta)
    }
}
```

---

## Key Takeaways

1. **Θ(g(n)) = tight bound**: growth is EXACTLY g(n), both upper and lower
2. **Use Θ when best case = worst case** (same growth rate)
3. **Merge sort is Θ(n log n)** — always does n log n work
4. **Linear search is NOT Θ(n)** — best case is O(1), so bounds don't match
5. **In interviews**, most people say "O(n)" when they mean "Θ(n)" — that's OK
6. **Θ is more precise** than O: Θ(n) gives more information than O(n)

> **Next up:** Big Omega Notation →
