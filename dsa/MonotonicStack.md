# Phase 6: Stack — Monotonic Stack

## Overview

A **monotonic stack** maintains elements in strictly increasing or decreasing order. When a new element violates the monotonicity, we pop elements until the invariant is restored.

**Use cases:**
- Next Greater Element / Next Smaller Element
- Previous Greater / Previous Smaller Element
- Largest Rectangle in Histogram
- Trapping Rain Water
- Stock span problems

| Type | Invariant | Use |
|------|-----------|-----|
| Monotonic Increasing | bottom → top: small → large | Find next/prev smaller |
| Monotonic Decreasing | bottom → top: large → small | Find next/prev greater |

---

## Example 1: Next Greater Element

```go
package main

import "fmt"

// For each element, find the first larger element to its right
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1 // default: no greater element
    }

    stack := []int{} // monotonic decreasing stack (stores indices)

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    tests := [][]int{
        {4, 5, 2, 25},
        {13, 7, 6, 12},
        {1, 2, 3, 4},
        {4, 3, 2, 1},
    }

    for _, nums := range tests {
        fmt.Printf("%v → %v\n", nums, nextGreaterElement(nums))
    }
    // [4,5,2,25] → [5,25,25,-1]
    // [13,7,6,12] → [-1,12,12,-1]
}
```

---

## Example 2: Next Smaller Element

```go
package main

import "fmt"

func nextSmallerElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic increasing stack

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] < nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    nums := []int{4, 8, 5, 2, 25}
    fmt.Printf("%v → %v\n", nums, nextSmallerElement(nums))
    // [4,8,5,2,25] → [2,5,2,-1,-1]

    nums2 := []int{1, 3, 2, 4}
    fmt.Printf("%v → %v\n", nums2, nextSmallerElement(nums2))
    // [1,3,2,4] → [-1,2,-1,-1]
}
```

---

## Example 3: Previous Greater Element

```go
package main

import "fmt"

func prevGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic decreasing stack

    for i := 0; i < n; i++ {
        // Pop elements smaller or equal
        for len(stack) > 0 && nums[stack[len(stack)-1]] <= nums[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) > 0 {
            result[i] = nums[stack[len(stack)-1]]
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    nums := []int{10, 4, 2, 20, 40, 12, 30}
    fmt.Printf("%v\n→ %v\n", nums, prevGreaterElement(nums))
    // [-1, 10, 4, -1, -1, 40, 40]
}
```

---

## Example 4: Daily Temperatures (LeetCode 739)

```go
package main

import "fmt"

// For each day, how many days until a warmer temperature?
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := []int{} // decreasing stack of indices

    for i := 0; i < n; i++ {
        for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
            prev := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[prev] = i - prev
        }
        stack = append(stack, i)
    }

    return result
}

func main() {
    temps := []int{73, 74, 75, 71, 69, 72, 76, 73}
    fmt.Println(dailyTemperatures(temps))
    // [1, 1, 4, 2, 1, 1, 0, 0]

    temps2 := []int{30, 40, 50, 60}
    fmt.Println(dailyTemperatures(temps2))
    // [1, 1, 1, 0]

    temps3 := []int{30, 60, 90}
    fmt.Println(dailyTemperatures(temps3))
    // [1, 1, 0]
}
```

---

## Example 5: Largest Rectangle in Histogram (LeetCode 84)

```go
package main

import "fmt"

func largestRectangleArea(heights []int) int {
    stack := []int{}
    maxArea := 0
    n := len(heights)

    for i := 0; i <= n; i++ {
        h := 0
        if i < n {
            h = heights[i]
        }

        for len(stack) > 0 && h < heights[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            width := i
            if len(stack) > 0 {
                width = i - stack[len(stack)-1] - 1
            }

            area := heights[top] * width
            if area > maxArea {
                maxArea = area
            }
        }
        stack = append(stack, i)
    }

    return maxArea
}

func main() {
    tests := [][]int{
        {2, 1, 5, 6, 2, 3},
        {2, 4},
        {1, 1, 1, 1},
        {6, 2, 5, 4, 5, 1, 6},
    }

    for _, h := range tests {
        fmt.Printf("heights=%v → maxArea=%d\n", h, largestRectangleArea(h))
    }
    // [2,1,5,6,2,3] → 10
    // [2,4] → 4
    // [1,1,1,1] → 4
}
```

