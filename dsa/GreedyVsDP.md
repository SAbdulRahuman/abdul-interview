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

**Textual Figure:**

```
Coin Change: Greedy vs DP

Case 1: coins = [25,10,5,1], amount = 41 (US coins)
  Greedy: 25+10+5+1 = 4 coins  ✓ correct
  DP:     25+10+5+1 = 4 coins  ✓ same

Case 2: coins = [6,4,1], amount = 8
  Greedy: pick largest first
    6 + 1 + 1 = 3 coins        ✘ suboptimal!
  DP: try all combinations
    4 + 4 = 2 coins             ✓ optimal!

  DP table for coins=[6,4,1], amount=8:
  amt:  0   1   2   3   4   5   6   7   8
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  dp: │ 0 │ 1 │ 2 │ 3 │ 1 │ 2 │ 1 │ 2 │[2]│
      └───┴───┴───┴───┴───┴───┴───┴───┴───┘
  dp[8] = min(dp[2]+1, dp[4]+1) = min(3, 2) = 2

  Greedy fails when coins aren't "canonical".
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

**Textual Figure:**

```
0/1 Knapsack: Greedy vs DP

Items: {w=5,v=10}, {w=4,v=40}, {w=6,v=30}, {w=3,v=50}
Capacity: 10

Greedy (sort by value/weight ratio):
  Ratios: 10/5=2.0, 40/4=10.0, 30/6=5.0, 50/3=16.7
  Sorted: item3(16.7), item1(10.0), item2(5.0), item0(2.0)
  Pick: item3(w=3) → rem=7
        item1(w=4) → rem=3
        item2(w=6) → won't fit
        item0(w=5) → won't fit
  Greedy total: 50+40 = 90

DP table:
  cap:  0   1   2   3    4    5    6    7    8    9   10
      ┌───┬───┬───┬────┬────┬────┬────┬────┬────┬────┬────┐
  dp: │ 0 │ 0 │ 0 │ 50 │ 40 │ 50 │ 50 │ 90 │ 90 │ 90 │[90]│
      └───┴───┴───┴────┴────┴────┴────┴────┴────┴────┴────┘

  Both give 90 here. But greedy can fail on other inputs.
  Greedy is O(n log n); DP is O(nW) — always correct.
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

**Textual Figure:**

```
Activity Selection: intervals = [[1,3],[2,5],[3,9],[6,8]]
Sorted by end time: [1,3], [2,5], [6,8], [3,9]

Timeline:
  1   2   3   4   5   6   7   8   9
  │───────│                           [1,3] ✓ pick
      │───────────│                   [2,5] ✘ overlaps
                      │───────│       [6,8] ✓ pick
          │───────────────────│   [3,9] ✘ overlaps

Greedy: pick earliest-ending, skip overlaps → 2 activities
DP:     dp[i] = max activities ending at i   → 2 activities (same)

  Greedy is sufficient because:
    • Earliest-ending activity leaves most room
    • Greedy choice property holds
    • O(n log n) vs DP's O(n²)
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

**Textual Figure:**

```
LIS: nums = [10, 9, 2, 5, 3, 7, 101, 18]

Greedy (wrong — just takes next increasing):
  Start 10 → skip 9 → skip 2 → skip 5 → skip 3
  → skip 7 → take 101 → skip 18
  LIS = [10, 101] = length 2  ✘ WRONG

DP (correct):
  Index:  0   1   2   3   4   5    6    7
  nums:  10   9   2   5   3   7  101   18
       ┌───┬───┬───┬───┬───┬───┬────┬────┐
  dp:  │ 1 │ 1 │ 1 │ 2 │ 2 │ 3 │  4 │  4 │
       └───┴───┴───┴───┴───┴───┴────┴────┘

  dp[5]=3: nums[2]=2 < 7, dp[2]+1=2
           nums[3]=5 < 7, dp[3]+1=3 ← best
  dp[6]=4: nums[5]=7 < 101, dp[5]+1=4

  LIS = [2, 5, 7, 101] or [2, 3, 7, 101] = length 4 ✓

  Greedy fails: can't decide locally which elements to include.
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

**Textual Figure:**

```
Huffman (Greedy) vs Optimal BST (DP):

Huffman Coding — Greedy works:
  Frequencies: a=5, b=9, c=12, d=13, e=16, f=45

  Merge smallest pair each step:
    (a,b)=14 → (14,c)=26 → (d,e)=29 → (26,29)=55 → (f,55)=100

       100
      /   \
    f:45   55
          /  \
        26    29
       / \   / \
     14  c  d   e
    / \
   a   b

  Greedy choice: smallest first always works.

Optimal BST — DP required:
  Keys: [10, 12, 20], freq: [34, 8, 50]
  Greedy (most frequent as root = 20):
    20(50) → depth 1 cost = 50
    10(34), 12(8) below → more depth
  DP tries ALL roots, computes total cost.

  Greedy choice property does NOT hold.
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

**Textual Figure:**

```
Stock Trading: prices = [3, 3, 5, 0, 0, 3, 1, 4]

