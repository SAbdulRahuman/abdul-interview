# Phase 15: Dynamic Programming — Tabulation

## Overview

**Tabulation** is the bottom-up DP approach: fill a table iteratively from base cases to the final answer. No recursion needed — deterministic, cache-friendly, and easy to optimize for space.

| Aspect | Detail |
|--------|--------|
| **Approach** | Bottom-up iterative |
| **Order** | Must solve subproblems in correct dependency order |
| **Storage** | Array/table indexed by state parameters |
| **Pros** | No recursion overhead; easy space optimization |
| **Cons** | Computes all states (even unreachable ones); order must be determined |

---

## Example 1: Fibonacci Tabulated

```go
package main

import "fmt"

func fib(n int) int {
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
		fmt.Printf("fib(%2d) = %d\n", n, fib(n))
	}
}
```

**Textual Figure:**

```
Fibonacci Tabulation: fib(6)

Index:   0   1   2   3   4   5   6
       ┌───┬───┬───┬───┬───┬───┬───┐
  dp:  │ 0 │ 1 │ 1 │ 2 │ 3 │ 5 │ 8 │
       └───┴───┴───┴───┴───┴───┴───┘

Fill order (left → right):

  dp[0]=0  dp[1]=1  (base cases)
  dp[2] = dp[1] + dp[0] = 1 + 0 = 1
  dp[3] = dp[2] + dp[1] = 1 + 1 = 2
  dp[4] = dp[3] + dp[2] = 2 + 1 = 3
  dp[5] = dp[4] + dp[3] = 3 + 2 = 5
  dp[6] = dp[5] + dp[4] = 5 + 3 = 8  ← answer

  Each cell depends on two previous cells:
  dp[i-2] ──┐
            ├──→ dp[i] = dp[i-1] + dp[i-2]
  dp[i-1] ──┘
```

---

## Example 2: Climbing Stairs — 1D Table

```go
package main

import "fmt"

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
		fmt.Printf("n=%2d: %d ways\n", n, climbStairs(n))
	}
}
```

**Textual Figure:**

```
Climbing Stairs: n = 5

Step:    1   2   3   4   5
       ┌───┬───┬───┬───┬───┐
  dp:  │ 1 │ 2 │ 3 │ 5 │ 8 │
       └───┴───┴───┴───┴───┘

Trace:
  dp[1] = 1                        (base: 1 way)
  dp[2] = 2                        (base: 2 ways)
  dp[3] = dp[2] + dp[1] = 2+1 = 3  (1+1+1, 1+2, 2+1)
  dp[4] = dp[3] + dp[2] = 3+2 = 5
  dp[5] = dp[4] + dp[3] = 5+3 = 8  ← answer

  Decision tree collapsed into table:
        ┌─ take 1 step → dp[i-1]
  dp[i]─┤
        └─ take 2 steps → dp[i-2]
```

---

## Example 3: Coin Change — 1D Table (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ {
		dp[i] = amount + 1 // sentinel for "impossible"
	}

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
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36)) // 3
	fmt.Println(coinChange([]int{2}, 3))              // -1
}
```

**Textual Figure:**

```
Coin Change: coins = {1, 5, 10, 25}, amount = 36

Amount:  0   1   2   3   4   5   6  ...  10  ...  25  ...  36
       ┌───┬───┬───┬───┬───┬───┬───┬───┬────┬───┬────┬───┬───┐
  dp:  │ 0 │ 1 │ 2 │ 3 │ 4 │ 1 │ 2 │...│  1 │...│  1 │...│ 3 │
       └───┴───┴───┴───┴───┴───┴───┴───┴────┴───┴────┴───┴───┘

For amount=36:
  dp[36] = min over all coins c where c ≤ 36:
    c=1:  dp[35]+1 = 3+1 = 4
    c=5:  dp[31]+1 = 3+1 = 4
    c=10: dp[26]+1 = 2+1 = 3  ← min
    c=25: dp[11]+1 = 2+1 = 3
  dp[36] = 3  (25 + 10 + 1)

  Transition per cell:
  for each coin c:
    dp[w] = min(dp[w], dp[w-c] + 1)
              ↑ skip      ↑ use coin c
```

---

## Example 4: Longest Common Subsequence — 2D Table (LeetCode 1143)

```go
package main

import "fmt"

