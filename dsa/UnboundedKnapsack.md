# Phase 15: Dynamic Programming — Unbounded Knapsack

## Overview

The **Unbounded Knapsack** problem allows each item to be used **unlimited** times. This single change flips the inner loop direction from reverse to forward.

| Aspect | 0/1 Knapsack | Unbounded Knapsack |
|--------|-------------|-------------------|
| **Item usage** | At most once | Unlimited |
| **1D loop** | w = W → wt[i] (reverse) | w = wt[i] → W (forward) |
| **Transition** | dp[w] = max(dp[w], dp[w-wt]+val) | Same formula, forward loop |
| **Time** | O(n·W) | O(n·W) |

---

## Example 1: Classic Unbounded Knapsack

```go
package main

import "fmt"

func unboundedKnapsack(weights, values []int, W int) int {
	dp := make([]int, W+1)

	for i := 0; i < len(weights); i++ {
		for w := weights[i]; w <= W; w++ { // forward!
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[W]
}

func main() {
	weights := []int{2, 3, 5}
	values := []int{3, 4, 7}
	fmt.Println("Max value:", unboundedKnapsack(weights, values, 10)) // 15
}
```

---

## Example 2: Coin Change — Minimum Coins (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i := range dp { dp[i] = amount + 1 }
	dp[0] = 0

	for _, coin := range coins {
		for w := coin; w <= amount; w++ {
			if dp[w-coin]+1 < dp[w] {
				dp[w] = dp[w-coin] + 1
			}
		}
	}

	if dp[amount] > amount { return -1 }
	return dp[amount]
}

func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36)) // 3 (25+10+1)
	fmt.Println(coinChange([]int{2}, 3))              // -1
}
```

---

## Example 3: Coin Change II — Count Ways (LeetCode 518)

```go
package main

import "fmt"

func change(amount int, coins []int) int {
	dp := make([]int, amount+1)
	dp[0] = 1

	for _, coin := range coins {
		for w := coin; w <= amount; w++ {
			dp[w] += dp[w-coin]
		}
	}
	return dp[amount]
}

func main() {
	fmt.Println(change(5, []int{1, 2, 5}))    // 4
	fmt.Println(change(10, []int{2, 5, 3, 6})) // 5
}
```

---

## Example 4: Rod Cutting

```go
package main

import "fmt"

