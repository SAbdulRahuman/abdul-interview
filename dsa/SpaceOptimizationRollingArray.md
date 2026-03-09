# Phase 15: Dynamic Programming — Space Optimization & Rolling Array

## Overview

Many DP problems use 2D tables where each row only depends on the previous row. We can reduce O(n·m) space to O(m) using a **rolling array** or O(1) space using variables.

| Technique | Space Reduction | When to Use |
|-----------|----------------|-------------|
| **Two rows** | O(n·m) → O(m) | dp[i] depends on dp[i-1] only |
| **Single row + reverse** | O(n·m) → O(m) | 0/1 knapsack style |
| **Rolling variables** | O(m) → O(1) | dp[i] depends on dp[i-1] and dp[i-2] |
| **In-place** | O(n·m) → O(m) | When overwriting doesn't corrupt needed values |

---

## Example 1: Fibonacci — O(1) Space

```go
package main

import "fmt"

// From O(n) array to O(1) two variables
func fibArray(n int) int {
	if n <= 1 { return n }
	dp := make([]int, n+1)
	dp[0], dp[1] = 0, 1
	for i := 2; i <= n; i++ { dp[i] = dp[i-1] + dp[i-2] }
	return dp[n]
}

func fibOptimized(n int) int {
	if n <= 1 { return n }
	a, b := 0, 1
	for i := 2; i <= n; i++ {
		a, b = b, a+b
	}
	return b
}

func main() {
	for i := 0; i <= 10; i++ {
		fmt.Printf("fib(%d) = %d (array) = %d (optimized)\n",
			i, fibArray(i), fibOptimized(i))
	}
}
```

---

## Example 2: Unique Paths — Two-Row Rolling

```go
package main

import "fmt"

// 2D approach: O(m*n) space
func uniquePaths2D(m, n int) int {
	dp := make([][]int, m)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][0] = 1
	}
	for j := 0; j < n; j++ { dp[0][j] = 1 }
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[m-1][n-1]
}

// 1D approach: O(n) space — reuse single row
func uniquePaths1D(m, n int) int {
	dp := make([]int, n)
	for j := range dp { dp[j] = 1 }

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[j] += dp[j-1] // dp[j] already has "above" value
		}
	}
	return dp[n-1]
}

func main() {
	fmt.Println(uniquePaths2D(3, 7), uniquePaths1D(3, 7)) // 28 28
	fmt.Println(uniquePaths2D(7, 3), uniquePaths1D(7, 3)) // 28 28
}
```

---

## Example 3: LCS — Two-Row Rolling

```go
package main

import "fmt"

// Full 2D: O(m*n)
func lcs2D(s, t string) int {
	m, n := len(s), len(t)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s[i-1] == t[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[m][n]
}

// Space optimized: O(min(m,n))
func lcsOptimized(s, t string) int {
	if len(s) < len(t) { s, t = t, s } // ensure t is shorter
	m, n := len(s), len(t)

	prev := make([]int, n+1)
	curr := make([]int, n+1)

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s[i-1] == t[j-1] {
				curr[j] = prev[j-1] + 1
			} else {
				curr[j] = max(prev[j], curr[j-1])
			}
		}
		prev, curr = curr, prev
		for j := range curr { curr[j] = 0 }
	}
	return prev[n]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	s, t := "abcde", "ace"
	fmt.Println(lcs2D(s, t), lcsOptimized(s, t)) // 3 3
}
```

---

## Example 4: Edit Distance — Two-Row

```go
package main

import "fmt"

func editDistance2Row(s, t string) int {
	m, n := len(s), len(t)
	prev := make([]int, n+1)
	curr := make([]int, n+1)

	for j := 0; j <= n; j++ { prev[j] = j }

	for i := 1; i <= m; i++ {
		curr[0] = i
		for j := 1; j <= n; j++ {
			if s[i-1] == t[j-1] {
				curr[j] = prev[j-1]
			} else {
				curr[j] = 1 + min3(prev[j], curr[j-1], prev[j-1])
			}
		}
		prev, curr = curr, prev
		curr = make([]int, n+1) // or reuse
	}
	return prev[n]
}

func min3(a, b, c int) int {
	if a < b { if a < c { return a }; return c }
	if b < c { return b }; return c
}

func main() {
	fmt.Println(editDistance2Row("horse", "ros"))          // 3
	fmt.Println(editDistance2Row("intention", "execution")) // 5
}
```

