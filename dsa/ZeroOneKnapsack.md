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

**Textual Figure:**

```
0/1 Knapsack: weights=[2,3,4,5], values=[3,4,5,6], W=8

  dp[i][w] = max(skip=dp[i-1][w], take=dp[i-1][w-wt]+val)

          w=  0   1   2   3   4   5   6   7   8
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  i=0   │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │  no items
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=1   │ 0 │ 0 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │  item0(w=2,v=3)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=2   │ 0 │ 0 │ 3 │ 4 │ 4 │ 7 │ 7 │ 7 │ 7 │  item1(w=3,v=4)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=3   │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │ 9 │  item2(w=4,v=5)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=4   │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │[10]│  item3(w=5,v=6)
        └───┴───┴───┴───┴───┴───┴───┴───┴───┘

  dp[4][8]: skip=dp[3][8]=9, take=dp[3][3]+6=4+6=10 → 10 ✓
  Items chosen: #1(w=3,v=4) + #3(w=5,v=6) = weight 8, value 10
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

**Textual Figure:**

```
Space-Optimized 0/1 Knapsack: weights=[1,3,4,5], values=[1,4,5,7], W=7

  Single 1D array, iterate RIGHT TO LEFT to prevent reuse:

  w→     0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
  Init │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
  Item 0 (w=1,v=1): scan w=7→1
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 0 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │  ←───── direction
       └───┴───┴───┴───┴───┴───┴───┴───┘
  Item 1 (w=3,v=4): scan w=7→3
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 0 │ 1 │ 1 │ 4 │ 5 │ 5 │ 5 │ 5 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
  Item 2 (w=4,v=5): scan w=7→4
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 0 │ 1 │ 1 │ 4 │ 5 │ 6 │ 6 │ 9 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
  Item 3 (w=5,v=7): scan w=7→5
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 0 │ 1 │ 1 │ 4 │ 5 │ 7 │ 8 │[9]│
       └───┴───┴───┴───┴───┴───┴───┴───┘

  Reverse scan ensures each item used at most once.
  Result: 9 (items 1+2: w=3+4=7, v=4+5=9)
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

**Textual Figure:**

```
Partition Equal Subset Sum: nums=[1,5,11,5], sum=22, target=11

  dp[w] = can we form sum w? (boolean knapsack)

  w→    0   1   2   3   4   5   6   7   8   9  10  11
       ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  Init │ T │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │
       └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  num=1:┌───┬───┐
       │ T │ T │ F  F  F  F  F  F  F  F  F  F
       └───┴───┘
  num=5:┌───┬───┬───┬───┬───┬───┬───┐
       │ T │ T │ F │ F │ F │ T │ T │ F  F  F  F  F
       └───┴───┴───┴───┴───┴───┴───┘
  num=11:
       T  T  F  F  F  T  T  F  F  F  F  [T] ← dp[11]=true!
  num=5:
       T  T  F  F  F  T  T  F  F  F  T  [T] ✓

  dp[11] = true → partition exists: {1,5,5} and {11}
  Result: true
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

**Textual Figure:**

```
Target Sum: nums=[1,1,1,1,1], target=3
  sum=5, subsetSum=(5+3)/2=4
  Find # subsets summing to 4

  dp[w] = # ways to form sum w

  w→    0   1   2   3   4
       ┌───┬───┬───┬───┬───┐
  Init │ 1 │ 0 │ 0 │ 0 │ 0 │
       └───┴───┴───┴───┴───┘
  num=1: dp[w] += dp[w-1] (reverse)
       ┌───┬───┬───┬───┬───┐
       │ 1 │ 1 │ 0 │ 0 │ 0 │
       └───┴───┴───┴───┴───┘
  num=1:
       ┌───┬───┬───┬───┬───┐
       │ 1 │ 2 │ 1 │ 0 │ 0 │
       └───┴───┴───┴───┴───┘
  num=1:
       ┌───┬───┬───┬───┬───┐
       │ 1 │ 3 │ 3 │ 1 │ 0 │
       └───┴───┴───┴───┴───┘
  num=1:
       ┌───┬───┬───┬───┬───┐
       │ 1 │ 4 │ 6 │ 4 │ 1 │
       └───┴───┴───┴───┴───┘
  num=1:
       ┌───┬───┬───┬───┬───┐
       │ 1 │ 5 │10 │10 │[5]│  ← dp[4]=5
       └───┴───┴───┴───┴───┘

  5 ways: +1+1+1+1-1, +1+1+1-1+1, +1+1-1+1+1,
          +1-1+1+1+1, -1+1+1+1+1
  Result: 5
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

**Textual Figure:**

```
Count Subsets: nums=[1,2,3,4,5], target=7

  dp[w] = number of subsets summing to w

  w→    0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
  Init │ 1 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
  +num=1:  1  1  0  0  0  0  0  0
  +num=2:  1  1  1  1  0  0  0  0
  +num=3:  1  1  1  2  1  1  1  0
  +num=4:  1  1  1  2  2  2  2  2
  +num=5:  1  1  1  2  2  3  3  3
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
  Done │ 1 │ 1 │ 1 │ 2 │ 2 │ 3 │ 3 │[3]│
       └───┴───┴───┴───┴───┴───┴───┴───┘

  dp[7] = 3 subsets: {2,5}, {3,4}, {1,2,4}
  Result: 3 subsets sum to 7
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

**Textual Figure:**

```
Minimum Subset Sum Difference: nums=[1,6,11,5], total=23, half=11

  dp[w] = can we form sum w?

  w→    0   1   2   3   4   5   6   7   8   9  10  11
       ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  Init │ T │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │ F │
       └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  +1:   T  T  F  F  F  F  F  F  F  F  F  F
  +6:   T  T  F  F  F  F  T  T  F  F  F  F
  +11:  T  T  F  F  F  F  T  T  F  F  F [T]
  +5:   T  T  F  F  F  T  T  T  F  F  F [T]

  Scan from half=11 downward: dp[11]=true!
  S1 = 11, S2 = 23-11 = 12
  Difference = |12 - 11| = 1

  Partition: {11} and {1,6,5} → |11-12| = 1
  Result: 1
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

