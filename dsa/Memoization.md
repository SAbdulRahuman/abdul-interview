# Phase 15: Dynamic Programming — Memoization

## Overview

**Memoization** is the top-down DP approach: solve problems recursively but cache results to avoid recomputation. It's often the easiest way to add DP to an existing recursive solution.

| Aspect | Detail |
|--------|--------|
| **Approach** | Top-down recursive with caching |
| **When computed** | Lazily — only when needed |
| **Storage** | Map or array indexed by state |
| **Pros** | Easy to implement from recursion; only computes reachable states |
| **Cons** | Recursion overhead; stack overflow risk for deep recursion |

---

## Example 1: Basic Memoization with Map

```go
package main

import "fmt"

func fib(n int, memo map[int]int) int {
	if n <= 1 { return n }
	if v, ok := memo[n]; ok { return v }

	memo[n] = fib(n-1, memo) + fib(n-2, memo)
	return memo[n]
}

func main() {
	memo := map[int]int{}
	for i := 0; i <= 15; i++ {
		fmt.Printf("fib(%2d) = %d\n", i, fib(i, memo))
	}
}
```

**Textual Figure:**

```
Memoization Call Tree for fib(5):

                      fib(5)
                     ╱      ╲
                fib(4)      fib(3) ← CACHE HIT ✓
               ╱      ╲
          fib(3)      fib(2) ← CACHE HIT ✓
         ╱      ╲
    fib(2)      fib(1)=1
   ╱      ╲
fib(1)=1  fib(0)=0

Memo Map After fib(5):
┌─────────┬───────┬────────────────────────────────┐
│   Key   │ Value │ How Computed                     │
├─────────┼───────┼────────────────────────────────┤
│ memo[2] │   1   │ fib(1)+fib(0) = 1+0              │
│ memo[3] │   2   │ fib(2)+fib(1) = 1+1              │
│ memo[4] │   3   │ fib(3)+fib(2) = 2+1 (cache hit)  │
│ memo[5] │   5   │ fib(4)+fib(3) = 3+2 (cache hit)  │
└─────────┴───────┴────────────────────────────────┘

Only 6 unique calls instead of 15 naive recursive calls!
```

---

## Example 2: Memoization with Array (Faster)

```go
package main

import "fmt"

func fib(n int, memo []int) int {
	if n <= 1 { return n }
	if memo[n] != -1 { return memo[n] }

	memo[n] = fib(n-1, memo) + fib(n-2, memo)
	return memo[n]
}

func main() {
	n := 40
	memo := make([]int, n+1)
	for i := range memo { memo[i] = -1 }

	fmt.Printf("fib(%d) = %d\n", n, fib(n, memo))
}
```

**Textual Figure:**

```
Array-Based Memo for fib(7):

Index:    0    1    2    3    4    5    6    7
        ┌────┬────┬────┬────┬────┬────┬────┬────┐
Init:   │ -1 │ -1 │ -1 │ -1 │ -1 │ -1 │ -1 │ -1 │  (-1 = not computed)
        └────┴────┴────┴────┴────┴────┴────┴────┘

After fib(7) completes (filled top-down):
        ┌────┬────┬────┬────┬────┬────┬────┬────┐
memo:   │  0 │  1 │  1 │  2 │  3 │  5 │  8 │ 13 │
        └────┴────┴────┴────┴────┴────┴────┴────┘
          ↑    ↑    ↑                   ↑     ↑
        base base  1+0                5+3   8+5

Fill order: 0→1→2→3→4→5→6→7 (deepest call returns first)
Array lookup O(1) with better cache locality than map.
```

---

## Example 3: Coin Change Memoized (LeetCode 322)

```go
package main

import "fmt"

func coinChange(coins []int, amount int) int {
	memo := make([]int, amount+1)
	for i := range memo { memo[i] = -2 } // -2 = not computed

	var solve func(amt int) int
	solve = func(amt int) int {
		if amt == 0 { return 0 }
		if amt < 0 { return -1 }
		if memo[amt] != -2 { return memo[amt] }

		best := -1
		for _, c := range coins {
			sub := solve(amt - c)
			if sub >= 0 {
				if best == -1 || sub+1 < best {
					best = sub + 1
				}
			}
		}

		memo[amt] = best
		return best
	}

	return solve(amount)
}

func main() {
	fmt.Println(coinChange([]int{1, 5, 10, 25}, 36))  // 3
	fmt.Println(coinChange([]int{2}, 3))                // -1
	fmt.Println(coinChange([]int{1, 3, 4}, 6))          // 2
}
```

