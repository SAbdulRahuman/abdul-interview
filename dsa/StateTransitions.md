# Phase 15: Dynamic Programming — State Transitions

## Overview

A **state transition** (recurrence relation) defines how to compute the value of a DP state from previously computed states. Designing the right state and transitions is the core skill of DP.

| Component | Question to Ask |
|-----------|----------------|
| **State** | What information uniquely describes a subproblem? |
| **Transition** | How does this state relate to smaller states? |
| **Base case** | What states can be answered directly? |
| **Answer** | Which state(s) contain the final answer? |

---

## Example 1: 1D Transition — House Robber (LeetCode 198)

```go
package main

import "fmt"

// State: dp[i] = max money robbing houses 0..i
// Transition: dp[i] = max(dp[i-1], dp[i-2] + nums[i])
// Base: dp[0] = nums[0], dp[1] = max(nums[0], nums[1])

func rob(nums []int) int {
	n := len(nums)
	if n == 0 { return 0 }
	if n == 1 { return nums[0] }

	dp := make([]int, n)
	dp[0] = nums[0]
	dp[1] = max(nums[0], nums[1])

	for i := 2; i < n; i++ {
		dp[i] = max(dp[i-1], dp[i-2]+nums[i])
	}
	return dp[n-1]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(rob([]int{2, 7, 9, 3, 1}))   // 12
	fmt.Println(rob([]int{1, 2, 3, 1}))       // 4
}
```

---

## Example 2: 2D Transition — Unique Paths with Obstacles (LeetCode 63)

```go
package main

import "fmt"

// State: dp[i][j] = number of paths to cell (i,j)
// Transition: dp[i][j] = dp[i-1][j] + dp[i][j-1] (if not obstacle)
// Base: dp[0][0] = 1 if no obstacle

func uniquePathsWithObstacles(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	if grid[0][0] == 1 { return 0 }

	dp := make([][]int, m)
	for i := range dp { dp[i] = make([]int, n) }
	dp[0][0] = 1

	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			if grid[i][j] == 1 { dp[i][j] = 0; continue }
			if i > 0 { dp[i][j] += dp[i-1][j] }
			if j > 0 { dp[i][j] += dp[i][j-1] }
		}
	}
	return dp[m-1][n-1]
}

func main() {
	grid := [][]int{
		{0, 0, 0},
		{0, 1, 0},
		{0, 0, 0},
	}
	fmt.Println(uniquePathsWithObstacles(grid)) // 2
}
```

---

## Example 3: Multi-Choice Transition — Decode Ways (LeetCode 91)

```go
package main

import "fmt"

// State: dp[i] = number of ways to decode s[0..i-1]
// Transition: dp[i] += dp[i-1] (if s[i-1] is valid single digit)
//             dp[i] += dp[i-2] (if s[i-2..i-1] is valid two-digit)

func numDecodings(s string) int {
	n := len(s)
	if s[0] == '0' { return 0 }

	dp := make([]int, n+1)
	dp[0], dp[1] = 1, 1

	for i := 2; i <= n; i++ {
		one := s[i-1] - '0'
		two := (s[i-2]-'0')*10 + s[i-1] - '0'

		if one >= 1 { dp[i] += dp[i-1] }
		if two >= 10 && two <= 26 { dp[i] += dp[i-2] }
	}
	return dp[n]
}

func main() {
	tests := []string{"12", "226", "06", "11106"}
	for _, s := range tests {
		fmt.Printf("decode(\"%s\") = %d\n", s, numDecodings(s))
	}
}
```

---

## Example 4: Min/Max Transition — Minimum Path Sum (LeetCode 64)

```go
package main

import "fmt"

// State: dp[i][j] = minimum sum path from (0,0) to (i,j)
// Transition: dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])

func minPathSum(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	dp := make([][]int, m)
	for i := range dp { dp[i] = make([]int, n) }

	dp[0][0] = grid[0][0]
	for j := 1; j < n; j++ { dp[0][j] = dp[0][j-1] + grid[0][j] }
	for i := 1; i < m; i++ { dp[i][0] = dp[i-1][0] + grid[i][0] }

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])
		}
	}
	return dp[m-1][n-1]
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	grid := [][]int{{1,3,1},{1,5,1},{4,2,1}}
	fmt.Println("Min path sum:", minPathSum(grid)) // 7
}
```

