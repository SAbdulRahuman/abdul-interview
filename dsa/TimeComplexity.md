# Phase 1: Algorithm Complexity — Time Complexity

## What is Time Complexity?

Time complexity describes **how the runtime of an algorithm grows** as the input size (`n`) increases. We use **Big O notation** to express the worst-case upper bound.

It does **not** measure actual seconds — it measures the **rate of growth** of operations.

---

## Big O Notation Cheat Sheet (Best → Worst)

| Big O | Name | Example |
|-------|------|---------|
| O(1) | Constant | Array index access |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Loop through array |
| O(n log n) | Linearithmic | Merge sort, Quick sort (avg) |
| O(n²) | Quadratic | Nested loops |
| O(n³) | Cubic | Triple nested loops |
| O(2ⁿ) | Exponential | Recursive Fibonacci |
| O(n!) | Factorial | Permutations |

---

## Rules for Calculating Big O

1. **Drop constants**: O(2n) → O(n)
2. **Drop lower-order terms**: O(n² + n) → O(n²)
3. **Different inputs → different variables**: Two separate arrays → O(a + b), not O(n)
4. **Nested loops multiply**: Loop inside loop → O(n × m)
5. **Sequential steps add**: One loop then another → O(n + m)

---

## Example 1: O(1) — Constant Time

The number of operations **does not change** regardless of input size.

```go
package main

import "fmt"

// O(1) - Constant time
// No matter how big the array is, accessing by index is always one operation
func getFirst(nums []int) int {
    return nums[0]
}

func main() {
    nums := []int{10, 20, 30, 40, 50}
    fmt.Println(getFirst(nums)) // 10
}
```

**Why O(1)?** — We perform exactly **1 operation** regardless of whether the slice has 5 elements or 5 million. Array index access is a direct memory offset calculation.

**More O(1) examples:**
- Map lookup: `val := m["key"]`
- Stack push/pop
- Checking if a number is even: `n % 2 == 0`

---

## Example 2: O(n) — Linear Time

The number of operations grows **proportionally** to the input size.

```go
package main

import "fmt"

// O(n) - Linear time
// We visit every element exactly once
func sum(nums []int) int {
    total := 0
    for _, num := range nums { // runs n times
        total += num           // O(1) per iteration
    }
    return total
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println(sum(nums)) // 15
}
```

**Why O(n)?** — The loop runs `n` times (once per element). If the array doubles in size, the runtime roughly doubles.

---

## Example 3: O(n) — Linear Search

```go
package main

import "fmt"

// O(n) - Linear time (worst case)
// In the worst case, the target is the last element or not present
func linearSearch(nums []int, target int) int {
    for i, num := range nums { // worst case: n iterations
        if num == target {
            return i
        }
    }
    return -1
}

func main() {
    nums := []int{5, 3, 8, 1, 9, 2, 7}
    fmt.Println(linearSearch(nums, 7))  // 6
    fmt.Println(linearSearch(nums, 99)) // -1
}
```

**Why O(n)?** — Best case is O(1) (found at index 0), but **Big O measures worst case**. The target might be at the end or not exist, requiring us to check all `n` elements.

---

## Example 4: O(n²) — Quadratic Time (Nested Loops)

The number of operations grows as the **square** of the input size.

```go
package main

import "fmt"

// O(n²) - Quadratic time
// Bubble Sort: compare every pair, swap if needed
func bubbleSort(nums []int) []int {
    n := len(nums)
    for i := 0; i < n; i++ {           // outer loop: n times
        for j := 0; j < n-1-i; j++ {   // inner loop: ~n times
            if nums[j] > nums[j+1] {
                nums[j], nums[j+1] = nums[j+1], nums[j]
            }
        }
    }
    return nums
}

func main() {
    nums := []int{5, 3, 8, 1, 2}
    fmt.Println(bubbleSort(nums)) // [1 2 3 5 8]
}
```

