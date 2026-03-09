# Phase 2: Arrays — Prefix Sum

## What is Prefix Sum?

A **prefix sum** (cumulative sum) array stores the running total of elements. It allows answering **range sum queries in O(1)** after O(n) preprocessing.

```
Original:   [a₀, a₁, a₂, a₃, a₄]
Prefix Sum: [a₀, a₀+a₁, a₀+a₁+a₂, ...]

Sum(L..R) = prefix[R] - prefix[L-1]
```

---

## Time Complexity

| Operation | Without Prefix Sum | With Prefix Sum |
|-----------|-------------------|-----------------|
| Build | — | O(n) |
| Range sum query | O(n) | O(1) |
| Point update | O(1) | O(n) to rebuild |

---

## Example 1: Building a Prefix Sum Array

```go
package main

import "fmt"

func buildPrefixSum(nums []int) []int {
    prefix := make([]int, len(nums)+1)
    for i, v := range nums {
        prefix[i+1] = prefix[i] + v
    }
    return prefix
}

func rangeSum(prefix []int, l, r int) int {
    return prefix[r+1] - prefix[l]
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}
    prefix := buildPrefixSum(nums)
    fmt.Println("Prefix:", prefix) // [0 3 4 8 9 14 23 25 31]

    // Sum of nums[2..5] = 4+1+5+9 = 19
    fmt.Println("Sum(2,5):", rangeSum(prefix, 2, 5)) // 19

    // Sum of nums[0..7] = entire array = 31
    fmt.Println("Sum(0,7):", rangeSum(prefix, 0, 7)) // 31

    // Sum of single element nums[3] = 1
    fmt.Println("Sum(3,3):", rangeSum(prefix, 3, 3)) // 1
}
```

---

## Example 2: Subarray Sum Equals K

```go
package main

import "fmt"

// Count subarrays with sum == k using prefix sum + hash map
// O(n) time, O(n) space
func subarraySum(nums []int, k int) int {
    count := 0
    sum := 0
    prefixCount := map[int]int{0: 1}

    for _, num := range nums {
        sum += num
        // If (sum - k) was seen before, those subarrays sum to k
        if c, ok := prefixCount[sum-k]; ok {
            count += c
        }
        prefixCount[sum]++
    }
    return count
}

func main() {
    fmt.Println(subarraySum([]int{1, 1, 1}, 2))         // 2: [1,1] at two positions
    fmt.Println(subarraySum([]int{1, 2, 3}, 3))         // 2: [1,2] and [3]
    fmt.Println(subarraySum([]int{1, -1, 0}, 0))        // 3: [1,-1], [-1,0], [1,-1,0]
    fmt.Println(subarraySum([]int{3, 4, 7, 2, -3, 1, 4, 2}, 7)) // 4
}
```

---

## Example 3: Running Average with Prefix Sum

```go
package main

import "fmt"

func main() {
    scores := []int{85, 90, 78, 92, 88, 76, 95, 80, 91, 87}

    // Build prefix sum
    prefix := make([]int, len(scores)+1)
    for i, v := range scores {
        prefix[i+1] = prefix[i] + v
    }

    // Average of scores[2..6] = (78+92+88+76+95) / 5
    l, r := 2, 6
    sum := prefix[r+1] - prefix[l]
    avg := float64(sum) / float64(r-l+1)
    fmt.Printf("Average of scores[%d..%d] = %.2f\n", l, r, avg) // 85.80

    // Moving average with window size 3
    windowSize := 3
    fmt.Println("Moving averages:")
    for i := 0; i+windowSize <= len(scores); i++ {
        s := prefix[i+windowSize] - prefix[i]
        fmt.Printf("  Window [%d..%d]: %.2f\n", i, i+windowSize-1, float64(s)/float64(windowSize))
    }
}
```

---

## Example 4: Product Except Self (Prefix & Suffix Products)

```go
package main

import "fmt"

func productExceptSelf(nums []int) []int {
    n := len(nums)
    result := make([]int, n)

    // Prefix products: result[i] = product of nums[0..i-1]
    result[0] = 1
    for i := 1; i < n; i++ {
        result[i] = result[i-1] * nums[i-1]
    }

    // Suffix products: multiply from right
    suffix := 1
    for i := n - 1; i >= 0; i-- {
        result[i] *= suffix
        suffix *= nums[i]
    }

    return result
}

func main() {
    fmt.Println(productExceptSelf([]int{1, 2, 3, 4}))    // [24 12 8 6]
    fmt.Println(productExceptSelf([]int{-1, 1, 0, -3, 3})) // [0 0 9 0 0]
    fmt.Println(productExceptSelf([]int{2, 3, 5}))        // [15 10 6]
}
```

