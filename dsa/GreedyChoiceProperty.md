# Phase 17: Greedy Algorithms — Greedy Choice Property

## Overview

The **Greedy Choice Property** states that a globally optimal solution can be reached by making locally optimal (greedy) choices at each step. Combined with **optimal substructure**, it guarantees correctness of greedy algorithms.

| Concept | Detail |
|---------|--------|
| **Greedy Choice** | Pick the locally best option at each step |
| **Optimal Substructure** | Optimal solution contains optimal sub-solutions |
| **Proof technique** | Show greedy stays ahead, or exchange argument |

---

## Example 1: Coin Change — Greedy Works (Canonical Denominations)

```go
package main

import "fmt"

func coinChangeGreedy(coins []int, amount int) []int {
	result := []int{}
	// Coins must be sorted descending
	for _, coin := range coins {
		for amount >= coin {
			result = append(result, coin)
			amount -= coin
		}
	}
	if amount != 0 {
		return nil // greedy fails
	}
	return result
}

func main() {
	// US coins — greedy works (canonical system)
	coins := []int{25, 10, 5, 1}
	fmt.Println(coinChangeGreedy(coins, 41)) // [25 10 5 1]
	fmt.Println("Total coins:", len(coinChangeGreedy(coins, 41))) // 4
}
```

---

## Example 2: Coin Change — Greedy Fails (Non-Canonical)

```go
package main

import "fmt"

func coinChangeGreedy(coins []int, amount int) ([]int, bool) {
	result := []int{}
	rem := amount
	for _, coin := range coins {
		for rem >= coin {
			result = append(result, coin)
			rem -= coin
		}
	}
	return result, rem == 0
}

func main() {
	// Non-canonical denominations — greedy fails
	coins := []int{6, 4, 1}
	greedy, _ := coinChangeGreedy(coins, 8)
	fmt.Println("Greedy for 8:", greedy) // [6 1 1] = 3 coins

	fmt.Println("Optimal for 8: [4 4] = 2 coins")
	fmt.Println()
	fmt.Println("Lesson: Greedy choice property must be PROVEN,")
	fmt.Println("not assumed. It doesn't hold for all coin systems.")
}
```

---

## Example 3: Jump Game — Greedy Proof

```go
package main

import "fmt"

// Can you reach the last index?
// Greedy: maintain farthest reachable position
func canJump(nums []int) bool {
	farthest := 0
	for i := 0; i < len(nums); i++ {
		if i > farthest {
			return false
		}
		if i+nums[i] > farthest {
			farthest = i + nums[i]
		}
	}
	return true
}

func main() {
	fmt.Println(canJump([]int{2, 3, 1, 1, 4})) // true
	fmt.Println(canJump([]int{3, 2, 1, 0, 4})) // false

	fmt.Println()
	fmt.Println("Why greedy works:")
	fmt.Println("  • At each index, we only need to know the farthest we can reach")
	fmt.Println("  • If farthest >= last index, answer is true")
	fmt.Println("  • If we're stuck (i > farthest), answer is false")
	fmt.Println("  • Greedy choice: always extend farthest reach")
}
```

---

## Example 4: Minimum Jump Game II

```go
package main

import "fmt"

func jump(nums []int) int {
	jumps, curEnd, farthest := 0, 0, 0
	for i := 0; i < len(nums)-1; i++ {
		if i+nums[i] > farthest {
			farthest = i + nums[i]
		}
		if i == curEnd {
			jumps++
			curEnd = farthest
		}
	}
	return jumps
}

func main() {
	fmt.Println(jump([]int{2, 3, 1, 1, 4})) // 2
	fmt.Println(jump([]int{2, 3, 0, 1, 4})) // 2

	fmt.Println()
	fmt.Println("Greedy choice: at each 'level', jump to farthest reachable")
	fmt.Println("Similar to BFS — each jump is a level")
}
```

---

## Example 5: Assign Cookies (LC 455)

```go
package main

import (
	"fmt"
	"sort"
)

func findContentChildren(greed []int, cookies []int) int {
	sort.Ints(greed)
	sort.Ints(cookies)

	child, cookie := 0, 0
	for child < len(greed) && cookie < len(cookies) {
		if cookies[cookie] >= greed[child] {
			child++ // satisfied
		}
		cookie++
	}
	return child
}

func main() {
	fmt.Println(findContentChildren([]int{1, 2, 3}, []int{1, 1}))    // 1
	fmt.Println(findContentChildren([]int{1, 2}, []int{1, 2, 3}))    // 2

	fmt.Println()
	fmt.Println("Greedy choice: assign smallest sufficient cookie to")
	fmt.Println("least greedy child. This leaves bigger cookies for")
	fmt.Println("greedier children — optimal by exchange argument.")
}
```

---

## Example 6: Best Time to Buy and Sell Stock II

