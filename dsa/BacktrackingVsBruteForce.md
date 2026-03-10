# Phase 14: Backtracking — Backtracking vs Brute Force

## Overview

**Brute force** examines every possible solution. **Backtracking** prunes branches early, skipping entire subtrees that can't lead to valid solutions. Both explore the same state space, but backtracking does it intelligently.

| Aspect | Brute Force | Backtracking |
|--------|-------------|-------------|
| Explores | All possibilities | Only promising paths |
| Pruning | None | Yes — constraints checked early |
| When fails | After full construction | As soon as constraint violated |
| Efficiency | Generates all, filters later | Cuts branches during generation |
| Time | Always worst case | Often much better in practice |

---

## Example 1: Subsets That Sum to Target

```go
package main

import "fmt"

var bruteNodes, backtrackNodes int

// Brute force: generate ALL subsets, then check each
func bruteForce(nums []int, target int) [][]int {
	all := [][]int{}

	var generate func(idx int, subset []int)
	generate = func(idx int, subset []int) {
		bruteNodes++
		if idx == len(nums) {
			// Check after full construction
			sum := 0
			for _, v := range subset { sum += v }
			if sum == target {
				all = append(all, append([]int{}, subset...))
			}
			return
		}
		generate(idx+1, append(subset, nums[idx]))
		generate(idx+1, subset)
	}

	generate(0, []int{})
	return all
}

// Backtracking: prune when sum exceeds target (positive numbers)
func backtrack(nums []int, target int) [][]int {
	result := [][]int{}

	var search func(idx, sum int, path []int)
	search = func(idx, sum int, path []int) {
		backtrackNodes++
		if sum == target {
			result = append(result, append([]int{}, path...))
			return
		}
		if sum > target { return } // PRUNE
		if idx == len(nums) { return }

		search(idx+1, sum+nums[idx], append(path, nums[idx]))
		search(idx+1, sum, path)
	}

	search(0, 0, []int{})
	return result
}

func main() {
	nums := []int{3, 7, 1, 8, 4, 12, 5, 2, 9, 6}
	target := 15

	bruteNodes = 0
	r1 := bruteForce(nums, target)

	backtrackNodes = 0
	r2 := backtrack(nums, target)

	fmt.Printf("Brute force:  %d solutions, %d nodes explored\n", len(r1), bruteNodes)
	fmt.Printf("Backtracking: %d solutions, %d nodes explored\n", len(r2), backtrackNodes)
	fmt.Printf("Nodes saved:  %d (%.1f%%)\n",
		bruteNodes-backtrackNodes,
		100.0*(1.0-float64(backtrackNodes)/float64(bruteNodes)))
}
```

**Textual Figure:**

```
nums = [3, 7, 1], target = 4   (simplified to 3 elements)

  Brute Force (explores ALL 2³ = 8 subsets, checks sum at leaves):
  ─────────────────────────────────────────────────────────────────
                            idx=0
                      ┌──────┴──────┐
                 include 3        exclude 3
                   idx=1            idx=1
               ┌─────┴─────┐   ┌─────┴─────┐
          include 7    excl 7  incl 7    excl 7
            idx=2       idx=2  idx=2      idx=2
           ┌──┴──┐    ┌──┴──┐ ┌──┴──┐   ┌──┴──┐
          +1   skip  +1  skip +1  skip  +1   skip
         {3,7,1} {3,7} {3,1} {3} {7,1} {7}  {1}  {}
         sum=11  sum=10 sum=4✓ s=3 s=8   s=7  s=1  s=0
          ✗       ✗      ✓    ✗    ✗     ✗    ✗    ✗

         Nodes explored: 15 (all internal + leaves)

  Backtracking (prunes when sum > target):
  ────────────────────────────────────────
                            idx=0, sum=0
                      ┌──────┴──────┐
                 +3 (sum=3)     skip (sum=0)
                   idx=1            idx=1
               ┌─────┴─────┐   ┌─────┴─────┐
          +7 (sum=10)  skip    +7 (sum=7)   skip
           PRUNED ✂    idx=2    PRUNED ✂    idx=2
          sum>4!     ┌──┴──┐   sum>4!     ┌──┴──┐
                   +1   skip             +1    skip
                 sum=4✓  sum=3          sum=1   sum=0
                   ✓       ✗              ✗       ✗

         Nodes explored: 11 (2 pruned branches saved)

  Result: {3, 1} → sum = 4 ✓
```

