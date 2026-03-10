# Phase 7: Queue — Monotonic Queue

## Overview

A **monotonic queue** (usually implemented with a **deque**) maintains elements in strictly increasing or decreasing order. When a new element arrives, elements that violate the monotonic property are removed from the back before insertion.

| Property | Monotonic Increasing | Monotonic Decreasing |
|----------|---------------------|---------------------|
| Front | Minimum | Maximum |
| When to remove from back | New element < back | New element > back |
| Typical use | Sliding window min | Sliding window max |

**Time Complexity:** Each element is pushed and popped at most once → **O(n)** total for processing n elements, i.e. **amortized O(1)** per operation.

---

## Example 1: Sliding Window Maximum (LeetCode 239 — Monotonic Decreasing Deque)

```go
package main

import "fmt"

func maxSlidingWindow(nums []int, k int) []int {
    deque := []int{} // stores indices; front has max's index
    result := []int{}

    for i, num := range nums {
        // Remove indices out of window
        for len(deque) > 0 && deque[0] < i-k+1 {
            deque = deque[1:]
        }
        // Remove smaller elements from back (maintain decreasing order)
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= num {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1, 3, -1, -3, 5, 3, 6, 7}, 3},
        {[]int{1}, 1},
        {[]int{1, -1}, 1},
        {[]int{9, 11}, 2},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %v\n", t.nums, t.k, maxSlidingWindow(t.nums, t.k))
    }
    // [3,3,5,5,6,7], [1], [1,-1], [11]
}
```

**Textual Figure:**
```
nums = [1, 3, -1, -3, 5, 3, 6, 7],  k = 3

Maintain DECREASING deque of indices (front = max)

Step   i   nums[i]   Deque (values)         Window        Result
─────────────────────────────────────────────────────────────────
 1     0     1       [1]                     [1,_,_]        -
 2     1     3       [3]  ← 1≤3 popped      [1,3,_]        -
 3     2    -1       [3,-1]                  [1,3,-1]       3
 4     3    -3       [3,-1,-3]               [3,-1,-3]      3
 5     4     5       [5]  ← all≤5 popped     [-1,-3,5]      5
 6     5     3       [5,3]                   [-3,5,3]       5
 7     6     6       [6]  ← all≤6 popped     [5,3,6]        6
 8     7     7       [7]  ← 6≤7 popped       [3,6,7]        7

                                              Result = [3,3,5,5,6,7]

Deque invariant (decreasing order):
┌───┬───┬───┐
│ 5 │ 3 │   │  ← front is always window max
└───┴───┴───┘
  ▲           ▲
front        back
(max)     (new elements
           must be ≤ front)
```

