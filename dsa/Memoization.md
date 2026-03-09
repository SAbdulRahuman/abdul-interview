# Phase 15: Dynamic Programming — Memoization

## Overview

**Memoization** is the top-down DP approach: solve problems recursively but cache results to avoid recomputation. It's often the easiest way to add DP to an existing recursive solution.

| Aspect | Detail |
|--------|--------|
| **Approach** | Top-down recursive with caching |
| **When computed** | Lazily — only when needed |
| **Storage** | Map or array indexed by state |
| **Pros** | Easy to implement from recursion; only computes reachable states |
| **Cons** | Recursion overhead; stack overflow risk for deep recursion |

---

## Example 1: Basic Memoization with Map

```go
package main

import "fmt"

func fib(n int, memo map[int]int) int {
	if n <= 1 { return n }
	if v, ok := memo[n]; ok { return v }

	memo[n] = fib(n-1, memo) + fib(n-2, memo)
	return memo[n]
}

func main() {
	memo := map[int]int{}
	for i := 0; i <= 15; i++ {
		fmt.Printf("fib(%2d) = %d\n", i, fib(i, memo))
	}
}
```

---

## Example 2: Memoization with Array (Faster)

```go
package main

import "fmt"

func fib(n int, memo []int) int {
	if n <= 1 { return n }
	if memo[n] != -1 { return memo[n] }

	memo[n] = fib(n-1, memo) + fib(n-2, memo)
	return memo[n]
}

func main() {
	n := 40
	memo := make([]int, n+1)
	for i := range memo { memo[i] = -1 }

	fmt.Printf("fib(%d) = %d\n", n, fib(n, memo))
}
```

---

## Example 3: Coin Change Memoized (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	memo := make([]int, amount+1)
	for i := range memo { memo[i] = -2 } // -2 = not computed

	var solve func(amt int) int
	solve = func(amt int) int {
		if amt == 0 { return 0 }
		if amt < 0 { return -1 }
		if memo[amt] != -2 { return memo[amt] }

		best := -1
		for _, c := range coins {
			sub := solve(amt - c)
			if sub >= 0 {
				if best == -1 || sub+1 < best {
					best = sub + 1
				}
			}
		}

		memo[amt] = best
		return best
	}

	return solve(amount)
}

func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36))  // 3
	fmt.Println(coinChange([]int{2}, 3))                // -1
	fmt.Println(coinChange([]int{1, 3, 4}, 6))          // 2
}
```

---

## Example 4: Grid Minimum Path Sum (LeetCode 64)

```go
package main

import "fmt"

