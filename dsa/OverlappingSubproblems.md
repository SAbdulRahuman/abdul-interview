# Phase 15: Dynamic Programming — Overlapping Subproblems

## Overview

**Overlapping subproblems** means the same subproblem is solved multiple times during recursion. DP exploits this by storing results (memoization or tabulation) so each subproblem is solved only once.

| Property | Meaning |
|----------|---------|
| **Overlapping** | Same function called with same arguments repeatedly |
| **Without DP** | Exponential time (redundant recomputation) |
| **With DP** | Polynomial time (each state computed once) |
| **Memoization** | Top-down: cache results in map/array |
| **Tabulation** | Bottom-up: fill table iteratively |

---

## Example 1: Fibonacci — Seeing the Overlap

```go
package main

import "fmt"

var calls int

func fibNaive(n int) int {
	calls++
	if n <= 1 { return n }
	return fibNaive(n-1) + fibNaive(n-2)
}

func main() {
	for n := 5; n <= 30; n += 5 {
		calls = 0
		result := fibNaive(n)
		fmt.Printf("fib(%2d) = %-10d  calls = %d\n", n, result, calls)
	}
	// fib(30) makes 2.6M+ calls — exponential!
}
```

**Textual Figure:**

```
Naive Fibonacci Recursion Tree for fib(5):

                         fib(5)
                       ╱        ╲
                  fib(4)         fib(3)
                 ╱      ╲        ╱      ╲
            fib(3)   fib(2)   fib(2)   fib(1)
           ╱    ╲    ╱    ╲    ╱    ╲      1
       fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
       ╱    ╲    1      1     0      1     0
   fib(1) fib(0)
      1     0

Call counts for fib(5):  15 total calls
┌─────────┬───────┬───────────────┐
│   Call  │ Count │ Redundancy    │
├─────────┼───────┼───────────────┤
│ fib(5)  │   1   │               │
│ fib(4)  │   1   │               │
│ fib(3)  │   2   │ 1 redundant   │
│ fib(2)  │   3   │ 2 redundant   │
│ fib(1)  │   5   │ 4 redundant   │
│ fib(0)  │   3   │ 2 redundant   │
└─────────┴───────┴───────────────┘

Growth: fib(n) makes ~O(2^n) calls — 9 of 15 are redundant!
```

---

## Example 2: Fibonacci — Memoized (Top-Down)

```go
package main

import "fmt"

func fibMemo(n int, memo map[int]int) int {
	if n <= 1 { return n }
	if v, ok := memo[n]; ok { return v }

	memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo)
	return memo[n]
}

func main() {
	memo := map[int]int{}
	for n := 0; n <= 10; n++ {
		fmt.Printf("fib(%d) = %d\n", n, fibMemo(n, memo))
	}

	// Even fib(100) is instant now
	fmt.Printf("fib(50) = %d\n", fibMemo(50, memo))
}
```

**Textual Figure:**

```
Memoized Fibonacci Call Tree for fib(6):

                    fib(6)
                   ╱      ╲
              fib(5)      fib(4) ← CACHE HIT ✓
             ╱      ╲
        fib(4)      fib(3) ← CACHE HIT ✓
       ╱      ╲
  fib(3)      fib(2) ← CACHE HIT ✓
  ╱      ╲
fib(2)  fib(1)=1
╱    ╲
fib(1) fib(0)
  1      0

Memo map after computation:
┌─────┬───────┬─────────┬───────────────┐
│ Key │ Value │  From   │ Cache Hit?    │
├─────┼───────┼─────────┼───────────────┤
│  2  │   1   │  1+0    │ reused 2x     │
│  3  │   2   │  1+1    │ reused 2x     │
│  4  │   3   │  2+1    │ reused 1x     │
│  5  │   5   │  3+2    │               │
│  6  │   8   │  5+3    │               │
└─────┴───────┴─────────┴───────────────┘

Total calls: 11 (vs 25 naive) — O(n) with memoization
```

