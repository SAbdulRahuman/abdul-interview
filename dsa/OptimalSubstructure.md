# Phase 15: Dynamic Programming — Optimal Substructure

## Overview

A problem has **optimal substructure** if the optimal solution contains optimal solutions to its subproblems. This is the second key property (along with overlapping subproblems) that makes DP applicable.

| Concept | Meaning |
|---------|---------|
| **Optimal substructure** | Best solution = best sub-solutions combined |
| **Why it matters** | Guarantees we can build optimal answer from smaller optimal answers |
| **How to prove** | Show that using a non-optimal sub-solution would contradict optimality |
| **Counterexample** | Longest simple path: optimal sub-paths may not compose |

---

## Example 1: Shortest Path Has Optimal Substructure

```go
package main

import (
	"fmt"
	"math"
)

// Shortest path from 0 to n-1 in a DAG
// Optimal substructure: shortest(0→n) = min(shortest(0→k) + w(k,n))

func shortestPathDAG(adj [][][2]int, n int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[0] = 0

	// Process in topological order (0, 1, 2, ..., n-1 for a simple DAG)
	for u := 0; u < n; u++ {
		if dist[u] == math.MaxInt32 { continue }
		for _, edge := range adj[u] {
			v, w := edge[0], edge[1]
			if dist[u]+w < dist[v] {
				dist[v] = dist[u] + w // Optimal substructure:
				// dist[v] builds on optimal dist[u]
			}
		}
	}
	return dist
}

func main() {
	// DAG: 0→1(2), 0→2(4), 1→2(1), 1→3(7), 2→3(3)
	adj := [][][2]int{
		{{1, 2}, {2, 4}},
		{{2, 1}, {3, 7}},
		{{3, 3}},
		{},
	}

	dist := shortestPathDAG(adj, 4)
	for i, d := range dist {
		fmt.Printf("Shortest path to %d: %d\n", i, d)
	}
	// 0→1→2→3 = 2+1+3 = 6 (optimal sub-paths: 0→1=2, 0→2=3)
}
```

**Textual Figure:**

```
Shortest Path DAG — Optimal Substructure:

  Graph: 0─▶2─▷2(w=2)  0─▶2─▷4(w=4)  1─▶2─▷1(w=1)
         1─▶3─▷7(w=7)  2─▶3─▷3(w=3)

        0 ──▷2── 1 ──▷7── 3
        │           │         ↑
        └──▷4── 2 ─▷1╦──▷3──┘
                    ║
          1 ──▷1──╝

  Processing (topological order):
  ┌──────┬──────┬─────────────────────────────┐
  │ Node │ dist │ Explanation                 │
  ├──────┼──────┼─────────────────────────────┤
  │  0   │   0  │ source                      │
  │  1   │   2  │ 0→1 = 2                     │
  │  2   │   3  │ min(0→2=4, 0→1→2=2+1=3)   │
  │  3   │   6  │ min(1→3=2+7, 2→3=3+3=6)   │
  └──────┴──────┴─────────────────────────────┘

  Optimal substructure: shortest(0→3) uses shortest(0→2)=3,
  then shortest(0→2) uses shortest(0→1)=2. Each sub-path is optimal.

  Path: 0 ─▷2─→ 1 ─▷1─→ 2 ─▷3─→ 3  (total = 6)
```

---

## Example 2: Longest Increasing Subsequence (LeetCode 300)

```go
package main

import "fmt"

// LIS has optimal substructure:
// LIS ending at i = 1 + max(LIS ending at j) for all j < i where nums[j] < nums[i]

func lengthOfLIS(nums []int) int {
	n := len(nums)
	dp := make([]int, n)
	for i := range dp { dp[i] = 1 }

	best := 1
	for i := 1; i < n; i++ {
		for j := 0; j < i; j++ {
			if nums[j] < nums[i] && dp[j]+1 > dp[i] {
				dp[i] = dp[j] + 1
			}
		}
		if dp[i] > best { best = dp[i] }
	}
	return best
}

func main() {
	tests := [][]int{
		{10, 9, 2, 5, 3, 7, 101, 18},
		{0, 1, 0, 3, 2, 3},
		{7, 7, 7, 7, 7, 7, 7},
	}

	for _, nums := range tests {
		fmt.Printf("LIS of %v = %d\n", nums, lengthOfLIS(nums))
	}
}
```

