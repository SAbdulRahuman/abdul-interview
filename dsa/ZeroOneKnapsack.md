# Phase 15: Dynamic Programming — 0/1 Knapsack

## Overview

The **0/1 Knapsack** problem: given items with weights and values, and a capacity W, pick items to maximize total value without exceeding W. Each item is either taken (1) or not (0).

| Aspect | Detail |
|--------|--------|
| **State** | dp[i][w] = max value using items 0..i-1 with capacity w |
| **Transition** | dp[i][w] = max(dp[i-1][w], dp[i-1][w-wt[i]] + val[i]) |
| **Time** | O(n·W) |
| **Space** | O(n·W) → optimizable to O(W) |

---

## Example 1: Classic 0/1 Knapsack

```go
package main

import "fmt"

func knapsack(weights, values []int, W int) int {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, W+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= W; w++ {
			dp[i][w] = dp[i-1][w] // skip item i
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

## Example 2: Space-Optimized 0/1 Knapsack

```go
package main

import "fmt"

func knapsack(weights, values []int, W int) int {
	dp := make([]int, W+1)

	for i := 0; i < len(weights); i++ {
		for w := W; w >= weights[i]; w-- { // reverse!
			take := dp[w-weights[i]] + values[i]
			if take > dp[w] { dp[w] = take }
		}
	}
	return dp[W]
}

func main() {
	weights := []int{1, 3, 4, 5}
	values := []int{1, 4, 5, 7}
	fmt.Println("Max value:", knapsack(weights, values, 7)) // 9
}
```

---

## Example 3: Partition Equal Subset Sum (LeetCode 416)

```go
package main

import "fmt"

// Can we partition array into two subsets with equal sum?
// Equivalent to: find a subset summing to totalSum/2

func canPartition(nums []int) bool {
	sum := 0
	for _, n := range nums { sum += n }
	if sum%2 != 0 { return false }
	target := sum / 2

	dp := make([]bool, target+1)
	dp[0] = true

	for _, num := range nums {
		for w := target; w >= num; w-- {
			dp[w] = dp[w] || dp[w-num]
		}
	}
	return dp[target]
}

func main() {
	fmt.Println(canPartition([]int{1, 5, 11, 5}))  // true
	fmt.Println(canPartition([]int{1, 2, 3, 5}))   // false
}
```

---

## Example 4: Target Sum (LeetCode 494)

```go
package main

import "fmt"

// Assign + or - to each number to reach target
// Equivalent to: find subset P where sum(P) = (totalSum + target) / 2

func findTargetSumWays(nums []int, target int) int {
	sum := 0
	for _, n := range nums { sum += n }
	if (sum+target)%2 != 0 || sum+target < 0 { return 0 }
	subsetSum := (sum + target) / 2

	dp := make([]int, subsetSum+1)
	dp[0] = 1

	for _, num := range nums {
		for w := subsetSum; w >= num; w-- {
			dp[w] += dp[w-num]
		}
	}
	return dp[subsetSum]
}

func main() {
	fmt.Println(findTargetSumWays([]int{1, 1, 1, 1, 1}, 3)) // 5
	fmt.Println(findTargetSumWays([]int{1}, 1))              // 1
}
```

---

## Example 5: Count Subsets with Given Sum

```go
package main

import "fmt"

func countSubsets(nums []int, target int) int {
	dp := make([]int, target+1)
	dp[0] = 1

	for _, num := range nums {
		for w := target; w >= num; w-- {
			dp[w] += dp[w-num]
		}
	}
	return dp[target]
}

func main() {
	nums := []int{1, 2, 3, 4, 5}
	for t := 1; t <= 15; t++ {
		c := countSubsets(nums, t)
		if c > 0 {
			fmt.Printf("Sum %2d: %d subsets\n", t, c)
		}
	}
}
```

---

## Example 6: Minimum Subset Sum Difference

```go
package main

import "fmt"

// Partition into two subsets minimizing |sum1 - sum2|
func minSubsetDiff(nums []int) int {
	total := 0
	for _, n := range nums { total += n }
	half := total / 2

	dp := make([]bool, half+1)
	dp[0] = true

	for _, num := range nums {
		for w := half; w >= num; w-- {
			dp[w] = dp[w] || dp[w-num]
		}
	}

	// Find largest achievable sum <= half
	for w := half; w >= 0; w-- {
		if dp[w] {
			return total - 2*w
		}
	}
	return total
}

func main() {
	fmt.Println(minSubsetDiff([]int{1, 6, 11, 5}))  // 1
	fmt.Println(minSubsetDiff([]int{3, 1, 4, 2, 2})) // 0
}
```

---

## Example 7: Ones and Zeroes (LeetCode 474)

```go
package main

