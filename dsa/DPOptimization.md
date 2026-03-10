# Phase 15: Dynamic Programming — DP Optimization

## Overview

**DP optimization** techniques reduce time or space complexity of DP solutions beyond the basic approach. Common optimizations include space reduction, monotonic queue/stack, divide and conquer optimization, and Knuth's optimization.

| Optimization | Reduces | From → To |
|-------------|---------|-----------|
| Rolling array | Space | O(n·m) → O(m) |
| Two variables | Space | O(n) → O(1) |
| Binary search on DP | Time | O(n²) → O(n log n) |
| Monotonic queue | Time | O(n·k) → O(n) |
| Convex hull trick | Time | O(n²) → O(n) |

---

## Example 1: Space Optimization — Two Variables

```go
package main

import "fmt"

// Fibonacci O(1) space
func fib(n int) int {
	if n <= 1 { return n }
	prev2, prev1 := 0, 1

	for i := 2; i <= n; i++ {
		curr := prev1 + prev2
		prev2, prev1 = prev1, curr
	}
	return prev1
}

func main() {
	for n := 0; n <= 15; n++ {
		fmt.Printf("fib(%2d) = %d\n", n, fib(n))
	}
}
```

**Textual Figure:**

```
Fibonacci O(1) space — two sliding variables:

  n:    0   1   2   3   4   5   6   7
  fib:  0   1   1   2   3   5   8  13

  ┌──────┬───────┬───────┬──────────────────┐
  │ Step │ prev2 │ prev1 │ curr = prev2+prev1│
  ├──────┼───────┼───────┼──────────────────┤
  │ init │   0   │   1   │                  │
  │ i=2  │   0   │   1   │ 0+1 = 1          │
  │      │   1   │   1   │ (shift →)         │
  │ i=3  │   1   │   1   │ 1+1 = 2          │
  │      │   1   │   2   │ (shift →)         │
  │ i=4  │   1   │   2   │ 1+2 = 3          │
  │      │   2   │   3   │ (shift →)         │
  │ i=5  │   2   │   3   │ 2+3 = 5          │
  │      │   3   │   5   │ (shift →)         │
  │ i=6  │   3   │   5   │ 3+5 = 8          │
  │      │   5   │   8   │ (shift →)         │
  └──────┴───────┴───────┴──────────────────┘

  Only 2 variables at any time → O(1) space
  Result: fib(6) = 8
```

---

## Example 2: Rolling Array — 2D → 1D

```go
package main

import "fmt"

// 0/1 Knapsack: O(n·W) space → O(W) space using rolling array
func knapsack(weights, values []int, W int) int {
	dp := make([]int, W+1)

	for i := 0; i < len(weights); i++ {
		// Iterate W → weights[i] (reverse!) to avoid using item twice
		for w := W; w >= weights[i]; w-- {
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[W]
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	fmt.Println("Max value:", knapsack(weights, values, 8)) // 10
}
```

**Textual Figure:**

```
0/1 Knapsack rolling 1D array — weights=[2,3,4,5], values=[3,4,5,6], W=8

  Reverse iteration prevents reusing same item:

  dp[w] initial: [0, 0, 0, 0, 0, 0, 0, 0, 0]
                  w: 0  1  2  3  4  5  6  7  8

  Item 0 (w=2, v=3): scan w=8→2
    dp: [0, 0, 3, 3, 3, 3, 3, 3, 3]

  Item 1 (w=3, v=4): scan w=8→3
    dp: [0, 0, 3, 4, 4, 7, 7, 7, 7]

  Item 2 (w=4, v=5): scan w=8→4
    dp: [0, 0, 3, 4, 5, 7, 8, 9, 9]

  Item 3 (w=5, v=6): scan w=8→5
    dp: [0, 0, 3, 4, 5, 7, 8, 9,10]
                                    ↑
                              max value = 10

  Reverse scan ensures dp[w-wt] reads old row values
```

**Why reverse?** Ensures each item is used at most once — we read from dp[w-wt] before writing dp[w].