**Textual Figure:**

```
LIS for [10, 9, 2, 5, 3, 7, 101, 18]:

  nums:  10   9   2   5   3   7  101  18
  dp:     1   1   1   2   2   3   4    4

  Step-by-step:
  ┌─────┬───────┬────┬─────────────────────────────┐
  │  i  │ nums[i]│ dp │ Best LIS ending at i        │
  ├─────┼───────┼────┼─────────────────────────────┤
  │  0  │  10   │  1 │ [10]                        │
  │  1  │   9   │  1 │ [9]  (9<10, can't extend)   │
  │  2  │   2   │  1 │ [2]                          │
  │  3  │   5   │  2 │ [2,5] (extend dp[2])        │
  │  4  │   3   │  2 │ [2,3] (extend dp[2])        │
  │  5  │   7   │  3 │ [2,5,7] or [2,3,7]          │
  │  6  │ 101   │  4 │ [2,5,7,101] ← best           │
  │  7  │  18   │  4 │ [2,5,7,18]                   │
  └─────┴───────┴────┴─────────────────────────────┘

  Optimal substructure: LIS ending at i = 1 + max(LIS ending at j)
  for all j < i where nums[j] < nums[i].
  dp[6] = 1 + dp[5] = 1 + 3 = 4 (uses optimal dp[5])

  Result: LIS length = 4
```

---

## Example 3: Longest Common Subsequence (LeetCode 1143)

```go
package main

import "fmt"

// LCS has optimal substructure:
// If s1[i]==s2[j]: LCS(i,j) = 1 + LCS(i-1,j-1)
// Else: LCS(i,j) = max(LCS(i-1,j), LCS(i,j-1))
// Optimal sub-solutions compose into optimal solution

func longestCommonSubsequence(text1, text2 string) int {
	m, n := len(text1), len(text2)
	dp := make([][]int, m+1)
	for i := range dp { dp[i] = make([]int, n+1) }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if text1[i-1] == text2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[m][n]
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	tests := [][2]string{
		{"abcde", "ace"},
		{"abc", "abc"},
		{"abc", "def"},
	}

	for _, t := range tests {
		fmt.Printf("LCS(\"%s\", \"%s\") = %d\n",
			t[0], t[1], longestCommonSubsequence(t[0], t[1]))
	}
}
```

**Textual Figure:**

```
LCS DP Table for "abcde" vs "ace":

  Recurrence:
    match:    dp[i][j] = dp[i-1][j-1] + 1
    mismatch: dp[i][j] = max(dp[i-1][j], dp[i][j-1])

        │  "" │  a  │  c  │  e  │
   ─────┼────┼────┼────┼────┤
    ""  │  0 │  0 │  0 │  0 │
    a   │  0 │ ↗1 │  1 │  1 │  a==a → diagonal+1
    b   │  0 │  1 │  1 │  1 │  b≠c → max(top,left)
    c   │  0 │  1 │ ↗2 │  2 │  c==c → diagonal+1
    d   │  0 │  1 │  2 │  2 │  d≠e → max(top,left)
    e   │  0 │  1 │  2 │ ↗3 │  e==e → diagonal+1

  Optimal substructure decomposition:
    LCS("abcde","ace") = 1 + LCS("abcd","ac")    [↗ match 'e']
    LCS("abcd","ac")   = 1 + LCS("ab","a")       [↗ match 'c']
    LCS("ab","a")      = 1 + LCS("","")          [↗ match 'a']

  Result: LCS = 3 ("ace")
```

---

## Example 4: Edit Distance (LeetCode 72)

