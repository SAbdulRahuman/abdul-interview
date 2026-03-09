# Phase 1: Algorithm Complexity — Master Theorem

## What is the Master Theorem?

The Master Theorem provides a **direct formula** to solve recurrences of the form:

```
T(n) = a · T(n/b) + O(nᵏ)
```

Where:
- **a** = number of subproblems (recursive calls)
- **b** = factor by which input shrinks
- **nᵏ** = work done outside recursion

## The Three Cases

Compare **log_b(a)** with **k**:

| Case | Condition | Solution |
|------|-----------|----------|
| Case 1 | log_b(a) > k | T(n) = O(n^(log_b(a))) |
| Case 2 | log_b(a) = k | T(n) = O(nᵏ · log n) |
| Case 3 | log_b(a) < k | T(n) = O(nᵏ) |

---

## Example 1: Binary Search — T(n) = T(n/2) + O(1)

```go
package main

import "fmt"

// T(n) = 1·T(n/2) + O(n⁰)
// a=1, b=2, k=0
// log₂(1) = 0 = k → Case 2
// Solution: O(n⁰ · log n) = O(log n)

func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}

func main() {
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15, 17, 19}
    fmt.Println(binarySearch(nums, 13)) // 6
    fmt.Println("Master Theorem: a=1, b=2, k=0 → Case 2 → O(log n)")
}
```

---

## Example 2: Merge Sort — T(n) = 2T(n/2) + O(n)

```go
package main

import "fmt"

// T(n) = 2T(n/2) + O(n¹)
// a=2, b=2, k=1
// log₂(2) = 1 = k → Case 2
// Solution: O(n¹ · log n) = O(n log n)

func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSort(nums[:mid])   // T(n/2)
    right := mergeSort(nums[mid:])  // T(n/2)
    return merge(left, right)       // O(n)
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
    fmt.Println("Master Theorem: a=2, b=2, k=1 → Case 2 → O(n log n)")
}
```

---

## Example 3: Tree Traversal — T(n) = 2T(n/2) + O(1)

```go
package main

import "fmt"

// T(n) = 2T(n/2) + O(n⁰)
// a=2, b=2, k=0
// log₂(2) = 1 > 0 = k → Case 1
// Solution: O(n^(log₂2)) = O(n)

type Node struct {
    Val   int
    Left  *Node
    Right *Node
}

func countNodes(root *Node) int {
    if root == nil {
        return 0
    }
    left := countNodes(root.Left)   // T(n/2)
    right := countNodes(root.Right) // T(n/2)
    return 1 + left + right         // O(1) work
}

func main() {
    root := &Node{1,
        &Node{2, &Node{4, nil, nil}, &Node{5, nil, nil}},
        &Node{3, &Node{6, nil, nil}, &Node{7, nil, nil}},
    }
    fmt.Println("Nodes:", countNodes(root)) // 7
    fmt.Println("Master Theorem: a=2, b=2, k=0 → Case 1 → O(n)")
}
```

---

## Example 4: Strassen's Matrix Multiplication — T(n) = 7T(n/2) + O(n²)

```go
package main

import (
    "fmt"
    "math"
)

// T(n) = 7T(n/2) + O(n²)
// a=7, b=2, k=2
// log₂(7) ≈ 2.807 > 2 = k → Case 1
// Solution: O(n^(log₂7)) = O(n^2.807)

// Standard matrix multiply: O(n³)
func matrixMultiply(A, B [][]int) [][]int {
    n := len(A)
    C := make([][]int, n)
    for i := range C {
        C[i] = make([]int, n)
        for j := 0; j < n; j++ {
            for k := 0; k < n; k++ {
                C[i][j] += A[i][k] * B[k][j]
            }
        }
    }
    return C
}

func main() {
    A := [][]int{{1, 2}, {3, 4}}
    B := [][]int{{5, 6}, {7, 8}}
    C := matrixMultiply(A, B)
    fmt.Println(C) // [[19 22] [43 50]]

    fmt.Printf("Standard:  O(n³) = O(n^%.3f)\n", 3.0)
    fmt.Printf("Strassen:  O(n^log₂7) = O(n^%.3f)\n", math.Log2(7))
    fmt.Println("Master Theorem: a=7, b=2, k=2 → Case 1 → O(n^2.807)")
}
```

---

## Example 5: Quickselect Average — T(n) = T(n/2) + O(n)

