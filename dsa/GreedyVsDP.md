# Phase 17: Greedy Algorithms — Greedy vs DP

## Overview

Both **Greedy** and **Dynamic Programming** solve optimization problems, but they differ fundamentally in approach. Understanding when each applies is crucial for choosing the right algorithm.

| Aspect | Greedy | DP |
|--------|--------|-----|
| **Choices** | One locally optimal choice | Explore all choices |
| **Subproblems** | Never revisit | Overlapping, memoized |
| **Proof** | Greedy choice property needed | Optimal substructure suffices |
| **Time** | Usually O(n log n) or O(n) | Usually O(n²) or O(nW) |

---

## Example 1: Coin Change — Greedy vs DP

```go
package main

import (
	"fmt"
	"math"
)

// Greedy — works for some coin systems
func coinGreedy(coins []int, amount int) int {
	count := 0
	for _, c := range coins {
		count += amount / c
		amount %= c
	}
	if amount != 0 { return -1 }
	return count
}

// DP — always correct
func coinDP(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i := range dp { dp[i] = math.MaxInt32 }
	dp[0] = 0

	for i := 1; i <= amount; i++ {
		for _, c := range coins {
			if c <= i && dp[i-c]+1 < dp[i] {
				dp[i] = dp[i-c] + 1
			}
		}
	}
	if dp[amount] == math.MaxInt32 { return -1 }
	return dp[amount]
}

func main() {
	// Case 1: Greedy works (canonical coins)
	coins1 := []int{25, 10, 5, 1}
	fmt.Printf("US coins, 41¢: Greedy=%d, DP=%d\n",
		coinGreedy(coins1, 41), coinDP(coins1, 41))

	// Case 2: Greedy fails
	coins2 := []int{6, 4, 1}
	fmt.Printf("Custom coins, 8: Greedy=%d, DP=%d\n",
		coinGreedy(coins2, 8), coinDP(coins2, 8))
	// Greedy: 6+1+1=3 coins, DP: 4+4=2 coins
}
```

---

## Example 2: Knapsack — Greedy vs DP

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct{ W, V int }

func knapsackGreedy(items []Item, cap int) int {
	sort.Slice(items, func(i, j int) bool {
		return float64(items[i].V)/float64(items[i].W) >
			float64(items[j].V)/float64(items[j].W)
	})
	total, rem := 0, cap
	for _, it := range items {
		if it.W <= rem {
			total += it.V
			rem -= it.W
		}
	}
	return total
}

func knapsackDP(items []Item, cap int) int {
	dp := make([]int, cap+1)
	for _, it := range items {
		for w := cap; w >= it.W; w-- {
			if dp[w-it.W]+it.V > dp[w] {
				dp[w] = dp[w-it.W] + it.V
			}
		}
	}
	return dp[cap]
}

func main() {
	items := []Item{{10, 60}, {20, 100}, {30, 120}}
	fmt.Printf("0/1 Knapsack (cap=50): Greedy=%d, DP=%d\n",
		knapsackGreedy(items, 50), knapsackDP(items, 50))

	// Greedy may miss optimal: ratio-based picks aren't always best for 0/1
	items2 := []Item{{5, 10}, {4, 40}, {6, 30}, {3, 50}}
	fmt.Printf("0/1 Knapsack (cap=10): Greedy=%d, DP=%d\n",
		knapsackGreedy(items2, 10), knapsackDP(items2, 10))
}
```

---

## Example 3: Activity Selection — Greedy Sufficient

```go
package main

import (
	"fmt"
	"sort"
)

func activitySelectionGreedy(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})
	count := 1
	end := intervals[0][1]
	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] >= end {
			count++
			end = intervals[i][1]
		}
	}
	return count
}

func activitySelectionDP(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})
	n := len(intervals)
	dp := make([]int, n)
	for i := range dp { dp[i] = 1 }

	for i := 1; i < n; i++ {
		dp[i] = dp[i-1]
		for j := i - 1; j >= 0; j-- {
			if intervals[j][1] <= intervals[i][0] {
				if dp[j]+1 > dp[i] { dp[i] = dp[j] + 1 }
				break
			}
		}
	}
	return dp[n-1]
}

func main() {
	intervals := [][]int{{1, 3}, {2, 5}, {3, 9}, {6, 8}}
	fmt.Println("Greedy:", activitySelectionGreedy(intervals)) // same
	fmt.Println("DP:", activitySelectionDP(intervals))         // same
	fmt.Println("Greedy is sufficient — and O(n log n) vs DP's O(n²)")
}
```

---

## Example 4: Longest Increasing Subsequence — DP Required

```go
package main

import "fmt"