---

## Example 2: Permutations — Filter vs Prune

```go
package main

import "fmt"

var bruteCount, btCount int

// Brute force: generate all arrangements, filter valid permutations
func brutePerms(nums []int) [][]int {
	result := [][]int{}
	n := len(nums)

	var generate func(path []int)
	generate = func(path []int) {
		bruteCount++
		if len(path) == n {
			// Check: are all elements unique positions?
			used := map[int]bool{}
			for _, v := range path {
				if used[v] { return }
				used[v] = true
			}
			result = append(result, append([]int{}, path...))
			return
		}
		for _, num := range nums {
			generate(append(path, num))
		}
	}

	generate([]int{})
	return result
}

// Backtracking: check used BEFORE adding
func btPerms(nums []int) [][]int {
	result := [][]int{}
	used := make([]bool, len(nums))

	var search func(path []int)
	search = func(path []int) {
		btCount++
		if len(path) == len(nums) {
			result = append(result, append([]int{}, path...))
			return
		}
		for i, num := range nums {
			if used[i] { continue } // PRUNE: already used
			used[i] = true
			search(append(path, num))
			used[i] = false
		}
	}

	search([]int{})
	return result
}

func main() {
	nums := []int{1, 2, 3, 4}

	bruteCount = 0
	r1 := brutePerms(nums)

	btCount = 0
	r2 := btPerms(nums)

	fmt.Printf("Brute force:  %d permutations, %d nodes\n", len(r1), bruteCount)
	fmt.Printf("Backtracking: %d permutations, %d nodes\n", len(r2), btCount)
	fmt.Printf("Brute explores n^n = %d^%d paths at leaves\n", len(nums), len(nums))
	fmt.Printf("Backtracking explores n! = %d paths at leaves\n", factorial(len(nums)))
}

func factorial(n int) int {
	if n <= 1 { return 1 }
	return n * factorial(n-1)
}
```

**Textual Figure:**

```
nums = [1, 2, 3]   n=3

  Brute Force (tries n^n = 27 leaf nodes — picks from ALL elements each time):
  ───────────────────────────────────────────────────────────────────────────
                              []
              ┌───────────────┼───────────────┐
             +1              +2              +3
          ┌───┼───┐       ┌───┼───┐       ┌───┼───┐
         +1  +2  +3      +1  +2  +3      +1  +2  +3
        ┌┼┐ ┌┼┐ ┌┼┐    ┌┼┐ ┌┼┐ ┌┼┐    ┌┼┐ ┌┼┐ ┌┼┐
        123 123 123    123 123 123    123 123 123
        ─── ─── ───    ─── ─── ───    ─── ─── ───
  [1,1,1]✗ [1,2,3]✓  ... 27 leaves total, only 6 valid

  Backtracking (only picks UNUSED elements — generates n! = 6 leaves):
  ────────────────────────────────────────────────────────────────────
                              []
              ┌───────────────┼───────────────┐
             +1              +2              +3
          (used:{1})      (used:{2})      (used:{3})
          ┌─────┐         ┌─────┐         ┌─────┐
         +2    +3        +1    +3        +1    +2
          │     │         │     │         │     │
         +3    +2        +3    +1        +2    +1
       [1,2,3] [1,3,2] [2,1,3] [2,3,1] [3,1,2] [3,2,1]
         ✓       ✓       ✓       ✓       ✓       ✓

  Brute: 27 leaves → 6 valid (22% hit rate)
  Backtracking: 6 leaves → 6 valid (100% hit rate)
  Nodes: 40 (brute) vs 16 (backtrack) → 2.5x speedup
```

---

## Example 3: N-Queens — Full Enumeration vs Backtracking