---

## Example 3: Fibonacci — Tabulated (Bottom-Up)

```go
package main

import "fmt"

func fibTab(n int) int {
	if n <= 1 { return n }
	dp := make([]int, n+1)
	dp[0], dp[1] = 0, 1
	for i := 2; i <= n; i++ {
		dp[i] = dp[i-1] + dp[i-2]
	}
	return dp[n]
}

func main() {
	for n := 0; n <= 15; n++ {
		fmt.Printf("fib(%2d) = %d\n", n, fibTab(n))
	}
}
```

**Textual Figure:**

```
Tabulated Fibonacci — Bottom-Up Table Construction:

Step-by-step for fib(7):

  Index:  0    1    2    3    4    5    6    7
        ┌────┬────┬────┬────┬────┬────┬────┬────┐
  Init: │  0 │  1 │  0 │  0 │  0 │  0 │  0 │  0 │
        └────┴────┴────┴────┴────┴────┴────┴────┘
  i=2:  dp[2] = dp[1]+dp[0] = 1+0 = 1
  i=3:  dp[3] = dp[2]+dp[1] = 1+1 = 2
  i=4:  dp[4] = dp[3]+dp[2] = 2+1 = 3
  i=5:  dp[5] = dp[4]+dp[3] = 3+2 = 5
  i=6:  dp[6] = dp[5]+dp[4] = 5+3 = 8
  i=7:  dp[7] = dp[6]+dp[5] = 8+5 = 13

        ┌────┬────┬────┬────┬────┬────┬────┬────┐
  Done: │  0 │  1 │  1 │  2 │  3 │  5 │  8 │ 13 │
        └────┴────┴────┴────┴────┴────┴────┴────┘
          ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑
        base base  ──────fill left to right─────→

No recursion, no cache lookups — just a simple loop. O(n) time, O(n) space.
```

---

## Example 4: Climbing Stairs (LeetCode 70)

```go
package main

import "fmt"

// Ways to climb n stairs taking 1 or 2 steps at a time
// Overlapping: ways(n) = ways(n-1) + ways(n-2) — same as Fibonacci

func climbStairs(n int) int {
	if n <= 2 { return n }
	dp := make([]int, n+1)
	dp[1], dp[2] = 1, 2
	for i := 3; i <= n; i++ {
		dp[i] = dp[i-1] + dp[i-2]
	}
	return dp[n]
}

func main() {
	for n := 1; n <= 10; n++ {
		fmt.Printf("Stairs %2d: %d ways\n", n, climbStairs(n))
	}
}
```

**Textual Figure:**

```
Climbing Stairs DP Table (n=6):

  Recurrence: dp[i] = dp[i-1] + dp[i-2]  (same as Fibonacci!)

  Step:   1    2    3    4    5    6
        ┌────┬────┬────┬────┬────┬────┐
  dp:   │  1 │  2 │  3 │  5 │  8 │ 13 │
        └────┴────┴────┴────┴────┴────┘
        base base  1+2  2+3  3+5  5+8

  Overlap: to compute ways(5), we need ways(4) and ways(3)
           to compute ways(4), we also need ways(3) ← overlap!

  Overlapping subproblem tree for ways(5):
              ways(5)
             ╱       ╲
        ways(4)     ways(3) ← computed twice without DP!
       ╱      ╲     ╱      ╲
  ways(3) ways(2) ways(2) ways(1)
      ↑              ↑
   overlap!       overlap!

  Result: 13 ways to climb 6 stairs
```

---

## Example 5: Counting Overlapping Calls

```go
package main

import "fmt"

func main() {
	// Track how many times each subproblem is called in naive Fibonacci
	counts := map[int]int{}

	var fib func(n int) int
	fib = func(n int) int {
		counts[n]++
		if n <= 1 { return n }
		return fib(n-1) + fib(n-2)
	}

	fib(10)

	fmt.Println("Subproblem call counts for fib(10):")
	for n := 0; n <= 10; n++ {
		bar := ""
		for i := 0; i < counts[n] && i < 50; i++ { bar += "█" }
		fmt.Printf("  fib(%2d): %3d times  %s\n", n, counts[n], bar)
	}
}
```