---

## Example 5: 0/1 Knapsack — Single Row Reverse

```go
package main

import "fmt"

// The classic single-row optimization for 0/1 knapsack
func knapsack(weights, values []int, W int) int {
	dp := make([]int, W+1)

	for i := 0; i < len(weights); i++ {
		// REVERSE iteration ensures each item used at most once
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

	// Why reverse? Forward would let you pick same item multiple times
	// dp[4] uses dp[2] which already includes item 0
	// Reverse ensures dp[2] is still the OLD value when computing dp[4]
}
```

---

## Example 6: Min Path Sum — In-Place Optimization

```go
package main

import "fmt"

// Use the input grid itself as DP table — O(1) extra space
func minPathSum(grid [][]int) int {
	m, n := len(grid), len(grid[0])

	// Fill first row
	for j := 1; j < n; j++ { grid[0][j] += grid[0][j-1] }
	// Fill first column
	for i := 1; i < m; i++ { grid[i][0] += grid[i-1][0] }

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			grid[i][j] += min(grid[i-1][j], grid[i][j-1])
		}
	}
	return grid[m-1][n-1]
}

// Single-row version (doesn't modify input)
func minPathSumOptimized(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	dp := make([]int, n)
	dp[0] = grid[0][0]
	for j := 1; j < n; j++ { dp[j] = dp[j-1] + grid[0][j] }

	for i := 1; i < m; i++ {
		dp[0] += grid[i][0]
		for j := 1; j < n; j++ {
			dp[j] = grid[i][j] + min(dp[j], dp[j-1])
		}
	}
	return dp[n-1]
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	grid := [][]int{{1, 3, 1}, {1, 5, 1}, {4, 2, 1}}
	fmt.Println(minPathSumOptimized(grid)) // 7
}
```

---

## Example 7: Longest Palindromic Subsequence — Space Optimized

```go
package main

import "fmt"

// Standard O(n²) space
func lpsStandard(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := range dp { dp[i] = make([]int, n); dp[i][i] = 1 }
	for l := 2; l <= n; l++ {
		for i := 0; i+l-1 < n; i++ {
			j := i + l - 1
			if s[i] == s[j] { dp[i][j] = dp[i+1][j-1] + 2 } else { dp[i][j] = max(dp[i+1][j], dp[i][j-1]) }
		}
	}
	return dp[0][n-1]
}

// O(n) space using rolling row
func lpsOptimized(s string) int {
	n := len(s)
	dp := make([]int, n)
	for i := range dp { dp[i] = 1 }

	for l := 2; l <= n; l++ {
		prev := 0
		for i := 0; i+l-1 < n; i++ {
			j := i + l - 1
			temp := dp[i]
			if s[i] == s[j] {
				dp[i] = prev + 2
				if l == 2 { dp[i] = 2 }
			} else {
				dp[i] = max(dp[i+1], dp[i])
			}
			prev = temp
		}
	}
	return dp[0]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(lpsStandard("bbbab"), lpsOptimized("bbbab"))     // 4 4
	fmt.Println(lpsStandard("racecar"), lpsOptimized("racecar")) // 7 7
}
```

---

## Example 8: House Robber — O(1) Space

