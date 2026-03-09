# Phase 2: Arrays — Kadane's Algorithm

## What is Kadane's Algorithm?

Kadane's algorithm finds the **maximum sum contiguous subarray** in O(n) time. It maintains a running sum, resetting when the sum becomes negative.

```
Core idea: at each position, decide:
  - Extend the current subarray (currentSum + nums[i])
  - Start a new subarray (nums[i])
  
  currentSum = max(nums[i], currentSum + nums[i])
```

---

## Example 1: Basic Kadane's Algorithm

```go
package main

import (
    "fmt"
    "math"
)

func maxSubArray(nums []int) int {
    maxSum := math.MinInt64
    currentSum := 0

    for _, num := range nums {
        currentSum += num
        if currentSum > maxSum {
            maxSum = currentSum
        }
        if currentSum < 0 {
            currentSum = 0
        }
    }
    return maxSum
}

func main() {
    fmt.Println(maxSubArray([]int{-2, 1, -3, 4, -1, 2, 1, -5, 4})) // 6 → [4,-1,2,1]
    fmt.Println(maxSubArray([]int{1}))                                // 1
    fmt.Println(maxSubArray([]int{5, 4, -1, 7, 8}))                  // 23 → entire array
    fmt.Println(maxSubArray([]int{-1}))                               // -1
    fmt.Println(maxSubArray([]int{-2, -1}))                           // -1
}
```

---

## Example 2: Return the Subarray Itself

```go
package main

import (
    "fmt"
    "math"
)

func maxSubArrayWithIndices(nums []int) (int, int, int) {
    maxSum := math.MinInt64
    currentSum := 0
    start, end, tempStart := 0, 0, 0

    for i, num := range nums {
        currentSum += num
        if currentSum > maxSum {
            maxSum = currentSum
            start = tempStart
            end = i
        }
        if currentSum < 0 {
            currentSum = 0
            tempStart = i + 1
        }
    }
    return maxSum, start, end
}

func main() {
    nums := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
    sum, start, end := maxSubArrayWithIndices(nums)
    fmt.Printf("Max sum: %d, subarray: %v (indices %d to %d)\n",
        sum, nums[start:end+1], start, end)
    // Max sum: 6, subarray: [4 -1 2 1] (indices 3 to 6)

    nums2 := []int{5, 4, -1, 7, 8}
    sum2, s2, e2 := maxSubArrayWithIndices(nums2)
    fmt.Printf("Max sum: %d, subarray: %v\n", sum2, nums2[s2:e2+1])
    // Max sum: 23, subarray: [5 4 -1 7 8]
}
```

---

## Example 3: Maximum Product Subarray (Variant)

```go
package main

import (
    "fmt"
    "math"
)

func maxProduct(nums []int) int {
    maxProd := math.MinInt64
    curMax, curMin := 1, 1

    for _, num := range nums {
        // A negative number flips max and min
        if num < 0 {
            curMax, curMin = curMin, curMax
        }

        if num > curMax*num {
            curMax = num
        } else {
            curMax = curMax * num
        }
        if num < curMin*num {
            curMin = num
        } else {
            curMin = curMin * num
        }

        if curMax > maxProd {
            maxProd = curMax
        }
    }
    return maxProd
}

func main() {
    fmt.Println(maxProduct([]int{2, 3, -2, 4}))   // 6 → [2,3]
    fmt.Println(maxProduct([]int{-2, 0, -1}))     // 0
    fmt.Println(maxProduct([]int{-2, 3, -4}))     // 24 → [-2,3,-4]
    fmt.Println(maxProduct([]int{-1, -2, -3, 0})) // 6 → [-1,-2,-3]...wait, actually [-1,-2,-3]=−6? No: (-1)*(-2)=2, 2*(-3)=−6; max subarray product is 2
    fmt.Println(maxProduct([]int{2, -5, -2, -4, 3})) // 24
}
```

**Why track min?** A negative min × negative number = large positive.

---

## Example 4: Minimum Sum Subarray (Inverted Kadane)

