# Phase 15: Dynamic Programming — Tabulation

## Overview

**Tabulation** is the bottom-up DP approach: fill a table iteratively from base cases to the final answer. No recursion needed — deterministic, cache-friendly, and easy to optimize for space.

| Aspect | Detail |
|--------|--------|
| **Approach** | Bottom-up iterative |
| **Order** | Must solve subproblems in correct dependency order |
| **Storage** | Array/table indexed by state parameters |
| **Pros** | No recursion overhead; easy space optimization |
| **Cons** | Computes all states (even unreachable ones); order must be determined |

---

## Example 1: Fibonacci Tabulated

```go
package main

import "fmt"

func fib(n int) int {
	if n <= 1 { return n }
	dp := make([]int, n+1)
	dp[0], dp[1] = 0, 1

	for i := 2; i <= n; i++ {
		dp[i] = dp[i-1] + dp[i-2]
	}
	return dp[n]
}

func main() {
	for n := 0; n <= 15; n++ {
		fmt.Printf("fib(%2d) = %d\n", n, fib(n))
	}
}
```

---

## Example 2: Climbing Stairs — 1D Table

```go
package main

import "fmt"

func climbStairs(n int) int {
	if n <= 2 { return n }
	dp := make([]int, n+1)
	dp[1], dp[2] = 1, 2

	for i := 3; i <= n; i++ {
		dp[i] = dp[i-1] + dp[i-2]
	}
	return dp[n]
}

func main() {
	for n := 1; n <= 10; n++ {
		fmt.Printf("n=%2d: %d ways\n", n, climbStairs(n))
	}
}
```

---

## Example 3: Coin Change — 1D Table (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ {
		dp[i] = amount + 1 // sentinel for "impossible"
	}

	for i := 1; i <= amount; i++ {
		for _, c := range coins {
			if c <= i && dp[i-c]+1 < dp[i] {
				dp[i] = dp[i-c] + 1
			}
		}
	}

	if dp[amount] > amount { return -1 }
	return dp[amount]
}

func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36)) // 3
	fmt.Println(coinChange([]int{2}, 3))              // -1
}
```

---

## Example 4: Longest Common Subsequence — 2D Table (LeetCode 1143)

```go
package main

import "fmt"

func longestCommonSubsequence(s1, s2 string) int {
	m, n := len(s1), len(s2)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	// Fill table bottom-up
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s1[i-1] == s2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				if dp[i-1][j] > dp[i][j-1] {
					dp[i][j] = dp[i-1][j]
				} else {
					dp[i][j] = dp[i][j-1]
				}
			}
		}
	}

	return dp[m][n]
}

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace")) // 3
	fmt.Println(longestCommonSubsequence("abc", "abc"))   // 3
	fmt.Println(longestCommonSubsequence("abc", "def"))   // 0
}
```

---

## Example 5: Edit Distance — 2D Table (LeetCode 72)

```go
package main

import "fmt"

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = i // delete all chars
	}
	for j := 0; j <= n; j++ {
		dp[0][j] = j // insert all chars
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if word1[i-1] == word2[j-1] {
				dp[i][j] = dp[i-1][j-1]
			} else {
				dp[i][j] = 1 + min3(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
			}
		}
	}
	return dp[m][n]
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

## Example 6: 0/1 Knapsack — 2D Table

```go
package main

import "fmt"

func knapsack(weights, values []int, W int) int {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, W+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= W; w++ {
			dp[i][w] = dp[i-1][w] // skip item
			if weights[i-1] <= w {
				take := dp[i-1][w-weights[i-1]] + values[i-1]
				if take > dp[i][w] { dp[i][w] = take }
			}
		}
	}
	return dp[n][W]
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	fmt.Println("Max value:", knapsack(weights, values, 8)) // 10
}
```

---

## Example 7: Unique Paths — 2D Table (LeetCode 62)

