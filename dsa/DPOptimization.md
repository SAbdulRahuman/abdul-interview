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

---

## Key Takeaways

1. **Space**: if dp[i] only depends on dp[i-1], use rolling array or two variables
2. **0/1 Knapsack**: reverse iteration converts 2D to 1D in O(W) space
3. **LIS in O(n log n)**: maintain tails array with binary search
4. **Monotonic queue**: O(n) sliding window min/max replaces O(n·k) naive approach
5. Always check: "does dp[i][j] really need the full table, or just a few rows?"

> **Next up:** 0/1 Knapsack →
