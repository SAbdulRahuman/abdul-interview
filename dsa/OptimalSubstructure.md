# Phase 15: Dynamic Programming — Optimal Substructure

## Overview

A problem has **optimal substructure** if the optimal solution contains optimal solutions to its subproblems. This is the second key property (along with overlapping subproblems) that makes DP applicable.

| Concept | Meaning |
|---------|---------|
| **Optimal substructure** | Best solution = best sub-solutions combined |
| **Why it matters** | Guarantees we can build optimal answer from smaller optimal answers |
| **How to prove** | Show that using a non-optimal sub-solution would contradict optimality |
| **Counterexample** | Longest simple path: optimal sub-paths may not compose |

---

## Example 1: Shortest Path Has Optimal Substructure

```go
package main

import (
	"fmt"
	"math"
)

// Shortest path from 0 to n-1 in a DAG
// Optimal substructure: shortest(0→n) = min(shortest(0→k) + w(k,n))

func shortestPathDAG(adj [][][2]int, n int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[0] = 0

	// Process in topological order (0, 1, 2, ..., n-1 for a simple DAG)
	for u := 0; u < n; u++ {
		if dist[u] == math.MaxInt32 { continue }
		for _, edge := range adj[u] {
			v, w := edge[0], edge[1]
			if dist[u]+w < dist[v] {
				dist[v] = dist[u] + w // Optimal substructure:
				// dist[v] builds on optimal dist[u]
			}
		}
	}
	return dist
}

func main() {
	// DAG: 0→1(2), 0→2(4), 1→2(1), 1→3(7), 2→3(3)
	adj := [][][2]int{
		{{1, 2}, {2, 4}},
		{{2, 1}, {3, 7}},
		{{3, 3}},
		{},
	}

	dist := shortestPathDAG(adj, 4)
	for i, d := range dist {
		fmt.Printf("Shortest path to %d: %d\n", i, d)
	}
	// 0→1→2→3 = 2+1+3 = 6 (optimal sub-paths: 0→1=2, 0→2=3)
}
```

---

## Example 2: Longest Increasing Subsequence (LeetCode 300)

```go
package main

import "fmt"

// LIS has optimal substructure:
// LIS ending at i = 1 + max(LIS ending at j) for all j < i where nums[j] < nums[i]

func lengthOfLIS(nums []int) int {
	n := len(nums)
	dp := make([]int, n)
	for i := range dp { dp[i] = 1 }

	best := 1
	for i := 1; i < n; i++ {
		for j := 0; j < i; j++ {
			if nums[j] < nums[i] && dp[j]+1 > dp[i] {
				dp[i] = dp[j] + 1
			}
		}
		if dp[i] > best { best = dp[i] }
	}
	return best
}

func main() {
	tests := [][]int{
		{10, 9, 2, 5, 3, 7, 101, 18},
		{0, 1, 0, 3, 2, 3},
		{7, 7, 7, 7, 7, 7, 7},
	}

	for _, nums := range tests {
		fmt.Printf("LIS of %v = %d\n", nums, lengthOfLIS(nums))
	}
}
```

---

## Example 3: Longest Common Subsequence (LeetCode 1143)

```go
package main

import "fmt"

// LCS has optimal substructure:
// If s1[i]==s2[j]: LCS(i,j) = 1 + LCS(i-1,j-1)
// Else: LCS(i,j) = max(LCS(i-1,j), LCS(i,j-1))
// Optimal sub-solutions compose into optimal solution

func longestCommonSubsequence(text1, text2 string) int {
	m, n := len(text1), len(text2)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if text1[i-1] == text2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[m][n]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	tests := [][2]string{
		{"abcde", "ace"},
		{"abc", "abc"},
		{"abc", "def"},
	}

	for _, t := range tests {
		fmt.Printf("LCS(\"%s\", \"%s\") = %d\n",
			t[0], t[1], longestCommonSubsequence(t[0], t[1]))
	}
}
```

---

## Example 4: Edit Distance (LeetCode 72)