**Why?** The deque front always holds the maximum of the current window. We remove from the back anything ≤ new element (they'll never be the max while the new element is in the window).

---

## Example 2: Sliding Window Minimum

```go
package main

import "fmt"

func minSlidingWindow(nums []int, k int) []int {
    deque := []int{} // monotonic increasing deque (stores indices)
    result := []int{}

    for i, num := range nums {
        // Remove out-of-window indices
        for len(deque) > 0 && deque[0] < i-k+1 {
            deque = deque[1:]
        }
        // Remove LARGER elements from back (maintain increasing order)
        for len(deque) > 0 && nums[deque[len(deque)-1]] >= num {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}

func main() {
    nums := []int{8, 5, 10, 7, 9, 4, 15, 12, 90, 13}
    fmt.Printf("Window min (k=4): %v\n", minSlidingWindow(nums, 4))
    // [5, 5, 4, 4, 4, 4, 12]
}
```

**Textual Figure:**
```
nums = [8, 5, 10, 7, 9, 4, 15, 12, 90, 13],  k = 4

Maintain INCREASING deque (front = min)

Step   i   nums[i]   Deque (values)       Window             Min
──────────────────────────────────────────────────────────────────
 1     0     8       [8]                   [8,_,_,_]           -
 2     1     5       [5]  ← 8≥5 popped    [8,5,_,_]           -
 3     2    10       [5,10]                [8,5,10,_]          -
 4     3     7       [5,7] ← 10≥7 popped  [8,5,10,7]          5
 5     4     9       [5,7,9]               [5,10,7,9]          5
 6     5     4       [4]  ← all≥4 popped   [10,7,9,4]          4
 7     6    15       [4,15]                [7,9,4,15]          4
 8     7    12       [4,12] ←15≥12 popped  [9,4,15,12]         4
 9     8    90       [4,12,90]             [4,15,12,90]        4
10     9    13       [12,13] ← 4 expired   [15,12,90,13]      12
                       90≥13 popped

Result = [5, 5, 4, 4, 4, 4, 12]

Deque invariant (increasing order):
┌───┬───┬────┐
│ 4 │ 12│ 90 │  ← front is always window min
└───┴───┴────┘
  ▲            ▲
front         back
(min)      (new elements
            must be ≥ front)
```

---

## Example 3: Sum of Subarray Minimums (LeetCode 907)

For every subarray, sum the minimum element. Use monotonic stack + contribution counting.

```go
package main

import "fmt"

func sumSubarrayMins(arr []int) int {
    const MOD = 1_000_000_007
    n := len(arr)
    // left[i] = number of subarrays ending at i where arr[i] is min
    // right[i] = number of subarrays starting at i where arr[i] is min
    left := make([]int, n)
    right := make([]int, n)

    // Monotonic increasing stack for left boundary
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

    // Monotonic increasing stack for right boundary
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

    total := 0
    for i := 0; i < n; i++ {
        total = (total + arr[i]*left[i]*right[i]) % MOD
    }
    return total
}

func main() {
    fmt.Println(sumSubarrayMins([]int{3, 1, 2, 4}))   // 17
    fmt.Println(sumSubarrayMins([]int{11, 81, 94, 43, 3})) // 444
}
```

**Textual Figure:**
```
arr = [3, 1, 2, 4]

Contribution counting: each element × (left span) × (right span)

  Index:    0     1     2     3
  Value:   [3]   [1]   [2]   [4]

Monotonic stack finds boundaries:
┌───────┬───────┬────────┬────────┬──────────────┐
│ Value │ left  │ right  │ Count  │ Contribution │
│       │ span  │ span   │ l × r  │  val × count │
├───────┼───────┼────────┼────────┼──────────────┤
│   3   │   1   │   1    │   1    │   3 × 1 = 3  │
│   1   │   2   │   3    │   6    │   1 × 6 = 6  │
│   2   │   1   │   2    │   2    │   2 × 2 = 4  │
│   4   │   1   │   1    │   1    │   4 × 1 = 4  │
└───────┴───────┴────────┴────────┴──────────────┘
                              Total = 3+6+4+4 = 17  ✓

Subarray mins verification:
 [3]=3  [1]=1  [2]=2  [4]=4
 [3,1]=1  [1,2]=1  [2,4]=2
 [3,1,2]=1  [1,2,4]=1
 [3,1,2,4]=1
 Sum = 3+1+2+4+1+1+2+1+1+1 = 17  ✓
```

---

## Example 4: Longest Continuous Subarray With Absolute Diff ≤ Limit (LeetCode 1438)

Uses two monotonic deques — one for max, one for min.

```go
package main

import "fmt"

func longestSubarray(nums []int, limit int) int {
    maxDeque := []int{} // decreasing (front = max)
    minDeque := []int{} // increasing (front = min)
    left := 0
    result := 0

    for right := 0; right < len(nums); right++ {
        // Maintain max deque (decreasing)
        for len(maxDeque) > 0 && nums[maxDeque[len(maxDeque)-1]] <= nums[right] {
            maxDeque = maxDeque[:len(maxDeque)-1]
        }
        maxDeque = append(maxDeque, right)

        // Maintain min deque (increasing)
        for len(minDeque) > 0 && nums[minDeque[len(minDeque)-1]] >= nums[right] {
            minDeque = minDeque[:len(minDeque)-1]
        }
        minDeque = append(minDeque, right)

        // Shrink window while diff exceeds limit
        for nums[maxDeque[0]]-nums[minDeque[0]] > limit {
            left++
            if maxDeque[0] < left {
                maxDeque = maxDeque[1:]
            }
            if minDeque[0] < left {
                minDeque = minDeque[1:]
            }
        }

        if right-left+1 > result {
            result = right - left + 1
        }
    }
    return result
}

func main() {
    tests := []struct {
        nums  []int
        limit int
    }{
        {[]int{8, 2, 4, 7}, 4},
        {[]int{10, 1, 2, 4, 7, 2}, 5},
        {[]int{4, 2, 2, 2, 4, 4, 2, 2}, 0},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, limit=%d → %d\n", t.nums, t.limit, longestSubarray(t.nums, t.limit))
    }
    // 2, 4, 3
}
```

**Textual Figure:**
```
nums = [8, 2, 4, 7],  limit = 4

Two deques: maxDeque (decreasing), minDeque (increasing)
Expand right, shrink left when max-min > limit

Step  left  right  Window     maxDeque  minDeque  max  min  diff  Action
──────────────────────────────────────────────────────────────────────────
 1     0     0     [8]        [8]       [8]        8    8    0    expand
 2     0     1     [8,2]      [8,2]     [2]        8    2    6    shrink!
 3     1     1     [2]        [2]       [2]        2    2    0    expand
 4     1     2     [2,4]      [4]       [2,4]      4    2    2    expand →len=2
 5     1     3     [2,4,7]    [7]       [2,4,7]    7    2    5    shrink!
 6     2     3     [4,7]      [7]       [4,7]      7    4    3    expand
 7     2     4     end

Best = len([2,4,7,_]) at step 5 → but diff>limit
       len([8,2]) at step 2 → diff>limit
       Longest valid windows: [2,4] or [4,7] → length = 2  ✓

  maxDeque (decreasing):      minDeque (increasing):
  ┌───┬───┐                   ┌───┬───┬───┐
  │ 7 │   │                   │ 2 │ 4 │ 7 │
  └───┴───┘                   └───┴───┴───┘
  front=max                   front=min
```

---

## Example 5: Jump Game VI (LeetCode 1696)

```go
package main

import "fmt"

func maxResult(nums []int, k int) int {
    n := len(nums)
    dp := make([]int, n)
    dp[0] = nums[0]

    deque := []int{0} // monotonic decreasing deque of indices (by dp value)

    for i := 1; i < n; i++ {
        // Remove out-of-range indices
        for len(deque) > 0 && deque[0] < i-k {
            deque = deque[1:]
        }

        dp[i] = dp[deque[0]] + nums[i]

        // Maintain decreasing order by dp value
        for len(deque) > 0 && dp[deque[len(deque)-1]] <= dp[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)
    }

    return dp[n-1]
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1, -1, -2, 4, -7, 3}, 2},
        {[]int{10, -5, -2, 4, 0, 3}, 3},
        {[]int{1, -5, -20, 4, -1, 3, -6, -3}, 2},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %d\n", t.nums, t.k, maxResult(t.nums, t.k))
    }
    // 7, 17, 0
}
```

**Textual Figure:**
```
nums = [1, -1, -2, 4, -7, 3],  k = 2

dp[i] = nums[i] + max(dp[j]) for j in [i-k, i-1]
Deque maintains decreasing order of dp values

i   nums[i]   dp[i]                  Deque (idx→dp)     Max from deque
─────────────────────────────────────────────────────────────────────────
0     1        1                      [0→1]               -
1    -1        -1 + dp[0] = 0         [0→1, 1→0]          dp[0]=1
2    -2        -2 + dp[0] = -1        [0→1, 1→0, 2→-1]    dp[0]=1
                                      → 0 expires (2-0≥k)
                                      [1→0, 2→-1]         dp[1]=0
               recalc: -2+0 = -2      [1→0, 2→-1]→[2→-2]  
               Actually: dp[2]=-2+max(dp[0],dp[1])=-2+1=-1
3     4        4 + max(dp[1],dp[2])   Deque front: dp[1]=0
               = 4 + 0 = 4            [3→4]               dp[1]=0
4    -7        -7 + max(dp[2],dp[3])  Deque front: dp[3]=4
               = -7 + 4 = -3          [3→4, 4→-3]         dp[3]=4
5     3        3 + max(dp[3],dp[4])   Deque front: dp[3]=4
               = 3 + 4 = 7            [5→7]               dp[3]=4

dp = [1, 0, -1, 4, -3, 7]  →  answer = dp[5] = 7  ✓

Deque acts as sliding window max of dp values:
┌──────────────────────────────────┐
│  Window size ≤ k (look back k)   │
│  Deque front = best dp to jump   │
│  from within the last k steps    │
└──────────────────────────────────┘
```

---

## Example 6: Shortest Subarray with Sum at Least K (LeetCode 862)

```go
package main

import "fmt"

func shortestSubarray(nums []int, k int) int {
    n := len(nums)
    prefix := make([]int64, n+1)
    for i := 0; i < n; i++ {
        prefix[i+1] = prefix[i] + int64(nums[i])
    }

    deque := []int{} // monotonic increasing deque of prefix indices
    result := n + 1

    for i := 0; i <= n; i++ {
        // Check if we found a valid subarray
        for len(deque) > 0 && prefix[i]-prefix[deque[0]] >= int64(k) {
            length := i - deque[0]
            if length < result {
                result = length
            }
            deque = deque[1:]
        }

        // Maintain increasing order of prefix sums
        for len(deque) > 0 && prefix[deque[len(deque)-1]] >= prefix[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)
    }

    if result == n+1 {
        return -1
    }
    return result
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1}, 1},
        {[]int{1, 2}, 4},
        {[]int{2, -1, 2}, 3},
        {[]int{84, -37, 32, 40, 95}, 167},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %d\n", t.nums, t.k, shortestSubarray(t.nums, t.k))
    }
    // 1, -1, 3, 3
}
```

**Textual Figure:**
```
nums = [2, -1, 2],  k = 3

prefix = [0, 2, 1, 3]
         p0 p1 p2 p3

Deque stores indices with INCREASING prefix values.
For each i, check front: prefix[i] - prefix[front] ≥ k → record length.

  i   prefix[i]  Deque (idx→prefix)     Check front         result
─────────────────────────────────────────────────────────────────
  0     0         [0→0]                  -                   ∞
  1     2         [0→0, 1→2]             2-0=2 < 3           ∞
  2     1         [0→0, 2→1]             1-0=1 < 3           ∞
                  (1→2 popped: 2≥1)
  3     3         [0→0, 2→1, 3→3]       3-0=3 ≥ 3 → len=3   3
                  pop 0; 3-1=2 < 3
                  [2→1, 3→3]

Answer = 3 (subarray [2,-1,2] sums to 3 ≥ k)

Why increasing prefix?
  prefix:   0     2     1     3
            │                 │
            └───────────────┘
            diff = 3 ≥ k  → subarray length = 3-0 = 3
```

---

## Example 7: Constrained Subsequence Sum (LeetCode 1425)

```go
package main

import "fmt"

func constrainedSubsetSum(nums []int, k int) int {
    n := len(nums)
    dp := make([]int, n)
    copy(dp, nums)

    deque := []int{} // decreasing deque by dp value
    result := dp[0]

    for i := 0; i < n; i++ {
        // Remove out-of-window
        for len(deque) > 0 && deque[0] < i-k {
            deque = deque[1:]
        }

        if len(deque) > 0 && dp[deque[0]] > 0 {
            dp[i] = nums[i] + dp[deque[0]]
        }

        // Maintain decreasing order
        for len(deque) > 0 && dp[deque[len(deque)-1]] <= dp[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        if dp[i] > result {
            result = dp[i]
        }
    }
    return result
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{10, 2, -10, 5, 20}, 2},
        {[]int{-1, -2, -3}, 1},
        {[]int{10, -2, -10, -5, 20}, 2},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %d\n", t.nums, t.k, constrainedSubsetSum(t.nums, t.k))
    }
    // 37, -1, 23
}
```

**Textual Figure:**
```
nums = [10, 2, -10, 5, 20],  k = 2

dp[i] = nums[i] + max(0, max(dp[j]) for j in [i-k..i-1])
Deque stores indices in DECREASING dp value order.

  i   nums[i]  Best from deque   dp[i]        Deque (idx→dp)
─────────────────────────────────────────────────────────────
  0    10       -                 10           [0→10]
  1     2       dp[0]=10          12           [1→12]
  2   -10       dp[1]=12           2           [1→12, 2→2]
  3     5       dp[1]=12          17           [3→17]
                (1 expires at i=3)
                dp[2]=2           7→use dp[2]  
                Actually max(dp[1],dp[2])=12   
                dp[3]=5+12=17                  [3→17]
  4    20       dp[3]=17          37           [4→37]

dp = [10, 12, 2, 17, 37]
result = max(dp) = 37  ✓

Subsequence: [10, 2, 5, 20] → sum = 37
  indices:    0   1  3  4  (each gap ≤ k=2)  ✓
```

---

## Example 8: Max Value of Equation (LeetCode 1499)

Given points sorted by x, find max `yi + yj + |xi - xj|` where `|xi - xj| ≤ k`.

Since points are sorted: `|xi - xj| = xj - xi`, so maximize `(yi - xi) + (yj + xj)`.

```go
package main

import "fmt"

func findMaxValueOfEquation(points [][]int, k int) int {
    // Monotonic decreasing deque of (y - x, x)
    type Pair struct {
        Val int // y - x
        X   int
    }

    deque := []Pair{}
    result := -(1 << 60)

    for _, p := range points {
        x, y := p[0], p[1]

        // Remove elements outside window [x-k, x]
        for len(deque) > 0 && x-deque[0].X > k {
            deque = deque[1:]
        }

        if len(deque) > 0 {
            val := deque[0].Val + y + x
            if val > result {
                result = val
            }
        }

        // Maintain decreasing order of (y - x)
        cur := y - x
        for len(deque) > 0 && deque[len(deque)-1].Val <= cur {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, Pair{cur, x})
    }

    return result
}

func main() {
    tests := []struct {
        points [][]int
        k      int
    }{
        {[][]int{{1, 3}, {2, 0}, {5, 10}, {6, -10}}, 1},
        {[][]int{{0, 0}, {3, 0}, {9, 2}}, 3},
    }

    for _, t := range tests {
        fmt.Printf("k=%d → %d\n", t.k, findMaxValueOfEquation(t.points, t.k))
    }
    // 4, 3
}
```

**Textual Figure:**
```
points = [[1,3], [2,0], [5,10], [6,-10]],  k = 1

Maximize: (yi - xi) + (yj + xj)  where xj - xi ≤ k
Rewrite as: val_i = y - x (stored), score = val_front + yj + xj

Deque stores DECREASING values of (y - x)

  j   point   yj-xj   Deque (val,x)              Score           best
─────────────────────────────────────────────────────────────────────
  0   [1,3]    2      [(2,1)]                     -              -∞
  1   [2,0]   -2      [(2,1),(-2,2)]              2+0+2=4        4
  2   [5,10]   5      [(5,5)]                     -2+10+5=13     
                      expire (2,1): 5-1>1          but (2,1) expired
                      expire (-2,2): 5-2>1         all expired!
                      no score this step           4
  3   [6,-10] -16     [(5,5),(-16,6)]             5+(-10)+6=1    4

Answer = 4  (points [1,3] and [2,0]: 3+0+|1-2| = 4)  ✓
```

---

## Example 9: Number of Visible People in a Queue (LeetCode 1944)

```go
package main

import "fmt"

func canSeePersonsCount(heights []int) []int {
    n := len(heights)
    result := make([]int, n)
    stack := []int{} // monotonic decreasing stack of indices

    for i := n - 1; i >= 0; i-- {
        count := 0
        // Pop all shorter people — person i can see them
        for len(stack) > 0 && heights[stack[len(stack)-1]] < heights[i] {
            stack = stack[:len(stack)-1]
            count++
        }
        // If stack not empty, person i can also see the next taller person
        if len(stack) > 0 {
            count++
        }
        result[i] = count
        stack = append(stack, i)
    }
    return result
}

func main() {
    tests := [][]int{
        {10, 6, 8, 5, 11, 9},
        {5, 1, 2, 3, 10},
    }

    for _, h := range tests {
        fmt.Printf("heights=%v → %v\n", h, canSeePersonsCount(h))
    }
    // [3,1,2,1,1,0]
    // [4,1,1,1,0]
}
```

**Textual Figure:**
```
heights = [10, 6, 8, 5, 11, 9]
             0  1  2  3   4  5

Monotonic stack from RIGHT (decreasing traversal):
Person i can see everyone they pop + top of remaining stack.

  i   h[i]   Stack (heights)   Popped    +Top?   result[i]
─────────────────────────────────────────────────────────────
  5    9     [9]               0         no       0
  4   11     [11]              9(1)      no       1
  3    5     [5,11]            0         +11      1
  2    8     [8,11]            5(1)      +11      2
  1    6     [6,8,11]          0         +8       1
  0   10     [10,11]           6,8 (2)   +11      3

result = [3, 1, 2, 1, 1, 0]  ✓

Visualization (who can person 0 see?):
  10   6   8   5   11   9
  [0]──▶[1]─▶[2]      Sees 6 (shorter, popped)
   │       │          Sees 8 (shorter, popped)
   └───────┴──▶[4]    Sees 11 (taller, stops)
   Total: 3 people
```

---

## Example 10: Monotonic Queue Generic Implementation

```go
package main

import "fmt"

type MonoQueue struct {
    deque []int
    less  func(a, b int) bool // true means a has higher priority
}

// NewDecreasingQueue: front is always the max
func NewDecreasingQueue() *MonoQueue {
    return &MonoQueue{less: func(a, b int) bool { return a > b }}
}

// NewIncreasingQueue: front is always the min
func NewIncreasingQueue() *MonoQueue {
    return &MonoQueue{less: func(a, b int) bool { return a < b }}
}

func (mq *MonoQueue) Push(val int) {
    for len(mq.deque) > 0 && !mq.less(mq.deque[len(mq.deque)-1], val) {
        mq.deque = mq.deque[:len(mq.deque)-1]
    }
    mq.deque = append(mq.deque, val)
}

func (mq *MonoQueue) PopFront(val int) {
    // Remove front if it equals the value being removed from the window
    if len(mq.deque) > 0 && mq.deque[0] == val {
        mq.deque = mq.deque[1:]
    }
}

func (mq *MonoQueue) Front() int {
    return mq.deque[0]
}

func (mq *MonoQueue) Empty() bool {
    return len(mq.deque) == 0
}

// ---- Sliding window max using the generic MonoQueue ----

func slidingWindowMax(nums []int, k int) []int {
    mq := NewDecreasingQueue()
    result := []int{}

    for i, num := range nums {
        mq.Push(num)
        if i >= k {
            mq.PopFront(nums[i-k]) // remove element leaving the window
        }
        if i >= k-1 {
            result = append(result, mq.Front())
        }
    }
    return result
}

// ---- Sliding window min ----

func slidingWindowMin(nums []int, k int) []int {
    mq := NewIncreasingQueue()
    result := []int{}

    for i, num := range nums {
        mq.Push(num)
        if i >= k {
            mq.PopFront(nums[i-k])
        }
        if i >= k-1 {
            result = append(result, mq.Front())
        }
    }
    return result
}

func main() {
    nums := []int{1, 3, -1, -3, 5, 3, 6, 7}
    k := 3
    fmt.Printf("Window max (k=%d): %v\n", k, slidingWindowMax(nums, k))
    fmt.Printf("Window min (k=%d): %v\n", k, slidingWindowMin(nums, k))
    // Max: [3 3 5 5 6 7]
    // Min: [-1 -3 -3 -3 3 3]
}
```

**Textual Figure:**
```
Generic Monotonic Queue Pattern:

           Push(val)                       PopFront(val)
           │                                │
           ▼                                ▼
  ┌─────────────────────────────────────────┐
  │  front                          back  │
  │  ┌───┬───┬───┬───┬───┐              │
  │  │ 5 │ 3 │ 2 │ 1 │   │  ← Decreasing │
  │  └───┴───┴───┴───┴───┘    (for max)  │
  │       ▲                              │
  │  ┌───┬───┬───┬───┬───┐              │
  │  │ 1 │ 2 │ 3 │ 5 │   │  ← Increasing │
  │  └───┴───┴───┴───┴───┘    (for min)  │
  └─────────────────────────────────────────┘

nums = [1, 3, -1, -3, 5, 3, 6, 7],  k = 3

Decreasing Queue (Max):           Increasing Queue (Min):
Window [1,3,-1]:  [3,-1]→max=3    [−1]     →min=−1
Window [3,-1,-3]: [3,-1,-3]→max=3 [−3]     →min=−3
Window [-1,-3,5]: [5]    →max=5    [−3,5]   →min=−3
Window [-3,5,3]:  [5,3]  →max=5    [−3,3]   →min=−3
Window [5,3,6]:   [6]    →max=6    [3,6]    →min=3
Window [3,6,7]:   [7]    →max=7    [3,6,7]  →min=3

Max: [3,3,5,5,6,7]    Min: [-1,-3,-3,-3,3,3]  ✓
```

---

## Monotonic Queue Patterns

| Problem | Monotone Order | Key Insight |
|---------|---------------|-------------|
| Sliding window max | Decreasing | Front = max in window |
| Sliding window min | Increasing | Front = min in window |
| Shortest subarray ≥ K | Increasing prefix | Prefix sums + deque |
| Jump Game VI | Decreasing dp | Best reachable dp value |
| Max diff in window | Both deques | Two deques simultaneously |
| Subarray min sum | Increasing (stack) | Contribution counting |

## Key Takeaways

1. A monotonic queue processes n elements in **O(n)** total — each element enters and leaves at most once
2. **Decreasing deque** → front is always the window's **maximum**
3. **Increasing deque** → front is always the window's **minimum**
4. Store **indices** (not values) when you need to check if elements are within the window boundary
5. The pattern works even with negative numbers (unlike two-pointer approaches)
6. **Two deques** — one for max, one for min — solve "longest subarray with bounded diff"
7. Monotonic queue is the sliding window version of monotonic stack

> **Next up:** BFS Queue Pattern →