**Why O(n²)?** — Two nested loops each running ~n times. Total operations ≈ n × n = n². If `n = 10` → 100 ops. If `n = 1000` → 1,000,000 ops. It grows **very fast**.

**Breakdown:**
```
Outer loop runs: n times
  Inner loop runs: (n-1) + (n-2) + ... + 1 = n(n-1)/2 ≈ n²/2
Drop constant → O(n²)
```

---

## Example 5: O(log n) — Logarithmic Time

The input is **halved** at each step.

```go
package main

import "fmt"

// O(log n) - Logarithmic time
// Binary Search: eliminate half the array each step
func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {               // runs log₂(n) times
        mid := left + (right-left)/2  // avoid overflow

        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1            // discard left half
        } else {
            right = mid - 1           // discard right half
        }
    }
    return -1
}

func main() {
    // Array MUST be sorted for binary search
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15}
    fmt.Println(binarySearch(nums, 7))  // 3
    fmt.Println(binarySearch(nums, 6))  // -1
}
```

**Why O(log n)?** — Each iteration cuts the search space in half.

```
n = 16 → 8 → 4 → 2 → 1  (4 steps = log₂(16))
n = 1,000,000 → only ~20 steps!
```

**Key insight:** Any time you see "halving" or "doubling", think O(log n).

---

## Example 6: O(n log n) — Linearithmic Time

Divide the problem in half (log n) but process all elements at each level (n).

```go
package main

import "fmt"

// O(n log n) - Linearithmic time
// Merge Sort: divide array in half recursively, then merge
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }

    mid := len(nums) / 2
    left := mergeSort(nums[:mid])   // T(n/2)
    right := mergeSort(nums[mid:])  // T(n/2)

    return merge(left, right)       // O(n) merge step
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0

    for i < len(left) && j < len(right) { // O(n) - visit each element once
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }

    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}

func main() {
    nums := []int{38, 27, 43, 3, 9, 82, 10}
    fmt.Println(mergeSort(nums)) // [3 9 10 27 38 43 82]
}
```

**Why O(n log n)?**

```
Level 0:  [8 elements]              → merge all n elements
Level 1:  [4] [4]                   → merge all n elements  
Level 2:  [2] [2] [2] [2]          → merge all n elements
Level 3:  [1] [1] [1] [1] [1]...  → merge all n elements

Levels = log₂(n), work per level = O(n)
Total = O(n × log n)
```

---

## Example 7: O(2ⁿ) — Exponential Time

Each call **branches into two** recursive calls.

```go
package main

import "fmt"

// O(2ⁿ) - Exponential time
// Naive recursive Fibonacci
func fib(n int) int {
    if n <= 1 {     // base case
        return n
    }
    return fib(n-1) + fib(n-2) // two recursive calls each time
}

func main() {
    fmt.Println(fib(10)) // 55
    fmt.Println(fib(20)) // 6765
    // fib(50) would take VERY long — don't try it!
}
```

**Why O(2ⁿ)?** — Each call spawns 2 more calls, creating a binary tree of calls:

```
                fib(5)
              /        \
          fib(4)       fib(3)
         /    \        /    \
      fib(3) fib(2) fib(2) fib(1)
      /   \
   fib(2) fib(1)
```

```
n = 10  → ~1,024 calls
n = 20  → ~1,048,576 calls
n = 50  → ~1,125,899,906,842,624 calls  ← impossibly slow!
```

**Fix:** Use memoization (Dynamic Programming) to bring it down to O(n):

```go
// O(n) - with memoization
func fibDP(n int) int {
    if n <= 1 {
        return n
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 0, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}
```

---

## Example 8: O(n³) — Cubic Time

Three nested loops — common in matrix multiplication and brute-force 3-sum.

```go
package main

import "fmt"

// O(n³) - Cubic time
// Find all unique triplets that sum to a target
func threeSumBruteForce(nums []int, target int) [][]int {
    n := len(nums)
    var result [][]int

    for i := 0; i < n; i++ {                  // n
        for j := i + 1; j < n; j++ {          // n
            for k := j + 1; k < n; k++ {      // n
                if nums[i]+nums[j]+nums[k] == target {
                    result = append(result, []int{nums[i], nums[j], nums[k]})
                }
            }
        }
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6}
    fmt.Println(threeSumBruteForce(nums, 10))
    // [[1 3 6] [1 4 5] [2 3 5]]
}
```