```go
package main

import "fmt"

func uniquePaths(m, n int) int {
	dp := make([][]int, m)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = 1 }
	}

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[m-1][n-1]
}

func main() {
	fmt.Println(uniquePaths(3, 7)) // 28
	fmt.Println(uniquePaths(3, 2)) // 3
}
```

---

## Example 8: Longest Palindromic Subsequence — 2D Table (LeetCode 516)

```go
package main

import "fmt"

func longestPalinSubseq(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = 1
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			if s[i] == s[j] {
				dp[i][j] = dp[i+1][j-1] + 2
			} else {
				if dp[i+1][j] > dp[i][j-1] {
					dp[i][j] = dp[i+1][j]
				} else {
					dp[i][j] = dp[i][j-1]
				}
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(longestPalinSubseq("bbbab"))   // 4 ("bbbb")
	fmt.Println(longestPalinSubseq("cbbd"))    // 2 ("bb")
}
```

---

## Example 9: Tabulation Order Matters

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Tabulation: Filling Order ===")
	fmt.Println()

	orders := []struct{ problem, order, reason string }{
		{"Fibonacci", "i = 2 → n", "dp[i] depends on dp[i-1] and dp[i-2]"},
		{"LCS", "i = 1→m, j = 1→n", "dp[i][j] depends on dp[i-1][j-1], dp[i-1][j], dp[i][j-1]"},
		{"LPS", "length = 2→n", "dp[i][j] depends on dp[i+1][j-1] (inner diagonal)"},
		{"Knapsack", "i = 1→n, w = 0→W", "dp[i][w] depends on dp[i-1][*]"},
		{"MCM", "length = 2→n", "dp[i][j] depends on dp[i][k] and dp[k+1][j] for k < j"},
		{"Edit Distance", "i = 1→m, j = 1→n", "dp[i][j] depends on dp[i-1][*] and dp[i][j-1]"},
	}

	for _, o := range orders {
		fmt.Printf("%-18s fill: %-18s because %s\n", o.problem, o.order, o.reason)
	}

	fmt.Println()
	fmt.Println("Rule: Process states in topological order of dependencies.")
	fmt.Println("When dp[i][j] depends on dp[i-1][*], fill row by row.")
	fmt.Println("When dp[i][j] depends on dp[i+1][*], fill bottom to top or by length.")
}
```

---

## Example 10: Tabulation Building Blocks Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Tabulation Summary ===")
	fmt.Println()

	steps := []struct{ step, description string }{
		{"1. Define state", "What does dp[i] or dp[i][j] represent?"},
		{"2. Base cases", "What are the trivial answers? (dp[0], dp[0][0], etc.)"},
		{"3. Recurrence", "How does dp[i][j] relate to smaller subproblems?"},
		{"4. Fill order", "What order ensures all dependencies are ready?"},
		{"5. Answer", "Where is the final answer? dp[n]? dp[n][m]? max(dp[*])?"},
		{"6. Optimize", "Can we reduce space? (rolling array, two variables)"},
	}

	for _, s := range steps {
		fmt.Printf("%-20s %s\n", s.step, s.description)
	}

	fmt.Println()
	fmt.Println("Common table shapes:")
	fmt.Println("  1D: dp[n+1]         — Fibonacci, climbing stairs, coin change")
	fmt.Println("  2D: dp[m+1][n+1]    — LCS, edit distance, knapsack")
	fmt.Println("  2D: dp[n][n]        — LPS, MCM, interval DP")
	fmt.Println("  3D: dp[n][n][k]     — rare; DP with extra parameter")
}
```

---

## Key Takeaways

1. Tabulation fills a table iteratively from base cases → no recursion, no stack overflow
2. Must determine correct fill order based on state dependencies
3. 1D tables for sequence problems, 2D for two-sequence or interval problems
4. All states are computed — even unreachable ones (unlike memoization)
5. Easier to apply space optimization (rolling array) compared to memoization

> **Next up:** State Transitions →
