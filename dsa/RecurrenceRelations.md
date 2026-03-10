# Phase 1: Algorithm Complexity — Recurrence Relations

## What is a Recurrence Relation?

A recurrence relation defines a function in terms of **itself with smaller inputs**. They naturally arise from **recursive algorithms** and help us express and solve time complexity.

**General form:** T(n) = a·T(n/b) + f(n)
- **a** = number of recursive calls
- **n/b** = size of each subproblem
- **f(n)** = work done outside recursion

---

## Example 1: Linear Recursion — T(n) = T(n-1) + O(1)

```go
package main

import "fmt"

// T(n) = T(n-1) + O(1)
// Each call does O(1) work and makes 1 recursive call with n-1
func factorial(n int) int {
    if n <= 1 { // base case: T(0) = T(1) = O(1)
        return 1
    }
    return n * factorial(n-1) // T(n-1) + O(1) multiplication
}

// Unrolling:
// T(n) = T(n-1) + 1
//      = T(n-2) + 1 + 1
//      = T(n-3) + 1 + 1 + 1
//      = ... = T(0) + n
//      = O(n)

func main() {
    for _, n := range []int{1, 5, 10, 15, 20} {
        fmt.Printf("factorial(%2d) = %d\n", n, factorial(n))
    }
}
```

**Solution:** T(n) = O(n)

**Textual Figure — Unrolling T(n) = T(n-1) + O(1):**

```
  Call chain for factorial(5):

  factorial(5)  ─▶  5 × factorial(4)
  factorial(4)  ─▶  4 × factorial(3)
  factorial(3)  ─▶  3 × factorial(2)
  factorial(2)  ─▶  2 × factorial(1)
  factorial(1)  ─▶  1  (base case)

  Work per call: O(1)      (just one multiplication)
  Number of calls: n

  T(n) = T(n-1) + 1
       = T(n-2) + 1 + 1
       = T(n-3) + 1 + 1 + 1
       = ...                      Unrolling k steps:
       = T(n-k) + k
       = T(0) + n  (when k=n)     ┌───────────────┐
       = O(n)                     │  Pattern: n   │
                                  │  constant-    │
  Work ▲                          │  cost steps   │
  1    │ ■ ■ ■ ■ ■               └───────────────┘
       └───────────▶ calls
       1  2  3  4  5
  Total = 1+1+1+1+1 = 5 = n = O(n)
```

---

## Example 2: Binary Recursion — T(n) = 2T(n-1) + O(1)

```go
package main

import "fmt"

// T(n) = 2T(n-1) + O(1)
// Naive Fibonacci: 2 recursive calls, each reducing by 1
func fib(n int) int {
    if n <= 1 {
        return n
    }
    return fib(n-1) + fib(n-2) // 2 branches
}

// Unrolling:
// T(n) = 2T(n-1) + 1
//      = 2(2T(n-2) + 1) + 1 = 4T(n-2) + 3
//      = 4(2T(n-3) + 1) + 3 = 8T(n-3) + 7
//      = 2^k * T(n-k) + (2^k - 1)
// When k = n: T(n) = 2^n * T(0) + 2^n - 1 = O(2^n)

func main() {
    for n := 1; n <= 10; n++ {
        fmt.Printf("fib(%2d) = %d\n", n, fib(n))
    }
}
```

**Solution:** T(n) = O(2ⁿ)

**Textual Figure — Exponential Recursion Tree for fib(5):**

```
  T(n) = 2T(n-1) + O(1) → doubles at each level

                    fib(5)
                   /      \
              fib(4)        fib(3)
             /     \        /    \
         fib(3)  fib(2)  fib(2) fib(1)
         /    \    / \
     fib(2) fib(1) ...    ← Explosive growth!

  Level 0:  1 call        = 2⁰
  Level 1:  2 calls       = 2¹
  Level 2:  4 calls       = 2²
  Level 3:  8 calls       = 2³
  ...         ...           ...
  Level n:  2ⁿ calls      = 2ⁿ

  Total calls: 2⁰ + 2¹ + 2² + ... + 2ⁿ = 2ⁿ⁺¹ - 1 = O(2ⁿ)

  Unrolling:
  T(n) = 2T(n-1) + 1
       = 2[2T(n-2)+1] + 1  = 4T(n-2) + 3
       = 4[2T(n-3)+1] + 3  = 8T(n-3) + 7
       = 2ᵏ·T(n-k) + (2ᵏ - 1)
       = 2ⁿ·T(0) + (2ⁿ - 1) = O(2ⁿ)
```