---

## Example 6: Trapping Rain Water (Monotonic Stack)

```go
package main

import "fmt"

func trap(height []int) int {
    stack := []int{}
    water := 0

    for i := 0; i < len(height); i++ {
        for len(stack) > 0 && height[i] > height[stack[len(stack)-1]] {
            bottom := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            if len(stack) == 0 {
                break
            }

            left := stack[len(stack)-1]
            width := i - left - 1
            h := min(height[left], height[i]) - height[bottom]
            water += width * h
        }
        stack = append(stack, i)
    }

    return water
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    tests := [][]int{
        {0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1},
        {4, 2, 0, 3, 2, 5},
    }

    for _, h := range tests {
        fmt.Printf("height=%v → water=%d\n", h, trap(h))
    }
    // [0,1,0,2,1,0,1,3,2,1,2,1] → 6
    // [4,2,0,3,2,5] → 9
}
```

---

## Example 7: Stock Span Problem

```go
package main

import "fmt"

// Stock span: number of consecutive days before (including today)
// where price was ≤ today's price
func stockSpan(prices []int) []int {
    n := len(prices)
    spans := make([]int, n)
    stack := []int{} // decreasing stack of indices

    for i := 0; i < n; i++ {
        for len(stack) > 0 && prices[stack[len(stack)-1]] <= prices[i] {
            stack = stack[:len(stack)-1]
        }

        if len(stack) == 0 {
            spans[i] = i + 1
        } else {
            spans[i] = i - stack[len(stack)-1]
        }
        stack = append(stack, i)
    }

    return spans
}

func main() {
    prices := []int{100, 80, 60, 70, 60, 75, 85}
    fmt.Println("Prices:", prices)
    fmt.Println("Spans: ", stockSpan(prices))
    // [1, 1, 1, 2, 1, 4, 6]
}
```

---

## Example 8: Remove K Digits (LeetCode 402)

```go
package main

import (
    "fmt"
    "strings"
)

func removeKdigits(num string, k int) string {
    stack := []byte{}

    for i := 0; i < len(num); i++ {
        // Remove digits from stack that are greater than current
        for k > 0 && len(stack) > 0 && stack[len(stack)-1] > num[i] {
            stack = stack[:len(stack)-1]
            k--
        }
        stack = append(stack, num[i])
    }

    // Remove remaining k digits from end
    stack = stack[:len(stack)-k]

    // Remove leading zeros
    result := strings.TrimLeft(string(stack), "0")
    if result == "" {
        return "0"
    }
    return result
}

func main() {
    tests := []struct {
        num string
        k   int
    }{
        {"1432219", 3},
        {"10200", 1},
        {"10", 2},
        {"9", 1},
        {"112", 1},
    }

    for _, t := range tests {
        fmt.Printf("removeKdigits(%q, %d) = %q\n", t.num, t.k, removeKdigits(t.num, t.k))
    }
    // "1432219",3 → "1219"
    // "10200",1 → "200"
    // "10",2 → "0"
}
```

---

## Example 9: Asteroid Collision (LeetCode 735)

