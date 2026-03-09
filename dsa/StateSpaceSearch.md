# Phase 14: Backtracking — State Space Search

## Overview

**State space search** is the systematic exploration of all possible states (configurations) of a problem to find valid solutions. Backtracking explores this space as a tree, making choices and undoing them when a dead end is reached.

| Term | Meaning |
|------|---------|
| **State** | Current partial solution configuration |
| **Choice** | Decision that transitions to a new state |
| **Constraint** | Rules that valid states must satisfy |
| **Goal state** | A complete, valid solution |
| **Backtrack** | Undo last choice and try alternatives |

---

## Example 1: Subsets as State Space (LeetCode 78)

```go
package main

import "fmt"

func subsets(nums []int) [][]int {
	result := [][]int{}

	var explore func(idx int, state []int)
	explore = func(idx int, state []int) {
		// Every state is a valid subset
		result = append(result, append([]int{}, state...))

		for i := idx; i < len(nums); i++ {
			// Choice: include nums[i]
			state = append(state, nums[i])
			explore(i+1, state)
			// Backtrack: undo choice
			state = state[:len(state)-1]
		}
	}

	explore(0, []int{})
	return result
}

func main() {
	result := subsets([]int{1, 2, 3})
	fmt.Println("Subsets:")
	for _, s := range result {
		fmt.Println(" ", s)
	}
}
```

---

## Example 2: State Space Template

```go
package main

import "fmt"

func backtrack(choices []int, target int) [][]int {
	result := [][]int{}

	var search func(state []int, remaining int, start int)
	search = func(state []int, remaining int, start int) {
		// Check: is this a goal state?
		if remaining == 0 {
			result = append(result, append([]int{}, state...))
			return
		}
		// Check: can we prune?
		if remaining < 0 { return }

		// Expand: try each choice
		for i := start; i < len(choices); i++ {
			// Make choice
			state = append(state, choices[i])
			// Explore deeper
			search(state, remaining-choices[i], i)
			// Undo choice (backtrack)
			state = state[:len(state)-1]
		}
	}

	search([]int{}, target, 0)
	return result
}

func main() {
	result := backtrack([]int{2, 3, 6, 7}, 7)
	fmt.Println("Combinations summing to 7:")
	for _, r := range result { fmt.Println(" ", r) }
}
```

---

## Example 3: State = Board Configuration (N-Queens)

```go
package main

import "fmt"

func solveNQueens(n int) [][]string {
	result := [][]string{}
	board := make([][]byte, n)
	for i := range board {
		board[i] = make([]byte, n)
		for j := range board[i] { board[i][j] = '.' }
	}

	cols := make([]bool, n)
	diag1 := make([]bool, 2*n)
	diag2 := make([]bool, 2*n)

	var search func(row int)
	search = func(row int) {
		if row == n {
			sol := make([]string, n)
			for i := range board { sol[i] = string(board[i]) }
			result = append(result, sol)
			return
		}

		for col := 0; col < n; col++ {
			if cols[col] || diag1[row-col+n] || diag2[row+col] { continue }

			// Modify state
			board[row][col] = 'Q'
			cols[col] = true; diag1[row-col+n] = true; diag2[row+col] = true

			search(row + 1)

			// Restore state
			board[row][col] = '.'
			cols[col] = false; diag1[row-col+n] = false; diag2[row+col] = false
		}
	}

	search(0)
	return result
}

func main() {
	solutions := solveNQueens(4)
	fmt.Println("4-Queens solutions:", len(solutions))
	for _, sol := range solutions {
		for _, row := range sol { fmt.Println(row) }
		fmt.Println()
	}
}
```

---

## Example 4: State = Grid Position (Word Search — LeetCode 79)

```go
package main

import "fmt"

func exist(board [][]byte, word string) bool {
	rows, cols := len(board), len(board[0])
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	var search func(r, c, idx int) bool
	search = func(r, c, idx int) bool {
		if idx == len(word) { return true } // goal state
		if r < 0 || r >= rows || c < 0 || c >= cols { return false }
		if board[r][c] != word[idx] { return false }

		// Mark visited (modify state)
		temp := board[r][c]
		board[r][c] = '#'

		for _, d := range dirs {
			if search(r+d[0], c+d[1], idx+1) { return true }
		}

		// Restore state
		board[r][c] = temp
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
	fmt.Println(exist(board, "ABCCED")) // true
	fmt.Println(exist(board, "SEE"))     // true
	fmt.Println(exist(board, "ABCB"))    // false
}
```

---

## Example 5: State = String Being Built (Generate Parentheses — LeetCode 22)

```go
package main

import "fmt"

func generateParenthesis(n int) []string {
	result := []string{}

	var search func(state []byte, open, close int)
	search = func(state []byte, open, close int) {
		if len(state) == 2*n {
			result = append(result, string(state))
			return
		}

		if open < n {
			search(append(state, '('), open+1, close)
		}
		if close < open {
			search(append(state, ')'), open, close+1)
		}
	}

	search([]byte{}, 0, 0)
	return result
}

func main() {
	result := generateParenthesis(3)
	for _, s := range result { fmt.Println(s) }
}
```

---

## Example 6: State = Digit Mapping (Letter Combinations — LeetCode 17)