---

## Example 3: Divide and Conquer — T(n) = 2T(n/2) + O(n)

```go
package main

import "fmt"

// T(n) = 2T(n/2) + O(n) → Merge Sort
// Split into 2 halves (2T(n/2)), merge all elements (O(n))
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSort(nums[:mid])   // T(n/2)
    right := mergeSort(nums[mid:])  // T(n/2)
    return merge(left, right)       // O(n) merge
}

func merge(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i]); i++
        } else {
            result = append(result, b[j]); j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}

func main() {
    nums := []int{38, 27, 43, 3, 9, 82, 10}
    fmt.Println(mergeSort(nums))
    // Recurrence tree:
    // Level 0: n work
    // Level 1: n/2 + n/2 = n work
    // Level 2: n/4 + n/4 + n/4 + n/4 = n work
    // ...log₂(n) levels → Total: n × log n = O(n log n)
}
```

**Solution:** T(n) = O(n log n)

**Textual Figure — Merge Sort Recurrence Tree:**

```
  T(n) = 2T(n/2) + O(n)

  Level 0:      [  n elements  ]           work = n
                /              \
  Level 1:  [n/2]            [n/2]          work = n/2 + n/2 = n
            /    \           /    \
  Level 2: [n/4] [n/4]   [n/4] [n/4]       work = 4 × n/4 = n
            ...    ...    ...    ...
  Level k:  2ᵏ nodes, each size n/2ᵏ        work = 2ᵏ × n/2ᵏ = n
            ...    ...    ...    ...
  Level log n: [1] [1] ... [1] [1]          work = n × 1 = n

  Each level does O(n) total work
  Number of levels: log₂(n)
  ────────────────────────────────
  Total: n × log n = O(n log n)

  Work per level:
  n   │ ███████████████  Level 0
  n   │ ███████████████  Level 1
  n   │ ███████████████  Level 2
  n   │ ███████████████  Level 3 (log n levels)
      └──────────────────
```

---

## Example 4: T(n) = T(n/2) + O(1) — Binary Search

```go
package main

import "fmt"

// T(n) = T(n/2) + O(1)
// 1 recursive call on half the input, O(1) comparison
func binarySearchRec(nums []int, target, left, right int) int {
    if left > right {
        return -1 // base case
    }
    mid := left + (right-left)/2     // O(1) work
    if nums[mid] == target {
        return mid
    } else if nums[mid] < target {
        return binarySearchRec(nums, target, mid+1, right)  // T(n/2)
    }
    return binarySearchRec(nums, target, left, mid-1)        // T(n/2)
}

// Unrolling:
// T(n) = T(n/2) + 1
//      = T(n/4) + 1 + 1
//      = T(n/8) + 1 + 1 + 1
//      = T(n/2^k) + k
// When n/2^k = 1 → k = log₂(n)
// T(n) = O(log n)

func main() {
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15}
    fmt.Println(binarySearchRec(nums, 7, 0, len(nums)-1)) // 3
}
```

**Solution:** T(n) = O(log n)

**Textual Figure — Binary Search Halving:**

```
  T(n) = T(n/2) + O(1)

  Search for 7 in [1, 3, 5, 7, 9, 11, 13, 15]

  Step 1: [1, 3, 5, 7, 9, 11, 13, 15]  mid=7, compare
                       ↑                  n=8 → O(1)
  Step 2: [1, 3, 5, 7]                   mid=3, too small
                    ↑                     n=4 → O(1)
  Step 3: [5, 7]                          mid=5, too small
               ↑                          n=2 → O(1)
  Step 4: [7]                             found!
           ↑                              n=1 → O(1)

  Unrolling:
  T(n) = T(n/2) + 1
       = T(n/4) + 1 + 1
       = T(n/8) + 1 + 1 + 1
       = T(n/2ᵏ) + k
  When n/2ᵏ = 1 → k = log₂(n)
  T(n) = O(log n)

  Search space shrinks:
  n    │ ████████████████
  n/2  │ ████████
  n/4  │ ████
  n/8  │ ██
  1    │ █  → done after log₂(n) steps
       └──────────────────
```

---

## Example 5: T(n) = T(n/2) + O(n) — Build Heap / Select