---

## Example 3: Rolling Array — LCS Space Optimized

```go
package main

import "fmt"

// LCS: O(m·n) → O(min(m,n)) space
func longestCommonSubsequence(s1, s2 string) int {
	if len(s1) < len(s2) { s1, s2 = s2, s1 }
	m, n := len(s1), len(s2)

	dp := make([]int, n+1)

	for i := 1; i <= m; i++ {
		prev := 0 // dp[i-1][j-1]
		for j := 1; j <= n; j++ {
			temp := dp[j]
			if s1[i-1] == s2[j-1] {
				dp[j] = prev + 1
			} else {
				if dp[j-1] > dp[j] { dp[j] = dp[j-1] }
			}
			prev = temp
		}
	}
	return dp[n]
}

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace")) // 3
}
```

**Textual Figure:**

```
LCS "abcde" vs "ace" — 1D rolling array, O(min(m,n)) space:

  dp[] tracks current row, prev saves dp[j] before overwrite

         ""   a   c   e
  init: [ 0,  0,  0,  0]

  i=1 (a):  a==a at j=1 → dp[1]=0+1=1
    dp: [ 0,  1,  1,  1]

  i=2 (b):  no matches
    dp: [ 0,  1,  1,  1]

  i=3 (c):  c==c at j=2 → dp[2]=prev+1=1+1=2
    dp: [ 0,  1,  2,  2]

  i=4 (d):  no matches
    dp: [ 0,  1,  2,  2]

  i=5 (e):  e==e at j=3 → dp[3]=prev+1=2+1=3
    dp: [ 0,  1,  2,  3]
                       ↑
  Result: dp[3] = 3  (LCS = "ace")
```

---

## Example 4: Binary Search Optimization — LIS O(n log n) (LeetCode 300)

```go
package main

import (
	"fmt"
	"sort"
)

// LIS: O(n²) → O(n log n) using patience sorting with binary search
func lengthOfLIS(nums []int) int {
	tails := []int{} // tails[i] = smallest tail for IS of length i+1

	for _, num := range nums {
		pos := sort.SearchInts(tails, num)
		if pos == len(tails) {
			tails = append(tails, num)
		} else {
			tails[pos] = num
		}
	}
	return len(tails)
}

func main() {
	tests := [][]int{
		{10, 9, 2, 5, 3, 7, 101, 18},
		{0, 1, 0, 3, 2, 3},
		{7, 7, 7, 7, 7, 7, 7},
		{1, 3, 6, 7, 9, 4, 10, 5, 6},
	}

	for _, nums := range tests {
		fmt.Printf("LIS of %v = %d\n", nums, lengthOfLIS(nums))
	}
}
```

**Textual Figure:**

```
LIS via patience sorting — tails[i] = smallest tail of IS length i+1:

  nums = [10, 9, 2, 5, 3, 7, 101, 18]

  ┌─────────┬────────────┬─────────────┬───────────┐
  │  num    │  action     │  tails        │  LIS len  │
  ├─────────┼────────────┼─────────────┼───────────┤
  │  10     │ append      │ [10]         │    1     │
  │   9     │ replace @0  │ [9]          │    1     │
  │   2     │ replace @0  │ [2]          │    1     │
  │   5     │ append      │ [2,5]        │    2     │
  │   3     │ replace @1  │ [2,3]        │    2     │
  │   7     │ append      │ [2,3,7]      │    3     │
  │ 101     │ append      │ [2,3,7,101]  │    4     │
  │  18     │ replace @3  │ [2,3,7,18]   │    4     │
  └─────────┴────────────┴─────────────┴───────────┘

  Binary search finds insertion point → O(log n) per element
  tails always sorted → SearchInts works correctly
  Result: len(tails) = 4
```

---

## Example 5: Monotonic Queue Optimization — Sliding Window Maximum (LeetCode 239)