---

## Example 5: 2D Prefix Sum

```go
package main

import "fmt"

func build2DPrefix(matrix [][]int) [][]int {
    rows, cols := len(matrix), len(matrix[0])
    prefix := make([][]int, rows+1)
    for i := range prefix {
        prefix[i] = make([]int, cols+1)
    }

    for i := 1; i <= rows; i++ {
        for j := 1; j <= cols; j++ {
            prefix[i][j] = matrix[i-1][j-1] +
                prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1]
        }
    }
    return prefix
}

// Sum of submatrix from (r1,c1) to (r2,c2) inclusive (0-indexed)
func regionSum(prefix [][]int, r1, c1, r2, c2 int) int {
    r1++; c1++; r2++; c2++
    return prefix[r2][c2] - prefix[r1-1][c2] - prefix[r2][c1-1] + prefix[r1-1][c1-1]
}

func main() {
    matrix := [][]int{
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    }

    prefix := build2DPrefix(matrix)

    // Sum of entire matrix
    fmt.Println("Full sum:", regionSum(prefix, 0, 0, 2, 3)) // 78

    // Sum of submatrix (1,1) to (2,3) = 6+7+8+10+11+12 = 54
    fmt.Println("Sub (1,1)-(2,3):", regionSum(prefix, 1, 1, 2, 3)) // 54

    // Sum of single element (0,2) = 3
    fmt.Println("Single (0,2):", regionSum(prefix, 0, 2, 0, 2)) // 3
}
```

**Why?** 2D prefix sums answer rectangular region sum queries in O(1) after O(n×m) preprocessing.

---

## Example 6: Count of Even Numbers in Range

```go
package main

import "fmt"

func main() {
    nums := []int{3, 8, 2, 7, 4, 6, 1, 5, 10, 9}

    // Prefix count of even numbers
    evenCount := make([]int, len(nums)+1)
    for i, v := range nums {
        evenCount[i+1] = evenCount[i]
        if v%2 == 0 {
            evenCount[i+1]++
        }
    }

    // Query: how many evens in nums[2..7]?
    l, r := 2, 7
    fmt.Printf("Evens in [%d..%d]: %d\n", l, r, evenCount[r+1]-evenCount[l]) // 3 (2,4,6)

    // Query: how many evens in nums[0..4]?
    l, r = 0, 4
    fmt.Printf("Evens in [%d..%d]: %d\n", l, r, evenCount[r+1]-evenCount[l]) // 3 (8,2,4)
}
```

---

## Example 7: Equilibrium Index

```go
package main

import "fmt"

// Find index where left sum == right sum
func equilibriumIndex(nums []int) int {
    totalSum := 0
    for _, v := range nums {
        totalSum += v
    }

    leftSum := 0
    for i, v := range nums {
        rightSum := totalSum - leftSum - v
        if leftSum == rightSum {
            return i
        }
        leftSum += v
    }
    return -1
}

func main() {
    fmt.Println(equilibriumIndex([]int{-7, 1, 5, 2, -4, 3, 0})) // 3
    // Left of 3: -7+1+5 = -1, Right of 3: -4+3+0 = -1  ✓
    fmt.Println(equilibriumIndex([]int{1, 2, 3}))               // -1
    fmt.Println(equilibriumIndex([]int{1, 3, 5, 2, 2}))        // 2
    fmt.Println(equilibriumIndex([]int{2, 0, 2}))               // 1
}
```

---

## Example 8: Prefix XOR — Range XOR Queries

```go
package main

import "fmt"

func main() {
    nums := []int{3, 5, 2, 8, 1, 4}

    // Build prefix XOR
    prefixXOR := make([]int, len(nums)+1)
    for i, v := range nums {
        prefixXOR[i+1] = prefixXOR[i] ^ v
    }

    // XOR of range [l, r] = prefixXOR[r+1] ^ prefixXOR[l]
    // Because x ^ x = 0 (XOR is its own inverse)
    l, r := 1, 4
    fmt.Printf("XOR of nums[%d..%d] = %d\n", l, r, prefixXOR[r+1]^prefixXOR[l])
    // 5 ^ 2 ^ 8 ^ 1 = 14

    l, r = 0, 5
    fmt.Printf("XOR of nums[%d..%d] = %d\n", l, r, prefixXOR[r+1]^prefixXOR[l])
    // 3 ^ 5 ^ 2 ^ 8 ^ 1 ^ 4 = 13
}
```