```go
package main

import "fmt"

// Edit distance has optimal substructure:
// edit(i,j) = min of:
//   edit(i-1,j) + 1     (delete from s1)
//   edit(i,j-1) + 1     (insert into s1)
//   edit(i-1,j-1) + cost (replace: 0 if match, 1 if not)

func minDistance(word1, word2 string) int {
	m, n := len(word1), len(word2)
	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
		dp[i][0] = i
	}
	for j := 0; j <= n; j++ { dp[0][j] = j }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			cost := 1
			if word1[i-1] == word2[j-1] { cost = 0 }

			dp[i][j] = min3(
				dp[i-1][j]+1,     // delete
				dp[i][j-1]+1,     // insert
				dp[i-1][j-1]+cost, // replace
			)
		}
	}
	return dp[m][n]
}

func min3(a, b, c int) int {
	if a < b { if a < c { return a }; return c }
	if b < c { return b }; return c
}

func main() {
	tests := [][2]string{
		{"horse", "ros"},
		{"intention", "execution"},
		{"", "abc"},
	}

	for _, t := range tests {
		fmt.Printf("edit(\"%s\", \"%s\") = %d\n",
			t[0], t[1], minDistance(t[0], t[1]))
	}
}
```

**Textual Figure:**

```
Edit Distance DP Table for "horse" → "ros":

  Recurrence:
    match:    dp[i][j] = dp[i-1][j-1]       (no op)
    mismatch: dp[i][j] = 1 + min(del, ins, rep)

          │  ""  │  r   │  o   │  s   │
   ──────┼─────┼─────┼─────┼─────┤
    ""    │  0  │  1  │  2  │  3  │  ← insert all
    h     │  1  │  1  │  2  │  3  │  h≠r → 1+min(0,1,1)=1
    o     │  2  │  2  │  1  │  2  │  o==o → dp[1][1]=1
    r     │  3  │  2  │  2  │  2  │  r==r → dp[1][0]+0? No: dp[2][1]=2
    s     │  4  │  3  │  3  │  2  │  s==s → dp[3][2]=2
    e     │  5  │  4  │  4  │ [3] │  e≠s → 1+min(2,3,2)=3
     ↑
   delete all

  Optimal substructure: edit("horse","ros") builds on
    edit("hors","ro")=2, edit("horse","ro")=3, edit("hors","ros")=3
    → 1 + min(3, 3, 2) = 3

  Operations: horse → rorse (replace h→r) → rose (delete r) → ros (delete e)
  Result: 3 edits
```

---

## Example 5: Rod Cutting — Classic Optimal Substructure

```go
package main

import "fmt"

// Rod cutting: given rod of length n and prices for each length,
// find maximum revenue from cutting
// Optimal substructure: revenue(n) = max(price[i] + revenue(n-i))

func cutRod(prices []int) int {
	n := len(prices)
	dp := make([]int, n+1)

	for length := 1; length <= n; length++ {
		for cut := 1; cut <= length; cut++ {
			val := prices[cut-1] + dp[length-cut]
			if val > dp[length] { dp[length] = val }
		}
	}
	return dp[n]
}

func main() {
	prices := []int{1, 5, 8, 9, 10, 17, 17, 20}
	fmt.Println("Max revenue for rod lengths:")
	for i := 1; i <= len(prices); i++ {
		fmt.Printf("  Length %d: %d\n", i, cutRod(prices[:i]))
	}
}
```

**Textual Figure:**