```go
package main

import "fmt"

// Max in sliding window of size k
// Naive DP: O(n·k), with deque: O(n)

func maxSlidingWindow(nums []int, k int) []int {
	deque := []int{} // indices, front = max
	result := []int{}

	for i := 0; i < len(nums); i++ {
		// Remove elements out of window
		for len(deque) > 0 && deque[0] < i-k+1 {
			deque = deque[1:]
		}

		// Remove smaller elements from back (they'll never be max)
		for len(deque) > 0 && nums[deque[len(deque)-1]] <= nums[i] {
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
	fmt.Println(maxSlidingWindow([]int{1,3,-1,-3,5,3,6,7}, 3))
	// [3 3 5 5 6 7]
}
```

**Textual Figure:**

```
Monotonic deque — sliding window max, k=3:

  nums = [1, 3, -1, -3, 5, 3, 6, 7]

  ┌───┬────────┬──────────────────────────┬───────────────┐
  │ i │ nums[i]│ deque (indices, front=max)│ result         │
  ├───┼────────┼──────────────────────────┼───────────────┤
  │ 0 │   1    │ [0]                      │                │
  │ 1 │   3    │ [1]     (1>0, pop 0)     │                │
  │ 2 │  -1    │ [1,2]                    │ [3]            │
  │ 3 │  -3    │ [1,2,3]                  │ [3,3]          │
  │ 4 │   5    │ [4]     (clear all)      │ [3,3,5]        │
  │ 5 │   3    │ [4,5]                    │ [3,3,5,5]      │
  │ 6 │   6    │ [6]     (clear all)      │ [3,3,5,5,6]    │
  │ 7 │   7    │ [7]     (clear all)      │ [3,3,5,5,6,7]  │
  └───┴────────┴──────────────────────────┴───────────────┘

  Deque invariant: front = index of current window max
  Back-pop: nums[back] ≤ nums[i] (they can never be max)
  Front-pop: index < i-k+1 (out of window)
  Result: [3, 3, 5, 5, 6, 7]
```

---

## Example 6: House Robber — O(1) Space

```go
package main

import "fmt"

func rob(nums []int) int {
	if len(nums) == 0 { return 0 }
	if len(nums) == 1 { return nums[0] }

	prev2, prev1 := 0, 0
	for _, num := range nums {
		curr := max(prev1, prev2+num)
		prev2, prev1 = prev1, curr
	}
	return prev1
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(rob([]int{2, 7, 9, 3, 1}))   // 12
	fmt.Println(rob([]int{1, 2, 3, 1}))       // 4
}
```

**Textual Figure:**

```
House Robber O(1) space — two rolling variables:

  nums = [2, 7, 9, 3, 1]

  ┌──────┬──────┬───────┬───────┬────────────────────────┐
  │ Step │  num │ prev2 │ prev1 │ curr = max(p1, p2+num) │
  ├──────┼──────┼───────┼───────┼────────────────────────┤
  │ init │      │   0   │   0   │                        │
  │ i=0  │  2   │   0   │   0   │ max(0, 0+2) = 2       │
  │      │      │   0   │   2   │ (shift →)              │
  │ i=1  │  7   │   0   │   2   │ max(2, 0+7) = 7       │
  │      │      │   2   │   7   │ (shift →)              │
  │ i=2  │  9   │   2   │   7   │ max(7, 2+9) = 11      │
  │      │      │   7   │  11   │ (shift →)              │
  │ i=3  │  3   │   7   │  11   │ max(11, 7+3) = 11     │
  │      │      │  11   │  11   │ (shift →)              │
  │ i=4  │  1   │  11   │  11   │ max(11, 11+1) = 12    │
  └──────┴──────┴───────┴───────┴────────────────────────┘

  Result: 12  (rob houses at index 0, 2, 4 → 2+9+1)
```

---

## Example 7: Edit Distance — Two-Row Optimization