func longestCommonSubsequence(s1, s2 string) int {
	m, n := len(s1), len(s2)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	// Fill table bottom-up
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if s1[i-1] == s2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				if dp[i-1][j] > dp[i][j-1] {
					dp[i][j] = dp[i-1][j]
				} else {
					dp[i][j] = dp[i][j-1]
				}
			}
		}
	}

	return dp[m][n]
}

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace")) // 3
	fmt.Println(longestCommonSubsequence("abc", "abc"))   // 3
	fmt.Println(longestCommonSubsequence("abc", "def"))   // 0
}
```

**Textual Figure:**

```
LCS Table: s1 = "abcde", s2 = "ace"

        ""   a    c    e
    ┌────┬────┬────┬────┐
 "" │  0 │  0 │  0 │  0 │
    ├────┼────┼────┼────┤
  a │  0 │ [1]│  1 │  1 │  a==a → dp[0][0]+1 = 1
    ├────┼────┼────┼────┤
  b │  0 │  1 │  1 │  1 │  no match → max(↑, ←)
    ├────┼────┼────┼────┤
  c │  0 │  1 │ [2]│  2 │  c==c → dp[1][1]+1 = 2
    ├────┼────┼────┼────┤
  d │  0 │  1 │  2 │  2 │  no match → max(↑, ←)
    ├────┼────┼────┼────┤
  e │  0 │  1 │  2 │ [3]│  e==e → dp[3][2]+1 = 3
    └────┴────┴────┴────┘

  Answer: dp[5][3] = 3  (LCS = "ace")

  Transition:
  match:      dp[i][j] = dp[i-1][j-1] + 1   (↖ + 1)
  no match:   dp[i][j] = max(dp[i-1][j], dp[i][j-1])  (max(↑, ←))
```

---

## Example 5: Edit Distance — 2D Table (LeetCode 72)

```go
package main

import "fmt"

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = i // delete all chars
	}
	for j := 0; j <= n; j++ {
		dp[0][j] = j // insert all chars
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if word1[i-1] == word2[j-1] {
				dp[i][j] = dp[i-1][j-1]
			} else {
				dp[i][j] = 1 + min3(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
			}
		}
	}
	return dp[m][n]
}

func min3(a, b, c int) int {
	if a < b && a < c { return a }
	if b < c { return b }
	return c
}

func main() {
	fmt.Println(minDistance("horse", "ros"))         // 3
	fmt.Println(minDistance("intention", "execution")) // 5
}
```

**Textual Figure:**

```
Edit Distance Table: word1 = "horse", word2 = "ros"

         ""   r    o    s
    ┌────┬────┬────┬────┐
 "" │  0 │  1 │  2 │  3 │  ← insert all
    ├────┼────┼────┼────┤
  h │  1 │  1 │  2 │  3 │  h≠r → 1+min(↑0,←1,↖0) = 1
    ├────┼────┼────┼────┤
  o │  2 │  2 │  1 │  2 │  o==o → ↖1 = 1
    ├────┼────┼────┼────┤
  r │  3 │  2 │  2 │  2 │  r==r → ↖2 (col r)
    ├────┼────┼────┼────┤
  s │  4 │  3 │  3 │  2 │  s==s → ↖2 = 2
    ├────┼────┼────┼────┤
  e │  5 │  4 │  4 │ [3]│  e≠s → 1+min(↑2,←4,↖3) = 3
    └────┴────┴────┴────┘

  Answer: dp[5][3] = 3

  Transition:
    match:    dp[i][j] = dp[i-1][j-1]          (no cost)
    mismatch: dp[i][j] = 1 + min(
                dp[i-1][j],    ← delete
                dp[i][j-1],    ← insert
                dp[i-1][j-1])  ← replace
```

---

## Example 6: 0/1 Knapsack — 2D Table

```go
package main

import "fmt"

func knapsack(weights, values []int, W int) int {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, W+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= W; w++ {
			dp[i][w] = dp[i-1][w] // skip item
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

         w=0  w=1  w=2  w=3  w=4  w=5  w=6  w=7  w=8
       ┌────┬────┬────┬────┬────┬────┬────┬────┬────┐
  i=0  │  0 │  0 │  0 │  0 │  0 │  0 │  0 │  0 │  0 │
       ├────┼────┼────┼────┼────┼────┼────┼────┼────┤
  i=1  │  0 │  0 │  3 │  3 │  3 │  3 │  3 │  3 │  3 │ item1(w=2,v=3)
       ├────┼────┼────┼────┼────┼────┼────┼────┼────┤
  i=2  │  0 │  0 │  3 │  4 │  4 │  7 │  7 │  7 │  7 │ item2(w=3,v=4)
       ├────┼────┼────┼────┼────┼────┼────┼────┼────┤
  i=3  │  0 │  0 │  3 │  4 │  5 │  7 │  8 │  9 │  9 │ item3(w=4,v=5)
       ├────┼────┼────┼────┼────┼────┼────┼────┼────┤
  i=4  │  0 │  0 │  3 │  4 │  5 │  7 │  8 │  9 │[10]│ item4(w=5,v=6)
       └────┴────┴────┴────┴────┴────┴────┴────┴────┘

  dp[4][8] = max(dp[3][8], dp[3][3]+6) = max(9, 4+6) = 10
  Answer: 10 (items 2+4: w=3+5=8, v=4+6=10)
```

---

## Example 7: Unique Paths — 2D Table (LeetCode 62)

```go
package main

import "fmt"

func uniquePaths(m, n int) int {
	dp := make([][]int, m)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = 1 }
	}

	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[m-1][n-1]
}