func minPathSum(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	memo := make([][]int, m)
	for i := range memo {
		memo[i] = make([]int, n)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(r, c int) int
	solve = func(r, c int) int {
		if r == m-1 && c == n-1 { return grid[r][c] }
		if r >= m || c >= n { return 1<<31 - 1 }
		if memo[r][c] != -1 { return memo[r][c] }

		right := solve(r, c+1)
		down := solve(r+1, c)

		best := grid[r][c]
		if right < down { best += right } else { best += down }

		memo[r][c] = best
		return best
	}

	return solve(0, 0)
}

func main() {
	grid := [][]int{
		{1, 3, 1},
		{1, 5, 1},
		{4, 2, 1},
	}
	fmt.Println("Min path sum:", minPathSum(grid)) // 7
}
```

---

## Example 5: Longest Common Subsequence — Memoized

```go
package main

import "fmt"

func longestCommonSubsequence(s1, s2 string) int {
	m, n := len(s1), len(s2)
	memo := make([][]int, m)
	for i := range memo {
		memo[i] = make([]int, n)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(i, j int) int
	solve = func(i, j int) int {
		if i == m || j == n { return 0 }
		if memo[i][j] != -1 { return memo[i][j] }

		if s1[i] == s2[j] {
			memo[i][j] = 1 + solve(i+1, j+1)
		} else {
			a := solve(i+1, j)
			b := solve(i, j+1)
			if a > b { memo[i][j] = a } else { memo[i][j] = b }
		}
		return memo[i][j]
	}

	return solve(0, 0)
}

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace"))   // 3
	fmt.Println(longestCommonSubsequence("abc", "def"))     // 0
}
```

---

## Example 6: Word Break — Memoized (LeetCode 139)

```go
package main

import "fmt"

func wordBreak(s string, wordDict []string) bool {
	wordSet := map[string]bool{}
	for _, w := range wordDict { wordSet[w] = true }

	memo := map[int]bool{}
	computed := map[int]bool{}

	var solve func(start int) bool
	solve = func(start int) bool {
		if start == len(s) { return true }
		if computed[start] { return memo[start] }

		computed[start] = true
		for end := start + 1; end <= len(s); end++ {
			if wordSet[s[start:end]] && solve(end) {
				memo[start] = true
				return true
			}
		}
		memo[start] = false
		return false
	}

	return solve(0)
}

func main() {
	fmt.Println(wordBreak("leetcode", []string{"leet", "code"}))       // true
	fmt.Println(wordBreak("applepenapple", []string{"apple", "pen"}))  // true
	fmt.Println(wordBreak("catsandog", []string{"cats","dog","sand","and","cat"})) // false
}
```

---

## Example 7: 2D State Memoization — Knapsack

```go
package main

import "fmt"

func knapsack(weights, values []int, capacity int) int {
	n := len(weights)
	memo := make([][]int, n)
	for i := range memo {
		memo[i] = make([]int, capacity+1)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(i, w int) int
	solve = func(i, w int) int {
		if i == n || w == 0 { return 0 }
		if memo[i][w] != -1 { return memo[i][w] }

		// Skip item i
		best := solve(i+1, w)

		// Take item i
		if weights[i] <= w {
			take := values[i] + solve(i+1, w-weights[i])
			if take > best { best = take }
		}

		memo[i][w] = best
		return best
	}

	return solve(0, capacity)
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	fmt.Println("Max value:", knapsack(weights, values, 8)) // 10
}
```

---

## Example 8: String Key Memoization — Partition

```go
package main

import "fmt"

func minCut(s string) int {
	n := len(s)

	// Precompute palindrome check
	isPalin := make([][]bool, n)
	for i := range isPalin { isPalin[i] = make([]bool, n) }
	for i := n - 1; i >= 0; i-- {
		for j := i; j < n; j++ {
			if s[i] == s[j] && (j-i <= 2 || isPalin[i+1][j-1]) {
				isPalin[i][j] = true
			}
		}
	}

	memo := make([]int, n)
	for i := range memo { memo[i] = -1 }

	var solve func(start int) int
	solve = func(start int) int {
		if start == n { return -1 }
		if memo[start] != -1 { return memo[start] }

		best := n - start // worst case: cut every character
		for end := start; end < n; end++ {
			if isPalin[start][end] {
				cuts := 1 + solve(end+1)
				if cuts < best { best = cuts }
			}
		}

		memo[start] = best
		return best
	}

	return solve(0)
}

func main() {
	fmt.Println(minCut("aab"))  // 1
	fmt.Println(minCut("a"))    // 0
	fmt.Println(minCut("ab"))   // 1
}
```

---

## Example 9: Memoization vs Tabulation — When to Choose

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Memoization vs Tabulation ===")
	fmt.Println()

	rows := []struct{ aspect, memo, tab string }{
		{"Direction", "Top-down (start from problem)", "Bottom-up (start from base cases)"},
		{"Implementation", "Recursive + cache", "Iterative + table"},
		{"States computed", "Only reachable states", "All states in table"},
		{"Stack overflow", "Risk for deep recursion", "No risk (iterative)"},
		{"Ease of writing", "Easy from recursion", "Requires understanding order"},
		{"Space optimization", "Harder", "Easier (rolling array)"},
		{"Constant factor", "Higher (function calls)", "Lower (simple loops)"},
		{"Best when", "Sparse state space", "Dense state space"},
	}

	fmt.Printf("%-20s %-32s %s\n", "Aspect", "Memoization", "Tabulation")
	fmt.Println(string(make([]byte, 85)))
	for _, r := range rows {
		fmt.Printf("%-20s %-32s %s\n", r.aspect, r.memo, r.tab)
	}
}
```

---

## Example 10: Generic Memoize Helper in Go

```go
package main

import "fmt"

// Generic memoization wrapper using closures

type MemoFunc func(n int) int

func memoize(f func(n int, recurse MemoFunc) int) MemoFunc {
	cache := map[int]int{}

	var memoized MemoFunc
	memoized = func(n int) int {
		if v, ok := cache[n]; ok { return v }
		result := f(n, memoized)
		cache[n] = result
		return result
	}
	return memoized
}

func main() {
	// Fibonacci using generic memoize
	fib := memoize(func(n int, recurse MemoFunc) int {
		if n <= 1 { return n }
		return recurse(n-1) + recurse(n-2)
	})

	for i := 0; i <= 15; i++ {
		fmt.Printf("fib(%2d) = %d\n", i, fib(i))
	}

	// Tribonacci using same helper
	trib := memoize(func(n int, recurse MemoFunc) int {
		if n == 0 { return 0 }
		if n <= 2 { return 1 }
		return recurse(n-1) + recurse(n-2) + recurse(n-3)
	})

	fmt.Println()
	for i := 0; i <= 10; i++ {
		fmt.Printf("trib(%2d) = %d\n", i, trib(i))
	}
}
```

---

## Key Takeaways

1. Memoization = recursion + cache — easiest way to add DP to existing recursive solution
2. Use map for sparse states, array for dense/bounded states (array is faster)
3. Only computes reachable states — better than tabulation when state space is sparse
4. Watch for stack overflow with deep recursion (Go default stack grows, but still bounded)
5. Converting from memoization to tabulation: identify the order states are needed, fill iteratively

> **Next up:** Tabulation →