// Greedy approach would fail for LIS
func lisGreedy(nums []int) int {
	if len(nums) == 0 { return 0 }
	count := 1
	prev := nums[0]
	for i := 1; i < len(nums); i++ {
		if nums[i] > prev {
			count++
			prev = nums[i]
		}
	}
	return count // WRONG — this is greedy, not optimal
}

func lisDP(nums []int) int {
	n := len(nums)
	if n == 0 { return 0 }
	dp := make([]int, n)
	for i := range dp { dp[i] = 1 }

	maxLen := 1
	for i := 1; i < n; i++ {
		for j := 0; j < i; j++ {
			if nums[j] < nums[i] && dp[j]+1 > dp[i] {
				dp[i] = dp[j] + 1
			}
		}
		if dp[i] > maxLen { maxLen = dp[i] }
	}
	return maxLen
}

func main() {
	nums := []int{10, 9, 2, 5, 3, 7, 101, 18}
	fmt.Println("Greedy (wrong):", lisGreedy(nums)) // may be wrong
	fmt.Println("DP (correct):", lisDP(nums))        // 4: [2,3,7,101] or [2,5,7,101]
}
```

---

## Example 5: Huffman vs Optimal BST

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Same Structure, Different Solutions ===")
	fmt.Println()

	fmt.Println("Huffman Coding (GREEDY):")
	fmt.Println("  • Merge two minimum frequencies")
	fmt.Println("  • Greedy choice: smallest first")
	fmt.Println("  • Proof: exchange argument works")
	fmt.Println("  • Time: O(n log n)")
	fmt.Println()

	fmt.Println("Optimal BST (DP):")
	fmt.Println("  • Find BST minimizing expected search time")
	fmt.Println("  • Must try ALL possible roots")
	fmt.Println("  • Greedy (most frequent as root) is WRONG")
	fmt.Println("  • Time: O(n³)")
	fmt.Println()

	fmt.Println("Why the difference?")
	fmt.Println("  Huffman: only combine adjacent elements")
	fmt.Println("  BST: root choice affects ENTIRE subtree structure")
	fmt.Println("  → No greedy choice property for BST")
}
```

---

## Example 6: Stock Trading — Greedy vs DP Variants

```go
package main

import "fmt"

// Unlimited transactions → Greedy works
func maxProfitUnlimited(prices []int) int {
	profit := 0
	for i := 1; i < len(prices); i++ {
		if prices[i] > prices[i-1] {
			profit += prices[i] - prices[i-1]
		}
	}
	return profit
}

// At most 2 transactions → DP needed
func maxProfitTwoTx(prices []int) int {
	n := len(prices)
	if n < 2 { return 0 }

	// Forward pass: max profit with one tx ending at or before i
	forward := make([]int, n)
	minPrice := prices[0]
	for i := 1; i < n; i++ {
		if prices[i] < minPrice { minPrice = prices[i] }
		forward[i] = forward[i-1]
		if prices[i]-minPrice > forward[i] {
			forward[i] = prices[i] - minPrice
		}
	}

	// Backward pass: max profit with one tx starting at or after i
	backward := make([]int, n)
	maxPrice := prices[n-1]
	for i := n - 2; i >= 0; i-- {
		if prices[i] > maxPrice { maxPrice = prices[i] }
		backward[i] = backward[i+1]
		if maxPrice-prices[i] > backward[i] {
			backward[i] = maxPrice - prices[i]
		}
	}

	maxProfit := 0
	for i := 0; i < n; i++ {
		total := forward[i] + backward[i]
		if total > maxProfit { maxProfit = total }
	}
	return maxProfit
}

func main() {
	prices := []int{3, 3, 5, 0, 0, 3, 1, 4}
	fmt.Println("Unlimited (greedy):", maxProfitUnlimited(prices)) // 8
	fmt.Println("At most 2 (DP):", maxProfitTwoTx(prices))         // 6
}
```

---

## Example 7: Greedy Fails — Subset Sum

```go
package main

import "fmt"

func subsetSumGreedy(nums []int, target int) bool {
	// Greedy: take largest that fits
	// This is WRONG
	remaining := target
	for i := len(nums) - 1; i >= 0 && remaining > 0; i-- {
		if nums[i] <= remaining {
			remaining -= nums[i]
		}
	}
	return remaining == 0
}

func subsetSumDP(nums []int, target int) bool {
	dp := make([]bool, target+1)
	dp[0] = true
	for _, num := range nums {
		for j := target; j >= num; j-- {
			if dp[j-num] { dp[j] = true }
		}
	}
	return dp[target]
}

func main() {
	nums := []int{3, 7, 1, 8, 4}
	target := 11

	fmt.Println("Greedy:", subsetSumGreedy(nums, target))
	fmt.Println("DP:", subsetSumDP(nums, target)) // true: 3+8=11 or 7+4=11

	// Counter-example
	nums2 := []int{1, 5, 6, 9}
	fmt.Println("\nFor target=11:")
	fmt.Println("Greedy:", subsetSumGreedy(nums2, 11)) // takes 9,1=10, fails
	fmt.Println("DP:", subsetSumDP(nums2, 11))          // true: 5+6=11
}
```