func main() {
	fmt.Println(uniquePaths(3, 7)) // 28
	fmt.Println(uniquePaths(3, 2)) // 3
}
```

**Textual Figure:**

```
Unique Paths: m=3, n=4

       c=0   c=1   c=2   c=3
     ┌─────┬─────┬─────┬─────┐
r=0  │  1  │  1  │  1  │  1  │  ← all 1 (only right)
     ├─────┼─────┼─────┼─────┤
r=1  │  1  │  2  │  3  │  4  │  dp[1][1] = ↑ 1 + ← 1 = 2
     ├─────┼─────┼─────┼─────┤
r=2  │  1  │  3  │  6  │ [10] │  dp[2][3] = ↑ 4 + ← 6 = 10
     └─────┴─────┴─────┴─────┘

  Transition: dp[i][j] = dp[i-1][j] + dp[i][j-1]
                          ↑ from above  ← from left

  Fill order: row by row, left to right
```

---

## Example 8: Longest Palindromic Subsequence — 2D Table (LeetCode 516)

```go
package main

import "fmt"

func longestPalinSubseq(s string) int {
	n := len(s)
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		dp[i][i] = 1
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			if s[i] == s[j] {
				dp[i][j] = dp[i+1][j-1] + 2
			} else {
				if dp[i+1][j] > dp[i][j-1] {
					dp[i][j] = dp[i+1][j]
				} else {
					dp[i][j] = dp[i][j-1]
				}
			}
		}
	}
	return dp[0][n-1]
}

func main() {
	fmt.Println(longestPalinSubseq("bbbab"))   // 4 ("bbbb")
	fmt.Println(longestPalinSubseq("cbbd"))    // 2 ("bb")
}
```

**Textual Figure:**

```
Longest Palindromic Subsequence: s = "bbbab"

         b    b    b    a    b
    ┌────┬────┬────┬────┬────┐
  b │  1 │  2 │  3 │  3 │ [4]│  i=0
    ├────┼────┼────┼────┼────┤
  b │    │  1 │  2 │  2 │  3 │  i=1
    ├────┼────┼────┼────┼────┤
  b │    │    │  1 │  1 │  2 │  i=2
    ├────┼────┼────┼────┼────┤
  a │    │    │    │  1 │  1 │  i=3
    ├────┼────┼────┼────┼────┤
  b │    │    │    │    │  1 │  i=4
    └────┴────┴────┴────┴────┘

  Fill order: by increasing length (diagonals)
  len=1: dp[i][i] = 1  (base)
  len=2: dp[0][1] = 2  (b==b → dp[1][0]+2)
  len=3: dp[0][2] = 3  (b==b → dp[1][1]+2)
  len=5: dp[0][4] = 4  (b==b → dp[1][3]+2)

  Transition:
    s[i]==s[j]:  dp[i][j] = dp[i+1][j-1] + 2
    s[i]≠s[j]:  dp[i][j] = max(dp[i+1][j], dp[i][j-1])
```

---

## Example 9: Tabulation Order Matters

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Tabulation: Filling Order ===")
	fmt.Println()

	orders := []struct{ problem, order, reason string }{
		{"Fibonacci", "i = 2 → n", "dp[i] depends on dp[i-1] and dp[i-2]"},
		{"LCS", "i = 1→m, j = 1→n", "dp[i][j] depends on dp[i-1][j-1], dp[i-1][j], dp[i][j-1]"},
		{"LPS", "length = 2→n", "dp[i][j] depends on dp[i+1][j-1] (inner diagonal)"},
		{"Knapsack", "i = 1→n, w = 0→W", "dp[i][w] depends on dp[i-1][*]"},
		{"MCM", "length = 2→n", "dp[i][j] depends on dp[i][k] and dp[k+1][j] for k < j"},
		{"Edit Distance", "i = 1→m, j = 1→n", "dp[i][j] depends on dp[i-1][*] and dp[i][j-1]"},
	}

	for _, o := range orders {
		fmt.Printf("%-18s fill: %-18s because %s\n", o.problem, o.order, o.reason)
	}

	fmt.Println()
	fmt.Println("Rule: Process states in topological order of dependencies.")
	fmt.Println("When dp[i][j] depends on dp[i-1][*], fill row by row.")
	fmt.Println("When dp[i][j] depends on dp[i+1][*], fill bottom to top or by length.")
}
```