```go
package main

import "fmt"

// Array version: O(n) space
func robArray(nums []int) int {
	n := len(nums)
	if n == 1 { return nums[0] }
	dp := make([]int, n)
	dp[0] = nums[0]
	dp[1] = max(nums[0], nums[1])
	for i := 2; i < n; i++ {
		dp[i] = max(dp[i-1], dp[i-2]+nums[i])
	}
	return dp[n-1]
}

// O(1) space: only need two previous values
func robOptimized(nums []int) int {
	prev2, prev1 := 0, 0
	for _, num := range nums {
		curr := max(prev1, prev2+num)
		prev2, prev1 = prev1, curr
	}
	return prev1
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	nums := []int{2, 7, 9, 3, 1}
	fmt.Println(robArray(nums), robOptimized(nums)) // 12 12

	nums2 := []int{1, 2, 3, 1}
	fmt.Println(robArray(nums2), robOptimized(nums2)) // 4 4
}
```

---

## Example 9: Coin Change — In-Place 1D

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i := range dp { dp[i] = amount + 1 }
	dp[0] = 0

	// Single pass per coin — naturally 1D
	for _, coin := range coins {
		for a := coin; a <= amount; a++ {
			if dp[a-coin]+1 < dp[a] {
				dp[a] = dp[a-coin] + 1
			}
		}
	}

	if dp[amount] > amount { return -1 }
	return dp[amount]
}

// Demonstrating why this is already space-optimal
func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36))  // 3
	fmt.Println(coinChange([]int{186, 419, 83, 408}, 6249)) // 20

	// Compare space usage
	fmt.Println("\nSpace analysis:")
	fmt.Printf("  2D table (coins × amount): %d × %d = %d cells\n", 4, 6249, 4*6249)
	fmt.Printf("  1D rolling:                %d cells\n", 6249)
	fmt.Printf("  Savings:                   %.1f%%\n", (1.0-1.0/4.0)*100)
}
```

---

## Example 10: Space Optimization Decision Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Space Optimization Decision Guide ===")
	fmt.Println()

	techniques := []struct{ pattern, before, after, how string }{
		{
			"Linear recurrence\n   (Fibonacci, climbing stairs)",
			"O(n)", "O(1)",
			"Use 2-3 variables instead of array",
		},
		{
			"Grid DP (paths, min cost)\n   Row depends on prev row only",
			"O(m×n)", "O(n)",
			"Single row, update left-to-right",
		},
		{
			"Two-string DP (LCS, edit dist)\n   dp[i] depends on dp[i-1]",
			"O(m×n)", "O(min(m,n))",
			"Two rows (prev + curr), swap",
		},
		{
			"0/1 Knapsack\n   Each item used at most once",
			"O(n×W)", "O(W)",
			"Single row, iterate capacity REVERSE",
		},
		{
			"Unbounded Knapsack\n   Items reusable",
			"O(n×W)", "O(W)",
			"Single row, iterate capacity FORWARD",
		},
		{
			"Interval DP\n   dp[i][j] needs dp[i+1] and dp[i]",
			"O(n²)", "O(n)",
			"Careful rolling with saved prev value",
		},
	}

	for i, t := range techniques {
		fmt.Printf("%d. %-40s\n", i+1, t.pattern)
		fmt.Printf("   Space: %s → %s\n", t.before, t.after)
		fmt.Printf("   How: %s\n\n", t.how)
	}

	fmt.Println("⚠ Trade-offs:")
	fmt.Println("  - Space optimization often prevents reconstruction (backtracking)")
	fmt.Println("  - Need full table to trace which items/choices were made")
	fmt.Println("  - For count-only or value-only answers, optimize aggressively")
	fmt.Println("  - For reconstruction, keep the 2D table or use Hirschberg's algorithm")
}
```

---

## Key Takeaways

1. If dp[i] only depends on dp[i-1], collapse to single row or two variables
2. Reverse iteration in 1D = 0/1 (each item once); forward = unbounded
3. Two-row technique: prev[] and curr[], swap after each outer iteration
4. In-place optimization uses the grid itself — saves space but modifies input
5. Trade-off: space optimization often loses the ability to reconstruct the solution
6. Always check which rows/cells are actually needed before optimizing

> **Phase 15 complete! Next up:** Phase 16 — Sorting →