```go
package main

import "fmt"

var bruteChecks, btChecks int

// Brute force: try all n^n placements
func bruteNQueens(n int) int {
	count := 0
	queens := make([]int, n)

	var generate func(row int)
	generate = func(row int) {
		bruteChecks++
		if row == n {
			if isValidPlacement(queens, n) { count++ }
			return
		}
		for col := 0; col < n; col++ {
			queens[row] = col
			generate(row + 1)
		}
	}

	generate(0)
	return count
}

func isValidPlacement(queens []int, n int) bool {
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if queens[i] == queens[j] { return false }
			if abs(queens[i]-queens[j]) == j-i { return false }
		}
	}
	return true
}

// Backtracking: check constraints incrementally
func btNQueens(n int) int {
	count := 0
	cols := make([]bool, n)
	d1 := make([]bool, 2*n)
	d2 := make([]bool, 2*n)

	var solve func(row int)
	solve = func(row int) {
		btChecks++
		if row == n { count++; return }
		for col := 0; col < n; col++ {
			if cols[col] || d1[row-col+n] || d2[row+col] { continue }
			cols[col], d1[row-col+n], d2[row+col] = true, true, true
			solve(row + 1)
			cols[col], d1[row-col+n], d2[row+col] = false, false, false
		}
	}

	solve(0)
	return count
}

func abs(x int) int { if x < 0 { return -x }; return x }

func main() {
	for n := 4; n <= 8; n++ {
		bruteChecks = 0
		r1 := bruteNQueens(n)

		btChecks = 0
		r2 := btNQueens(n)

		fmt.Printf("N=%d: solutions=%d  brute=%d nodes  bt=%d nodes  ratio=%.1fx\n",
			n, r1, bruteChecks, btChecks, float64(bruteChecks)/float64(btChecks))
		_ = r2
	}
}
```

**Textual Figure:**

```
4-Queens: Brute Force vs Backtracking

  Brute Force tries ALL 4⁴ = 256 placements, validates at end:
  ─────────────────────────────────────────────────────────────
    col: 0 1 2 3    0 1 2 3    0 1 2 3        0 1 2 3
  row 0: Q . . .    Q . . .    Q . . .   ...  . . . Q
  row 1: Q . . .    . Q . .    . . Q .   ...  . . Q .
  row 2: Q . . .    Q . . .    Q . . .   ...  . Q . .
  row 3: Q . . .    Q . . .    Q . . .   ...  Q . . .
         ✗ all      ✗ diag     ✗ col          ✗ check
         same col   conflict               all 256 placements

  Backtracking prunes row-by-row:
  ──────────────────────────────
  Row 0: try col 0
    ┌─────────────────────────────────────────┐
    │  Q . . .   Row 1: col 0 ✂ (same col)   │
    │  . . . .         col 1 ✂ (diagonal)     │
    │  . . . .         col 2 → place          │
    │  . . . .                                │
    └─────────────────────────────────────────┘
    ┌─────────────────────────────────────────┐
    │  Q . . .   Row 1: col 2 placed          │
    │  . . Q .   Row 2: col 0 ✂ (col)         │
    │  . . . .         col 1 → place          │
    │  . . . .         but Row 3 fails → back │
    └─────────────────────────────────────────┘

  First valid solution found:
    ┌───────────┐
    │ . Q . .   │  Row 0: col 1
    │ . . . Q   │  Row 1: col 3
    │ Q . . .   │  Row 2: col 0
    │ . . Q .   │  Row 3: col 2
    └───────────┘
    queens = [1, 3, 0, 2] ✓

  N=4: Brute=341 nodes, Backtrack=17 nodes → 20x speedup
  N=8: Brute=19,173,961, Backtrack=2,057 → 9,322x speedup!
```

---

## Example 4: Word Search — Generate All vs Prune (LeetCode 79)