**Textual Figure:**

```
Tabulation Fill Orders:

1. Forward Linear (Fibonacci, Climbing Stairs):
   ┌───┬───┬───┬───┬───┬───┐
   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │    i = 2 → n
   └───┴───┴───┴───┴───┴───┘
   ==========────→

2. Row-by-Row (LCS, Edit Distance, Knapsack):
   ┌───┬───┬───┐
   │ → │ → │ → │  row 0
   ├───┼───┼───┤
   │ → │ → │ → │  row 1
   ├───┼───┼───┤
   │ → │ → │ → │  row 2
   └───┴───┴───┘

3. By Diagonal Length (LPS, MCM):
   ┌───┬───┬───┬───┐
   │ 1ₒ │ 2ₐ │ 3ₐ │ 4ₐ │  diag 1 (base) → 2 → 3 → 4
   ├───┼───┼───┼───┤
   │    │ 1ₒ │ 2ₐ │ 3ₐ │
   ├───┼───┼───┼───┤
   │    │    │ 1ₒ │ 2ₐ │  subscript: fill-pass number
   ├───┼───┼───┼───┤
   │    │    │    │ 1ₒ │
   └───┴───┴───┴───┘

  Key: Always fill so all dependencies are computed first.
```

---

## Example 10: Tabulation Building Blocks Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Tabulation Summary ===")
	fmt.Println()

	steps := []struct{ step, description string }{
		{"1. Define state", "What does dp[i] or dp[i][j] represent?"},
		{"2. Base cases", "What are the trivial answers? (dp[0], dp[0][0], etc.)"},
		{"3. Recurrence", "How does dp[i][j] relate to smaller subproblems?"},
		{"4. Fill order", "What order ensures all dependencies are ready?"},
		{"5. Answer", "Where is the final answer? dp[n]? dp[n][m]? max(dp[*])?"},
		{"6. Optimize", "Can we reduce space? (rolling array, two variables)"},
	}

	for _, s := range steps {
		fmt.Printf("%-20s %s\n", s.step, s.description)
	}

	fmt.Println()
	fmt.Println("Common table shapes:")
	fmt.Println("  1D: dp[n+1]         — Fibonacci, climbing stairs, coin change")
	fmt.Println("  2D: dp[m+1][n+1]    — LCS, edit distance, knapsack")
	fmt.Println("  2D: dp[n][n]        — LPS, MCM, interval DP")
	fmt.Println("  3D: dp[n][n][k]     — rare; DP with extra parameter")
}
```

**Textual Figure:**

```
Tabulation Building Blocks — 6-Step Process:

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 1. Define    │──→│ 2. Base      │──→│ 3. Recurrence │
  │    State    │   │    Cases    │   │              │
  └──────────────┘   └──────────────┘   └───────┬──────┘
                                              │
  ┌──────────────┐   ┌──────────────┐   ┌───────┴──────┐
  │ 6. Optimize  │──←│ 5. Answer    │──←│ 4. Fill      │
  │    Space    │   │    Location │   │    Order     │
  └──────────────┘   └──────────────┘   └──────────────┘

  Table shapes:
  ┌─────────────┐   ┌─┬─┬─┬─┐       ┌─┬─┬─┬─┐
  │┌─┬─┬─┬─┬─┬─┐│   ├─┼─┼─┼─┤       ├─┼─┼─┼─┤
  │└─┴─┴─┴─┴─┴─┘│   ├─┼─┼─┼─┤       ├─┼─┼─┼─┤
  └─────────────┘   └─┴─┴─┴─┘       └─┴─┴─┴─┘
    1D: dp[n+1]      2D: dp[m+1][n+1]  2D: dp[n][n]
```

---

## Key Takeaways

1. Tabulation fills a table iteratively from base cases → no recursion, no stack overflow
2. Must determine correct fill order based on state dependencies
3. 1D tables for sequence problems, 2D for two-sequence or interval problems
4. All states are computed — even unreachable ones (unlike memoization)
5. Easier to apply space optimization (rolling array) compared to memoization

> **Next up:** State Transitions →