**Textual Figure:**

```
Memoized Call Tree for coinChange([1,3,4], 6):

                          solve(6)
                        ╱    │     ╲
                  coin=1│  coin=3  coin=4╲
                       ╱     │           ╲
                 solve(5)  solve(3)    solve(2)
                ╱  │  ╲    ╱   ╲        ╱
              s(4) s(2) s(1) s(2)↵ s(0) s(1)↵
              ...  ...   1  HIT✓   0   HIT✓

Memo Array (amt → min coins):
┌─────┬──────┬───────────────────┐
│ amt │ memo │ Best combination   │
├─────┼──────┼───────────────────┤
│  0  │   0  │ base case          │
│  1  │   1  │ 1 coin  (1)        │
│  2  │   2  │ 2 coins (1+1)      │
│  3  │   1  │ 1 coin  (3)        │
│  4  │   1  │ 1 coin  (4)        │
│  5  │   2  │ 2 coins (1+4)      │
│  6  │   2  │ 2 coins (3+3) ✓    │
└─────┴──────┴───────────────────┘

Result: 2 coins needed for amount 6
```

---

## Example 4: Grid Minimum Path Sum (LeetCode 64)

```go
package main

import "fmt"

func minPathSum(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	memo := make([][]int, m)
	for i := range memo {
		memo[i] = make([]int, n)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(r, c int) int
	solve = func(r, c int) int {
		if r == m-1 && c == n-1 { return grid[r][c] }
		if r >= m || c >= n { return 1<<31 - 1 }
		if memo[r][c] != -1 { return memo[r][c] }

		right := solve(r, c+1)
		down := solve(r+1, c)

		best := grid[r][c]
		if right < down { best += right } else { best += down }

		memo[r][c] = best
		return best
	}

	return solve(0, 0)
}

func main() {
	grid := [][]int{
		{1, 3, 1},
		{1, 5, 1},
		{4, 2, 1},
	}
	fmt.Println("Min path sum:", minPathSum(grid)) // 7
}
```

**Textual Figure:**

```
Grid Minimum Path Sum — Memoization:

Original Grid:            Memo Table (filled top-down):
┌───┬───┬───┐             ┌───┬───┬───┐
│ 1 │ 3 │ 1 │             │ 7 │ 6 │ 3 │ ← solve(0,0)=7
├───┼───┼───┤             ├───┼───┼───┤
│ 1 │ 5 │ 1 │             │ 8 │ 7 │ 2 │
├───┼───┼───┤             ├───┼───┼───┤
│ 4 │ 2 │ 1 │             │ 7 │ 3 │ 1 │ ← base: solve(2,2)=1
└───┴───┴───┘             └───┴───┴───┘

Optimal Path (cost = 7):
  ┌───┐───┐───┐
  │*1 │ 3 │*1 │     Path: (0,0)→(1,0)→(1,1)→(1,2)→(2,2)
  ├───┼───┼───┤     Cost: 1 + 1 + 5 + 1 + 1 = ... wait
  │*1 │*5 │*1 │     Actually: (0,0)→(0,1)→(0,2)→(1,2)→(2,2)
  ├───┼───┼───┤     Cost: 1 + 3 + 1 + 1 + 1 = 7 ✓
  │ 4 │ 2 │*1 │
  └───┴───┴───┘

Memo avoids recomputing overlapping cells reached
from both right-then-down and down-then-right.
```

---

## Example 5: Longest Common Subsequence — Memoized

```go
package main

import "fmt"

func longestCommonSubsequence(s1, s2 string) int {
	m, n := len(s1), len(s2)
	memo := make([][]int, m)
	for i := range memo {
		memo[i] = make([]int, n)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(i, j int) int
	solve = func(i, j int) int {
		if i == m || j == n { return 0 }
		if memo[i][j] != -1 { return memo[i][j] }

		if s1[i] == s2[j] {
			memo[i][j] = 1 + solve(i+1, j+1)
		} else {
			a := solve(i+1, j)
			b := solve(i, j+1)
			if a > b { memo[i][j] = a } else { memo[i][j] = b }
		}
		return memo[i][j]
	}

	return solve(0, 0)
}

func main() {
	fmt.Println(longestCommonSubsequence("abcde", "ace"))   // 3
	fmt.Println(longestCommonSubsequence("abc", "def"))     // 0
}
```

