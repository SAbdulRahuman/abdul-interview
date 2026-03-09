# Phase 15: Dynamic Programming — Interval DP

## Overview

**Interval DP** solves problems on contiguous ranges (intervals). The state is dp[i][j] representing the optimal answer for the subarray/substring from index i to j. We try all split points k in [i, j).

| Aspect | Detail |
|--------|--------|
| **State** | dp[i][j] = answer for range [i..j] |
| **Transition** | dp[i][j] = optimize over k: dp[i][k] ⊕ dp[k+1][j] + cost |
| **Fill order** | By increasing interval length |
| **Time** | O(n³) typical |

---

## Example 1: Matrix Chain Multiplication

```go
package main

import (
	"fmt"
	"math"
)

// dims[i] = rows of matrix i, dims[i+1] = cols of matrix i
func matrixChainOrder(dims []int) int {
	n := len(dims) - 1 // number of matrices
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt32 }
		dp[i][i] = 0
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
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
	fmt.Println("Min multiplications:", matrixChainOrder(dims)) // 4500

	dims2 := []int{40, 20, 30, 10, 30}
	fmt.Println("Min multiplications:", matrixChainOrder(dims2)) // 26000
}
```

---

## Example 2: Burst Balloons (LeetCode 312)

```go
package main

import "fmt"

func maxCoins(nums []int) int {
	n := len(nums)
	// Add boundary balloons with value 1
	vals := make([]int, n+2)
	vals[0], vals[n+1] = 1, 1
	copy(vals[1:], nums)

	m := n + 2
	dp := make([][]int, m)
	for i := range dp { dp[i] = make([]int, m) }

	// dp[i][j] = max coins from bursting all balloons between i and j (exclusive)
	for length := 2; length < m; length++ {
		for i := 0; i+length < m; i++ {
			j := i + length
			for k := i + 1; k < j; k++ {
				// k is the LAST balloon to burst in range (i,j)
				coins := dp[i][k] + dp[k][j] + vals[i]*vals[k]*vals[j]
				if coins > dp[i][j] { dp[i][j] = coins }
			}
		}
	}
	return dp[0][n+1]
}

func main() {
	fmt.Println(maxCoins([]int{3, 1, 5, 8})) // 167
	fmt.Println(maxCoins([]int{1, 5}))         // 10
}
```

---

## Example 3: Minimum Cost to Merge Stones (LeetCode 1000)

```go
package main

import (
	"fmt"
	"math"
)

func mergeStones(stones []int, k int) int {
	n := len(stones)
	if (n-1)%(k-1) != 0 { return -1 }

	prefix := make([]int, n+1)
	for i := 0; i < n; i++ { prefix[i+1] = prefix[i] + stones[i] }

	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
	}

	for length := k; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			dp[i][j] = math.MaxInt32

			for mid := i; mid < j; mid += k - 1 {
				val := dp[i][mid] + dp[mid+1][j]
				if val < dp[i][j] { dp[i][j] = val }
			}

			if (j-i)%(k-1) == 0 {
				dp[i][j] += prefix[j+1] - prefix[i]
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(mergeStones([]int{3, 2, 4, 1}, 2)) // 20
	fmt.Println(mergeStones([]int{3, 5, 1, 2, 6}, 3)) // 25
}
```

---

## Example 4: Optimal BST Construction

```go
package main

import (
	"fmt"
	"math"
)

// Given frequencies, build BST minimizing total search cost
func optimalBST(keys, freq []int) int {
	n := len(keys)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = freq[i]
	}

	sumFreq := func(i, j int) int {
		s := 0
		for k := i; k <= j; k++ { s += freq[k] }
		return s
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			dp[i][j] = math.MaxInt32

			for r := i; r <= j; r++ {
				left := 0
				if r > i { left = dp[i][r-1] }
				right := 0
				if r < j { right = dp[r+1][j] }
				cost := left + right + sumFreq(i, j)
				if cost < dp[i][j] { dp[i][j] = cost }
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	keys := []int{10, 12, 20}
	freq := []int{34, 8, 50}
	fmt.Println("Optimal BST cost:", optimalBST(keys, freq)) // 142
}
```

---

## Example 5: Minimum Score Triangulation (LeetCode 1039)

```go
package main

import (
	"fmt"
	"math"
)

func minScoreTriangulation(values []int) int {
	n := len(values)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt32 }
	}

	// Base: adjacent vertices — no triangle possible
	for i := 0; i < n-1; i++ { dp[i][i+1] = 0 }

	for length := 3; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			for k := i + 1; k < j; k++ {
				score := dp[i][k] + dp[k][j] + values[i]*values[k]*values[j]
				if score < dp[i][j] { dp[i][j] = score }
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(minScoreTriangulation([]int{1, 2, 3}))    // 6
	fmt.Println(minScoreTriangulation([]int{3, 7, 4, 5})) // 144
	fmt.Println(minScoreTriangulation([]int{1, 3, 1, 4, 1, 5})) // 13
}
```

---

## Example 6: Palindrome Removal (Min Removals to Empty)

```go
package main

import "fmt"

// Minimum number of palindromic removals to empty the array
func minRemovals(arr []int) int {
	n := len(arr)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = 1 // single element is a palindrome
	}

	for length := 2; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			dp[i][j] = dp[i+1][j] + 1 // worst: remove arr[i] alone

			for k := i + 1; k <= j; k++ {
				if arr[i] == arr[k] {
					left := 0
					if i+1 <= k-1 { left = dp[i+1][k-1] }
					right := 0
					if k+1 <= j { right = dp[k+1][j] }

					val := max(1, left) + right
					if val < dp[i][j] { dp[i][j] = val }
				}
			}
		}
	}
	return dp[0][n-1]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(minRemovals([]int{1, 3, 4, 1, 5})) // 3
	fmt.Println(minRemovals([]int{1, 2, 1}))         // 1
}
```