```go
package main

import "fmt"

// T(n) = T(n/2) + O(n¹)
// a=1, b=2, k=1
// log₂(1) = 0 < 1 = k → Case 3
// Solution: O(n¹) = O(n)

func quickSelect(nums []int, k int) int {
    if len(nums) == 1 {
        return nums[0]
    }
    pivot := nums[len(nums)/2]
    var lo, eq, hi []int
    for _, n := range nums { // O(n) partition
        if n < pivot {
            lo = append(lo, n)
        } else if n == pivot {
            eq = append(eq, n)
        } else {
            hi = append(hi, n)
        }
    }
    if k < len(lo) {
        return quickSelect(lo, k) // T(n/2) on average
    } else if k < len(lo)+len(eq) {
        return pivot
    }
    return quickSelect(hi, k-len(lo)-len(eq))
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6, 5}
    // Find 4th smallest (0-indexed)
    fmt.Println(quickSelect(nums, 4)) // 4
    fmt.Println("Master Theorem: a=1, b=2, k=1 → Case 3 → O(n)")
}
```

---

## Example 6: Karatsuba Multiplication — T(n) = 3T(n/2) + O(n)

```go
package main

import (
    "fmt"
    "math"
)

// T(n) = 3T(n/2) + O(n¹)
// a=3, b=2, k=1
// log₂(3) ≈ 1.585 > 1 = k → Case 1
// Solution: O(n^(log₂3)) = O(n^1.585)

// Karatsuba: multiply two n-digit numbers
// Instead of 4 multiplications, use only 3!
// x = xH·10^(n/2) + xL
// y = yH·10^(n/2) + yL
// x·y = z2·10^n + z1·10^(n/2) + z0
//   z2 = xH·yH
//   z0 = xL·yL
//   z1 = (xH+xL)(yH+yL) - z2 - z0  ← the trick: 3 multiplications, not 4

func karatsuba(x, y int) int {
    if x < 10 || y < 10 {
        return x * y
    }

    // Find number of digits
    n := 0
    temp := x
    for temp > 0 { n++; temp /= 10 }
    half := n / 2
    power := int(math.Pow(10, float64(half)))

    xH, xL := x/power, x%power
    yH, yL := y/power, y%power

    z2 := karatsuba(xH, yH)          // subproblem 1
    z0 := karatsuba(xL, yL)          // subproblem 2
    z1 := karatsuba(xH+xL, yH+yL) - z2 - z0  // subproblem 3

    return z2*power*power + z1*power + z0
}

func main() {
    fmt.Println(karatsuba(1234, 5678)) // 7006652
    fmt.Println(1234 * 5678)           // 7006652 — verify
    fmt.Printf("Karatsuba: O(n^log₂3) = O(n^%.3f)\n", math.Log2(3))
}
```

---

## Example 7: T(n) = 4T(n/2) + O(n²) — Case 2

```go
package main

import "fmt"

// T(n) = 4T(n/2) + O(n²)
// a=4, b=2, k=2
// log₂(4) = 2 = k → Case 2
// Solution: O(n² · log n)

// Example: Process all quadrants of a matrix, then combine
func processQuadrants(matrix [][]int, row, col, size, depth int) int {
    if size <= 1 {
        if row < len(matrix) && col < len(matrix[0]) {
            return matrix[row][col]
        }
        return 0
    }

    half := size / 2

    // O(n²) work at this level
    sum := 0
    for i := row; i < row+size && i < len(matrix); i++ {
        for j := col; j < col+size && j < len(matrix[0]); j++ {
            sum += matrix[i][j]
        }
    }

    // 4 recursive calls on quadrants (n/2 × n/2 each)
    processQuadrants(matrix, row, col, half, depth+1)
    processQuadrants(matrix, row, col+half, half, depth+1)
    processQuadrants(matrix, row+half, col, half, depth+1)
    processQuadrants(matrix, row+half, col+half, half, depth+1)

    return sum
}

func main() {
    matrix := [][]int{
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
        {13, 14, 15, 16},
    }
    result := processQuadrants(matrix, 0, 0, 4, 0)
    fmt.Println("Sum:", result)
    fmt.Println("Master Theorem: a=4, b=2, k=2 → Case 2 → O(n² log n)")
}
```

---

## Example 8: T(n) = 4T(n/2) + O(n) — Case 1

```go
package main

import (
    "fmt"
    "math"
)

// T(n) = 4T(n/2) + O(n¹)
// a=4, b=2, k=1
// log₂(4) = 2 > 1 = k → Case 1
// Solution: O(n^(log₂4)) = O(n²)

func fourWayRecurse(n, depth int) int {
    if n <= 1 { return 1 }

    // O(n) work at this level
    work := n

    // 4 recursive calls on n/2
    for i := 0; i < 4; i++ {
        work += fourWayRecurse(n/2, depth+1)
    }
    return work
}

func main() {
    for _, n := range []int{4, 8, 16, 32} {
        result := fourWayRecurse(n, 0)
        ratio := float64(result) / float64(n*n)
        fmt.Printf("n=%2d, work=%5d, work/n²=%.2f\n", n, result, ratio)
    }
    fmt.Printf("\nMaster: a=4, b=2, k=1 → Case 1 → O(n^%.1f) = O(n²)\n", math.Log2(4))
}
```