```go
package main

import "fmt"

var bruteOps, btOps int

// Brute force: generate all paths of length len(word), check match
func bruteWordSearch(board [][]byte, word string) bool {
	rows, cols := len(board), len(board[0])
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	var search func(r, c, idx int, visited [][]bool) bool
	search = func(r, c, idx int, visited [][]bool) bool {
		bruteOps++
		if idx == len(word) { return true }
		if r < 0 || r >= rows || c < 0 || c >= cols { return false }
		if visited[r][c] { return false }
		// No character check until end (brute force)
		visited[r][c] = true
		defer func() { visited[r][c] = false }()

		if idx < len(word) && board[r][c] != word[idx] {
			return false // still checks, but later in the process
		}

		for _, d := range dirs {
			if search(r+d[0], c+d[1], idx+1, visited) { return true }
		}
		return false
	}

	visited := make([][]bool, rows)
	for i := range visited { visited[i] = make([]bool, cols) }

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if search(r, c, 0, visited) { return true }
		}
	}
	return false
}

// Backtracking: prune immediately on character mismatch
func btWordSearch(board [][]byte, word string) bool {
	rows, cols := len(board), len(board[0])
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	var search func(r, c, idx int) bool
	search = func(r, c, idx int) bool {
		btOps++
		if idx == len(word) { return true }
		if r < 0 || r >= rows || c < 0 || c >= cols { return false }
		if board[r][c] != word[idx] { return false } // IMMEDIATE prune

		ch := board[r][c]
		board[r][c] = '#'
		for _, d := range dirs {
			if search(r+d[0], c+d[1], idx+1) { return true }
		}
		board[r][c] = ch
		return false
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if search(r, c, 0) { return true }
		}
	}
	return false
}

func main() {
	board := [][]byte{
		{'A','B','C','E'},
		{'S','F','C','S'},
		{'A','D','E','E'},
	}

	board2 := [][]byte{
		{'A','B','C','E'},
		{'S','F','C','S'},
		{'A','D','E','E'},
	}

	word := "ABCCED"

	bruteOps = 0
	r1 := bruteWordSearch(board, word)

	btOps = 0
	r2 := btWordSearch(board2, word)

	fmt.Printf("Brute force:  found=%v, ops=%d\n", r1, bruteOps)
	fmt.Printf("Backtracking: found=%v, ops=%d\n", r2, btOps)
}
```

**Textual Figure:**

```
Board:                  Word: "ABCCED"
  ┌───┬───┬───┬───┐
  │ A │ B │ C │ E │
  ├───┼───┼───┼───┤
  │ S │ F │ C │ S │
  ├───┼───┼───┼───┤
  │ A │ D │ E │ E │
  └───┴───┴───┴───┘

  Brute Force: generates all length-6 paths, checks at end
  ─────────────────────────────────────────────────────────
  Start (0,0)='A' → tries all 4 directions at each step
    (0,0)→(0,1)→(0,2)→(0,3)→... keeps going even if mismatch
    (0,0)→(1,0)→(2,0)→... explores 'A','S','A' path fully
    Many wasted ops on paths that diverge early

  Backtracking: prunes on FIRST character mismatch
  ────────────────────────────────────────────────
  Start (0,0)='A'=word[0] ✓
    → (0,1)='B'=word[1] ✓
      → (0,2)='C'=word[2] ✓
        → (0,3)='E' ≠ word[3]='C' ✂ PRUNE!
        → (1,2)='C'=word[3] ✓
          → (1,1)='F' ≠ word[4]='E' ✂ PRUNE!
          → (2,2)='E'=word[4] ✓
            → (2,1)='D'=word[5] ✓ FOUND!

  Path traced on board:
  ┌───┬───┬───┬───┐
  │ A→│→B→│→C │ E │   A(0,0) → B(0,1) → C(0,2)
  ├───┼───┼───┼───┤                       ↓
  │ S │ F │ C │ S │               C(1,2)←─┘
  ├───┼───┼───┼───┤               ↓
  │ A │ D←│←E │ E │   D(2,1) ← E(2,2)
  └───┴───┴───┴───┘

  Result: "ABCCED" found ✓
```

---

## Example 5: When Brute Force Is Better