```go
package main

import "fmt"

// T(n) = T(n/2) + O(n)
// Process all n elements, then recurse on half
func findMedian(nums []int) int {
    // Simplified: process all, then focus on half
    // This is the pattern of quickselect
    return quickSelect(nums, len(nums)/2)
}

func quickSelect(nums []int, k int) int {
    if len(nums) == 1 {
        return nums[0]
    }
    pivot := nums[len(nums)/2]
    var lo, hi, eq []int
    for _, n := range nums { // O(n) work
        if n < pivot {
            lo = append(lo, n)
        } else if n > pivot {
            hi = append(hi, n)
        } else {
            eq = append(eq, n)
        }
    }
    if k < len(lo) {
        return quickSelect(lo, k) // T(n/2) average
    } else if k < len(lo)+len(eq) {
        return pivot
    }
    return quickSelect(hi, k-len(lo)-len(eq))
}

// T(n) = T(n/2) + n
//      = T(n/4) + n/2 + n
//      = T(n/8) + n/4 + n/2 + n
//      = n + n/2 + n/4 + ... + 1 = 2n = O(n)

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6, 5}
    fmt.Println(findMedian(nums))
}
```

**Solution:** T(n) = O(n) — geometric series sums to 2n

**Textual Figure — Geometric Series for T(n) = T(n/2) + n:**

```
  T(n) = T(n/2) + n

  Level 0:  process n elements         work = n
  Level 1:  process n/2 elements       work = n/2
  Level 2:  process n/4 elements       work = n/4
  Level 3:  process n/8 elements       work = n/8
  ...          ...                     ...
  Level k:  process 1 element          work = 1

  Total = n + n/2 + n/4 + n/8 + ... + 1

  Visualizing the geometric series:
  n     │ ████████████████
  n/2   │ ████████
  n/4   │ ████
  n/8   │ ██
  n/16  │ █
        └────────────────
  Sum approaches 2n (geometric series: r=1/2)

  ┌─────────────────────────────────────┐
  │  n × (1 + 1/2 + 1/4 + 1/8 + ...)│
  │  = n × 2 = 2n = O(n)            │
  │                                  │
  │  Key insight: work HALVES each   │
  │  level → dominated by FIRST level│
  └─────────────────────────────────────┘
```

---

## Example 6: T(n) = 3T(n/3) + O(n) — Three-Way Divide

```go
package main

import "fmt"

// T(n) = 3T(n/3) + O(n)
// Divide into 3 parts, O(n) work at each level
func threeWayProcess(nums []int, depth int) int {
    indent := ""
    for i := 0; i < depth; i++ { indent += "  " }

    if len(nums) <= 1 {
        return len(nums)
    }

    fmt.Printf("%sProcessing %d elements\n", indent, len(nums))

    third := len(nums) / 3
    // O(n) work: sum all elements
    sum := 0
    for _, n := range nums { sum += n }

    // 3 recursive calls on n/3 each
    left := threeWayProcess(nums[:third], depth+1)
    mid := threeWayProcess(nums[third:2*third], depth+1)
    right := threeWayProcess(nums[2*third:], depth+1)

    return sum + left + mid + right
}

// T(n) = 3T(n/3) + n
// log₃(3) = 1, f(n) = n = n^1
// By Master theorem: a = b^k → O(n log n)

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
    threeWayProcess(nums, 0)
    fmt.Println("Complexity: O(n log n)")
}
```

**Textual Figure — Three-Way Divide Recurrence Tree:**

```
  T(n) = 3T(n/3) + O(n)

  Level 0:         [  9 elements  ]              work = 9
                 /       |        \
  Level 1:    [3]       [3]       [3]            work = 3+3+3 = 9
             / | \    / | \     / | \
  Level 2: [1][1][1][1][1][1] [1][1][1]          work = 9×1 = 9

  3 branches × n/3 each = n total at every level
  Number of levels: log₃(n)

  Work per level:
  n   │ ███████████████   Level 0
  n   │ ███████████████   Level 1
  n   │ ███████████████   Level 2 (log₃ n levels)
      └──────────────────

  Same pattern as merge sort but base-3:
  a=3, b=3, f(n)=n → log₃(3)=1, n¹=n
  a = bᵏ → O(n log n)
```

---

## Example 7: T(n) = 2T(n/2) + O(1) — Count Nodes