```
Rod Cutting — prices=[1,5,8,9,10,17,17,20]:

  dp[len] = max over all cuts (price[cut] + dp[len-cut])

  ┌────────┬───────┬────┬────────────────────────┐
  │ Length │ Price │ dp │ Best cut combination   │
  ├────────┼───────┼────┼────────────────────────┤
  │   1    │   1   │  1 │ [1]                    │
  │   2    │   5   │  5 │ [2]  (5 > 1+1)         │
  │   3    │   8   │  8 │ [3]  (8 > 5+1, 1+5)    │
  │   4    │   9   │ 10 │ [2+2] (5+5=10 > 9)     │
  │   5    │  10   │ 13 │ [2+3] (5+8=13 > 10)    │
  │   6    │  17   │ 17 │ [6]  (17 ≥ 5+8+1,...)  │
  │   7    │  17   │ 18 │ [1+6] (1+17=18 > 17)   │
  │   8    │  20   │ 22 │ [2+6] (5+17=22 > 20)   │
  └────────┴───────┴────┴────────────────────────┘

  Optimal substructure for dp[4]=10:
    dp[4] = max(price[1]+dp[3], price[2]+dp[2],
                price[3]+dp[1], price[4]+dp[0])
          = max(1+8, 5+5, 8+1, 9+0) = max(9,10,9,9) = 10
    Uses optimal dp[2]=5 → optimal sub-solution!
```

---

## Example 6: Matrix Chain Multiplication

```go
package main

import (
	"fmt"
	"math"
)

// Optimal substructure: MCM(i,j) = min over k of:
//   MCM(i,k) + MCM(k+1,j) + dims[i-1]*dims[k]*dims[j]

func matrixChainOrder(dims []int) int {
	n := len(dims) - 1
	dp := make([][]int, n)
	for i := range dp { dp[i] = make([]int, n) }

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			dp[i][j] = math.MaxInt32

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
	fmt.Println("Minimum multiplications:", matrixChainOrder(dims)) // 4500

	dims2 := []int{40, 20, 30, 10, 30}
	fmt.Println("Minimum multiplications:", matrixChainOrder(dims2)) // 26000
}
```

**Textual Figure:**

```
Matrix Chain Multiplication: dims=[10,30,5,60]
Matrices: A1(10x30), A2(30x5), A3(5x60)

  dp[i][j] = min cost to multiply matrices i..j

  DP Table (upper triangle):
          j=0    j=1     j=2
  i=0  │   0  │ 1500  │ 4500  │
  i=1  │      │   0   │ 9000  │
  i=2  │      │       │   0   │

  Computation:
  dp[0][1] = 10*30*5 = 1500             (A1·A2)
  dp[1][2] = 30*5*60 = 9000             (A2·A3)
  dp[0][2] = min(
    dp[0][0]+dp[1][2]+10*30*60 = 0+9000+18000 = 27000  (A1)·(A2·A3)
    dp[0][1]+dp[2][2]+10*5*60  = 1500+0+3000  = 4500   (A1·A2)·(A3)
  ) = 4500 ✓

  Optimal substructure:
    Best(A1·A2·A3) at k=1: Best(A1·A2) + Best(A3) + join cost
    = 1500 + 0 + 3000 = 4500
    Uses optimal dp[0][1]=1500!

  Parenthesization: (A1 × A2) × A3 = 4500 multiplications
```

---

## Example 7: Problem WITHOUT Optimal Substructure

```go
package main

import "fmt"

// Longest Simple Path does NOT have optimal substructure!
// The longest path from A→C via B doesn't mean
// using the longest A→B and longest B→C paths
// (they might share vertices → not simple)

func main() {
	fmt.Println("=== Longest Simple Path: NO Optimal Substructure ===")
	fmt.Println()
	fmt.Println("Graph: A - B - C - D, with edge A-C")
	fmt.Println()
	fmt.Println("Longest simple path A → D:")
	fmt.Println("  A → C → B → ... but B doesn't connect to D directly")
	fmt.Println("  A → B → C → D = length 3 (optimal)")
	fmt.Println()
	fmt.Println("If we split at B:")
	fmt.Println("  Longest A→B = A→C→B (length 2)")
	fmt.Println("  Longest B→D = B→C→D (length 2)")
	fmt.Println("  Combined: A→C→B→C→D — visits C twice! NOT SIMPLE")
	fmt.Println()
	fmt.Println("This is why longest simple path is NP-hard.")
	fmt.Println("Optimal sub-paths may conflict (share vertices).")
}
```