**Why O(n³)?** — Three nested loops → n × n × n. For `n = 100` → 1,000,000 operations. For `n = 1000` → 1,000,000,000 — very slow.

> **Interview tip:** 3Sum can be solved in O(n²) using sorting + two pointers!

---

## Example 9: O(n!) — Factorial Time

Generating all permutations.

```go
package main

import "fmt"

// O(n!) - Factorial time
// Generate all permutations of a slice
func permutations(nums []int, start int, result *[][]int) {
    if start == len(nums)-1 {
        perm := make([]int, len(nums))
        copy(perm, nums)
        *result = append(*result, perm)
        return
    }

    for i := start; i < len(nums); i++ {    // n choices, then n-1, then n-2...
        nums[start], nums[i] = nums[i], nums[start]  // swap
        permutations(nums, start+1, result)           // recurse
        nums[start], nums[i] = nums[i], nums[start]  // backtrack
    }
}

func main() {
    nums := []int{1, 2, 3}
    var result [][]int
    permutations(nums, 0, &result)

    for _, perm := range result {
        fmt.Println(perm)
    }
    // [1 2 3] [1 3 2] [2 1 3] [2 3 1] [3 2 1] [3 1 2]
}
```

**Why O(n!)?** — At each level, we have fewer choices:

```
Level 0: 3 choices
Level 1: 2 choices  
Level 2: 1 choice
Total = 3 × 2 × 1 = 3! = 6 permutations
```

```
n = 5   → 120 permutations
n = 10  → 3,628,800 permutations
n = 20  → 2,432,902,008,176,640,000 ← impossible
```

---

## Example 10: O(√n) — Square Root Time

Check divisors only up to the square root.

```go
package main

import (
    "fmt"
    "math"
)

// O(√n) - Square root time
// Check if a number is prime
func isPrime(n int) bool {
    if n < 2 {
        return false
    }
    // Only need to check up to √n
    // If n has a factor > √n, it must also have one < √n
    for i := 2; i <= int(math.Sqrt(float64(n))); i++ {
        if n%i == 0 {
            return false
        }
    }
    return true
}

func main() {
    fmt.Println(isPrime(2))   // true
    fmt.Println(isPrime(17))  // true
    fmt.Println(isPrime(100)) // false
    fmt.Println(isPrime(997)) // true
}
```

**Why O(√n)?** — The loop runs from 2 to √n. For `n = 1,000,000`, we only check ~1,000 numbers instead of 1,000,000.

---

## Example 11: O(n + m) — Two Independent Inputs

When you have **two different inputs**, use separate variables.

```go
package main

import "fmt"

// O(n + m) - where n = len(a), m = len(b)
// Merge two sorted arrays into one sorted array
func mergeSorted(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0

    for i < len(a) && j < len(b) { // O(n + m)
        if a[i] <= b[j] {
            result = append(result, a[i])
            i++
        } else {
            result = append(result, b[j])
            j++
        }
    }

    result = append(result, a[i:]...) // remaining from a
    result = append(result, b[j:]...) // remaining from b
    return result
}

func main() {
    a := []int{1, 3, 5, 7}
    b := []int{2, 4, 6, 8, 10}
    fmt.Println(mergeSorted(a, b)) // [1 2 3 4 5 6 7 8 10]
}
```

**Why O(n + m)?** — We iterate through both arrays once. Each element is visited exactly once. This is **NOT** O(n) because the arrays can be different sizes.

---

## Example 12: O(n × m) — Nested Loops with Different Inputs