```go
package main

import "fmt"

func main() {
	fmt.Println("=== When Brute Force Is Actually Better ===")
	fmt.Println()

	scenarios := []struct{ scenario, winner, reason string }{
		{"Very small input (n ≤ 5)", "Either",
			"Overhead of pruning logic not worth it"},
		{"Need ALL solutions, no pruning possible", "Brute Force",
			"No constraints to prune on; backtracking has overhead"},
		{"Counting (not enumerating)", "Brute Force / Math",
			"Formula may give O(1) answer vs O(2^n) enumeration"},
		{"Dense solution space", "Brute Force",
			"Most branches succeed, pruning rarely triggers"},
		{"Sparse solution space", "Backtracking",
			"Most branches fail early → massive pruning"},
		{"Strong constraints", "Backtracking",
			"Constraints prune huge subtrees quickly"},
		{"Need shortest path", "BFS (brute-level by level)",
			"BFS guarantees shortest; DFS backtracking doesn't"},
	}

	for i, s := range scenarios {
		fmt.Printf("%d. %s\n", i+1, s.scenario)
		fmt.Printf("   Winner: %s\n", s.winner)
		fmt.Printf("   Reason: %s\n\n", s.reason)
	}
}
```

**Textual Figure:**

```
  Decision Guide: When Brute Force Wins vs Backtracking Wins
  ══════════════════════════════════════════════════════════

  ┌──────────────────────────┐     ┌──────────────────────────┐
  │     BRUTE FORCE WINS     │     │    BACKTRACKING WINS     │
  ├──────────────────────────┤     ├──────────────────────────┤
  │ • Small input (n ≤ 5)    │     │ • Strong constraints     │
  │ • No constraints to      │     │ • Sparse solution space  │
  │   prune on               │     │ • Cheap constraint check │
  │ • Dense solutions (most  │     │ • Find any/first valid   │
  │   branches succeed)      │     │ • Early bounds available │
  │ • Counting via formula   │     │                          │
  │ • Need shortest path     │     │                          │
  └────────────┬─────────────┘     └────────────┬─────────────┘
               │                                │
               ▼                                ▼
  ┌──────────────────────────────────────────────────────────┐
  │           Solution Density Spectrum                      │
  │                                                          │
  │  Dense ◄────────────────────────────────────► Sparse     │
  │  (most valid)         ↑                  (few valid)     │
  │  Brute OK         BOTH OK             Backtrack wins     │
  │                                                          │
  │  Example:          Example:             Example:         │
  │  All subsets       Small perms          N-Queens          │
  │  of size k         (n ≤ 5)              Sudoku            │
  └──────────────────────────────────────────────────────────┘
```

---

## Example 6: Decision Tree Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Decision Tree: Brute Force vs Backtracking ===")
	fmt.Println()
	fmt.Println("Problem: Place 4 non-attacking rooks on 4×4 board")
	fmt.Println()

	fmt.Println("Brute Force approach:")
	fmt.Println("  - Generate all 4^4 = 256 placements")
	fmt.Println("  - Check each: are all columns unique?")
	fmt.Println("  - Valid: 4! = 24 out of 256 (9.4%)")
	fmt.Println()

	fmt.Println("Backtracking approach:")
	fmt.Println("  Row 0: try col 0,1,2,3 → pick col 0")
	fmt.Println("  Row 1: try col 1,2,3 (col 0 pruned) → pick col 1")
	fmt.Println("  Row 2: try col 2,3 (0,1 pruned) → pick col 2")
	fmt.Println("  Row 3: try col 3 (0,1,2 pruned) → pick col 3")
	fmt.Println("  → Explores ~4+3+2+1 = 10 nodes per path")
	fmt.Println()

	fmt.Println("Comparison:")
	fmt.Printf("  Brute force nodes:  256 leaves + internal = ~340\n")
	fmt.Printf("  Backtracking nodes: ~65 (varies by branching)\n")
	fmt.Printf("  Speedup: ~5x for n=4, grows exponentially\n")
}
```

**Textual Figure:**

```
4 Non-Attacking Rooks on 4×4 Board

  Brute Force Decision Tree (4⁴ = 256 leaves):
  ────────────────────────────────────────────
  Row 0:     0       1       2       3
  Row 1:   0 1 2 3  0 1 2 3  0 1 2 3  0 1 2 3
  Row 2:   ││││... (4 branches each → 4×4×4×4 paths)
  Row 3:   ││││... total = 256 leaves
           └─ check all 256 for unique columns
              Only 24 valid (4! = 24)

  Backtracking Decision Tree (~65 nodes):
  ────────────────────────────────────────────
  Row 0:    col=0         col=1        col=2      col=3
              │             │            │          │
  Row 1:   1  2  3       0  2  3      0  1  3    0  1  2
           │  │  │       │  │  │      │  │  │    │  │  │
  Row 2:  2  3 1  3 1  2  ...  (only unused columns)
           │  │ │  │ │  │
  Row 3:   3  2 3  1 2  1     (1 choice left each)

  Pruning illustration (Row 0=col 0):
    Row 0: col 0  →  R . . .
    Row 1: col 0  ✂ PRUNED (col 0 taken)
    Row 1: col 1  →  R . . .
                     . R . .
    Row 2: col 0  ✂ PRUNED
    Row 2: col 1  ✂ PRUNED
    Row 2: col 2  →  R . . .
                     . R . .
                     . . R .
    Row 3: col 3  →  R . . .  (only option left)
                     . R . .
                     . . R .
                     . . . R  ✓

  Summary: 256 nodes (brute) vs ~65 nodes (backtrack) → ~5x speedup