import "fmt"

// 2D Knapsack: items have two costs (number of 0s and 1s)
func findMaxForm(strs []string, m, n int) int {
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	for _, s := range strs {
		zeros, ones := 0, 0
		for _, c := range s {
			if c == '0' { zeros++ } else { ones++ }
		}

		// Reverse iteration for 0/1 knapsack
		for i := m; i >= zeros; i-- {
			for j := n; j >= ones; j-- {
				take := dp[i-zeros][j-ones] + 1
				if take > dp[i][j] { dp[i][j] = take }
			}
		}
	}
	return dp[m][n]
}

func main() {
	strs := []string{"10", "0001", "111001", "1", "0"}
	fmt.Println(findMaxForm(strs, 5, 3)) // 4
}
```

---

## Example 8: Knapsack with Item Tracking (Reconstruct Solution)

```go
package main

import "fmt"

func knapsackWithItems(weights, values []int, W int) (int, []int) {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, W+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= W; w++ {
			dp[i][w] = dp[i-1][w]
			if weights[i-1] <= w {
				take := dp[i-1][w-weights[i-1]] + values[i-1]
				if take > dp[i][w] { dp[i][w] = take }
			}
		}
	}

	// Backtrack to find which items were chosen
	items := []int{}
	w := W
	for i := n; i > 0; i-- {
		if dp[i][w] != dp[i-1][w] {
			items = append(items, i-1)
			w -= weights[i-1]
		}
	}

	return dp[n][W], items
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}

	maxVal, items := knapsackWithItems(weights, values, 8)
	fmt.Println("Max value:", maxVal)
	fmt.Println("Items chosen (0-indexed):", items)
	for _, i := range items {
		fmt.Printf("  Item %d: weight=%d, value=%d\n", i, weights[i], values[i])
	}
}
```

---

## Example 9: Bounded Knapsack (Limited Copies)

```go
package main

import "fmt"

// Each item can be used at most count[i] times
func boundedKnapsack(weights, values, counts []int, W int) int {
	dp := make([]int, W+1)

	for i := 0; i < len(weights); i++ {
		// Binary grouping: split count into powers of 2
		remaining := counts[i]
		k := 1
		for remaining > 0 {
			batch := min(k, remaining)
			bw, bv := weights[i]*batch, values[i]*batch

			for w := W; w >= bw; w-- {
				take := dp[w-bw] + bv
				if take > dp[w] { dp[w] = take }
			}

			remaining -= batch
			k *= 2
		}
	}
	return dp[W]
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	weights := []int{2, 3, 5}
	values := []int{3, 4, 7}
	counts := []int{3, 2, 1}
	fmt.Println("Max value:", boundedKnapsack(weights, values, counts, 10)) // 15
}
```

---

## Example 10: 0/1 Knapsack Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== 0/1 Knapsack Patterns ===")
	fmt.Println()

	patterns := []struct{ variant, key, example string }{
		{"Classic", "max value ≤ W", "Items with weight/value"},
		{"Subset Sum", "dp bool, target = sum", "Can sum to target?"},
		{"Equal Partition", "target = totalSum/2", "LC 416"},
		{"Target Sum", "subset = (sum+target)/2", "LC 494"},
		{"Count subsets", "dp counts ways", "Number of subsets = target"},
		{"Min diff", "find max achievable ≤ half", "Minimize |S1-S2|"},
		{"2D Knapsack", "two capacity dimensions", "LC 474 Ones and Zeroes"},
		{"Bounded", "binary grouping", "Each item used ≤ k times"},
		{"Reconstruct", "backtrack through dp table", "Which items chosen?"},
	}

	for i, p := range patterns {
		fmt.Printf("%d. %-18s %-30s %s\n", i+1, p.variant, p.key, p.example)
	}

	fmt.Println()
	fmt.Println("Key pattern: reverse iteration (for w = W; w >= wt; w--)")
	fmt.Println("ensures each item used at most once in 1D optimization.")
}
```

---

## Key Takeaways

1. 0/1 Knapsack: dp[i][w] = max(skip, take) — O(n·W) time, O(W) space optimized
2. Reverse iteration in 1D prevents reusing same item (0/1 property)
3. Subset sum, partition, target sum are all knapsack variants
4. For reconstruction: keep full 2D table, backtrack from dp[n][W]
5. Many problems reduce to: "pick items maximizing/minimizing something within a budget"

> **Next up:** Unbounded Knapsack →