```go
package main

import (
    "fmt"
    "math"
)

func minSubArray(nums []int) int {
    minSum := math.MaxInt64
    currentSum := 0

    for _, num := range nums {
        currentSum += num
        if currentSum < minSum {
            minSum = currentSum
        }
        if currentSum > 0 {
            currentSum = 0
        }
    }
    return minSum
}

func main() {
    fmt.Println(minSubArray([]int{3, -4, 2, -3, -1, 7, -5})) // -6 → [-4,2,-3,-1]
    fmt.Println(minSubArray([]int{1, 2, 3}))                   // 1
    fmt.Println(minSubArray([]int{-1, -2, -3}))                // -6
}
```

---

## Example 5: Maximum Circular Subarray Sum

```go
package main

import (
    "fmt"
    "math"
)

func maxSubarrayCircular(nums []int) int {
    // Case 1: max subarray is non-wrapping → normal Kadane
    totalSum := 0
    maxSum := math.MinInt64
    curMax := 0
    minSum := math.MaxInt64
    curMin := 0

    for _, num := range nums {
        totalSum += num

        curMax += num
        if curMax > maxSum {
            maxSum = curMax
        }
        if curMax < 0 {
            curMax = 0
        }

        curMin += num
        if curMin < minSum {
            minSum = curMin
        }
        if curMin > 0 {
            curMin = 0
        }
    }

    // Case 2: max subarray wraps around → totalSum - minSubarray
    if totalSum == minSum {
        return maxSum // all negative
    }
    wrappedMax := totalSum - minSum

    if wrappedMax > maxSum {
        return wrappedMax
    }
    return maxSum
}

func main() {
    fmt.Println(maxSubarrayCircular([]int{1, -2, 3, -2}))     // 3
    fmt.Println(maxSubarrayCircular([]int{5, -3, 5}))          // 10 → [5, 5] wrapping
    fmt.Println(maxSubarrayCircular([]int{-3, -2, -3}))        // -2
    fmt.Println(maxSubarrayCircular([]int{3, -1, 2, -1}))      // 4 → [2,-1,3]
    fmt.Println(maxSubarrayCircular([]int{3, -2, 2, -3}))      // 3
}
```

---

## Example 6: Maximum Sum of Two Non-Overlapping Subarrays

```go
package main

import "fmt"

func maxSumTwoNoOverlap(nums []int, firstLen, secondLen int) int {
    n := len(nums)
    prefix := make([]int, n+1)
    for i, v := range nums {
        prefix[i+1] = prefix[i] + v
    }

    rangeSum := func(l, r int) int { return prefix[r+1] - prefix[l] }

    maxSum := 0

    // Case 1: first window before second window
    maxFirst := 0
    for i := firstLen; i+secondLen <= n; i++ {
        firstEnd := i - 1
        firstStart := firstEnd - firstLen + 1
        s := rangeSum(firstStart, firstEnd)
        if s > maxFirst {
            maxFirst = s
        }
        secondSum := rangeSum(i, i+secondLen-1)
        if maxFirst+secondSum > maxSum {
            maxSum = maxFirst + secondSum
        }
    }

    // Case 2: second window before first window
    maxSecond := 0
    for i := secondLen; i+firstLen <= n; i++ {
        secondEnd := i - 1
        secondStart := secondEnd - secondLen + 1
        s := rangeSum(secondStart, secondEnd)
        if s > maxSecond {
            maxSecond = s
        }
        firstSum := rangeSum(i, i+firstLen-1)
        if maxSecond+firstSum > maxSum {
            maxSum = maxSecond + firstSum
        }
    }

    return maxSum
}

func main() {
    fmt.Println(maxSumTwoNoOverlap([]int{0, 6, 5, 2, 2, 5, 1, 9, 4}, 1, 2)) // 20
    fmt.Println(maxSumTwoNoOverlap([]int{3, 8, 1, 3, 2, 1, 8, 9, 0}, 3, 2)) // 29
}
```

---

## Example 7: Longest Subarray with Sum ≤ K