```go
package main

import "fmt"

func letterCombinations(digits string) []string {
	if len(digits) == 0 { return nil }

	phone := map[byte]string{
		'2': "abc", '3': "def", '4': "ghi", '5': "jkl",
		'6': "mno", '7': "pqrs", '8': "tuv", '9': "wxyz",
	}

	result := []string{}

	var search func(idx int, state []byte)
	search = func(idx int, state []byte) {
		if idx == len(digits) {
			result = append(result, string(state))
			return
		}

		for _, ch := range phone[digits[idx]] {
			search(idx+1, append(state, byte(ch)))
		}
	}

	search(0, []byte{})
	return result
}

func main() {
	fmt.Println(letterCombinations("23"))
	// [ad ae af bd be bf cd ce cf]
}
```

---

## Example 7: State = IP Address Parts (Restore IP — LeetCode 93)

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func restoreIpAddresses(s string) []string {
	result := []string{}

	var search func(start int, parts []string)
	search = func(start int, parts []string) {
		if len(parts) == 4 {
			if start == len(s) {
				result = append(result, strings.Join(parts, "."))
			}
			return
		}

		for length := 1; length <= 3; length++ {
			if start+length > len(s) { break }
			part := s[start : start+length]
			if len(part) > 1 && part[0] == '0' { break } // no leading zeros
			num, _ := strconv.Atoi(part)
			if num > 255 { break }

			search(start+length, append(parts, part))
		}
	}

	search(0, []string{})
	return result
}

func main() {
	fmt.Println(restoreIpAddresses("25525511135"))
	// [255.255.11.135 255.255.111.35]
}
```

---

## Example 8: BFS State Space Search

```go
package main

import "fmt"

// State space search using BFS (shortest path in state space)
// Example: Word Ladder (transform one word to another, one letter at a time)

func ladderLength(beginWord, endWord string, wordList []string) int {
	wordSet := map[string]bool{}
	for _, w := range wordList { wordSet[w] = true }
	if !wordSet[endWord] { return 0 }

	queue := []string{beginWord}
	visited := map[string]bool{beginWord: true}
	steps := 1

	for len(queue) > 0 {
		size := len(queue)
		for i := 0; i < size; i++ {
			word := queue[i]
			if word == endWord { return steps }

			// Try all single-character changes
			chars := []byte(word)
			for j := 0; j < len(chars); j++ {
				original := chars[j]
				for c := byte('a'); c <= 'z'; c++ {
					chars[j] = c
					next := string(chars)
					if wordSet[next] && !visited[next] {
						visited[next] = true
						queue = append(queue, next)
					}
				}
				chars[j] = original
			}
		}
		queue = queue[size:]
		steps++
	}
	return 0
}

func main() {
	words := []string{"hot","dot","dog","lot","log","cog"}
	fmt.Println(ladderLength("hit", "cog", words)) // 5
}
```

---

## Example 9: Implicit vs Explicit State

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Implicit vs Explicit State ===")
	fmt.Println()

	// Implicit state: using function parameters and call stack
	fmt.Println("1. Implicit State (most common):")
	fmt.Println("   - State stored in function arguments")
	fmt.Println("   - Backtracking = returning from recursive call")
	fmt.Println("   - Example: permute(nums, chosen, used)")

	fmt.Println()

	// Explicit state: using a struct
	type State struct {
		board    [4][4]int
		row      int
		queens   int
	}

	fmt.Println("2. Explicit State (BFS or complex state):")
	fmt.Println("   - State stored in a struct")
	fmt.Println("   - Queue/stack of states")
	fmt.Println("   - Example: State{board, row, queenCount}")

	fmt.Println()
	fmt.Println("When to use which:")
	fmt.Printf("%-25s %-15s\n", "Scenario", "Approach")
	fmt.Println("----------------------------------------")
	fmt.Printf("%-25s %-15s\n", "Find all solutions", "DFS (implicit)")
	fmt.Printf("%-25s %-15s\n", "Find any solution", "DFS (implicit)")
	fmt.Printf("%-25s %-15s\n", "Shortest path in states", "BFS (explicit)")
	fmt.Printf("%-25s %-15s\n", "Complex undo logic", "Explicit state")
}
```

---

## Example 10: State Space Search Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== State Space Search Summary ===")
	fmt.Println()

	concepts := []struct{ concept, description string }{
		{"State", "Current configuration of the partial solution"},
		{"Initial state", "Empty or starting configuration"},
		{"Goal test", "Check if current state is a complete solution"},
		{"Successor function", "Generate next states from current state"},
		{"State space tree", "Tree of all possible state transitions"},
		{"DFS exploration", "Go deep first, backtrack on failure"},
		{"BFS exploration", "Level by level, finds shortest path"},
		{"Visited tracking", "Avoid revisiting same state"},
		{"State encoding", "Represent state compactly (hash, bitmask)"},
		{"State space size", "Determines algorithm feasibility"},
	}

	for i, c := range concepts {
		fmt.Printf("%2d. %-22s %s\n", i+1, c.concept, c.description)
	}

	fmt.Println()
	fmt.Println("Template: search(state) {")
	fmt.Println("  if isGoal(state): record solution")
	fmt.Println("  for each choice in expand(state):")
	fmt.Println("    make choice → new state")
	fmt.Println("    search(new state)")
	fmt.Println("    undo choice (backtrack)")
	fmt.Println("}")
}
```

---

## Key Takeaways

1. State space search = exploring all possible configurations systematically
2. State captures everything needed to continue solving (board, position, choices made)
3. DFS for finding all/any solutions; BFS for shortest paths in state space
4. Backtracking = DFS with undo — the core pattern for constraint problems
5. State encoding matters for efficiency (visited sets, memoization)

> **Next up:** Pruning →