```go
package main

import "fmt"

// Edit distance has optimal substructure:
// edit(i,j) = min of:
//   edit(i-1,j) + 1     (delete from s1)
//   edit(i,j-1) + 1     (insert into s1)
//   edit(i-1,j-1) + cost (replace: 0 if match, 1 if not)

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = i
	}
	for j := 0; j <= n; j++ { dp[0][j] = j }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			cost := 1
			if word1[i-1] == word2[j-1] { cost = 0 }

			dp[i][j] = min3(
				dp[i-1][j]+1,     // delete
				dp[i][j-1]+1,     // insert
				dp[i-1][j-1]+cost, // replace
			)
		}
	}
	return dp[m][n]
}

func min3(a, b, c int) int {
	if a < b { if a < c { return a }; return c }
	if b < c { return b }; return c
}

func main() {
	tests := [][2]string{
		{"horse", "ros"},
		{"intention", "execution"},
		{"", "abc"},
	}

	for _, t := range tests {
		fmt.Printf("edit(\"%s\", \"%s\") = %d\n",
			t[0], t[1], minDistance(t[0], t[1]))
	}
}
```

---

## Example 5: Rod Cutting — Classic Optimal Substructure

```go
package main

import "fmt"

// Rod cutting: given rod of length n and prices for each length,
// find maximum revenue from cutting
// Optimal substructure: revenue(n) = max(price[i] + revenue(n-i))

func cutRod(prices []int) int {
	n := len(prices)
	dp := make([]int, n+1)

	for length := 1; length <= n; length++ {
		for cut := 1; cut <= length; cut++ {
			val := prices[cut-1] + dp[length-cut]
			if val > dp[length] { dp[length] = val }
		}
	}
	return dp[n]
}

func main() {
	prices := []int{1, 5, 8, 9, 10, 17, 17, 20}
	fmt.Println("Max revenue for rod lengths:")
	for i := 1; i <= len(prices); i++ {
		fmt.Printf("  Length %d: %d\n", i, cutRod(prices[:i]))
	}
}
```

---

## Example 6: Matrix Chain Multiplication

```go
package main

import (
	"fmt"
	"math"
)

// Optimal substructure: MCM(i,j) = min over k of:
//   MCM(i,k) + MCM(k+1,j) + dims[i-1]*dims[k]*dims[j]

func matrixChainOrder(dims []int) int {
	n := len(dims) - 1
	dp := make([][]int, n)
	for i := range dp { dp[i] = make([]int, n) }

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			dp[i][j] = math.MaxInt32

			for k := i; k < j; k++ {
				cost := dp[i][k] + dp[k+1][j] + dims[i]*dims[k+1]*dims[j+1]
				if cost < dp[i][j] { dp[i][j] = cost }
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	dims := []int{10, 30, 5, 60}
	fmt.Println("Minimum multiplications:", matrixChainOrder(dims)) // 4500

	dims2 := []int{40, 20, 30, 10, 30}
	fmt.Println("Minimum multiplications:", matrixChainOrder(dims2)) // 26000
}
```

---

## Example 7: Problem WITHOUT Optimal Substructure

```go
package main

import "fmt"

// Longest Simple Path does NOT have optimal substructure!
// The longest path from A→C via B doesn't mean
// using the longest A→B and longest B→C paths
// (they might share vertices → not simple)

func main() {
	fmt.Println("=== Longest Simple Path: NO Optimal Substructure ===")
	fmt.Println()
	fmt.Println("Graph: A - B - C - D, with edge A-C")
	fmt.Println()
	fmt.Println("Longest simple path A → D:")
	fmt.Println("  A → C → B → ... but B doesn't connect to D directly")
	fmt.Println("  A → B → C → D = length 3 (optimal)")
	fmt.Println()
	fmt.Println("If we split at B:")
	fmt.Println("  Longest A→B = A→C→B (length 2)")
	fmt.Println("  Longest B→D = B→C→D (length 2)")
	fmt.Println("  Combined: A→C→B→C→D — visits C twice! NOT SIMPLE")
	fmt.Println()
	fmt.Println("This is why longest simple path is NP-hard.")
	fmt.Println("Optimal sub-paths may conflict (share vertices).")
}
```