**Why?** Just like prefix sums enable range sum queries, prefix XOR enables range XOR queries in O(1).

---

## Example 9: Maximum Subarray Sum Using Prefix Sum

```go
package main

import (
    "fmt"
    "math"
)

// maxSubarraySum using prefix sum approach
// max(prefix[j] - prefix[i]) for all j > i
func maxSubarraySum(nums []int) int {
    prefix := make([]int, len(nums)+1)
    for i, v := range nums {
        prefix[i+1] = prefix[i] + v
    }

    maxSum := math.MinInt64
    minPrefix := prefix[0]

    for j := 1; j <= len(nums); j++ {
        if prefix[j]-minPrefix > maxSum {
            maxSum = prefix[j] - minPrefix
        }
        if prefix[j] < minPrefix {
            minPrefix = prefix[j]
        }
    }
    return maxSum
}

func main() {
    fmt.Println(maxSubarraySum([]int{-2, 1, -3, 4, -1, 2, 1, -5, 4})) // 6
    fmt.Println(maxSubarraySum([]int{1}))                                // 1
    fmt.Println(maxSubarraySum([]int{-1, -2, -3}))                      // -1
    fmt.Println(maxSubarraySum([]int{5, 4, -1, 7, 8}))                  // 23
}
```

---

## Example 10: Prefix Sum for Binary Array Queries

```go
package main

import "fmt"

func main() {
    // Binary array: count 1s in any range
    bits := []int{1, 0, 1, 1, 0, 0, 1, 1, 1, 0}

    prefix := make([]int, len(bits)+1)
    for i, v := range bits {
        prefix[i+1] = prefix[i] + v
    }

    // Count 1s in range [2, 8]
    l, r := 2, 8
    fmt.Printf("Count of 1s in [%d..%d]: %d\n", l, r, prefix[r+1]-prefix[l]) // 5

    // Count 0s in range [2, 8]
    total := r - l + 1
    ones := prefix[r+1] - prefix[l]
    fmt.Printf("Count of 0s in [%d..%d]: %d\n", l, r, total-ones) // 2

    // Is range [6, 8] all ones?
    l, r = 6, 8
    fmt.Printf("All ones in [%d..%d]? %v\n", l, r, prefix[r+1]-prefix[l] == r-l+1) // true
}
```

---

## Example 11: Prefix Min and Prefix Max

```go
package main

import "fmt"

func main() {
    nums := []int{5, 3, 8, 1, 9, 2, 7}

    // Prefix min: min of nums[0..i]
    prefixMin := make([]int, len(nums))
    prefixMin[0] = nums[0]
    for i := 1; i < len(nums); i++ {
        if nums[i] < prefixMin[i-1] {
            prefixMin[i] = nums[i]
        } else {
            prefixMin[i] = prefixMin[i-1]
        }
    }

    // Suffix max: max of nums[i..n-1]
    suffixMax := make([]int, len(nums))
    suffixMax[len(nums)-1] = nums[len(nums)-1]
    for i := len(nums) - 2; i >= 0; i-- {
        if nums[i] > suffixMax[i+1] {
            suffixMax[i] = nums[i]
        } else {
            suffixMax[i] = suffixMax[i+1]
        }
    }

    fmt.Println("nums:      ", nums)
    fmt.Println("prefixMin: ", prefixMin) // [5 3 3 1 1 1 1]
    fmt.Println("suffixMax: ", suffixMax) // [9 9 9 9 9 7 7]

    // Use: find first index where prefixMin < suffixMax
    for i := 0; i < len(nums)-1; i++ {
        if prefixMin[i] < suffixMax[i+1] {
            fmt.Printf("At index %d: min so far=%d < max ahead=%d\n", i, prefixMin[i], suffixMax[i+1])
            break
        }
    }
}
```

---

## Key Takeaways

1. **Build once O(n), query O(1)** — classic preprocessing pattern
2. **Range sum**: `sum(L..R) = prefix[R+1] - prefix[L]`
3. **Works for**: sum, XOR, count, min/max, product
4. **2D prefix sum** extends to rectangular region queries
5. **Prefix sum + hash map** solves "subarray sum equals K" in O(n)
6. **Limitation**: point updates require O(n) rebuild (use Fenwick tree for O(log n))

> **Next up:** Difference Arrays →