```

---

## Example 7: Empirical Comparison — Subset Sum

```go
package main

import (
	"fmt"
	"time"
)

func subsetSumBrute(nums []int, target int) int {
	count := 0
	n := len(nums)
	total := 1 << n

	for mask := 0; mask < total; mask++ {
		sum := 0
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				sum += nums[i]
			}
		}
		if sum == target { count++ }
	}
	return count
}

func subsetSumBacktrack(nums []int, target int) int {
	count := 0

	var search func(idx, sum int)
	search = func(idx, sum int) {
		if sum == target { count++; return }
		if sum > target || idx == len(nums) { return }

		search(idx+1, sum+nums[idx])
		search(idx+1, sum)
	}

	search(0, 0)
	return count
}

func main() {
	nums := []int{2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 3, 5, 7, 9, 11, 13, 15, 17, 19, 1}
	target := 50

	start := time.Now()
	r1 := subsetSumBrute(nums, target)
	t1 := time.Since(start)

	start = time.Now()
	r2 := subsetSumBacktrack(nums, target)
	t2 := time.Since(start)

	fmt.Printf("Brute force:  %d solutions in %v\n", r1, t1)
	fmt.Printf("Backtracking: %d solutions in %v\n", r2, t2)
	if t2 > 0 {
		fmt.Printf("Speedup: %.1fx\n", float64(t1)/float64(t2))
	}
}
```

**Textual Figure:**

```
nums = [2, 4, 6, 8, ...], target = 50

  Brute Force (bitmask enumeration — all 2^20 = 1,048,576 subsets):
  ────────────────────────────────────────────────────────────
  mask = 00000000000000000000  → sum=0    ≠ 50
  mask = 00000000000000000001  → sum=2    ≠ 50
  mask = 00000000000000000010  → sum=4    ≠ 50
  ...                          (checks ALL 1,048,576 masks)
  mask = 11111111111111111111  → sum=210  ≠ 50

  Backtracking (recursive with pruning on sum > target):
  ────────────────────────────────────────────────────────────
                        idx=0, sum=0
                  ┌───────┴───────┐
             +2 (sum=2)        skip
                idx=1           idx=1
           ┌─────┴─────┐       ...
      +4 (sum=6)     skip
         idx=2        idx=2
       ┌───┴───┐     ...
  +6 (sum=12)  skip
      ...       ...
  Eventually: sum = 2+4+6+8+10+12+8 = 50 → found!
  But if sum > 50, subtree is PRUNED ✂

  Example prune: {2,4,6,8,10,12,14} sum=56 > 50 ✂
    → Skips all 2^13 subsets of remaining elements!

  Result: Backtracking ~2–10x faster on this input
```

---

## Example 8: Backtracking With Memoization (Hybrid)

```go
package main