---

## Example 5: Multi-State Transition — Buy/Sell Stock with Cooldown (LeetCode 309)

```go
package main

import "fmt"

// State machine DP:
// hold[i] = max profit on day i while holding stock
// sold[i] = max profit on day i after just selling
// rest[i] = max profit on day i doing nothing (no stock, not just sold)
//
// Transitions:
// hold[i] = max(hold[i-1], rest[i-1] - price[i])  // keep or buy
// sold[i] = hold[i-1] + price[i]                   // sell
// rest[i] = max(rest[i-1], sold[i-1])              // wait

func maxProfit(prices []int) int {
	if len(prices) < 2 { return 0 }

	hold, sold, rest := -prices[0], 0, 0

	for i := 1; i < len(prices); i++ {
		prevHold, prevSold, prevRest := hold, sold, rest

		hold = max(prevHold, prevRest-prices[i])
		sold = prevHold + prices[i]
		rest = max(prevRest, prevSold)
	}

	return max(sold, rest)
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(maxProfit([]int{1, 2, 3, 0, 2})) // 3
	fmt.Println(maxProfit([]int{1}))               // 0
}
```

---

## Example 6: String Matching Transition — Wildcard Matching (LeetCode 44)

```go
package main

import "fmt"

// State: dp[i][j] = does s[0..i-1] match p[0..j-1]?
// Transitions:
//   p[j-1] == s[i-1] or '?': dp[i][j] = dp[i-1][j-1]
//   p[j-1] == '*': dp[i][j] = dp[i][j-1] (match empty) || dp[i-1][j] (match one more)
//   else: dp[i][j] = false

func isMatch(s, p string) bool {
	m, n := len(s), len(p)
	dp := make([][]bool, m+1)
	for i := range dp { dp[i] = make([]bool, n+1) }
	dp[0][0] = true

	for j := 1; j <= n; j++ {
		if p[j-1] == '*' { dp[0][j] = dp[0][j-1] }
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if p[j-1] == s[i-1] || p[j-1] == '?' {
				dp[i][j] = dp[i-1][j-1]
			} else if p[j-1] == '*' {
				dp[i][j] = dp[i][j-1] || dp[i-1][j]
			}
		}
	}
	return dp[m][n]
}

func main() {
	tests := [][2]string{
		{"aa", "a"},
		{"aa", "*"},
		{"cb", "?a"},
		{"adceb", "*a*b"},
	}
	for _, t := range tests {
		fmt.Printf("match(\"%s\", \"%s\") = %v\n", t[0], t[1], isMatch(t[0], t[1]))
	}
}
```

---

## Example 7: Interval Transition — Burst Balloons (LeetCode 312)

```go
package main

import "fmt"

// State: dp[i][j] = max coins from bursting balloons i+1..j-1
// Transition: dp[i][j] = max over k in (i,j) of:
//   dp[i][k] + dp[k][j] + nums[i]*nums[k]*nums[j]
// k is the LAST balloon to burst in range (i,j)

func maxCoins(nums []int) int {
	n := len(nums)
	padded := make([]int, n+2)
	padded[0], padded[n+1] = 1, 1
	copy(padded[1:], nums)

	dp := make([][]int, n+2)
	for i := range dp { dp[i] = make([]int, n+2) }

	for length := 2; length <= n+1; length++ {
		for i := 0; i+length <= n+1; i++ {
			j := i + length
			for k := i + 1; k < j; k++ {
				val := dp[i][k] + dp[k][j] + padded[i]*padded[k]*padded[j]
				if val > dp[i][j] { dp[i][j] = val }
			}
		}
	}
	return dp[0][n+1]
}

func main() {
	fmt.Println(maxCoins([]int{3, 1, 5, 8})) // 167
}
```

---

## Example 8: Transition with Multiple Previous States — Maximum Product (LeetCode 152)