```go
package main

import "fmt"

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	prev := make([]int, n+1)
	curr := make([]int, n+1)

	for j := 0; j <= n; j++ { prev[j] = j }

	for i := 1; i <= m; i++ {
		curr[0] = i
		for j := 1; j <= n; j++ {
			if word1[i-1] == word2[j-1] {
				curr[j] = prev[j-1]
			} else {
				curr[j] = 1 + min3(prev[j], curr[j-1], prev[j-1])
			}
		}
		prev, curr = curr, prev
	}
	return prev[n]
}

func min3(a, b, c int) int {
	if a < b && a < c { return a }
	if b < c { return b }
	return c
}

func main() {
	fmt.Println(minDistance("horse", "ros"))         // 3
	fmt.Println(minDistance("intention", "execution")) // 5
}
```

**Textual Figure:**

```
Edit Distance "horse" → "ros" — two-row rolling array:

  prev starts as base row, curr computed from prev:

         ""   r    o    s
  init: [ 0,  1,   2,   3]  ← prev (insert all)

  h →   [ 1,  1,   2,   3]  h≠r:1+min(0,1,1)=1
  o →   [ 2,  2,   1,   2]  o=o:diag=1
  r →   [ 3,  2,   2,   2]  r=r:diag=2
  s →   [ 4,  3,   3,   2]  s=s:diag=2
  e →   [ 5,  4,   4,   3]  e≠s:1+min(2,4,3)=3
                          ↑
  Only prev[] and curr[] needed → O(n) space
  After each row: swap(prev, curr)

  Result: 3  (horse → rorse → rose → ros)
```

---

## Example 8: DP with Prefix Sums

```go
package main

import "fmt"

// Minimum cost to merge stones (simplified)
// DP + prefix sums to compute range sums in O(1)

func minCostMerge(stones []int) int {
	n := len(stones)

	// Prefix sums
	prefix := make([]int, n+1)
	for i := 0; i < n; i++ {
		prefix[i+1] = prefix[i] + stones[i]
	}
	rangeSum := func(i, j int) int { return prefix[j+1] - prefix[i] }

	// dp[i][j] = min cost to merge stones[i..j] into one pile
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
	}

	for length := 2; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			dp[i][j] = 1<<31 - 1
			for k := i; k < j; k++ {
				cost := dp[i][k] + dp[k+1][j] + rangeSum(i, j)
				if cost < dp[i][j] { dp[i][j] = cost }
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(minCostMerge([]int{3, 1, 5, 8})) // 34
}
```

**Textual Figure:**

```
Interval DP + Prefix Sums — merge stones [3, 1, 5, 8]:

  prefix = [0, 3, 4, 9, 17]
  rangeSum(i,j) = prefix[j+1] - prefix[i]

  DP table dp[i][j] = min cost to merge stones[i..j]:
  ┌────┬─────┬─────┬─────┬─────┐
  │i\j │  0  │  1  │  2  │  3  │
  ├────┼─────┼─────┼─────┼─────┤
  │  0 │  0  │  4  │ 13  │ 30  │
  │  1 │     │  0  │  6  │ 20  │
  │  2 │     │     │  0  │ 13  │
  │  3 │     │     │     │  0  │
  └────┴─────┴─────┴─────┴─────┘

  Fill order (by length):
    len=2: dp[0][1]=4   dp[1][2]=6   dp[2][3]=13
    len=3: dp[0][2]=min(0+6+9, 4+0+9) = 13
           dp[1][3]=min(0+13+14, 6+0+14) = 20
    len=4: dp[0][3]=min(0+20+17, 4+13+17, 13+0+17) = 30

  Prefix sums: O(1) range sum vs O(n) per query
  Result: dp[0][3] = 30
```

---

## Example 9: Coin Change — Eliminating Redundant States

```go
package main

import "fmt"

// Number of ways to make change (not minimum coins)
// Optimization: order coins loop outside amount loop to avoid counting permutations

func coinWays(coins []int, amount int) int {
	dp := make([]int, amount+1)
	dp[0] = 1

	// Each coin processed once; avoids permutation counting
	for _, c := range coins {
		for a := c; a <= amount; a++ {
			dp[a] += dp[a-c]
		}
	}
	return dp[amount]
}

func main() {
	fmt.Println(coinWays([]int{1, 2, 5}, 5))  // 4 ways
	fmt.Println(coinWays([]int{1, 2, 5}, 11)) // 11 ways
}
```