---

## Example 8: When to Choose Greedy vs DP

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Decision Framework: Greedy vs DP ===")
	fmt.Println()

	fmt.Println("Ask these questions:")
	fmt.Println()

	fmt.Println("1. Does a locally optimal choice lead to global optimum?")
	fmt.Println("   YES → Try Greedy")
	fmt.Println("   NO  → Use DP")
	fmt.Println()

	fmt.Println("2. Are there overlapping subproblems?")
	fmt.Println("   YES → DP (memoize)")
	fmt.Println("   NO  → Greedy or D&C")
	fmt.Println()

	fmt.Println("3. Can you prove the greedy choice property?")
	fmt.Println("   YES → Greedy is safe")
	fmt.Println("   NO  → Default to DP")
	fmt.Println()

	fmt.Println("4. Does the problem have constraints on choices?")
	fmt.Println("   All-or-nothing → Usually DP (0/1 knapsack)")
	fmt.Println("   Fractionable   → Usually Greedy")
	fmt.Println()

	fmt.Println("Rule of thumb:")
	fmt.Println("  If unsure, start with DP — it's always safe")
	fmt.Println("  Then check if greedy works for optimization")
}
```

---

## Example 9: Same Problem, Different Constraints

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Same Problem Structure, Different Solutions ===")
	fmt.Println()

	fmt.Println("┌─────────────────────┬──────────┬─────────────────┐")
	fmt.Println("│ Problem             │ Greedy?  │ Why             │")
	fmt.Println("├─────────────────────┼──────────┼─────────────────┤")
	fmt.Println("│ Fractional Knapsack │ ✓ YES    │ Can take parts  │")
	fmt.Println("│ 0/1 Knapsack        │ ✗ NO     │ All or nothing  │")
	fmt.Println("├─────────────────────┼──────────┼─────────────────┤")
	fmt.Println("│ Activity (unweight) │ ✓ YES    │ Greedy choice   │")
	fmt.Println("│ Activity (weighted) │ ✗ NO     │ Value matters   │")
	fmt.Println("├─────────────────────┼──────────┼─────────────────┤")
	fmt.Println("│ Shortest path (+w)  │ ✓ YES    │ Dijkstra        │")
	fmt.Println("│ Shortest path (-w)  │ ✗ NO     │ Bellman-Ford    │")
	fmt.Println("├─────────────────────┼──────────┼─────────────────┤")
	fmt.Println("│ Stock (unlimited)   │ ✓ YES    │ Take all gains  │")
	fmt.Println("│ Stock (k trades)    │ ✗ NO     │ Must optimize k │")
	fmt.Println("├─────────────────────┼──────────┼─────────────────┤")
	fmt.Println("│ Coin change (US)    │ ✓ YES    │ Canonical coins │")
	fmt.Println("│ Coin change (any)   │ ✗ NO     │ Counter-examples│")
	fmt.Println("└─────────────────────┴──────────┴─────────────────┘")
}
```

---

## Example 10: Comprehensive Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Greedy vs DP — Complete Comparison ===")
	fmt.Println()

	fmt.Println("                    GREEDY              DP")
	fmt.Println("  ─────────────────────────────────────────────")
	fmt.Println("  Approach:         Top-down,           Bottom-up or")
	fmt.Println("                    one choice           top-down+memo")
	fmt.Println()
	fmt.Println("  Choices:          One best choice      All choices")
	fmt.Println("                    per step             explored")
	fmt.Println()
	fmt.Println("  Backtrack:        Never                Implicitly")
	fmt.Println()
	fmt.Println("  Correctness:      Must prove greedy    Always correct")
	fmt.Println("                    choice property      with opt. substr.")
	fmt.Println()
	fmt.Println("  Efficiency:       O(n) or O(n log n)   O(n²), O(nW)...")
	fmt.Println()
	fmt.Println("  Implementation:   Simple               More complex")
	fmt.Println()
	fmt.Println("  When to use:      Can prove greedy     When greedy")
	fmt.Println("                    choice property      fails")
	fmt.Println()
	fmt.Println("  Both require:     Optimal substructure")
	fmt.Println("  Only DP needs:    Overlapping subproblems")
	fmt.Println("  Only Greedy needs: Greedy choice property")
}
```

---

## Key Takeaways

1. Greedy makes one locally optimal choice; DP explores all choices
2. Greedy is faster but requires proof of correctness
3. When unsure, default to DP — it's always safe
4. Same problem with different constraints may need different approaches
5. Both require optimal substructure; greedy additionally needs greedy choice property

> **Next up:** Exchange Argument Proofs →