```go
package main

import "fmt"

// State: maxP[i] = max product ending at i
//        minP[i] = min product ending at i (needed for negative × negative)
// Transition:
//   maxP[i] = max(nums[i], maxP[i-1]*nums[i], minP[i-1]*nums[i])
//   minP[i] = min(nums[i], maxP[i-1]*nums[i], minP[i-1]*nums[i])

func maxProduct(nums []int) int {
	maxP, minP, result := nums[0], nums[0], nums[0]

	for i := 1; i < len(nums); i++ {
		if nums[i] < 0 { maxP, minP = minP, maxP } // swap when negative

		maxP = max(nums[i], maxP*nums[i])
		minP = min(nums[i], minP*nums[i])

		if maxP > result { result = maxP }
	}
	return result
}

func max(a, b int) int { if a > b { return a }; return b }
func min(a, b int) int { if a < b { return a }; return b }

func main() {
	fmt.Println(maxProduct([]int{2, 3, -2, 4}))  // 6
	fmt.Println(maxProduct([]int{-2, 0, -1}))    // 0
	fmt.Println(maxProduct([]int{-2, 3, -4}))    // 24
}
```

---

## Example 9: Transition Diagram Visualization

```go
package main

import "fmt"

func main() {
	fmt.Println("=== State Transition Diagrams ===")
	fmt.Println()

	fmt.Println("1. House Robber (linear chain):")
	fmt.Println("   dp[i-2] ──→ dp[i] ←── dp[i-1]")
	fmt.Println("   (rob i)      ↑        (skip i)")
	fmt.Println()

	fmt.Println("2. LCS (2D grid):")
	fmt.Println("   dp[i-1][j-1] ──→ dp[i][j] (match)")
	fmt.Println("   dp[i-1][j]   ──→ dp[i][j] (skip in s1)")
	fmt.Println("   dp[i][j-1]   ──→ dp[i][j] (skip in s2)")
	fmt.Println()

	fmt.Println("3. Stock with Cooldown (state machine):")
	fmt.Println("   HOLD ──sell──→ SOLD ──wait──→ REST ──buy──→ HOLD")
	fmt.Println("   HOLD ──hold──→ HOLD")
	fmt.Println("   REST ──rest──→ REST")
	fmt.Println()

	fmt.Println("4. Burst Balloons (interval):")
	fmt.Println("   dp[i][j] ←── dp[i][k] + dp[k][j] + cost(i,k,j)")
	fmt.Println("   for all k in (i,j)")
}
```

---

## Example 10: State Transition Design Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Designing State Transitions ===")
	fmt.Println()

	steps := []struct{ step, question string }{
		{"1. Identify state", "What parameters define a subproblem?"},
		{"2. Define value", "What does dp[state] represent? (count? min? max?)"},
		{"3. Find transitions", "How can I reach this state from smaller states?"},
		{"4. Set base cases", "What states are trivially known?"},
		{"5. Determine order", "What fill order ensures all deps are ready?"},
		{"6. Extract answer", "Where is the final answer in the table?"},
	}

	for _, s := range steps {
		fmt.Printf("%-22s %s\n", s.step, s.question)
	}

	fmt.Println()
	fmt.Println("Common transition patterns:")
	fmt.Printf("%-20s %s\n", "Pattern", "Example")
	fmt.Println("------------------------------------------")
	fmt.Printf("%-20s %s\n", "Linear:  dp[i-1]", "Fibonacci, house robber")
	fmt.Printf("%-20s %s\n", "Grid:    dp[i-1][j]", "Paths, LCS, edit dist")
	fmt.Printf("%-20s %s\n", "Knapsack: dp[i-1][w]", "0/1 knapsack, subset sum")
	fmt.Printf("%-20s %s\n", "Interval: dp[i][k]", "MCM, burst balloons")
	fmt.Printf("%-20s %s\n", "Machine:  state→state", "Stock problems")
}
```

---

## Key Takeaways

1. State transition = recurrence relation — the heart of every DP solution
2. Define what dp[i] or dp[i][j] **means** before writing transitions
3. Every transition must only depend on already-computed states
4. Multi-state transitions (hold/sold/rest) model state machines
5. Interval transitions dp[i][j] from dp[i][k] and dp[k+1][j] handle range problems

> **Next up:** DP Optimization →