**Textual Figure:**

```
Overlapping Call Counts for fib(10):

  fib( 0):  34 times  ██████████████████████████████████
  fib( 1):  55 times  ██████████████████████████████████████████████████
  fib( 2):  34 times  ██████████████████████████████████
  fib( 3):  21 times  █████████████████████
  fib( 4):  13 times  █████████████
  fib( 5):   8 times  ████████
  fib( 6):   5 times  █████
  fib( 7):   3 times  ███
  fib( 8):   2 times  ██
  fib( 9):   1 times  █
  fib(10):   1 times  █
                       └──────────────────────────────────┘
  Total: 177 calls for just fib(10)!
  With memoization: only 11 calls (one per unique state)
  Smaller subproblems have MORE redundant calls —
  call count follows Fibonacci pattern itself!
```

---

## Example 6: Coin Change (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	// dp[i] = minimum coins needed for amount i
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ { dp[i] = amount + 1 }

	for i := 1; i <= amount; i++ {
		for _, c := range coins {
			if c <= i && dp[i-c]+1 < dp[i] {
				dp[i] = dp[i-c] + 1
			}
		}
	}

	if dp[amount] > amount { return -1 }
	return dp[amount]
}

func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36)) // 3 (25+10+1)
	fmt.Println(coinChange([]int{2}, 3))              // -1
	fmt.Println(coinChange([]int{1, 3, 4}, 6))        // 2 (3+3)
}
```

**Textual Figure:**

```
Coin Change DP Table: coins=[1,3,4], amount=6

  dp[i] = minimum coins needed for amount i

  Amount:  0    1    2    3    4    5    6
         ┌────┬────┬────┬────┬────┬────┬────┐
  Init:  │  0 │  ∞ │  ∞ │  ∞ │  ∞ │  ∞ │  ∞ │
         └────┴────┴────┴────┴────┴────┴────┘

  i=1: try coin 1: dp[0]+1=1        → dp[1]=1
  i=2: try coin 1: dp[1]+1=2        → dp[2]=2
  i=3: try coin 1: dp[2]+1=3
       try coin 3: dp[0]+1=1        → dp[3]=1  ← better!
  i=4: try coin 1: dp[3]+1=2
       try coin 3: dp[1]+1=2
       try coin 4: dp[0]+1=1        → dp[4]=1
  i=5: try coin 1: dp[4]+1=2
       try coin 3: dp[2]+1=3
       try coin 4: dp[1]+1=2        → dp[5]=2
  i=6: try coin 1: dp[5]+1=3
       try coin 3: dp[3]+1=2
       try coin 4: dp[2]+1=3        → dp[6]=2  (3+3)

         ┌────┬────┬────┬────┬────┬────┬────┐
  Done: │  0 │  1 │  2 │  1 │  1 │  2 │  2 │
         └────┴────┴────┴────┴────┴────┴────┘

  Result: 2 coins (3+3)
```

---

## Example 7: Unique Paths (LeetCode 62)

```go
package main

import "fmt"

func uniquePaths(m, n int) int {
	dp := make([][]int, m)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][0] = 1
	}
	for j := 0; j < n; j++ { dp[0][j] = 1 }

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[m-1][n-1]
}