**Textual Figure:**

```
Memoized LCS("abcde", "ace") — Call Tree:

              solve(0,0)
                  │ s1[0]='a' == s2[0]='a' → match!
             1 + solve(1,1)
                ╱            ╲
  s1[1]='b' ≠ s2[1]='c'      (no match)
    ╱                  ╲
solve(2,1)          solve(1,2)
    │                   │
  'c'=='c' match     'b'≠'e'
    │                 ╱      ╲
1+solve(3,2)     solve(2,2)  solve(1,3)=0
    │                │
  'd'≠'e'        'c'≠'e'
  ╱      ╲        ╱      ╲
s(4,2)  s(3,3)  s(3,2)↵  s(2,3)=0
  │      =0     HIT✓
 'e'=='e'
  │
1+s(5,3)=0

Memo Table (sparse, only reachable states filled):
        j→  0('a')  1('c')  2('e')
  i↓
  0 'a'  │   3    │        │       │
  1 'b'  │        │   2    │   0   │
  2 'c'  │        │   2    │   1   │
  3 'd'  │        │        │   1   │
  4 'e'  │        │        │   1   │

Result: LCS = 3 ("ace")
```

---

## Example 6: Word Break — Memoized (LeetCode 139)

```go
package main

import "fmt"

func wordBreak(s string, wordDict []string) bool {
	wordSet := map[string]bool{}
	for _, w := range wordDict { wordSet[w] = true }

	memo := map[int]bool{}
	computed := map[int]bool{}

	var solve func(start int) bool
	solve = func(start int) bool {
		if start == len(s) { return true }
		if computed[start] { return memo[start] }

		computed[start] = true
		for end := start + 1; end <= len(s); end++ {
			if wordSet[s[start:end]] && solve(end) {
				memo[start] = true
				return true
			}
		}
		memo[start] = false
		return false
	}

	return solve(0)
}

func main() {
	fmt.Println(wordBreak("leetcode", []string{"leet", "code"}))       // true
	fmt.Println(wordBreak("applepenapple", []string{"apple", "pen"}))  // true
	fmt.Println(wordBreak("catsandog", []string{"cats","dog","sand","and","cat"})) // false
}
```

**Textual Figure:**

```
Word Break Memoization for "leetcode", dict=["leet","code"]:

Call Tree:
  solve(0)
    │ try s[0:1]="l"  ✖
    │ try s[0:2]="le" ✖
    │ try s[0:3]="lee" ✖
    │ try s[0:4]="leet" ✔ → solve(4)
    │                        │ try s[4:5]="c"    ✖
    │                        │ try s[4:6]="co"   ✖
    │                        │ try s[4:7]="cod"  ✖
    │                        │ try s[4:8]="code" ✔ → solve(8)
    │                        │                        │ start==len(s) → true!
    │                        └→ return true
    └→ return true

Memo Cache:
┌───────┬───────┬──────────────────┐
│ start │ result │ Reason            │
├───────┼───────┼──────────────────┤
│   8   │  true  │ base case (end)   │
│   4   │  true  │ "code" → solve(8) │
│   0   │  true  │ "leet" → solve(4) │
└───────┴───────┴──────────────────┘

Segmentation: "leet" | "code" → true
```

---

## Example 7: 2D State Memoization — Knapsack

```go
package main

import "fmt"

func knapsack(weights, values []int, capacity int) int {
	n := len(weights)
	memo := make([][]int, n)
	for i := range memo {
		memo[i] = make([]int, capacity+1)
		for j := range memo[i] { memo[i][j] = -1 }
	}

	var solve func(i, w int) int
	solve = func(i, w int) int {
		if i == n || w == 0 { return 0 }
		if memo[i][w] != -1 { return memo[i][w] }

		// Skip item i
		best := solve(i+1, w)

		// Take item i
		if weights[i] <= w {
			take := values[i] + solve(i+1, w-weights[i])
			if take > best { best = take }
		}

		memo[i][w] = best
		return best
	}

	return solve(0, capacity)
}

func main() {
	weights := []int{2, 3, 4, 5}
	values := []int{3, 4, 5, 6}
	fmt.Println("Max value:", knapsack(weights, values, 8)) // 10
}
```