```go
package main

import "fmt"

type Node struct {
    Val   int
    Left  *Node
    Right *Node
}

// T(n) = 2T(n/2) + O(1)
// Visit both children, O(1) work per node
func countNodes(root *Node) int {
    if root == nil {
        return 0 // base case
    }
    return 1 + countNodes(root.Left) + countNodes(root.Right)
    // O(1) + T(n/2) + T(n/2) = 2T(n/2) + O(1)
}

// Unrolling:
// T(n) = 2T(n/2) + 1
// By Master theorem: a=2, b=2, k=0
// log₂(2) = 1 > 0 = k → T(n) = O(n^(log₂2)) = O(n)

func buildTree(vals []int, i int) *Node {
    if i >= len(vals) || vals[i] == 0 { return nil }
    return &Node{
        Val:   vals[i],
        Left:  buildTree(vals, 2*i+1),
        Right: buildTree(vals, 2*i+2),
    }
}

func main() {
    root := buildTree([]int{1, 2, 3, 4, 5, 6, 7}, 0)
    fmt.Printf("Nodes: %d\n", countNodes(root)) // 7
    // T(n) = O(n) — must visit every node
}
```

**Textual Figure — Tree Traversal T(n) = 2T(n/2) + O(1):**

```
  T(n) = 2T(n/2) + O(1)
  Visit each node, O(1) work per node

  Tree:         1          Level 0: 1 node,  O(1) work
               / \
              2   3        Level 1: 2 nodes, O(2) work
             / \ / \
            4  5 6  7      Level 2: 4 nodes, O(4) work

  Level 0:  1 node  × O(1) = O(1)     = 2⁰
  Level 1:  2 nodes × O(1) = O(2)     = 2¹
  Level 2:  4 nodes × O(1) = O(4)     = 2²
  ...           ...         ...
  Level k:  2ᵏ              = 2ᵏ

  Total = 1 + 2 + 4 + ... + n/2 + n
  ┌────────────────────────────────────────┐
  │  = 2ⁿ⁺¹ - 1 = 2n - 1 = O(n)             │
  │                                         │
  │  Key: O(1) per node, n nodes → O(n)      │
  │  Dominated by leaves (bottom level)       │
  └────────────────────────────────────────┘
```

---

## Example 8: T(n) = T(n-1) + O(n) — Selection Sort

```go
package main

import "fmt"

// T(n) = T(n-1) + O(n)
// Find min in n elements (O(n)), then sort remaining n-1
func selectionSortRecursive(nums []int, start int) {
    if start >= len(nums)-1 { return }

    // O(n-start) work: find minimum
    minIdx := start
    for i := start + 1; i < len(nums); i++ {
        if nums[i] < nums[minIdx] {
            minIdx = i
        }
    }
    nums[start], nums[minIdx] = nums[minIdx], nums[start]

    selectionSortRecursive(nums, start+1) // T(n-1)
}

// Unrolling:
// T(n) = T(n-1) + n
//      = T(n-2) + (n-1) + n
//      = T(n-3) + (n-2) + (n-1) + n
//      = 1 + 2 + ... + n = n(n+1)/2 = O(n²)

func main() {
    nums := []int{64, 25, 12, 22, 11}
    selectionSortRecursive(nums, 0)
    fmt.Println(nums) // [11 12 22 25 64]
}
```

**Solution:** T(n) = O(n²) — arithmetic sum

**Textual Figure — Arithmetic Sum for T(n) = T(n-1) + n:**

```
  T(n) = T(n-1) + O(n)

  Selection sort on [64, 25, 12, 22, 11]:

  Call 1: scan 5 elements → find min(11)    work = 5
  Call 2: scan 4 elements → find min(12)    work = 4
  Call 3: scan 3 elements → find min(22)    work = 3
  Call 4: scan 2 elements → find min(25)    work = 2
  Call 5: scan 1 element  → done            work = 1

  Stacking the work:
  5 │ █████
  4 │ ████
  3 │ ███         Total = 5+4+3+2+1
  2 │ ██                 = n(n+1)/2
  1 │ █                  = O(n²)
    └──────────

  This forms a triangle:
  ●  Total area = ½ × base × height = ½ × n × n = O(n²)

  Contrast with T(n) = T(n/2) + n:
  ──────────────────────────────────────
  T(n-1)+n: n + (n-1) + (n-2) + ... → O(n²)  (arithmetic)
  T(n/2)+n: n + n/2 + n/4 + ...     → O(n)    (geometric)
```