import "fmt"

// When backtracking revisits identical subproblems, memoize!
// This is where backtracking meets DP.

func canPartitionBacktrack(nums []int) bool {
	sum := 0
	for _, n := range nums { sum += n }
	if sum%2 != 0 { return false }
	target := sum / 2

	// Pure backtracking (exponential without memo)
	nodes := 0
	var bt func(idx, remaining int) bool
	bt = func(idx, remaining int) bool {
		nodes++
		if remaining == 0 { return true }
		if remaining < 0 || idx == len(nums) { return false }
		return bt(idx+1, remaining-nums[idx]) || bt(idx+1, remaining)
	}

	r1 := bt(0, target)
	fmt.Printf("Pure backtracking: %v, nodes=%d\n", r1, nodes)

	// Backtracking + memoization (polynomial)
	memo := map[[2]int]bool{}
	memoNodes := 0
	var btMemo func(idx, remaining int) bool
	btMemo = func(idx, remaining int) bool {
		memoNodes++
		if remaining == 0 { return true }
		if remaining < 0 || idx == len(nums) { return false }

		key := [2]int{idx, remaining}
		if v, ok := memo[key]; ok { return v }

		result := btMemo(idx+1, remaining-nums[idx]) || btMemo(idx+1, remaining)
		memo[key] = result
		return result
	}

	r2 := btMemo(0, target)
	fmt.Printf("With memoization:  %v, nodes=%d\n", r2, memoNodes)

	return
}

func main() {
	canPartitionBacktrack([]int{1, 5, 11, 5})
	fmt.Println()
	canPartitionBacktrack([]int{1, 2, 3, 4, 5, 6, 7})
}
```

**Textual Figure:**

```
nums = [1, 5, 11, 5], sum=22, target=11

  Pure Backtracking (many repeated subproblems):
  ─────────────────────────────────────────────────
                   (idx=0, rem=11)
              ┌───────┴───────┐
         -1 (rem=10)      skip (rem=11)
            idx=1              idx=1
         ┌───┴───┐        ┌───┴───┐
    -5(rem=5) skip(10) -5(rem=6) skip(11)
       idx=2    idx=2     idx=2 ◄── SAME
      ┌─┴─┐  ┌─┴─┐   ┌─┴─┐      subproblem!
   -11 skip -11 skip  -11 skip
   rem=-6 rem=5  ...
    │     │
   <0│   ┌┴┐    (idx=2,rem=5) computed TWICE
  prune -5 skip   (idx=2,rem=10) computed TWICE

  With Memoization (cache key = (idx, remaining)):
  ─────────────────────────────────────────────────
   memo[(2,5)]  = true  ← cached, reused!
   memo[(2,10)] = false ← cached, reused!

   Pure backtracking: ~15 nodes
   With memoization:  ~10 nodes (overlaps eliminated)

   Key insight: backtracking + memo = DP!
```

---

## Example 9: Complexity Comparison Table

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Complexity: Brute Force vs Backtracking ===")
	fmt.Println()

	problems := []struct {
		problem, bruteTime, btTime, note string
	}{
		{"Permutations", "O(n^n)", "O(n!)",
			"n^n >> n! (e.g., 10^10 vs 3.6M)"},
		{"N-Queens", "O(n^n)", "O(n!)",
			"Pruning makes it even faster in practice"},
		{"Subset Sum", "O(2^n)", "O(2^n) worst",
			"Pruning helps when target is small"},
		{"Sudoku", "O(9^81)", "O(9^empty)",
			"Constraints reduce 9^81 to manageable"},
		{"Graph Coloring", "O(m^n)", "< O(m^n)",
			"Adjacency constraints prune heavily"},
		{"Combination Sum", "O(inf)", "O(target/min)",
			"Bounded by target, not by n"},
	}

	fmt.Printf("%-18s %-12s %-14s %s\n", "Problem", "Brute", "Backtrack", "Notes")
	fmt.Println("---------------------------------------------------------------")
	for _, p := range problems {
		fmt.Printf("%-18s %-12s %-14s %s\n", p.problem, p.bruteTime, p.btTime, p.note)
	}
}
```