---

## Example 9: Cases That DON'T Apply to Master Theorem

```go
package main

import "fmt"

// The Master Theorem does NOT apply to these patterns:

// 1. Non-equal subproblem sizes: T(n) = T(n/3) + T(2n/3) + O(n)
// (Subproblems must be same size n/b)

// 2. Subtract recurrences: T(n) = T(n-1) + O(n)
// (Must be T(n/b), not T(n-c))

// 3. Non-polynomial f(n): T(n) = 2T(n/2) + O(n log n)
// (Extended master theorem needed for log factors)

// 4. Variable number of subproblems: T(n) = nT(n/2) + O(n)

// For these, use:
// - Recursion tree method
// - Substitution (guess and prove by induction)
// - Akra-Bazzi theorem (generalization)

func subtractRecurrence(n int) int {
    if n <= 0 { return 0 }
    work := 0
    for i := 0; i < n; i++ { work++ } // O(n)
    return work + subtractRecurrence(n-1) // T(n-1) — NOT T(n/b)!
}

// T(n) = T(n-1) + n = n + (n-1) + ... + 1 = n(n+1)/2 = O(n²)
// Must solve by unrolling, not Master Theorem

func main() {
    for _, n := range []int{10, 100, 1000} {
        result := subtractRecurrence(n)
        expected := n * (n + 1) / 2
        fmt.Printf("n=%4d, work=%d, n(n+1)/2=%d, match=%v\n",
            n, result, expected, result == expected)
    }
}
```

---

## Example 10: Master Theorem Decision Flowchart

```go
package main

import (
    "fmt"
    "math"
)

type Recurrence struct {
    A    int
    B    int
    K    int
    Name string
}

func solve(r Recurrence) string {
    logBA := math.Log(float64(r.A)) / math.Log(float64(r.B))
    k := float64(r.K)

    var caseStr, solution string

    if logBA > k+0.001 {
        caseStr = "Case 1 (log_b(a) > k)"
        solution = fmt.Sprintf("O(n^%.3f)", logBA)
    } else if math.Abs(logBA-k) < 0.001 {
        caseStr = "Case 2 (log_b(a) = k)"
        if r.K == 0 {
            solution = "O(log n)"
        } else {
            solution = fmt.Sprintf("O(n^%d · log n)", r.K)
        }
    } else {
        caseStr = "Case 3 (log_b(a) < k)"
        solution = fmt.Sprintf("O(n^%d)", r.K)
    }

    return fmt.Sprintf("%-20s a=%d, b=%d, k=%d | log_%d(%d)=%.2f | %-25s → %s",
        r.Name, r.A, r.B, r.K, r.B, r.A, logBA, caseStr, solution)
}

func main() {
    recurrences := []Recurrence{
        {1, 2, 0, "Binary Search"},
        {2, 2, 0, "Tree Traversal"},
        {2, 2, 1, "Merge Sort"},
        {4, 2, 2, "Quad Process"},
        {1, 2, 1, "Quickselect"},
        {3, 2, 1, "Karatsuba"},
        {7, 2, 2, "Strassen"},
        {4, 2, 1, "4-way Recurse"},
        {8, 2, 3, "3D Process"},
        {9, 3, 2, "9-way / 3-split"},
    }

    fmt.Println("=== Master Theorem Solutions ===")
    fmt.Println()
    for _, r := range recurrences {
        fmt.Println(solve(r))
    }
}
```

---

## Quick Reference

```
T(n) = a·T(n/b) + O(nᵏ)

Step 1: Compute log_b(a)
Step 2: Compare with k

  log_b(a) > k  →  O(n^(log_b(a)))     [recursion dominates]
  log_b(a) = k  →  O(nᵏ · log n)       [balanced]
  log_b(a) < k  →  O(nᵏ)               [combine step dominates]
```

---

## Key Takeaways

1. **Master Theorem** solves T(n) = aT(n/b) + O(nᵏ) in one step
2. **Three cases**: compare log_b(a) with k
3. **Case 1**: recursion dominates → O(n^(log_b(a)))
4. **Case 2**: balanced → O(nᵏ log n)
5. **Case 3**: combine dominates → O(nᵏ)
6. **Doesn't apply** to subtract recurrences or unequal splits
7. **Common results**: binary search O(log n), merge sort O(n log n)

> **Next up:** Asymptotic Analysis →