```go
package main

import "fmt"

// O(n × m) - where n = rows, m = cols
// Print every cell in a 2D matrix
func printMatrix(matrix [][]int) {
    for i := 0; i < len(matrix); i++ {          // n rows
        for j := 0; j < len(matrix[i]); j++ {   // m cols
            fmt.Printf("%d ", matrix[i][j])
        }
        fmt.Println()
    }
}

func main() {
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }
    printMatrix(matrix)
}
```

**Why O(n × m)?** — The outer loop runs `n` times, the inner loop runs `m` times for each outer iteration. If rows ≠ cols, it's O(n × m), not O(n²).

---

## Example 13: Amortized O(1) — Dynamic Array (Slice Append)

```go
package main

import "fmt"

// Amortized O(1) per append
// Go slices double in capacity when full
func buildSlice(n int) []int {
    var nums []int // starts with cap = 0

    for i := 0; i < n; i++ {
        nums = append(nums, i) // usually O(1), occasionally O(n) when resizing
    }
    return nums
}

func main() {
    nums := buildSlice(10)
    fmt.Println(nums) // [0 1 2 3 4 5 6 7 8 9]
}
```

**Why Amortized O(1)?**

Most appends are O(1). When the slice is full, Go allocates a new, larger backing array and copies everything — that single append is O(n). But this happens so rarely that **averaged over all operations**, each append is O(1).

```
Capacity growth: 0 → 1 → 2 → 4 → 8 → 16 → 32 ...
Copies happen:      1   2   4   8   16  (total copies ≈ 2n)
Average per op:  2n / n = O(1) amortized
```

---

## Common Algorithm Complexities — Summary

| Algorithm | Time | Space |
|-----------|------|-------|
| Array access | O(1) | O(1) |
| Hash map get/set | O(1) avg | O(n) |
| Binary search | O(log n) | O(1) |
| Linear search | O(n) | O(1) |
| Merge sort | O(n log n) | O(n) |
| Quick sort | O(n log n) avg | O(log n) |
| Bubble sort | O(n²) | O(1) |
| Fibonacci (naive) | O(2ⁿ) | O(n) |
| Permutations | O(n!) | O(n!) |

---

## How to Identify Time Complexity — Quick Rules

| Pattern | Complexity |
|---------|-----------|
| No loops, direct access | O(1) |
| Single loop over n | O(n) |
| Loop that halves/doubles | O(log n) |
| Nested loop (same input) | O(n²) |
| Nested loop (different inputs) | O(n × m) |
| Divide & conquer + merge | O(n log n) |
| Two branches per recursive call | O(2ⁿ) |
| Generating all arrangements | O(n!) |

---

## Practice: Analyze These

Try to determine the time complexity before checking the answers.

```go
// Problem 1: What is the time complexity?
func mystery1(n int) int {
    count := 0
    i := 1
    for i < n {
        count++
        i *= 2  // doubles each time
    }
    return count
}
// Answer: O(log n) — i doubles each iteration, so loop runs log₂(n) times


// Problem 2: What is the time complexity?
func mystery2(nums []int) int {
    count := 0
    for i := 0; i < len(nums); i++ {
        for j := i; j < len(nums); j++ { // starts at i, not 0
            count++
        }
    }
    return count
}
// Answer: O(n²) — inner loop runs n + (n-1) + (n-2) + ... + 1 = n(n+1)/2 ≈ n²/2 → O(n²)


// Problem 3: What is the time complexity?
func mystery3(n int) int {
    if n <= 0 {
        return 0
    }
    return mystery3(n/3) + mystery3(n/3) // two calls, each with n/3
}
// Answer: O(n^(log₃2)) ≈ O(n^0.63) — Master theorem: a=2, b=3, work per level = O(1)
```

---

## Key Takeaways

1. **Big O measures growth rate**, not exact time
2. **Always analyze the worst case** unless told otherwise
3. **Drop constants and lower terms** — O(3n + 100) = O(n)
4. **Different inputs = different variables** — O(a + b) or O(a × b)
5. **Recursive algorithms**: draw the call tree, count total nodes
6. **Know your common complexities**: O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)

> **Next up:** Space Complexity →