**Textual Figure:**

```
Longest Simple Path — NO Optimal Substructure:

  Graph:  A ─── B ─── C ─── D
          │           │
          └───────────┘

  Longest simple A→D: A → B → C → D  (length 3) ✓

  If we split at B:
  ┌───────────────────────┬───────────────────────┐
  │ Longest A→B:           │ Longest B→D:           │
  │ A → C → B (length 2)   │ B → C → D (length 2)   │
  └───────────────────────┴───────────────────────┘

  Combined: A → C → B → C → D
                   ↑       ↑
                   C appears TWICE! Not a simple path!

  ✔ Shortest path:  HAS optimal substructure
  ✖ Longest simple: NO optimal substructure (NP-hard)

  The sub-paths conflict by sharing vertex C.
```

---

## Example 8: 0/1 Knapsack

```go
package main

import "fmt"

// 0/1 Knapsack has optimal substructure:
// opt(i, w) = max(opt(i-1, w), opt(i-1, w-wt[i]) + val[i])

func knapsack(weights, values []int, capacity int) int {
	n := len(weights)
	dp := make([][]int, n+1)
	for i := range dp { dp[i] = make([]int, capacity+1) }

	for i := 1; i <= n; i++ {
		for w := 0; w <= capacity; w++ {
			dp[i][w] = dp[i-1][w] // don't take item i
			if weights[i-1] <= w {
				take := dp[i-1][w-weights[i-1]] + values[i-1]
				if take > dp[i][w] { dp[i][w] = take }
			}
		}
	}
	return dp[n][capacity]
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	capacity := 8

	fmt.Println("Max value:", knapsack(weights, values, capacity)) // 10
}
```

**Textual Figure:**

```
0/1 Knapsack: weights=[2,3,4,5], values=[3,4,5,6], W=8

  dp[i][w] = max(dp[i-1][w], dp[i-1][w-wt[i]] + val[i])

          w=  0   1   2   3   4   5   6   7   8
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  i=0   │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │  (no items)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=1   │ 0 │ 0 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │ 3 │  item0(w=2,v=3)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=2   │ 0 │ 0 │ 3 │ 4 │ 4 │ 7 │ 7 │ 7 │ 7 │  item1(w=3,v=4)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=3   │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │ 9 │  item2(w=4,v=5)
        ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  i=4   │ 0 │ 0 │ 3 │ 4 │ 5 │ 7 │ 8 │ 9 │[10]│  item3(w=5,v=6)
        └───┴───┴───┴───┴───┴───┴───┴───┴───┘

  dp[4][8] = max(dp[3][8], dp[3][3]+6) = max(9, 4+6) = 10
  Optimal substructure: dp[4][8] uses optimal dp[3][3]=4

  Result: 10 (items 1 and 3: w=3+5=8, v=4+6=10)
```

---

## Example 9: Proving Optimal Substructure (Cut-and-Paste)

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Cut-and-Paste Proof Technique ===")
	fmt.Println()
	fmt.Println("To prove optimal substructure:")
	fmt.Println()
	fmt.Println("1. Assume we have an optimal solution S* to the problem")
	fmt.Println("2. Identify the subproblem within S*")
	fmt.Println("3. Suppose the sub-solution is NOT optimal")
	fmt.Println("4. 'Cut' the non-optimal sub-solution out")
	fmt.Println("5. 'Paste' in the truly optimal sub-solution")
	fmt.Println("6. Show the new solution is strictly better")
	fmt.Println("7. This contradicts S* being optimal → contradiction!")
	fmt.Println("8. Therefore, sub-solutions must be optimal ✓")
	fmt.Println()
	fmt.Println("Example with shortest path:")
	fmt.Println("  Optimal path: u → ... → v → ... → w")
	fmt.Println("  Subpath u → v must be shortest")
	fmt.Println("  If not, replace with shorter u → v path")
	fmt.Println("  Total path becomes shorter → contradicts optimality")
}
```

**Textual Figure:**

```
Cut-and-Paste Proof — Shortest Path Example:

  Given: Optimal path P* from u to w via v:

    u ───P1*───▶ v ───P2*───▶ w
    │───── d1 ─────│───── d2 ─────│
    total cost = d1 + d2

  Suppose P1* is NOT the shortest u→v path.
  Then ∃ a shorter path P1' with cost d1' < d1:

    u ───P1'───▶ v ───P2*───▶ w
    │───── d1'─────│───── d2 ─────│
    total cost = d1' + d2 < d1 + d2  ← CONTRADICTION!

  Step 1: Assume optimal solution P*
  Step 2: Identify sub-solution P1*
  Step 3: Suppose P1* is not optimal
  Step 4: CUT P1* from P*
  Step 5: PASTE optimal P1' in its place
  Step 6: New path is shorter → contradicts P* being optimal
  Step 7: ∴ P1* must be optimal. QED ✓