**Textual Figure:**

```
Coin combinations (not permutations) — coins=[1,2,5], amount=5:

  Outer loop: coins, Inner loop: amounts
  dp[a] = number of ways to make amount a

  dp init: [1, 0, 0, 0, 0, 0]  (1 way to make 0)
            a: 0  1  2  3  4  5

  Coin 1 (a=1→5): use only 1-cent coins
    dp: [1, 1, 1, 1, 1, 1]

  Coin 2 (a=2→5):
    dp[2]+=dp[0]=2  dp[3]+=dp[1]=2  dp[4]+=dp[2]=3  dp[5]+=dp[3]=3
    dp: [1, 1, 2, 2, 3, 3]

  Coin 5 (a=5→5):
    dp[5]+=dp[0]=3+1=4
    dp: [1, 1, 2, 2, 3, 4]

  Ways: {1+1+1+1+1, 1+1+1+2, 1+2+2, 5}
  Result: dp[5] = 4
```

---

## Example 10: Optimization Techniques Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== DP Optimization Techniques ===")
	fmt.Println()

	techniques := []struct{ technique, from, to, when string }{
		{"Two variables", "O(n)", "O(1)", "dp[i] depends on dp[i-1], dp[i-2]"},
		{"Rolling array", "O(n·m)", "O(m)", "dp[i] depends only on dp[i-1]"},
		{"Reverse iteration", "O(n·W)", "O(W)", "0/1 knapsack space opt"},
		{"Binary search", "O(n²)", "O(n log n)", "LIS, monotonic property"},
		{"Monotonic queue", "O(n·k)", "O(n)", "Sliding window min/max"},
		{"Prefix sums", "O(n³) per query", "O(n²)", "Range sum in interval DP"},
		{"Convex hull trick", "O(n²)", "O(n)", "Linear transition functions"},
		{"D&C optimization", "O(n²k)", "O(nk log n)", "Monotone SMAWK property"},
	}

	fmt.Printf("%-20s %-10s %-12s %s\n", "Technique", "From", "To", "When")
	fmt.Println(string(make([]byte, 70)))
	for _, t := range techniques {
		fmt.Printf("%-20s %-10s %-12s %s\n", t.technique, t.from, t.to, t.when)
	}
}
```

**Textual Figure:**

```
DP Optimization decision flowchart:

  ┌─────────────────────────────┐
  │ Does dp[i] depend only on  │
  │ dp[i-1] and dp[i-2]?       │
  └────────────┬────────────────┘
       yes │   no
  ┌──────┴──────┐  ┌────────────────────────┐
  │ Two variables │  │ Does dp[i][j] depend     │
  │ O(n) → O(1)   │  │ only on row i-1?         │
  └──────────────┘  └───────────┬────────────┘
                        yes │   no
                   ┌──────┴────────┐  ┌──────────────┐
                   │ Rolling array   │  │ Check for:   │
                   │ O(n·m) → O(m)   │  │ • Monotonic Q │
                   └───────────────┘  │ • Binary srch │
                                       │ • CHT / D&C   │
                                       └──────────────┘

  Complexity reduction:
    Space: O(n·m) → O(m) → O(1)
    Time:  O(n²)  → O(n log n) binary search
           O(n·k) → O(n)     monotonic deque
```

---

## Key Takeaways

1. **Space**: if dp[i] only depends on dp[i-1], use rolling array or two variables
2. **0/1 Knapsack**: reverse iteration converts 2D to 1D in O(W) space
3. **LIS in O(n log n)**: maintain tails array with binary search
4. **Monotonic queue**: O(n) sliding window min/max replaces O(n·k) naive approach
5. Always check: "does dp[i][j] really need the full table, or just a few rows?"

> **Next up:** 0/1 Knapsack →