func main() {
	tests := [][2]int{{3,7}, {3,2}, {7,3}, {3,3}}
	for _, t := range tests {
		fmt.Printf("Grid %dx%d: %d paths\n", t[0], t[1], uniquePaths(t[0], t[1]))
	}
}
```

**Textual Figure:**

```
Unique Paths — DP Table for 3x4 grid:

  dp[i][j] = dp[i-1][j] + dp[i][j-1]

         col 0  col 1  col 2  col 3
        ┌─────┬─────┬─────┬─────┐
  row 0 │  1  │  1  │  1  │  1  │  ← only right
        ├─────┼─────┼─────┼─────┤
  row 1 │  1  │  2  │  3  │  4  │
        ├─────┼─────┼─────┼─────┤
  row 2 │  1  │  3  │  6  │ 10  │  ← answer
        └─────┴─────┴─────┴─────┘
         ↑ only down

  dp[1][1] = dp[0][1] + dp[1][0] = 1+1 = 2
  dp[1][2] = dp[0][2] + dp[1][1] = 1+2 = 3
  dp[2][2] = dp[1][2] + dp[2][1] = 3+3 = 6
  dp[2][3] = dp[1][3] + dp[2][2] = 4+6 = 10

  Overlap: cell (1,1) is needed by both (1,2) and (2,1)
  → Without DP, it would be recomputed.
  Result: 10 unique paths in 3x4 grid
```

---

## Example 8: House Robber (LeetCode 198)

```go
package main

import "fmt"

func rob(nums []int) int {
	n := len(nums)
	if n == 0 { return 0 }
	if n == 1 { return nums[0] }

	dp := make([]int, n)
	dp[0] = nums[0]
	dp[1] = max(nums[0], nums[1])

	for i := 2; i < n; i++ {
		// Overlapping: dp[i] depends on dp[i-1] and dp[i-2]
		dp[i] = max(dp[i-1], dp[i-2]+nums[i])
	}
	return dp[n-1]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	fmt.Println(rob([]int{1, 2, 3, 1}))       // 4
	fmt.Println(rob([]int{2, 7, 9, 3, 1}))    // 12
	fmt.Println(rob([]int{2, 1, 1, 2}))       // 4
}
```

**Textual Figure:**

```
House Robber DP for nums=[2, 7, 9, 3, 1]:

  dp[i] = max(dp[i-1], dp[i-2] + nums[i])

  House:    0     1     2     3     4
  Value:   [2]   [7]   [9]   [3]   [1]
          ┌─────┬─────┬─────┬─────┬─────┐
  dp:     │  2  │  7  │ 11  │ 11  │ 12  │
          └─────┴─────┴─────┴─────┴─────┘

  dp[0] = 2               (base: rob house 0)
  dp[1] = max(2, 7) = 7   (base: best of house 0 or 1)
  dp[2] = max(dp[1], dp[0]+9) = max(7, 2+9) = 11  ← rob 0,2
  dp[3] = max(dp[2], dp[1]+3) = max(11, 7+3) = 11 ← still rob 0,2
  dp[4] = max(dp[3], dp[2]+1) = max(11, 11+1) = 12 ← rob 0,2,4

  Overlap: dp[3] needs dp[2] AND dp[1]
           dp[4] needs dp[3] AND dp[2] ← dp[2] used twice!

  Result: 12 (rob houses 0, 2, 4 → 2+9+1=12)
```

---

## Example 9: Detecting Overlapping Subproblems

```go
package main

import "fmt"

func main() {
	fmt.Println("=== How to Detect Overlapping Subproblems ===")
	fmt.Println()

	checks := []struct{ check, means string }{
		{"Same function called with same args multiple times",
			"Overlap exists — use memoization"},
		{"Recursion tree has repeated nodes",
			"Overlap exists — fib(n-1) and fib(n-2) both call fib(n-3)"},
		{"Number of distinct states << total recursive calls",
			"High overlap — DP will help significantly"},
		{"Each subproblem has unique arguments",
			"No overlap — memoization won't help (e.g., merge sort)"},
		{"State can be described by 1-3 parameters",
			"DP-friendly — can build table of manageable size"},
	}

	for i, c := range checks {
		fmt.Printf("%d. %s\n", i+1, c.check)
		fmt.Printf("   → %s\n\n", c.means)
	}
}
```

**Textual Figure:**

```
Overlap Detection Decision Tree:

  Given a recursive solution:
              │
  ┌───────────┴───────────┐
  │ Draw the recursion tree │
  └───────────┬───────────┘
              │
   ┌─────────┴──────────┐
   │ Repeated nodes exist?  │
   └────┬──────────┬─────┘
    YES │          NO │
   ┌────┴────┐  ┌───┴──────────┐
   │ OVERLAP  │  │ No overlap    │
   │ Use DP!  │  │ (e.g., merge  │
   └────┬────┘  │  sort — each   │
        │       │  subarray is   │
   ┌────┴────┐  │  unique)       │
   │ # unique │  └───────────────┘
   │ states   │
   │ << total │
   │ calls?  │
   └────┬────┘
    YES │
   ┌────┴──────────────┐
   │ High overlap ratio  │
   │ DP gives big speedup│
   └───────────────────┘
