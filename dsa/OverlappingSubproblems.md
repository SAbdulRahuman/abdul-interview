# Phase 15: Dynamic Programming — Overlapping Subproblems

## Overview

**Overlapping subproblems** means the same subproblem is solved multiple times during recursion. DP exploits this by storing results (memoization or tabulation) so each subproblem is solved only once.

| Property | Meaning |
|----------|---------|
| **Overlapping** | Same function called with same arguments repeatedly |
| **Without DP** | Exponential time (redundant recomputation) |
| **With DP** | Polynomial time (each state computed once) |
| **Memoization** | Top-down: cache results in map/array |
| **Tabulation** | Bottom-up: fill table iteratively |

---

## Example 1: Fibonacci — Seeing the Overlap

```go
package main

import "fmt"

var calls int

func fibNaive(n int) int {
	calls++
	if n <= 1 { return n }
	return fibNaive(n-1) + fibNaive(n-2)
}

func main() {
	for n := 5; n <= 30; n += 5 {
		calls = 0
		result := fibNaive(n)
		fmt.Printf("fib(%2d) = %-10d  calls = %d\n", n, result, calls)
	}
	// fib(30) makes 2.6M+ calls — exponential!
}
```

---

## Example 2: Fibonacci — Memoized (Top-Down)

```go
package main

import "fmt"

func fibMemo(n int, memo map[int]int) int {
	if n <= 1 { return n }
	if v, ok := memo[n]; ok { return v }

	memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo)
	return memo[n]
}

func main() {
	memo := map[int]int{}
	for n := 0; n <= 10; n++ {
		fmt.Printf("fib(%d) = %d\n", n, fibMemo(n, memo))
	}

	// Even fib(100) is instant now
	fmt.Printf("fib(50) = %d\n", fibMemo(50, memo))
}
```

---

## Example 3: Fibonacci — Tabulated (Bottom-Up)

```go
package main

import "fmt"

func fibTab(n int) int {
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
		fmt.Printf("fib(%2d) = %d\n", n, fibTab(n))
	}
}
```

---

## Example 4: Climbing Stairs (LeetCode 70)

```go
package main

import "fmt"

// Ways to climb n stairs taking 1 or 2 steps at a time
// Overlapping: ways(n) = ways(n-1) + ways(n-2) — same as Fibonacci

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
		fmt.Printf("Stairs %2d: %d ways\n", n, climbStairs(n))
	}
}
```

---

## Example 5: Counting Overlapping Calls

```go
package main

import "fmt"

func main() {
	// Track how many times each subproblem is called in naive Fibonacci
	counts := map[int]int{}

	var fib func(n int) int
	fib = func(n int) int {
		counts[n]++
		if n <= 1 { return n }
		return fib(n-1) + fib(n-2)
	}

	fib(10)

	fmt.Println("Subproblem call counts for fib(10):")
	for n := 0; n <= 10; n++ {
		bar := ""
		for i := 0; i < counts[n] && i < 50; i++ { bar += "█" }
		fmt.Printf("  fib(%2d): %3d times  %s\n", n, counts[n], bar)
	}
}
```

---

## Example 6: Coin Change (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	// dp[i] = minimum coins needed for amount i
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ { dp[i] = amount + 1 }

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
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36)) // 3 (25+10+1)
	fmt.Println(coinChange([]int{2}, 3))              // -1
	fmt.Println(coinChange([]int{1, 3, 4}, 6))        // 2 (3+3)
}
```

---

## Example 7: Unique Paths (LeetCode 62)

```go
package main

import "fmt"

func uniquePaths(m, n int) int {
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

func main() {
	tests := [][2]int{{3,7}, {3,2}, {7,3}, {3,3}}
	for _, t := range tests {
		fmt.Printf("Grid %dx%d: %d paths\n", t[0], t[1], uniquePaths(t[0], t[1]))
	}
}
```

---

## Example 8: House Robber (LeetCode 198)

```go
package main

import "fmt"

func rob(nums []int) int {
	n := len(nums)
	if n == 0 { return 0 }
	if n == 1 { return nums[0] }

	dp := make([]int, n)
	dp[0] = nums[0]
	dp[1] = max(nums[0], nums[1])

	for i := 2; i < n; i++ {
		// Overlapping: dp[i] depends on dp[i-1] and dp[i-2]
		dp[i] = max(dp[i-1], dp[i-2]+nums[i])
	}
	return dp[n-1]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(rob([]int{1, 2, 3, 1}))       // 4
	fmt.Println(rob([]int{2, 7, 9, 3, 1}))    // 12
	fmt.Println(rob([]int{2, 1, 1, 2}))       // 4
}
```

---

## Example 9: Detecting Overlapping Subproblems

```go
package main

import "fmt"

func main() {
	fmt.Println("=== How to Detect Overlapping Subproblems ===")
	fmt.Println()

	checks := []struct{ check, means string }{
		{"Same function called with same args multiple times",
			"Overlap exists — use memoization"},
		{"Recursion tree has repeated nodes",
			"Overlap exists — fib(n-1) and fib(n-2) both call fib(n-3)"},
		{"Number of distinct states << total recursive calls",
			"High overlap — DP will help significantly"},
		{"Each subproblem has unique arguments",
			"No overlap — memoization won't help (e.g., merge sort)"},
		{"State can be described by 1-3 parameters",
			"DP-friendly — can build table of manageable size"},
	}

	for i, c := range checks {
		fmt.Printf("%d. %s\n", i+1, c.check)
		fmt.Printf("   → %s\n\n", c.means)
	}
}
```

---

## Example 10: Overlap Comparison Table

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Overlapping Subproblems Summary ===")
	fmt.Println()

	problems := []struct{ problem, naive, dp, overlap string }{
		{"Fibonacci", "O(2^n)", "O(n)", "fib(k) called many times"},
		{"Climbing Stairs", "O(2^n)", "O(n)", "Same as Fibonacci"},
		{"Coin Change", "O(S^n)", "O(n·S)", "amount k reached many ways"},
		{"House Robber", "O(2^n)", "O(n)", "rob(i) = rob(i-1) or rob(i-2)+val"},
		{"Unique Paths", "O(2^(m+n))", "O(m·n)", "cell (i,j) reached from left and top"},
		{"LCS", "O(2^(m+n))", "O(m·n)", "LCS(i,j) computed repeatedly"},
		{"Edit Distance", "O(3^max)", "O(m·n)", "edit(i,j) called from 3 directions"},
	}

	fmt.Printf("%-16s %-10s %-8s %s\n", "Problem", "Naive", "DP", "Why overlap?")
	fmt.Println("---------------------------------------------------------")
	for _, p := range problems {
		fmt.Printf("%-16s %-10s %-8s %s\n", p.problem, p.naive, p.dp, p.overlap)
	}

	fmt.Println()
	fmt.Println("Key insight: if naive recursion is exponential but there")
	fmt.Println("are only polynomial distinct states → DP makes it polynomial.")
}
```

---

## Key Takeaways

1. Overlapping subproblems = same computation repeated → store results
2. Fibonacci: O(2^n) naive → O(n) with DP because only n distinct states
3. Detect overlap: draw recursion tree, look for repeated nodes
4. Two approaches: memoization (top-down, lazy) or tabulation (bottom-up, iterative)
5. If n distinct states and each takes O(1) to compute → O(n) total

> **Next up:** Optimal Substructure →