```

---

## Example 10: Optimal Substructure Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Optimal Substructure Summary ===")
	fmt.Println()

	has := []struct{ problem, recurrence string }{
		{"Shortest Path", "dist(v) = min(dist(u) + w(u,v))"},
		{"LIS", "LIS(i) = 1 + max(LIS(j) : j<i, a[j]<a[i])"},
		{"LCS", "LCS(i,j) = 1+LCS(i-1,j-1) or max(...)"},
		{"Edit Distance", "ed(i,j) = min(ed(i-1,j), ed(i,j-1), ed(i-1,j-1))+cost"},
		{"Rod Cutting", "rev(n) = max(price[i] + rev(n-i))"},
		{"Knapsack", "opt(i,w) = max(skip, take)"},
		{"MCM", "MCM(i,j) = min_k(MCM(i,k) + MCM(k+1,j) + cost)"},
	}

	doesNot := []struct{ problem, reason string }{
		{"Longest simple path", "Sub-paths may share vertices"},
		{"Cheapest simple cycle", "Sub-cycles may overlap"},
		{"Unweighted longest path", "NP-hard, no polynomial decomposition"},
	}

	fmt.Println("Problems WITH optimal substructure:")
	for _, p := range has {
		fmt.Printf("  %-18s %s\n", p.problem, p.recurrence)
	}

	fmt.Println("\nProblems WITHOUT optimal substructure:")
	for _, p := range doesNot {
		fmt.Printf("  %-25s %s\n", p.problem, p.reason)
	}
}
```

**Textual Figure:**

```
Optimal Substructure Taxonomy:

  ┌────────────────────────────────────────────────┐
  │            HAS Optimal Substructure            │
  ├────────────────┬───────────────┬───────────────┤
  │ Shortest Path  │ LIS / LCS       │ Knapsack      │
  │ Edit Distance  │ Rod Cutting     │ MCM           │
  └────────────────┴───────────────┴───────────────┘
         │                  │                │
    dist(v) =           LIS(i) =          opt(i,w) =
    min(dist(u)+w)    1+max(LIS(j))     max(skip, take)

  ┌────────────────────────────────────────────────┐
  │         NO Optimal Substructure              │
  ├────────────────┬───────────────┬───────────────┤
  │ Longest simple │ Cheapest       │ Unweighted    │
  │ path           │ simple cycle   │ longest path  │
  └────────────────┴───────────────┴───────────────┘
    Sub-paths may share vertices → NP-hard!
```

---

## Key Takeaways

1. Optimal substructure: the optimal solution to the whole contains optimal solutions to subproblems
2. Prove it via cut-and-paste: replacing a sub-solution with a better one would improve the whole
3. Both overlapping subproblems AND optimal substructure must hold for DP to work
4. Longest SIMPLE path does NOT have optimal substructure — this is why it's NP-hard
5. The recurrence relation directly encodes the optimal substructure

> **Next up:** Memoization →