```

---

## Example 10: Overlap Comparison Table

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Overlapping Subproblems Summary ===")
	fmt.Println()

	problems := []struct{ problem, naive, dp, overlap string }{
		{"Fibonacci", "O(2^n)", "O(n)", "fib(k) called many times"},
		{"Climbing Stairs", "O(2^n)", "O(n)", "Same as Fibonacci"},
		{"Coin Change", "O(S^n)", "O(n·S)", "amount k reached many ways"},
		{"House Robber", "O(2^n)", "O(n)", "rob(i) = rob(i-1) or rob(i-2)+val"},
		{"Unique Paths", "O(2^(m+n))", "O(m·n)", "cell (i,j) reached from left and top"},
		{"LCS", "O(2^(m+n))", "O(m·n)", "LCS(i,j) computed repeatedly"},
		{"Edit Distance", "O(3^max)", "O(m·n)", "edit(i,j) called from 3 directions"},
	}

	fmt.Printf("%-16s %-10s %-8s %s\n", "Problem", "Naive", "DP", "Why overlap?")
	fmt.Println("---------------------------------------------------------")
	for _, p := range problems {
		fmt.Printf("%-16s %-10s %-8s %s\n", p.problem, p.naive, p.dp, p.overlap)
	}

	fmt.Println()
	fmt.Println("Key insight: if naive recursion is exponential but there")
	fmt.Println("are only polynomial distinct states → DP makes it polynomial.")
}
```

**Textual Figure:**

```
Overlap Summary — Naive vs DP:

┌─────────────────┬───────────┬────────┬────────────────────┐
│ Problem         │ Naive     │ DP     │ Speedup              │
├─────────────────┼───────────┼────────┼────────────────────┤
│ Fibonacci       │ O(2^n)    │ O(n)   │ 2^30 ≈ 10^9 → 30    │
│ Climbing Stairs │ O(2^n)    │ O(n)   │ Same as Fibonacci    │
│ Coin Change     │ O(S^n)    │ O(n·S) │ Exponential → poly   │
│ House Robber    │ O(2^n)    │ O(n)   │ Exponential → linear │
│ Unique Paths    │ O(2^(m+n))│ O(m·n) │ Exponential → poly   │
│ LCS             │ O(2^(m+n))│ O(m·n) │ Exponential → poly   │
│ Edit Distance   │ O(3^max)  │ O(m·n) │ Exponential → poly   │
└─────────────────┴───────────┴────────┴────────────────────┘

Key insight:  Exponential naive → Polynomial DP
              when # unique states << # total calls

  Naive calls:   ██████████████████████████  O(2^n)
  Unique states: ███                           O(n)
  Savings:       ───────────────────────  All redundant!
```

---

## Key Takeaways

1. Overlapping subproblems = same computation repeated → store results
2. Fibonacci: O(2^n) naive → O(n) with DP because only n distinct states
3. Detect overlap: draw recursion tree, look for repeated nodes
4. Two approaches: memoization (top-down, lazy) or tabulation (bottom-up, iterative)
5. If n distinct states and each takes O(1) to compute → O(n) total

> **Next up:** Optimal Substructure →