```go
package main

import "fmt"

func asteroidCollision(asteroids []int) []int {
    stack := []int{}

    for _, a := range asteroids {
        alive := true

        for alive && a < 0 && len(stack) > 0 && stack[len(stack)-1] > 0 {
            top := stack[len(stack)-1]

            if top < -a {
                // Top asteroid explodes
                stack = stack[:len(stack)-1]
            } else if top == -a {
                // Both explode
                stack = stack[:len(stack)-1]
                alive = false
            } else {
                // Current asteroid explodes
                alive = false
            }
        }

        if alive {
            stack = append(stack, a)
        }
    }

    return stack
}

func main() {
    tests := [][]int{
        {5, 10, -5},
        {8, -8},
        {10, 2, -5},
        {-2, -1, 1, 2},
        {1, -2, -2, -2},
    }

    for _, t := range tests {
        fmt.Printf("%v → %v\n", t, asteroidCollision(t))
    }
    // [5,10,-5] → [5,10]
    // [8,-8] → []
    // [10,2,-5] → [10]
    // [-2,-1,1,2] → [-2,-1,1,2]
}
```

---

## Example 10: Next Greater Element II (Circular Array, LeetCode 503)

```go
package main

import "fmt"

func nextGreaterElements(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // monotonic decreasing

    // Traverse twice to handle circular
    for i := 0; i < 2*n; i++ {
        idx := i % n
        for len(stack) > 0 && nums[idx] > nums[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = nums[idx]
        }
        if i < n {
            stack = append(stack, i)
        }
    }

    return result
}

func main() {
    tests := [][]int{
        {1, 2, 1},
        {1, 2, 3, 4, 3},
        {5, 4, 3, 2, 1},
    }

    for _, nums := range tests {
        fmt.Printf("%v → %v\n", nums, nextGreaterElements(nums))
    }
    // [1,2,1] → [2,-1,2]
    // [1,2,3,4,3] → [2,3,4,-1,4]
    // [5,4,3,2,1] → [-1,5,5,5,5]
}
```

---

## Example 11: Sum of Subarray Minimums (LeetCode 907)

```go
package main

import "fmt"

func sumSubarrayMins(arr []int) int {
    const MOD = 1_000_000_007
    n := len(arr)

    // left[i]  = number of subarrays ending at i where arr[i] is min
    // right[i] = number of subarrays starting at i where arr[i] is min
    left := make([]int, n)
    right := make([]int, n)

    // Previous less element
    stack := []int{}
    for i := 0; i < n; i++ {
        for len(stack) > 0 && arr[stack[len(stack)-1]] >= arr[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) == 0 {
            left[i] = i + 1
        } else {
            left[i] = i - stack[len(stack)-1]
        }
        stack = append(stack, i)
    }

    // Next less element
    stack = stack[:0]
    for i := n - 1; i >= 0; i-- {
        for len(stack) > 0 && arr[stack[len(stack)-1]] > arr[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) == 0 {
            right[i] = n - i
        } else {
            right[i] = stack[len(stack)-1] - i
        }
        stack = append(stack, i)
    }

    result := 0
    for i := 0; i < n; i++ {
        result = (result + arr[i]*left[i]*right[i]) % MOD
    }

    return result
}

func main() {
    tests := [][]int{
        {3, 1, 2, 4},
        {11, 81, 94, 43, 3},
    }

    for _, arr := range tests {
        fmt.Printf("arr=%v → sum=%d\n", arr, sumSubarrayMins(arr))
    }
    // [3,1,2,4] → 17
    // [11,81,94,43,3] → 444
}
```

---

## Monotonic Stack Pattern Summary

| Problem | Stack Type | Direction |
|---------|-----------|-----------|
| Next Greater | Decreasing | Left → Right |
| Next Smaller | Increasing | Left → Right |
| Prev Greater | Decreasing | Left → Right |
| Prev Smaller | Increasing | Left → Right |
| Histogram | Increasing | Left → Right |
| Rain Water | Decreasing | Left → Right |
| Stock Span | Decreasing | Left → Right |

## Key Takeaways

1. **Monotonic decreasing stack** → finds next/prev **greater** element
2. **Monotonic increasing stack** → finds next/prev **smaller** element
3. Stack stores **indices** (not values) — enables distance/width calculation
4. Every element is pushed and popped **at most once** → O(n) total
5. For **circular arrays**, traverse `2*n` elements using modulo
6. **Histogram/rain water**: think about what each bar "contributes" when popped

> **Next up:** Expression Evaluation →