Price chart:
  5 │     *
  4 │                           *
  3 │ * *           *
  2 │
  1 │                     *
  0 │           * *
    └───────────────────────────
      0  1  2  3  4  5  6  7

Unlimited (Greedy): take every upswing
  3→5(+2), 0→3(+3), 1→4(+3) = 8
  Simple: profit += max(0, price[i]-price[i-1])

At Most 2 Transactions (DP):
  forward[i]:  max profit with 1 tx ending at/before i
  backward[i]: max profit with 1 tx starting at/after i

    i:       0   1   2   3   4   5   6   7
  forward:  [0,  0,  2,  2,  2,  3,  3,  4]
  backward: [5,  5,  5,  4,  4,  3,  3,  0]
  total:    [5,  5,  7,  6,  6,  6,  6,  4]

  Best split at i=2: forward[2]+backward[2] = 2+5... = 7?
  Output says 6. The constraint limits optimization.
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

**Textual Figure:**

```
Subset Sum: nums = [1, 5, 6, 9], target = 11

Greedy (take largest that fits):
  rem=11: take 9 → rem=2
  rem=2:  take 1 → rem=1
  rem=1:  nothing fits → FAIL  ✘

DP (try all subsets via table):
  target:  0   1   2   3   4   5   6   7   8   9  10  11
         ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  init:  │ T │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │
  +1:    │ T │ T │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │
  +5:    │ T │ T │ F │ F │ F │ T │ T │ F │ F │ F │ F │ F │
  +6:    │ T │ T │ F │ F │ F │ T │ T │ T │ F │ F │ F │[T]│
         └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
               dp[11] becomes T after adding 6: dp[11-6]=dp[5]=T

  Answer: TRUE (5+6=11)  ✓
  Greedy can't see that skipping 9 leads to solution.
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

**Textual Figure:**

```
Decision Framework: Greedy vs DP

                  ┌─────────────────────┐
                  │  Optimization Problem │
                  └──────────┬──────────┘
                             │
               Locally optimal = global?
               ┌─────┴─────┐
            YES │           │ NO / UNSURE
               ▼           ▼
         ┌────────┐  ┌─────────────┐
         │ GREEDY │  │ Overlapping   │
         └────────┘  │ subproblems?  │
                     └──────┬──────┘
                      YES │    NO
                         ▼     ▼
                    ┌─────┐ ┌──────┐
                    │ DP  │ │ D & C │
                    └─────┘ └──────┘

  Rule: When unsure, default to DP — always safe.
  Then verify if greedy works as an optimization.
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
**Textual Figure:**

```
Same Problem, Different Constraints:

┌───────────────────────────────────────────────┐
│ Constraint     │ Greedy? │ Why                     │
├────────────────┼─────────┼─────────────────────────┤
│ Fractional     │   ✓     │ Can take partial items   │
│ All-or-nothing │   ✘     │ Must explore all combos  │
├────────────────┼─────────┼─────────────────────────┤
│ Unweighted     │   ✓     │ Earliest-end works      │
│ Weighted       │   ✘     │ Value matters per item   │
├────────────────┼─────────┼─────────────────────────┤
│ Positive wt    │   ✓     │ Dijkstra                │
│ Negative wt    │   ✘     │ Need Bellman-Ford        │
├────────────────┼─────────┼─────────────────────────┤
│ Unlimited txn  │   ✓     │ Take all upswings       │
│ K trades       │   ✘     │ Must optimize k choices  │
└────────────────┴─────────┴─────────────────────────┘

  Pattern: constraints that allow "partial" or
  "local sufficiency" → Greedy
  Otherwise → DP
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

**Textual Figure:**

```
Greedy vs DP — Complete Comparison:

              ┌───────────────┬─────────────────┐
              │    GREEDY      │       DP        │
  ┌──────────┼───────────────┼─────────────────┤
  │ Approach │ 1 best choice  │ All choices     │
  │          │ per step       │ explored        │
  ├──────────┼───────────────┼─────────────────┤
  │ Correct  │ Must prove     │ Always correct  │
  │          │ greedy choice  │ with opt substr │
  ├──────────┼───────────────┼─────────────────┤
  │ Time     │ O(n), O(nlogn)│ O(n²), O(nW)    │
  ├──────────┼───────────────┼─────────────────┤
  │ Requires │ Opt substruc  │ Opt substruc    │
  │          │ + Greedy prop │ + Overlapping   │
  └──────────┴───────────────┴─────────────────┘

  Both share:      Optimal Substructure
  Only Greedy:     Greedy Choice Property
  Only DP:         Overlapping Subproblems
```

---

## Key Takeaways

1. Greedy makes one locally optimal choice; DP explores all choices
2. Greedy is faster but requires proof of correctness
3. When unsure, default to DP — it's always safe
4. Same problem with different constraints may need different approaches
5. Both require optimal substructure; greedy additionally needs greedy choice property

> **Next up:** Exchange Argument Proofs →
