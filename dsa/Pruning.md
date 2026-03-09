# Phase 14: Backtracking — Pruning

## Overview

**Pruning** eliminates branches of the search tree that cannot possibly lead to valid solutions. By cutting away hopeless subtrees early, pruning can reduce exponential search spaces dramatically.

| Pruning Type | Strategy |
|-------------|----------|
| **Feasibility pruning** | Skip choices violating constraints |
| **Bound pruning** | Skip if best possible outcome < current best |
| **Symmetry pruning** | Skip symmetric/duplicate branches |
| **Ordering pruning** | Sort choices so pruning triggers earlier |
| **Alpha-beta pruning** | Game tree pruning (minimax) |

---

## Example 1: Basic Pruning — Combination Sum (LeetCode 39)

```go
package main

import (
	"fmt"
	"sort"
)

func combinationSum(candidates []int, target int) [][]int {
	sort.Ints(candidates) // Sort to enable pruning
	result := [][]int{}

	var backtrack func(start, remaining int, path []int)
	backtrack = func(start, remaining int, path []int) {
		if remaining == 0 {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := start; i < len(candidates); i++ {
			// PRUNING: if current candidate exceeds remaining, skip all further
			if candidates[i] > remaining {
				break // sorted, so all candidates[i+1..] are also too large
			}

			path = append(path, candidates[i])
			backtrack(i, remaining-candidates[i], path)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, target, []int{})
	return result
}

func main() {
	fmt.Println(combinationSum([]int{2, 3, 6, 7}, 7))
	// [[2 2 3] [7]]
}
```

**Why?** Sorting + break prunes all branches where the remaining sum is exceeded.

---

## Example 2: Duplicate Pruning — Combination Sum II (LeetCode 40)

```go
package main

import (
	"fmt"
	"sort"
)

func combinationSum2(candidates []int, target int) [][]int {
	sort.Ints(candidates)
	result := [][]int{}

	var backtrack func(start, remaining int, path []int)
	backtrack = func(start, remaining int, path []int) {
		if remaining == 0 {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := start; i < len(candidates); i++ {
			if candidates[i] > remaining { break }

			// PRUNING: skip duplicates at the same level
			if i > start && candidates[i] == candidates[i-1] {
				continue
			}

			path = append(path, candidates[i])
			backtrack(i+1, remaining-candidates[i], path)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, target, []int{})
	return result
}

func main() {
	fmt.Println(combinationSum2([]int{10, 1, 2, 7, 6, 1, 5}, 8))
	// [[1 1 6] [1 2 5] [1 7] [2 6]]
}
```

**Why?** `i > start && candidates[i] == candidates[i-1]` avoids generating duplicate subsets.

---

## Example 3: Constraint Pruning — N-Queens

```go
package main

import "fmt"

func totalNQueens(n int) int {
	count := 0
	cols := make([]bool, n)
	diag1 := make([]bool, 2*n)
	diag2 := make([]bool, 2*n)

	var solve func(row int)
	solve = func(row int) {
		if row == n { count++; return }

		for col := 0; col < n; col++ {
			// PRUNING: skip columns/diagonals already attacked
			if cols[col] || diag1[row-col+n] || diag2[row+col] {
				continue
			}

			cols[col] = true
			diag1[row-col+n] = true
			diag2[row+col] = true

			solve(row + 1)

			cols[col] = false
			diag1[row-col+n] = false
			diag2[row+col] = false
		}
	}

	solve(0)
	return count
}

func main() {
	for n := 1; n <= 10; n++ {
		fmt.Printf("N=%2d: %d solutions\n", n, totalNQueens(n))
	}
}
```

**Why?** Column + diagonal checks prune invalid placements immediately instead of checking after all queens are placed.

---

## Example 4: Bound Pruning — 0/1 Knapsack with Pruning

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct{ weight, value int }

func knapsackPruned(items []Item, capacity int) int {
	// Sort by value/weight ratio (descending) for better pruning
	sort.Slice(items, func(i, j int) bool {
		return float64(items[i].value)/float64(items[i].weight) >
			float64(items[j].value)/float64(items[j].weight)
	})

	bestValue := 0

	// Upper bound: fractional knapsack relaxation
	upperBound := func(idx, curWeight, curValue int) float64 {
		bound := float64(curValue)
		w := curWeight
		for i := idx; i < len(items); i++ {
			if w+items[i].weight <= capacity {
				w += items[i].weight
				bound += float64(items[i].value)
			} else {
				bound += float64(items[i].value) * float64(capacity-w) / float64(items[i].weight)
				break
			}
		}
		return bound
	}

	var search func(idx, weight, value int)
	search = func(idx, weight, value int) {
		if value > bestValue { bestValue = value }
		if idx == len(items) { return }

		// BOUND PRUNING: if upper bound can't beat current best, skip
		if upperBound(idx, weight, value) <= float64(bestValue) {
			return
		}

		// Include item
		if weight+items[idx].weight <= capacity {
			search(idx+1, weight+items[idx].weight, value+items[idx].value)
		}
		// Exclude item
		search(idx+1, weight, value)
	}

	search(0, 0, 0)
	return bestValue
}