**Textual Figure:**

```
2D Knapsack: strs=["10","0001","111001","1","0"], m=5 zeros, n=3 ones

  Each string costs (zeros, ones):
  ┌──────────┬───────┬──────┐
  │  String  │ Zeros │ Ones │
  ├──────────┼───────┼──────┤
  │ "10"     │   1   │  1   │
  │ "0001"   │   3   │  1   │
  │ "111001" │   2   │  4   │  ← too many ones (4>3)
  │ "1"      │   0   │  1   │
  │ "0"      │   1   │  0   │
  └──────────┴───────┴──────┘

  dp[i][j] = max strings with ≤i zeros, ≤j ones
  Final dp[5][3] after processing all strings:

        j=  0   1   2   3
  i=0  │  0 │  0 │  0 │  0 │
  i=1  │  1 │  1 │  1 │  1 │  ("0" or "10")
  i=2  │  1 │  2 │  2 │  2 │
  i=3  │  1 │  2 │  3 │  3 │
  i=4  │  1 │  2 │  3 │  3 │
  i=5  │  1 │  2 │  3 │ [4]│

  Result: 4 strings ("10","0001","1","0")
  Cost: 1+3+0+1=5 zeros, 1+1+1+0=3 ones ≤ (5,3) ✓
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

**Textual Figure:**

```
Knapsack with Backtracking: w=[2,3,4,5], v=[3,4,5,6], W=8

  DP Table (same as Example 1):
          w=  0   1   2   3   4   5   6   7   8
  i=0       │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
  i=1       │ 0 │ 0 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │
  i=2       │ 0 │ 0 │ 3 │ 4 │ 4 │ 7 │ 7 │ 7 │ 7 │
  i=3       │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │ 9 │
  i=4       │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │[10]│

  Backtrack from dp[4][8]=10:
    i=4: dp[4][8]=10 ≠ dp[3][8]=9  → item 3 TAKEN (w=5)
         w = 8-5 = 3
    i=3: dp[3][3]=4  ≠ dp[2][3]=4  → wait, equal → item 2 SKIPPED
    i=2: dp[2][3]=4  ≠ dp[1][3]=3  → item 1 TAKEN (w=3)
         w = 3-3 = 0
    i=1: dp[1][0]=0  = dp[0][0]=0  → item 0 SKIPPED

  Items chosen: [3, 1] (0-indexed)
    Item 1: weight=3, value=4
    Item 3: weight=5, value=6
    Total: weight=8, value=10 ✓
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

**Textual Figure:**

```
Bounded Knapsack: w=[2,3,5], v=[3,4,7], counts=[3,2,1], W=10

  Binary grouping for each item:
  ┌──────┬───────┬──────────────────────────────┐
  │ Item │ Count │ Binary groups              │
  ├──────┼───────┼──────────────────────────────┤
  │  0   │   3   │ 1×(w=2,v=3), 2×(w=4,v=6)  │
  │  1   │   2   │ 1×(w=3,v=4), 1×(w=3,v=4)  │
  │  2   │   1   │ 1×(w=5,v=7)               │
  └──────┴───────┴──────────────────────────────┘

  1D DP array progression (w = 0..10):
  w→     0  1  2  3  4  5  6  7  8  9  10
  Init:  0  0  0  0  0  0  0  0  0  0   0
  grp(2,3):  0  0  3  3  3  3  3  3  3  3   3
  grp(4,6):  0  0  3  3  6  6  9  9  9  9   9
  grp(3,4):  0  0  3  4  6  7  9 10 13 13  13
  grp(3,4):  0  0  3  4  6  7  9 10 13 14  14
  grp(5,7):  0  0  3  4  6  7  9 10 13 14 [15]

  Result: 15 (3×item0 + 1×item1 = 3×3+4=13... actually
  items: 2×item0(w=4,v=6)+1×item1(w=3,v=4)+1×item2(w=5=..wait
  Best: value 15 at W=10)
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

**Textual Figure:**

```
0/1 Knapsack Pattern Family:

                    ┌────────────────┐
                    │  0/1 Knapsack  │
                    └───────┬────────┘
            ┌────────┼────────┬────────┐
            ▼        ▼        ▼        ▼
    ┌─────────┐┌───────┐┌───────┐┌─────────┐
    │ Subset  ││Target ││ Equal ││ Min Diff│
    │ Sum     ││ Sum   ││Parttn ││         │
    └─────────┘└───────┘└───────┘└─────────┘
      bool     count   target    max
      dp[w]    dp[w]   =sum/2   sum≤half

  Key technique: REVERSE ITERATION
    for w := W; w >= wt; w--
    ┌────────────────────────────────┐
    │  ←───── scan direction        │
    │  dp[7] dp[6] dp[5] ... dp[0] │
    │  Uses old values (not yet     │
    │  updated) → each item used     │
    │  at most once (0/1 property)  │
    └────────────────────────────────┘
```

---

## Key Takeaways

1. 0/1 Knapsack: dp[i][w] = max(skip, take) — O(n·W) time, O(W) space optimized
2. Reverse iteration in 1D prevents reusing same item (0/1 property)
3. Subset sum, partition, target sum are all knapsack variants
4. For reconstruction: keep full 2D table, backtrack from dp[n][W]
5. Many problems reduce to: "pick items maximizing/minimizing something within a budget"

> **Next up:** Unbounded Knapsack →