---

## Example 7: Boolean Parenthesization

```go
package main

import "fmt"

// Count ways to parenthesize boolean expression to get true
func countParenthesizations(symbols string, operators string) int {
	n := len(symbols)
	trueDP := make([][]int, n)
	falseDP := make([][]int, n)
	for i := range trueDP {
		trueDP[i] = make([]int, n)
		falseDP[i] = make([]int, n)
		if symbols[i] == 'T' { trueDP[i][i] = 1 } else { falseDP[i][i] = 1 }
	}

	for length := 2; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			for k := i; k < j; k++ {
				lt, lf := trueDP[i][k], falseDP[i][k]
				rt, rf := trueDP[k+1][j], falseDP[k+1][j]

				switch operators[k] {
				case '&':
					trueDP[i][j] += lt * rt
					falseDP[i][j] += lt*rf + lf*rt + lf*rf
				case '|':
					trueDP[i][j] += lt*rt + lt*rf + lf*rt
					falseDP[i][j] += lf * rf
				case '^':
					trueDP[i][j] += lt*rf + lf*rt
					falseDP[i][j] += lt*rt + lf*rf
				}
			}
		}
	}
	return trueDP[0][n-1]
}

func main() {
	fmt.Println(countParenthesizations("TFT", "&|"))   // 1
	fmt.Println(countParenthesizations("TTFT", "|&^")) // 4
}
```

---

## Example 8: Minimum Cost Tree From Leaf Values (LeetCode 1130)

```go
package main

import (
	"fmt"
	"math"
)

func mctFromLeafValues(arr []int) int {
	n := len(arr)
	// maxVal[i][j] = max leaf value in arr[i..j]
	maxVal := make([][]int, n)
	for i := range maxVal {
		maxVal[i] = make([]int, n)
		maxVal[i][i] = arr[i]
		for j := i + 1; j < n; j++ {
			maxVal[i][j] = maxVal[i][j-1]
			if arr[j] > maxVal[i][j] { maxVal[i][j] = arr[j] }
		}
	}

	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		// dp[i][i] = 0 (single leaf, no internal node cost)
	}

	for length := 2; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			dp[i][j] = math.MaxInt32
			for k := i; k < j; k++ {
				cost := dp[i][k] + dp[k+1][j] + maxVal[i][k]*maxVal[k+1][j]
				if cost < dp[i][j] { dp[i][j] = cost }
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(mctFromLeafValues([]int{6, 2, 4}))    // 32
	fmt.Println(mctFromLeafValues([]int{4, 11}))       // 44
}
```

---

## Example 9: Strange Printer (LeetCode 664)

```go
package main

import "fmt"

func strangePrinter(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = 1
	}

	for length := 2; length <= n; length++ {
		for i := 0; i+length-1 < n; i++ {
			j := i + length - 1
			dp[i][j] = dp[i][j-1] + 1 // print s[j] separately

			for k := i; k < j; k++ {
				if s[k] == s[j] {
					val := dp[i][k] + dp[k+1][j-1]
					if k+1 > j-1 { val = dp[i][k] }
					if val < dp[i][j] { dp[i][j] = val }
				}
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(strangePrinter("aaabbb"))  // 2
	fmt.Println(strangePrinter("aba"))     // 2
	fmt.Println(strangePrinter("abcabc"))  // 5
}
```

---

## Example 10: Interval DP Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Interval DP Patterns ===")
	fmt.Println()

	patterns := []struct{ problem, state, splitLogic string }{
		{"Matrix Chain Mult", "dp[i][j] = min cost to multiply M_i..M_j",
			"Split at k: dp[i][k] + dp[k+1][j] + dims cost"},
		{"Burst Balloons", "dp[i][j] = max coins in (i,j) exclusive",
			"k = LAST balloon burst in range"},
		{"Optimal BST", "dp[i][j] = min search cost for keys i..j",
			"r = root, cost = left + right + sum(freq)"},
		{"Triangulation", "dp[i][j] = min score to triangulate polygon i..j",
			"k = third vertex of triangle with edge (i,j)"},
		{"Boolean Parens", "true[i][j], false[i][j]",
			"Split at operator k, combine with &, |, ^"},
		{"Palindrome Removal", "dp[i][j] = min palindromic removals",
			"Match arr[i] with arr[k] if equal"},
		{"Strange Printer", "dp[i][j] = min turns to print s[i..j]",
			"Merge if s[k] == s[j]"},
	}

	for i, p := range patterns {
		fmt.Printf("%d. %s\n", i+1, p.problem)
		fmt.Printf("   State: %s\n", p.state)
		fmt.Printf("   Split: %s\n\n", p.splitLogic)
	}

	fmt.Println("Template:")
	fmt.Println("  for length := 2; length <= n; length++ {")
	fmt.Println("    for i := 0; i+length-1 < n; i++ {")
	fmt.Println("      j := i + length - 1")
	fmt.Println("      for k := i; k < j; k++ {")
	fmt.Println("        dp[i][j] = optimize(dp[i][k], dp[k+1][j], cost)")
	fmt.Println("      }")
	fmt.Println("    }")
	fmt.Println("  }")
}
```

---

## Key Takeaways

1. Interval DP state: dp[i][j] for range [i..j], filled by increasing length
2. Split at every k in [i..j): combine left and right subproblems + merge cost
3. MCM is the archetype — burst balloons reverses the thinking (last instead of first)
4. Time is O(n³) — acceptable for n ≤ 500
5. Pre-compute range queries (max, sum) for efficient transitions

> **Next up:** State Machine DP →