```go
package main

import "fmt"

func maxProfit(prices []int) int {
	profit := 0
	for i := 1; i < len(prices); i++ {
		if prices[i] > prices[i-1] {
			profit += prices[i] - prices[i-1]
		}
	}
	return profit
}

func main() {
	fmt.Println(maxProfit([]int{7, 1, 5, 3, 6, 4})) // 7
	fmt.Println(maxProfit([]int{1, 2, 3, 4, 5}))     // 4
	fmt.Println(maxProfit([]int{7, 6, 4, 3, 1}))     // 0

	fmt.Println()
	fmt.Println("Greedy choice: collect every positive price difference")
	fmt.Println("Equivalent to buying/selling at every local min/max")
}
```

---

## Example 7: Gas Station (LC 134)

```go
package main

import "fmt"

func canCompleteCircuit(gas []int, cost []int) int {
	totalGas, totalCost := 0, 0
	tank, start := 0, 0

	for i := 0; i < len(gas); i++ {
		totalGas += gas[i]
		totalCost += cost[i]
		tank += gas[i] - cost[i]
		if tank < 0 {
			start = i + 1 // greedy: restart from next station
			tank = 0
		}
	}

	if totalGas < totalCost { return -1 }
	return start
}

func main() {
	fmt.Println(canCompleteCircuit(
		[]int{1, 2, 3, 4, 5},
		[]int{3, 4, 5, 1, 2},
	)) // 3

	fmt.Println()
	fmt.Println("Greedy choice: if we can't reach station i+1 from start,")
	fmt.Println("then no station between start..i can be the answer either.")
	fmt.Println("So jump start to i+1.")
}
```

---

## Example 8: Partition Labels (LC 763)

```go
package main

import "fmt"

func partitionLabels(s string) []int {
	// Find last occurrence of each character
	last := [26]int{}
	for i, c := range s {
		last[c-'a'] = i
	}

	result := []int{}
	start, end := 0, 0
	for i, c := range s {
		if last[c-'a'] > end {
			end = last[c-'a']
		}
		if i == end {
			result = append(result, end-start+1)
			start = end + 1
		}
	}
	return result
}

func main() {
	fmt.Println(partitionLabels("ababcbacadefegdehijhklij"))
	// [9 7 8]

	fmt.Println()
	fmt.Println("Greedy: extend current partition to include all")
	fmt.Println("occurrences of each character seen so far")
}
```

---

## Example 9: Proving Greedy Choice Property

```go
package main

import "fmt"

func main() {
	fmt.Println("=== How to Prove Greedy Choice Property ===")
	fmt.Println()

	fmt.Println("Method 1: Greedy Stays Ahead")
	fmt.Println("  1. Define a measure of progress")
	fmt.Println("  2. Show greedy is at least as good as OPT at each step")
	fmt.Println("  3. By induction, greedy is optimal")
	fmt.Println()

	fmt.Println("Method 2: Exchange Argument")
	fmt.Println("  1. Assume OPT differs from greedy at step k")
	fmt.Println("  2. Show you can swap OPT's choice with greedy's")
	fmt.Println("  3. Prove the swap doesn't worsen the solution")
	fmt.Println("  4. Repeat until OPT = greedy")
	fmt.Println()

	fmt.Println("Method 3: Structural (Cut-and-Paste)")
	fmt.Println("  1. Assume optimal solution doesn't include greedy choice")
	fmt.Println("  2. 'Cut' the non-greedy part, 'paste' greedy choice")
	fmt.Println("  3. Show new solution is at least as good")
	fmt.Println()

	fmt.Println("Questions to ask:")
	fmt.Println("  • Can the problem be decomposed into steps?")
	fmt.Println("  • Is there a clear 'locally best' choice?")
	fmt.Println("  • Does making this choice preserve optimality?")
	fmt.Println("  • Does the subproblem have optimal substructure?")
}
```

---

## Example 10: Greedy vs Non-Greedy Decision

```go
package main

import "fmt"

func main() {
	fmt.Println("=== When Greedy Works vs When It Doesn't ===")
	fmt.Println()

	fmt.Println("✓ Greedy WORKS:")
	fmt.Println("  • Activity selection (earliest finish)")
	fmt.Println("  • Huffman coding")
	fmt.Println("  • Fractional knapsack")
	fmt.Println("  • Minimum spanning tree (Kruskal/Prim)")
	fmt.Println("  • Shortest path (Dijkstra, non-negative)")
	fmt.Println("  • Coin change (canonical denominations)")
	fmt.Println("  • Jump game")
	fmt.Println()

	fmt.Println("✗ Greedy FAILS (need DP):")
	fmt.Println("  • 0/1 Knapsack")
	fmt.Println("  • Coin change (arbitrary denominations)")
	fmt.Println("  • Longest path in general graph")
	fmt.Println("  • Edit distance")
	fmt.Println("  • Matrix chain multiplication")
	fmt.Println()

	fmt.Println("Key difference:")
	fmt.Println("  Greedy: make ONE choice per step, never backtrack")
	fmt.Println("  DP: consider ALL choices, use optimal substructure")
}
```

---

## Key Takeaways

1. Greedy choice property = locally optimal → globally optimal
2. Must be **proven** for each problem (exchange argument or stays-ahead)
3. Not all optimization problems have the greedy choice property
4. Greedy algorithms are typically O(n log n) or O(n) — very efficient
5. When greedy fails, consider DP as the alternative

> **Next up:** Activity Selection →