```go
package main

import "fmt"

func longestSubarraySum(nums []int, k int) int {
    left := 0
    sum := 0
    maxLen := 0

    for right := 0; right < len(nums); right++ {
        sum += nums[right]

        for sum > k && left <= right {
            sum -= nums[left]
            left++
        }

        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(longestSubarraySum([]int{3, 1, 2, 7, 4, 2, 1, 1, 5}, 8)) // 4 → [4,2,1,1]
    fmt.Println(longestSubarraySum([]int{1, 2, 3, 4, 5}, 9))              // 3 → [2,3,4]
    fmt.Println(longestSubarraySum([]int{1, 1, 1, 1, 1}, 3))              // 3
}
```

---

## Example 8: Kadane's on 2D — Maximum Sum Rectangle

```go
package main

import (
    "fmt"
    "math"
)

func maxSumRectangle(matrix [][]int) int {
    rows, cols := len(matrix), len(matrix[0])
    maxSum := math.MinInt64

    for left := 0; left < cols; left++ {
        temp := make([]int, rows)

        for right := left; right < cols; right++ {
            // Add column 'right' to running row sums
            for i := 0; i < rows; i++ {
                temp[i] += matrix[i][right]
            }

            // Apply 1D Kadane on temp[]
            curSum := 0
            curMax := math.MinInt64
            for _, v := range temp {
                curSum += v
                if curSum > curMax {
                    curMax = curSum
                }
                if curSum < 0 {
                    curSum = 0
                }
            }
            if curMax > maxSum {
                maxSum = curMax
            }
        }
    }
    return maxSum
}

func main() {
    matrix := [][]int{
        {1, 2, -1, -4, -20},
        {-8, -3, 4, 2, 1},
        {3, 8, 10, 1, 3},
        {-4, -1, 1, 7, -6},
    }
    fmt.Println(maxSumRectangle(matrix)) // 29 → submatrix cols 1-3, rows 1-3
}
```

---

## Example 9: Maximum Alternating Subarray Sum

```go
package main

import "fmt"

// max subarray sum where signs alternate: +a[i] -a[i+1] +a[i+2] ...
func maxAlternatingSum(nums []int) int {
    even, odd := 0, 0 // even = max sum ending at even position, etc.

    for _, num := range nums {
        newEven := odd + num
        newOdd := even - num

        if num > newEven {
            newEven = num
        }
        if -num > newOdd {
            newOdd = -num
        }

        even = newEven
        odd = newOdd
    }

    maxSum := even
    if odd > maxSum {
        maxSum = odd
    }
    return maxSum
}

func main() {
    fmt.Println(maxAlternatingSum([]int{4, 2, 5, 3}))    // 7 → +5−3+5 = no, +4−2+5 = 7
    fmt.Println(maxAlternatingSum([]int{5, 6, 7, 8}))    // 8
    fmt.Println(maxAlternatingSum([]int{6, 2, 1, 2, 4, 5})) // 10
}
```

---

## Example 10: Count Subarrays with Negative Sum

```go
package main

import "fmt"

func countNegativeSumSubarrays(nums []int) int {
    count := 0
    n := len(nums)

    // Using Kadane-style thinking with prefix sums
    // For efficiency with merge sort approach...
    // Here we show O(n²) for clarity
    for i := 0; i < n; i++ {
        sum := 0
        for j := i; j < n; j++ {
            sum += nums[j]
            if sum < 0 {
                count++
            }
        }
    }
    return count
}

func main() {
    fmt.Println(countNegativeSumSubarrays([]int{-1, 2, -3}))     // 3: [-1], [-3], [-1,2,-3]→-2
    fmt.Println(countNegativeSumSubarrays([]int{1, -2, 3}))      // 1: [-2]
    fmt.Println(countNegativeSumSubarrays([]int{-1, -2, -3}))    // 6: all subarrays
}
```

---

## Key Takeaways

1. **Core Kadane**: `currentSum = max(nums[i], currentSum + nums[i])`
2. **O(n) time, O(1) space** — the optimal for max subarray sum
3. **Track indices** by recording start/end when maxSum updates
4. **Product variant**: track both curMax and curMin (negatives flip)
5. **Circular variant**: `max(normalKadane, totalSum - minSubarray)`
6. **2D extension**: fix left/right columns, apply 1D Kadane on row sums

> **Next up:** In-Place Algorithms →