---

## Example 8: 0/1 Knapsack

```go
package main

import "fmt"

// 0/1 Knapsack has optimal substructure:
// opt(i, w) = max(opt(i-1, w), opt(i-1, w-wt[i]) + val[i])

func knapsack(weights, values []int, capacity int) int {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, capacity+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= capacity; w++ {
			dp[i][w] = dp[i-1][w] // don't take item i
			if weights[i-1] <= w {
				take := dp[i-1][w-weights[i-1]] + values[i-1]
				if take > dp[i][w] { dp[i][w] = take }
			}
		}
	}
	return dp[n][capacity]
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	capacity := 8

	fmt.Println("Max value:", knapsack(weights, values, capacity)) // 10
}
```

---

## Example 9: Proving Optimal Substructure (Cut-and-Paste)

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Cut-and-Paste Proof Technique ===")
	fmt.Println()
	fmt.Println("To prove optimal substructure:")
	fmt.Println()
	fmt.Println("1. Assume we have an optimal solution S* to the problem")
	fmt.Println("2. Identify the subproblem within S*")
	fmt.Println("3. Suppose the sub-solution is NOT optimal")
	fmt.Println("4. 'Cut' the non-optimal sub-solution out")
	fmt.Println("5. 'Paste' in the truly optimal sub-solution")
	fmt.Println("6. Show the new solution is strictly better")
	fmt.Println("7. This contradicts S* being optimal → contradiction!")
	fmt.Println("8. Therefore, sub-solutions must be optimal ✓")
	fmt.Println()
	fmt.Println("Example with shortest path:")
	fmt.Println("  Optimal path: u → ... → v → ... → w")
	fmt.Println("  Subpath u → v must be shortest")
	fmt.Println("  If not, replace with shorter u → v path")
	fmt.Println("  Total path becomes shorter → contradicts optimality")
}
```

---

## Example 10: Optimal Substructure Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Optimal Substructure Summary ===")
	fmt.Println()

	has := []struct{ problem, recurrence string }{
		{"Shortest Path", "dist(v) = min(dist(u) + w(u,v))"},
		{"LIS", "LIS(i) = 1 + max(LIS(j) : j<i, a[j]<a[i])"},
		{"LCS", "LCS(i,j) = 1+LCS(i-1,j-1) or max(...)"},
		{"Edit Distance", "ed(i,j) = min(ed(i-1,j), ed(i,j-1), ed(i-1,j-1))+cost"},
		{"Rod Cutting", "rev(n) = max(price[i] + rev(n-i))"},
		{"Knapsack", "opt(i,w) = max(skip, take)"},
		{"MCM", "MCM(i,j) = min_k(MCM(i,k) + MCM(k+1,j) + cost)"},
	}

	doesNot := []struct{ problem, reason string }{
		{"Longest simple path", "Sub-paths may share vertices"},
		{"Cheapest simple cycle", "Sub-cycles may overlap"},
		{"Unweighted longest path", "NP-hard, no polynomial decomposition"},
	}

	fmt.Println("Problems WITH optimal substructure:")
	for _, p := range has {
		fmt.Printf("  %-18s %s\n", p.problem, p.recurrence)
	}

	fmt.Println("\nProblems WITHOUT optimal substructure:")
	for _, p := range doesNot {
		fmt.Printf("  %-25s %s\n", p.problem, p.reason)
	}
}
```

---

## Key Takeaways

1. Optimal substructure: the optimal solution to the whole contains optimal solutions to subproblems
2. Prove it via cut-and-paste: replacing a sub-solution with a better one would improve the whole
3. Both overlapping subproblems AND optimal substructure must hold for DP to work
4. Longest SIMPLE path does NOT have optimal substructure — this is why it's NP-hard
5. The recurrence relation directly encodes the optimal substructure

> **Next up:** Memoization →