**Textual Figure:**

```
  Complexity Growth: Brute Force vs Backtracking
  ═════════════════════════════════════════════════════

  Permutations (n elements):     N-Queens:
  ────────────────────────     ────────────────────────
   n   Brute(n^n) Backtrack(n!)   n   Brute(n^n)   Backtrack
   3      27          6          4     256            17
   5    3,125        120          8  16.7M         2,057
  10    10^10    3,628,800       12   8.9×10^12     ~15,000

  ┌────────────────────────────────────────────────────┐
  │  Sudoku:                                          │
  │    Brute: 9^81 ≈ 1.97 × 10^77  (impossible!)       │
  │    Backtrack: 9^empty ≈ 9^51 but with constraints  │
  │    In practice: solves in < 1ms with pruning       │
  └────────────────────────────────────────────────────┘

  The gap between brute force and backtracking grows
  EXPONENTIALLY as n increases.
```

---

## Example 10: Decision Guide — When to Use Which

```go
package main

import "fmt"

func main() {
	fmt.Println("=== When to Use Brute Force vs Backtracking ===")
	fmt.Println()

	fmt.Println("Use BRUTE FORCE when:")
	fmt.Println("  1. Input is very small (n ≤ 5-8)")
	fmt.Println("  2. No constraints to prune on")
	fmt.Println("  3. Need to examine every possibility anyway")
	fmt.Println("  4. Solution density is high (most branches valid)")
	fmt.Println("  5. Simpler code is preferred for correctness")
	fmt.Println()

	fmt.Println("Use BACKTRACKING when:")
	fmt.Println("  1. Strong constraints exist (queens, sudoku)")
	fmt.Println("  2. Solution space is large but solutions sparse")
	fmt.Println("  3. Constraint checking is O(1) or cheap")
	fmt.Println("  4. Want to find first/any valid solution")
	fmt.Println("  5. Can prune early with bounds or feasibility")
	fmt.Println()

	fmt.Println("Use BACKTRACKING + MEMOIZATION when:")
	fmt.Println("  1. Overlapping subproblems exist")
	fmt.Println("  2. State can be encoded compactly")
	fmt.Println("  3. This is a bridge to dynamic programming")
	fmt.Println()

	fmt.Println("Mental model:")
	fmt.Println("  Brute Force = for loop over all candidates, filter")
	fmt.Println("  Backtracking = recursive build + constraint check + undo")
	fmt.Println("  DP = backtracking + memoization (when subproblems overlap)")
}
```

**Textual Figure:**

```
  Algorithm Selection Flowchart
  ═══════════════════════════════

                ┌─────────────────┐
                │  Has constraints │
                │  to prune on?    │
                └────────┬────────┘
               YES│         │NO
          ┌─────┴─────┐  ┌┴─────────────┐
          │ Backtracking │  │  BRUTE FORCE   │
          └──────┬─────┘  └──────────────┘
                 │
      ┌─────────┴─────────┐
      │ Overlapping          │
      │ subproblems?         │
      └─────────┬─────────┘
             YES│         │NO
        ┌─────┴─────┐  ┌┴─────────────┐
        │ Add memo    │  │ Pure backtrack │
        │ → becomes  │  │ (no memo)      │
        │   DP        │  └──────────────┘
        └────────────┘

  Mental Model:
    Brute Force  ─────────→  for-loop over all, filter
    Backtracking ─────────→  recursive build + prune + undo
    DP           ─────────→  backtracking + memoization
```

---

## Key Takeaways

1. Brute force generates everything and filters; backtracking prunes during generation
2. Backtracking reduces O(n^n) to O(n!) for permutations — exponential difference
3. The stronger the constraints, the more backtracking saves over brute force
4. For very small inputs, both perform similarly — use whichever is clearer
5. When backtracking has overlapping subproblems, add memoization → becomes DP
6. Backtracking is not always better: if most branches succeed, pruning overhead wasted

> **Next up:** Phase 15 — Dynamic Programming: Overlapping Subproblems →