---

## Example 9: T(n) = T(n/3) + T(2n/3) + O(n)

```go
package main

import "fmt"

// T(n) = T(n/3) + T(2n/3) + O(n)
// Uneven split — like quicksort with bad but consistent pivot
func unevenDivide(n, depth int) int {
    if n <= 1 { return 1 }

    indent := ""
    for i := 0; i < depth; i++ { indent += "  " }
    fmt.Printf("%sn=%d, splitting into %d and %d\n", indent, n, n/3, 2*n/3)

    left := unevenDivide(n/3, depth+1)     // T(n/3)
    right := unevenDivide(2*n/3, depth+1)  // T(2n/3)
    return n + left + right                 // O(n) combine
}

// At each level, total work = O(n)
// Deepest branch: n → 2n/3 → 4n/9 → 8n/27 → ...
// Depth of longest branch: log₃/₂(n) = O(log n)
// Total: O(n log n)

func main() {
    unevenDivide(27, 0)
    fmt.Println("\nComplexity: O(n log n)")
}
```

**Textual Figure — Uneven Split Recurrence:**

```
  T(n) = T(n/3) + T(2n/3) + O(n)

  Tree is NOT balanced — right side is deeper:

              n=27                          work = 27
             /      \
          n=9       n=18                    work = 9+18 = 27
         /   \      /    \
       n=3   n=6  n=6   n=12               work = 3+6+6+12 = 27
       / \    ...   ...   / \
      1   2             4    8
                              \
                               5
                                \
                                 3 ← deepest path

  Left branch depth:  log₃(n)    (divides by 3)
  Right branch depth: log₃₂(n)  (divides by 3/2)

  Every level still totals O(n)!
  ┌────────────────────────────────────────┐
  │  Depth = longest path = log₃₂(n)      │
  │  Work per level ≤ n                    │
  │  Total = O(n × log₃₂ n) = O(n log n)  │
  └────────────────────────────────────────┘
```

---

## Example 10: Building Recurrence from Code

```go
package main

import "fmt"

// Practice: derive the recurrence relation from code

// Example A: What's the recurrence?
func exampleA(n int) int {
    if n <= 0 { return 1 }
    return exampleA(n-1) + exampleA(n-1)
    // 2 calls with n-1 → T(n) = 2T(n-1) + O(1) → O(2ⁿ)
}

// Example B: What's the recurrence?
func exampleB(nums []int) int {
    if len(nums) <= 1 { return 0 }
    mid := len(nums) / 2
    sum := 0
    for _, n := range nums { sum += n } // O(n) work
    return sum + exampleB(nums[:mid])   // 1 call with n/2
    // T(n) = T(n/2) + O(n) → O(n)
}

// Example C: What's the recurrence?
func exampleC(n int) int {
    if n <= 1 { return 1 }
    return exampleC(n/2) + exampleC(n/2) + exampleC(n/2)
    // 3 calls with n/2 → T(n) = 3T(n/2) + O(1)
    // By Master theorem: log₂(3) ≈ 1.585 → O(n^1.585)
}

func main() {
    fmt.Println("Example A (n=10):", exampleA(10))
    fmt.Println("  Recurrence: T(n) = 2T(n-1) + O(1) → O(2ⁿ)")

    nums := []int{1, 2, 3, 4, 5, 6, 7, 8}
    fmt.Println("Example B:", exampleB(nums))
    fmt.Println("  Recurrence: T(n) = T(n/2) + O(n) → O(n)")

    fmt.Println("Example C (n=8):", exampleC(8))
    fmt.Println("  Recurrence: T(n) = 3T(n/2) + O(1) → O(n^1.585)")
}
```

**Textual Figure — Deriving Recurrences from Code:**