**Textual Figure:**

```
2D Knapsack Memoization: weights=[2,3,4,5], values=[3,4,5,6], W=8

Call Tree (top-down):
              solve(0, 8)
             ╱            ╲
       skip item 0      take item 0 (w=2,v=3)
       solve(1, 8)      solve(1, 6)
      ╱         ╲       ╱         ╲
   s(2,8)   s(2,5)   s(2,6)   s(2,3)
    ...      ...      ...      ...

Memo Table (only reachable states):
        w→  0   1   2   3   4   5   6   7   8
  i↓
  0       │ 0 │   │   │   │   │   │   │   │ 10│
  1       │ 0 │   │   │   │   │   │   │   │ 10│
  2       │ 0 │   │   │ 4 │ 5 │ 5 │ 9 │ 9 │ 10│
  3       │ 0 │   │   │   │   │ 6 │ 6 │   │ 6 │
  4 (end) │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │

Optimal: take items 1(v=4) + 2(v=5) = skip 0,3 → w=3+4=7≤8, val=9
Wait — actually items 0+1+2: w=2+3+4=9>8. Items 1+2: w=7, val=9.
But result is 10: items 0+2+3? w=2+4+5=11>8. Items 1+3: w=3+5=8, val=4+6=10 ✓

Result: max value = 10 (items 1 and 3, weight=8)
```

---

## Example 8: String Key Memoization — Partition

```go
package main

import "fmt"

func minCut(s string) int {
	n := len(s)

	// Precompute palindrome check
	isPalin := make([][]bool, n)
	for i := range isPalin { isPalin[i] = make([]bool, n) }
	for i := n - 1; i >= 0; i-- {
		for j := i; j < n; j++ {
			if s[i] == s[j] && (j-i <= 2 || isPalin[i+1][j-1]) {
				isPalin[i][j] = true
			}
		}
	}

	memo := make([]int, n)
	for i := range memo { memo[i] = -1 }

	var solve func(start int) int
	solve = func(start int) int {
		if start == n { return -1 }
		if memo[start] != -1 { return memo[start] }

		best := n - start // worst case: cut every character
		for end := start; end < n; end++ {
			if isPalin[start][end] {
				cuts := 1 + solve(end+1)
				if cuts < best { best = cuts }
			}
		}

		memo[start] = best
		return best
	}

	return solve(0)
}

func main() {
	fmt.Println(minCut("aab"))  // 1
	fmt.Println(minCut("a"))    // 0
	fmt.Println(minCut("ab"))   // 1
}
```

**Textual Figure:**

```
Palindrome Partition Min Cut for "aab":

Step 1 — isPalin table (s[i..j] palindrome?):
          j→  0('a')  1('a')  2('b')
  i↓
  0 'a'       true    true    false
  1 'a'                true   false
  2 'b'                        true

Step 2 — Memoized calls:
  solve(0): try palindromes starting at 0
    │ s[0..0]="a" is palindrome → 1 + solve(1)
    │   solve(1): try palindromes starting at 1
    │     │ s[1..1]="a" → 1 + solve(2)
    │     │   solve(2): s[2..2]="b" → 1 + solve(3)
    │     │     solve(3): return -1 (past end)
    │     │   → solve(2) = 0
    │     │ s[1..2]="ab" not palindrome
    │   → solve(1) = 1
    │ s[0..1]="aa" is palindrome → 1 + solve(2) = 1
    │ s[0..2]="aab" not palindrome
  → solve(0) = min(2, 1) = 1

Result: "aa" | "b" → 1 cut
```

---