// Given a rod of length n and prices for each length, find max profit
func rodCutting(prices []int, n int) int {
	dp := make([]int, n+1)

	for length := 1; length <= len(prices); length++ {
		for w := length; w <= n; w++ {
			take := dp[w-length] + prices[length-1]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[n]
}

func main() {
	prices := []int{1, 5, 8, 9, 10, 17, 17, 20}
	fmt.Println("Rod length 8, max profit:", rodCutting(prices, 8)) // 22
	fmt.Println("Rod length 4, max profit:", rodCutting(prices, 4)) // 10
}
```

---

## Example 5: Minimum Cost to Fill Weight (Exact)

```go
package main

import "fmt"

// Fill exactly W weight at minimum cost
func minCostFill(weights, costs []int, W int) int {
	const INF = 1<<31 - 1
	dp := make([]int, W+1)
	for i := range dp { dp[i] = INF }
	dp[0] = 0

	for i := 0; i < len(weights); i++ {
		for w := weights[i]; w <= W; w++ {
			if dp[w-weights[i]] != INF {
				cost := dp[w-weights[i]] + costs[i]
				if cost < dp[w] { dp[w] = cost }
			}
		}
	}

	if dp[W] == INF { return -1 }
	return dp[W]
}

func main() {
	weights := []int{1, 3, 5}
	costs := []int{2, 6, 7}
	fmt.Println("Min cost for weight 7:", minCostFill(weights, costs, 7)) // 13
}
```

---

## Example 6: Maximum Ribbon Cut

```go
package main

import "fmt"

// Cut ribbon into pieces of given lengths to maximize number of pieces
func maxRibbonCut(lengths []int, n int) int {
	dp := make([]int, n+1)
	for i := range dp { dp[i] = -1 }
	dp[0] = 0

	for _, l := range lengths {
		for w := l; w <= n; w++ {
			if dp[w-l] != -1 {
				pieces := dp[w-l] + 1
				if pieces > dp[w] { dp[w] = pieces }
			}
		}
	}

	if dp[n] == -1 { return 0 }
	return dp[n]
}

func main() {
	fmt.Println(maxRibbonCut([]int{2, 3, 5}, 5))  // 2
	fmt.Println(maxRibbonCut([]int{3, 5, 7}, 13)) // 3
}
```

---

## Example 7: Word Break (LeetCode 139)

```go
package main

import "fmt"

// Can string s be segmented into dictionary words? (unbounded usage)
func wordBreak(s string, wordDict []string) bool {
	words := make(map[string]bool)
	for _, w := range wordDict { words[w] = true }

	n := len(s)
	dp := make([]bool, n+1)
	dp[0] = true

	for i := 1; i <= n; i++ {
		for j := 0; j < i; j++ {
			if dp[j] && words[s[j:i]] {
				dp[i] = true
				break
			}
		}
	}
	return dp[n]
}

func main() {
	fmt.Println(wordBreak("leetcode", []string{"leet", "code"}))       // true
	fmt.Println(wordBreak("applepenapple", []string{"apple", "pen"}))  // true
	fmt.Println(wordBreak("catsandog", []string{"cats", "dog", "sand", "and", "cat"})) // false
}
```

---

## Example 8: Perfect Squares (LeetCode 279)

```go
package main

import "fmt"

// Minimum perfect squares that sum to n
func numSquares(n int) int {
	dp := make([]int, n+1)
	for i := range dp { dp[i] = n + 1 }
	dp[0] = 0

	for sq := 1; sq*sq <= n; sq++ {
		v := sq * sq
		for w := v; w <= n; w++ {
			if dp[w-v]+1 < dp[w] {
				dp[w] = dp[w-v] + 1
			}
		}
	}
	return dp[n]
}

func main() {
	fmt.Println(numSquares(12))  // 3 (4+4+4)
	fmt.Println(numSquares(13))  // 2 (4+9)
	fmt.Println(numSquares(100)) // 1 (100)
}
```

---

## Example 9: Unbounded Knapsack with Item Tracking

```go
package main

import "fmt"

func unboundedWithItems(weights, values []int, W int) (int, map[int]int) {
	dp := make([]int, W+1)
	choice := make([]int, W+1) // which item was last added at capacity w
	for i := range choice { choice[i] = -1 }

	for i := 0; i < len(weights); i++ {
		for w := weights[i]; w <= W; w++ {
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] {
				dp[w] = take
				choice[w] = i
			}
		}
	}

	// Reconstruct
	items := make(map[int]int) // item index → count
	w := W
	for w > 0 && choice[w] != -1 {
		items[choice[w]]++
		w -= weights[choice[w]]
	}

	return dp[W], items
}

func main() {
	weights := []int{2, 3, 5}
	values := []int{3, 4, 7}

	maxVal, items := unboundedWithItems(weights, values, 10)
	fmt.Println("Max value:", maxVal)
	for idx, count := range items {
		fmt.Printf("  Item %d (w=%d, v=%d) × %d\n", idx, weights[idx], values[idx], count)
	}
}
```

---

## Example 10: 0/1 vs Unbounded Comparison

```go
package main

import "fmt"

func knapsack01(weights, values []int, W int) int {
	dp := make([]int, W+1)
	for i := 0; i < len(weights); i++ {
		for w := W; w >= weights[i]; w-- { // REVERSE
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[W]
}

func knapsackUnbounded(weights, values []int, W int) int {
	dp := make([]int, W+1)
	for i := 0; i < len(weights); i++ {
		for w := weights[i]; w <= W; w++ { // FORWARD
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[W]
}

func main() {
	weights := []int{2, 3, 5}
	values := []int{3, 4, 7}

	fmt.Println("=== 0/1 vs Unbounded Knapsack ===")
	for W := 1; W <= 15; W++ {
		v01 := knapsack01(weights, values, W)
		vUB := knapsackUnbounded(weights, values, W)
		marker := ""
		if vUB > v01 { marker = " *" }
		fmt.Printf("W=%2d  0/1=%2d  Unbounded=%2d%s\n", W, v01, vUB, marker)
	}

	fmt.Println()
	fmt.Println("Key difference: LOOP DIRECTION")
	fmt.Println("  0/1:       for w := W; w >= wt; w--  (reverse)")
	fmt.Println("  Unbounded: for w := wt; w <= W; w++  (forward)")
}
```

---

## Key Takeaways

1. Unbounded knapsack = forward iteration — allows reusing same item
2. Coin change (min coins, count ways), rod cutting, word break are all unbounded variants
3. The only code difference from 0/1 is loop direction: forward vs reverse
4. For exact fill problems, initialize dp with INF (minimization) or -1 (maximization)
5. Reconstruction: track last-chosen item at each capacity, trace back

> **Next up:** DP on Strings →