```
  Example A: exampleA(n-1) + exampleA(n-1)
  ──────────────────────────────────────────
  2 calls with (n-1) → T(n) = 2T(n-1) + O(1)
  Pattern: each call DOUBLES, depth = n
  Total: 2ⁿ = O(2ⁿ)

  Example B: sum(nums) + exampleB(nums[:mid])
  ──────────────────────────────────────────
  O(n) loop + 1 call with n/2 → T(n) = T(n/2) + O(n)
  Geometric series: n + n/2 + n/4 + ... = 2n = O(n)

  Example C: 3 × exampleC(n/2)
  ──────────────────────────────────────────
  3 calls with n/2 → T(n) = 3T(n/2) + O(1)
  Master: a=3, b=2 → log₂(3) ≈ 1.585
  T(n) = O(n^1.585)

  How to read code for recurrences:
  ┌─────────────────────────────────────────────┐
  │ 1. Count recursive calls          → a      │
  │ 2. How does input size reduce?    → n/b    │
  │    or n-1?                                  │
  │ 3. What work is done per call?    → f(n)   │
  │ 4. Combine: T(n) = a·T(n/b) + f(n)         │
  └─────────────────────────────────────────────┘
```

---

## Example 11: Common Recurrences Summary

```go
package main

import "fmt"

func main() {
    recurrences := []struct {
        Recurrence string
        Solution   string
        Example    string
    }{
        {"T(n) = T(n-1) + O(1)", "O(n)", "Factorial, linked list traversal"},
        {"T(n) = T(n-1) + O(n)", "O(n²)", "Selection sort, insertion sort"},
        {"T(n) = 2T(n-1) + O(1)", "O(2ⁿ)", "Fibonacci, Tower of Hanoi"},
        {"T(n) = T(n/2) + O(1)", "O(log n)", "Binary search"},
        {"T(n) = T(n/2) + O(n)", "O(n)", "Quickselect (average)"},
        {"T(n) = 2T(n/2) + O(1)", "O(n)", "Tree traversal"},
        {"T(n) = 2T(n/2) + O(n)", "O(n log n)", "Merge sort"},
        {"T(n) = 2T(n/2) + O(n²)", "O(n²)", "Certain matrix algorithms"},
        {"T(n) = 3T(n/2) + O(1)", "O(n^1.585)", "Karatsuba (simplified)"},
        {"T(n) = 7T(n/2) + O(n²)", "O(n^2.807)", "Strassen's matrix multiply"},
    }

    fmt.Printf("%-30s %-15s %-35s\n", "Recurrence", "Solution", "Example")
    fmt.Println("--------------------------------------------------------------------------------")
    for _, r := range recurrences {
        fmt.Printf("%-30s %-15s %-35s\n", r.Recurrence, r.Solution, r.Example)
    }
}
```

**Textual Figure — Recurrence Families:**

```
  Subtract Recurrences (n → n-1):
  ──────────────────────────────────────────────────
  T(n) = T(n-1) + O(1)  →  Depth=n, work=1  →  O(n)
      ●─●─●─●─●─●  (linear chain)

  T(n) = T(n-1) + O(n)  →  Depth=n, work=n  →  O(n²)
      ●─●─●─●─●─●  (triangle sum: n+(n-1)+...+1)

  T(n) = 2T(n-1) + O(1)  →  Depth=n, branches=2  →  O(2ⁿ)
           ●                 (exponential tree)
          / \
         ●   ●
        / \ / \
       ● ● ● ●

  Divide Recurrences (n → n/b):
  ──────────────────────────────────────────────────
  T(n) = T(n/2) + O(1)   →  O(log n)     (binary search)
  T(n) = T(n/2) + O(n)   →  O(n)         (geometric sum)
  T(n) = 2T(n/2) + O(1)  →  O(n)         (tree traversal)
  T(n) = 2T(n/2) + O(n)  →  O(n log n)   (merge sort)
  T(n) = 3T(n/2) + O(1)  →  O(n^1.585)   (branching > splitting)

  Pattern:
  ┌─────────────────────────────────────────────────┐
  │  Subtract (n-1): arithmetic → polynomial/exp    │
  │  Divide (n/b):   geometric  → logarithmic       │
  │                                                  │
  │  More recursive calls (a↑) → faster growth       │
  │  Bigger split (b↑)         → slower growth       │
  │  More per-call work (f↑)   → depends on balance  │
  └─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Recurrence = recursive formula** for an algorithm's running time
2. **T(n) = aT(n/b) + f(n)** is the standard form for divide-and-conquer
3. **Unrolling**: expand the recurrence to spot the pattern
4. **Master theorem** solves T(n) = aT(n/b) + O(nᵏ) directly
5. **Subtract recurrences** (T(n-1)): often O(n) or O(2ⁿ)
6. **Divide recurrences** (T(n/b)): often O(log n) or O(n log n)

> **Next up:** Master Theorem →