## Example 9: Memoization vs Tabulation — When to Choose

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Memoization vs Tabulation ===")
	fmt.Println()

	rows := []struct{ aspect, memo, tab string }{
		{"Direction", "Top-down (start from problem)", "Bottom-up (start from base cases)"},
		{"Implementation", "Recursive + cache", "Iterative + table"},
		{"States computed", "Only reachable states", "All states in table"},
		{"Stack overflow", "Risk for deep recursion", "No risk (iterative)"},
		{"Ease of writing", "Easy from recursion", "Requires understanding order"},
		{"Space optimization", "Harder", "Easier (rolling array)"},
		{"Constant factor", "Higher (function calls)", "Lower (simple loops)"},
		{"Best when", "Sparse state space", "Dense state space"},
	}

	fmt.Printf("%-20s %-32s %s\n", "Aspect", "Memoization", "Tabulation")
	fmt.Println(string(make([]byte, 85)))
	for _, r := range rows {
		fmt.Printf("%-20s %-32s %s\n", r.aspect, r.memo, r.tab)
	}
}
```

**Textual Figure:**

```
Memoization vs Tabulation — Decision Flow:

                   DP Problem
                      │
            ┌─────────┴─────────┐
            │                   │
     Sparse state space?   Dense state space?
     Few states reachable   All states needed
            │                   │
            ▼                   ▼
     ┌─────────────┐    ┌─────────────┐
     │ MEMOIZATION │    │  TABULATION  │
     ├─────────────┤    ├─────────────┤
     │ Top-down    │    │ Bottom-up   │
     │ Recursive   │    │ Iterative   │
     │ Lazy eval   │    │ All states  │
     │ Stack risk  │    │ No stack    │
     │ Map/array   │    │ Table/array │
     └─────────────┘    └─────────────┘

Space optimization: Tabulation wins (rolling arrays)
Ease of coding:     Memoization wins (just add cache)
Performance:        Tabulation wins (no call overhead)
```

---

## Example 10: Generic Memoize Helper in Go

```go
package main

import "fmt"

// Generic memoization wrapper using closures

type MemoFunc func(n int) int

func memoize(f func(n int, recurse MemoFunc) int) MemoFunc {
	cache := map[int]int{}

	var memoized MemoFunc
	memoized = func(n int) int {
		if v, ok := cache[n]; ok { return v }
		result := f(n, memoized)
		cache[n] = result
		return result
	}
	return memoized
}

func main() {
	// Fibonacci using generic memoize
	fib := memoize(func(n int, recurse MemoFunc) int {
		if n <= 1 { return n }
		return recurse(n-1) + recurse(n-2)
	})

	for i := 0; i <= 15; i++ {
		fmt.Printf("fib(%2d) = %d\n", i, fib(i))
	}

	// Tribonacci using same helper
	trib := memoize(func(n int, recurse MemoFunc) int {
		if n == 0 { return 0 }
		if n <= 2 { return 1 }
		return recurse(n-1) + recurse(n-2) + recurse(n-3)
	})

	fmt.Println()
	for i := 0; i <= 10; i++ {
		fmt.Printf("trib(%2d) = %d\n", i, trib(i))
	}
}
```

**Textual Figure:**

```
Generic Memoize Pattern — Cache Flow:

   call memoized(n)
         │
    ┌────┴─────┐
    │ In cache? │
    └────┬─────┘
     yes│    no│
    ┌───┴─┐  ┌──┴───────────┐
    │return│  │ compute f(n) │
    │cached│  │ using recurse│
    └──────┘  └─────┬───────┘
                   │
            ┌─────┴───────┐
            │ store in cache│
            │ return result │
            └─────────────┘

Fibonacci via memoize():
  fib(5) → cache miss → compute fib(4)+fib(3)
  fib(4) → cache miss → compute fib(3)+fib(2)
  fib(3) → cache miss → compute fib(2)+fib(1)
  fib(2) → cache miss → compute fib(1)+fib(0) = 1
  fib(1) → base case = 1
  fib(0) → base case = 0
  fib(3) → CACHE HIT = 2  (reused by fib(4))
  fib(2) → CACHE HIT = 1  (reused by fib(3) call from fib(4))

Cache: {0:0, 1:1, 2:1, 3:2, 4:3, 5:5}
```

---

## Key Takeaways

1. Memoization = recursion + cache — easiest way to add DP to existing recursive solution
2. Use map for sparse states, array for dense/bounded states (array is faster)
3. Only computes reachable states — better than tabulation when state space is sparse
4. Watch for stack overflow with deep recursion (Go default stack grows, but still bounded)
5. Converting from memoization to tabulation: identify the order states are needed, fill iteratively

> **Next up:** Tabulation →