func main() {
	items := []Item{{2, 6}, {2, 10}, {3, 12}, {7, 13}, {1, 5}}
	fmt.Println("Max value:", knapsackPruned(items, 10)) // 33
}
```

---

## Example 5: Symmetry Pruning — Permutations Without Duplicates (LeetCode 47)

```go
package main

import (
	"fmt"
	"sort"
)

func permuteUnique(nums []int) [][]int {
	sort.Ints(nums)
	result := [][]int{}
	used := make([]bool, len(nums))

	var backtrack func(path []int)
	backtrack = func(path []int) {
		if len(path) == len(nums) {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := 0; i < len(nums); i++ {
			if used[i] { continue }

			// SYMMETRY PRUNING: skip if same value was already tried at this position
			if i > 0 && nums[i] == nums[i-1] && !used[i-1] {
				continue
			}

			used[i] = true
			path = append(path, nums[i])
			backtrack(path)
			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack([]int{})
	return result
}

func main() {
	fmt.Println(permuteUnique([]int{1, 1, 2}))
	// [[1 1 2] [1 2 1] [2 1 1]]
}
```

---

## Example 6: Ordering Pruning — Sudoku Solver (LeetCode 37)

```go
package main

import "fmt"

func solveSudoku(board *[9][9]byte) bool {
	// Find most constrained empty cell (MRV heuristic)
	minOptions, bestR, bestC := 10, -1, -1

	for r := 0; r < 9; r++ {
		for c := 0; c < 9; c++ {
			if board[r][c] != '.' { continue }
			options := countOptions(board, r, c)
			if options < minOptions {
				minOptions = options
				bestR, bestC = r, c
			}
		}
	}

	if bestR == -1 { return true } // all filled

	for d := byte('1'); d <= '9'; d++ {
		if isValid(board, bestR, bestC, d) {
			board[bestR][bestC] = d
			if solveSudoku(board) { return true }
			board[bestR][bestC] = '.'
		}
	}
	return false
}

func countOptions(board *[9][9]byte, r, c int) int {
	count := 0
	for d := byte('1'); d <= '9'; d++ {
		if isValid(board, r, c, d) { count++ }
	}
	return count
}

func isValid(board *[9][9]byte, row, col int, d byte) bool {
	for i := 0; i < 9; i++ {
		if board[row][i] == d { return false }
		if board[i][col] == d { return false }
		r := 3*(row/3) + i/3
		c := 3*(col/3) + i%3
		if board[r][c] == d { return false }
	}
	return true
}

func main() {
	board := [9][9]byte{
		{'5','3','.','.','7','.','.','.','.'},
		{'6','.','.','1','9','5','.','.','.'},
		{'.','9','8','.','.','.','.','6','.'},
		{'8','.','.','.','6','.','.','.','3'},
		{'4','.','.','8','.','3','.','.','1'},
		{'7','.','.','.','2','.','.','.','6'},
		{'.','6','.','.','.','.','2','8','.'},
		{'.','.','.','4','1','9','.','.','5'},
		{'.','.','.','.','8','.','.','7','9'},
	}

	solveSudoku(&board)
	for _, row := range board {
		fmt.Println(string(row[:]))
	}
}
```

**Why?** Choosing the most constrained cell first (MRV = Minimum Remaining Values) causes failures faster, pruning more branches.

---

## Example 7: Counting With & Without Pruning

```go
package main

import "fmt"

var nodesWithPruning, nodesWithoutPruning int

// Without pruning: generate all subsets and check
func bruteForceSum(nums []int, target, idx, sum int) bool {
	nodesWithoutPruning++
	if idx == len(nums) { return sum == target }
	return bruteForceSum(nums, target, idx+1, sum+nums[idx]) ||
		bruteForceSum(nums, target, idx+1, sum)
}

// With pruning: cut early when sum exceeds target (all positive)
func prunedSum(nums []int, target, idx, sum int) bool {
	nodesWithPruning++
	if sum == target { return true }
	if sum > target { return false } // PRUNING
	if idx == len(nums) { return false }

	return prunedSum(nums, target, idx+1, sum+nums[idx]) ||
		prunedSum(nums, target, idx+1, sum)
}

func main() {
	nums := []int{3, 7, 1, 8, 4, 12, 5, 2, 9, 6}

	nodesWithoutPruning = 0
	bruteForceSum(nums, 15, 0, 0)
	fmt.Printf("Without pruning: %d nodes explored\n", nodesWithoutPruning)

	nodesWithPruning = 0
	prunedSum(nums, 15, 0, 0)
	fmt.Printf("With pruning:    %d nodes explored\n", nodesWithPruning)

	fmt.Printf("Saved:           %.1f%% of work\n",
		100.0*(1.0-float64(nodesWithPruning)/float64(nodesWithoutPruning)))
}
```

---

## Example 8: Alpha-Beta Pruning (Game Trees)

```go
package main

import (
	"fmt"
	"math"
)

var nodesEvaluated int

// Minimax with alpha-beta pruning
func alphaBeta(depth int, isMax bool, alpha, beta float64, tree [][]int, nodeIdx int) float64 {
	nodesEvaluated++
	if depth == 0 {
		return float64(tree[0][nodeIdx])
	}

	if isMax {
		val := math.Inf(-1)
		for i := 0; i < 2; i++ {
			child := alphaBeta(depth-1, false, alpha, beta, tree, nodeIdx*2+i)
			if child > val { val = child }
			if val > alpha { alpha = val }
			// PRUNING: beta cutoff
			if alpha >= beta { break }
		}
		return val
	} else {
		val := math.Inf(1)
		for i := 0; i < 2; i++ {
			child := alphaBeta(depth-1, true, alpha, beta, tree, nodeIdx*2+i)
			if child < val { val = child }
			if val < beta { beta = val }
			// PRUNING: alpha cutoff
			if alpha >= beta { break }
		}
		return val
	}
}

func main() {
	// Leaf values of a game tree (depth 3, binary tree)
	leaves := [][]int{{3, 5, 6, 9, 1, 2, 0, -1}}

	nodesEvaluated = 0
	result := alphaBeta(3, true, math.Inf(-1), math.Inf(1), leaves, 0)
	fmt.Printf("Optimal value: %.0f\n", result)
	fmt.Printf("Nodes evaluated: %d (out of %d leaves)\n", nodesEvaluated, len(leaves[0]))
}
```

---

## Example 9: Row Constraint Pruning — Word Search II (LeetCode 212 Simplified)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	word     string
}

func findWords(board [][]byte, words []string) []string {
	root := &TrieNode{}
	for _, w := range words {
		cur := root
		for _, c := range w {
			idx := c - 'a'
			if cur.children[idx] == nil { cur.children[idx] = &TrieNode{} }
			cur = cur.children[idx]
		}
		cur.word = w
	}

	result := []string{}
	rows, cols := len(board), len(board[0])
	dirs := [4][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

	var dfs func(r, c int, node *TrieNode)
	dfs = func(r, c int, node *TrieNode) {
		if r < 0 || r >= rows || c < 0 || c >= cols { return }
		ch := board[r][c]
		if ch == '#' { return }

		next := node.children[ch-'a']
		if next == nil { return } // PRUNING: no word with this prefix

		if next.word != "" {
			result = append(result, next.word)
			next.word = "" // avoid duplicates
		}

		board[r][c] = '#'
		for _, d := range dirs {
			dfs(r+d[0], c+d[1], next)
		}
		board[r][c] = ch
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			dfs(r, c, root)
		}
	}
	return result
}

func main() {
	board := [][]byte{
		{'o', 'a', 'a', 'n'},
		{'e', 't', 'a', 'e'},
		{'i', 'h', 'k', 'r'},
		{'i', 'f', 'l', 'v'},
	}
	words := []string{"oath", "pea", "eat", "rain"}
	fmt.Println(findWords(board, words)) // [oath eat]
}
```

**Why?** Trie prefix matching prunes paths where no word starts with the current prefix.

---

## Example 10: Pruning Strategies Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Pruning Strategies Summary ===")
	fmt.Println()

	strategies := []struct{ name, how, example string }{
		{"Feasibility", "Reject choices violating constraints immediately",
			"N-Queens: skip attacked diagonals"},
		{"Bound", "Skip if best possible < current best known",
			"Knapsack: fractional relaxation upper bound"},
		{"Duplicate/Symmetry", "Skip identical branches",
			"Permutations: skip same value at same level"},
		{"Ordering", "Try most promising / most constrained first",
			"Sudoku: MRV heuristic"},
		{"Prefix/Trie", "Skip paths with invalid prefixes",
			"Word Search II: trie guides exploration"},
		{"Alpha-Beta", "Skip game tree branches that can't affect result",
			"Minimax: prune when alpha >= beta"},
		{"Domain reduction", "Propagate constraints to reduce future choices",
			"Arc consistency in CSPs"},
	}

	for i, s := range strategies {
		fmt.Printf("%d. %s Pruning\n", i+1, s.name)
		fmt.Printf("   How:  %s\n", s.how)
		fmt.Printf("   Example: %s\n\n", s.example)
	}

	fmt.Println("Key principle: The earlier you prune, the more work you save.")
	fmt.Println("Sort + early termination is the simplest and most effective pruning.")
}
```

---

## Key Takeaways

1. Pruning cuts search space by eliminating branches that can't lead to valid solutions
2. Sort inputs to enable early break/continue (the easiest and most impactful pruning)
3. Feasibility pruning: violates constraint? skip immediately
4. Bound pruning: can't beat current best? skip the entire subtree
5. Duplicate pruning: sorted + `i > start && nums[i] == nums[i-1]` eliminates repeated work
6. MRV heuristic: pick the most constrained variable first for faster failure detection

> **Next up:** Permutations →
