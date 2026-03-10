# Phase 14: Backtracking — Constraint Satisfaction

## Overview

A **Constraint Satisfaction Problem (CSP)** has variables, domains (possible values), and constraints. The goal is to assign values to all variables such that every constraint is satisfied. Backtracking with constraint propagation is the standard approach.

| Term | Meaning |
|------|---------|
| **Variable** | Something that needs a value |
| **Domain** | Set of possible values for a variable |
| **Constraint** | Rule restricting which value combinations are allowed |
| **Assignment** | Mapping of values to variables |
| **Consistent** | Assignment satisfies all constraints |

---

## Example 1: N-Queens as CSP

```go
package main

import "fmt"

// Variables: rows 0..n-1
// Domain: columns 0..n-1
// Constraints: no two queens share column or diagonal

func solveNQueens(n int) [][]int {
	result := [][]int{}
	queens := make([]int, n) // queens[row] = column

	cols := make([]bool, n)
	diag1 := make([]bool, 2*n)
	diag2 := make([]bool, 2*n)

	var solve func(row int)
	solve = func(row int) {
		if row == n {
			sol := make([]int, n)
			copy(sol, queens)
			result = append(result, sol)
			return
		}

		for col := 0; col < n; col++ {
			// Check constraints
			if cols[col] || diag1[row-col+n] || diag2[row+col] {
				continue
			}

			// Assign variable
			queens[row] = col
			cols[col] = true
			diag1[row-col+n] = true
			diag2[row+col] = true

			solve(row + 1)

			// Undo assignment
			cols[col] = false
			diag1[row-col+n] = false
			diag2[row+col] = false
		}
	}

	solve(0)
	return result
}

func main() {
	solutions := solveNQueens(8)
	fmt.Printf("8-Queens: %d solutions\n", len(solutions))
	fmt.Println("First solution:", solutions[0])
}
```

**Textual Figure:**

```
4-Queens as CSP  (Variables: rows, Domain: cols, Constraints: col/diag)

  Backtracking search with constraint checking:
  ───────────────────────────────────────────────────
  Row 0: try col 0      Row 0: try col 1
  ┌───┬───┬───┬───┐     ┌───┬───┬───┬───┐
  │ Q │   │   │   │     │   │ Q │   │   │
  ├───┼───┼───┼───┤     ├───┼───┼───┼───┤
  │   │   │ Q │   │     │   │   │   │ Q │
  ├───┼───┼───┼───┤     ├───┼───┼───┼───┤
  │   │   │   │   │     │ Q │   │   │   │
  │   Row 2: all  │     ├───┼───┼───┼───┤
  │   cols fail!   │     │   │   │ Q │   │
  └───┴───┴───┴───┘     └───┴───┴───┴───┘
  BACKTRACK ✂               queens=[1,3,0,2] ✓

  Constraint check per cell:
    cols[c]?           → same column as existing queen?
    diag1[row-col+n]?  → same \ diagonal?
    diag2[row+col]?    → same / diagonal?

    Row 0 col 1: cols[1]=F, diag1[1-1+4]=F, diag2[1+0]=F → OK
    Row 1 col 3: cols[3]=F, diag1[1-3+4]=F, diag2[1+3]=F → OK
    Row 2 col 0: cols[0]=F, diag1[2-0+4]=F, diag2[2+0]=F → OK
    Row 3 col 2: cols[2]=F, diag1[3-2+4]=F, diag2[3+2]=F → OK

  Solution: [1, 3, 0, 2]   (first of 2 for n=4)
  8-Queens: 92 solutions
```

---

## Example 2: Sudoku Solver (LeetCode 37)

```go
package main

import "fmt"

// Variables: empty cells
// Domain: digits 1-9
// Constraints: row, column, 3x3 box uniqueness

func solveSudoku(board *[9][9]byte) bool {
	r, c := findEmpty(board)
	if r == -1 { return true } // all filled = solved

	for d := byte('1'); d <= '9'; d++ {
		if isValid(board, r, c, d) {
			board[r][c] = d
			if solveSudoku(board) { return true }
			board[r][c] = '.' // undo
		}
	}
	return false
}

func findEmpty(board *[9][9]byte) (int, int) {
	for r := 0; r < 9; r++ {
		for c := 0; c < 9; c++ {
			if board[r][c] == '.' { return r, c }
		}
	}
	return -1, -1
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

	if solveSudoku(&board) {
		for _, row := range board {
			fmt.Println(string(row[:]))
		}
	}
}
```

**Textual Figure:**

```
Sudoku CSP: Variables=empty cells, Domain={1..9}, Constraints=row/col/box

  Initial board:                     Solved board:
  ┌───┬───┬───┰───┬───┬───┰───┬───┬───┐   ┌───┬───┬───┰───┬───┬───┰───┬───┬───┐
  │ 5 │ 3 │ . ┃ . │ 7 │ . ┃ . │ . │ . │   │ 5 │ 3 │ 4 ┃ 6 │ 7 │ 8 ┃ 9 │ 1 │ 2 │
  │ 6 │ . │ . ┃ 1 │ 9 │ 5 ┃ . │ . │ . │   │ 6 │ 7 │ 2 ┃ 1 │ 9 │ 5 ┃ 3 │ 4 │ 8 │
  │ . │ 9 │ 8 ┃ . │ . │ . ┃ . │ 6 │ . │   │ 1 │ 9 │ 8 ┃ 3 │ 4 │ 2 ┃ 5 │ 6 │ 7 │
  ┠───┼───┼───╋───┼───┼───╋───┼───┼───┨   ┠───┼───┼───╋───┼───┼───╋───┼───┼───┨
  │ 8 │ . │ . ┃ . │ 6 │ . ┃ . │ . │ 3 │   │ 8 │ 5 │ 9 ┃ 7 │ 6 │ 1 ┃ 4 │ 2 │ 3 │
  │ 4 │ . │ . ┃ 8 │ . │ 3 ┃ . │ . │ 1 │   │ 4 │ 2 │ 6 ┃ 8 │ 5 │ 3 ┃ 7 │ 9 │ 1 │
  │ 7 │ . │ . ┃ . │ 2 │ . ┃ . │ . │ 6 │   │ 7 │ 1 │ 3 ┃ 9 │ 2 │ 4 ┃ 8 │ 5 │ 6 │
  ┠───┼───┼───╋───┼───┼───╋───┼───┼───┨   ┠───┼───┼───╋───┼───┼───╋───┼───┼───┨
  │ . │ 6 │ . ┃ . │ . │ . ┃ 2 │ 8 │ . │   │ 9 │ 6 │ 1 ┃ 5 │ 3 │ 7 ┃ 2 │ 8 │ 4 │
  │ . │ . │ . ┃ 4 │ 1 │ 9 ┃ . │ . │ 5 │   │ 2 │ 8 │ 7 ┃ 4 │ 1 │ 9 ┃ 6 │ 3 │ 5 │
  │ . │ . │ . ┃ . │ 8 │ . ┃ . │ 7 │ 9 │   │ 3 │ 4 │ 5 ┃ 2 │ 8 │ 6 ┃ 1 │ 7 │ 9 │
  └───┴───┴───┸───┴───┴───┸───┴───┴───┘   └───┴───┴───┸───┴───┴───┸───┴───┴───┘

  Backtracking trace (first empty cell = (0,2)):
    Try '1': row has 5,3 ✓  col has 8 ✓  box has 5,3,6,9,8 ✓
      → but leads to dead end later → undo
    Try '4': row ✓  col ✓  box ✓
      → proceed to next empty cell (0,3)
      → eventually solves completely ✓
```

---

## Example 3: Graph Coloring

```go
package main

import "fmt"

// Variables: vertices
// Domain: colors 0..m-1
// Constraints: adjacent vertices must have different colors

func graphColoring(adj [][]int, m int) []int {
	n := len(adj)
	colors := make([]int, n)
	for i := range colors { colors[i] = -1 }

	var solve func(v int) bool
	solve = func(v int) bool {
		if v == n { return true }

		for c := 0; c < m; c++ {
			if isSafe(adj, colors, v, c) {
				colors[v] = c
				if solve(v + 1) { return true }
				colors[v] = -1
			}
		}
		return false
	}

	if solve(0) { return colors }
	return nil
}

func isSafe(adj [][]int, colors []int, v, c int) bool {
	for _, u := range adj[v] {
		if colors[u] == c { return false }
	}
	return true
}

func main() {
	// Graph: 0-1, 0-2, 1-2, 1-3, 2-3
	adj := [][]int{
		{1, 2},    // 0
		{0, 2, 3}, // 1
		{0, 1, 3}, // 2
		{1, 2},    // 3
	}

	for m := 1; m <= 4; m++ {
		result := graphColoring(adj, m)
		if result != nil {
			fmt.Printf("%d colors: %v ✓\n", m, result)
		} else {
			fmt.Printf("%d colors: no solution\n", m)
		}
	}
}
```

**Textual Figure:**

```
Graph:  0───1           Coloring with m colors
        |\ /|           Variables: vertices {0,1,2,3}
        | X |           Domain: colors {0..m-1}
        |/ \|           Constraint: adjacent ≠ same color
        2───3

  m=1: all vertices must be different but all connected → FAIL
  m=2: try vertex 0...
    v0=0, v1=1, v2=1 ✂ (v2 adj to v1, both=1) → try v2=0 │
    v2=0 ││ (adj to v0, both=0) ││→ FAIL││

  m=3: Backtracking search:
  ─────────────────────────────────────
    v0 = 0 (R)
      v1: 0(R)││adj to v0 │→ try 1(G) ✓
        v2: 0(R)││adj to v0,v1 → try 2(B) ✓
          v3: 0(R)││adj to v1(G),v2(B) → 0(R) ✓
            ✓ FOUND!

  Result:  (0)───(1)     R───G
            |\ /|         |\ /|
            | X |    →   | X |
            |/ \|        |/ \|
           (2)───(3)     B───R

  colors = [0, 1, 2, 0]  (R, G, B, R)  ✓  chromatic number = 3
```

---

## Example 4: Cryptarithmetic Puzzle (SEND + MORE = MONEY)

```go
package main

import "fmt"

func solveCryptarithmetic() {
	// Variables: S, E, N, D, M, O, R, Y
	// Domain: 0-9
	// Constraints: all different, SEND + MORE = MONEY, S≠0, M≠0

	used := make([]bool, 10)
	var S, E, N, D, M, O, R, Y int

	assign := func(vals [8]int) (int, int) {
		S, E, N, D = vals[0], vals[1], vals[2], vals[3]
		M, O, R, Y = vals[4], vals[5], vals[6], vals[7]
		send := S*1000 + E*100 + N*10 + D
		more := M*1000 + O*100 + R*10 + E
		money := M*10000 + O*1000 + N*100 + E*10 + Y
		return send + more, money
	}

	var solve func(idx int, vals [8]int) bool
	solve = func(idx int, vals [8]int) bool {
		if idx == 8 {
			sum, target := assign(vals)
			return sum == target
		}

		for d := 0; d <= 9; d++ {
			if used[d] { continue }
			// S and M can't be 0
			if (idx == 0 || idx == 4) && d == 0 { continue }

			used[d] = true
			vals[idx] = d
			if solve(idx+1, vals) {
				// Print solution
				if idx == 0 {
					sum, _ := assign(vals)
					send := vals[0]*1000 + vals[1]*100 + vals[2]*10 + vals[3]
					more := vals[4]*1000 + vals[5]*100 + vals[6]*10 + vals[1]
					_ = sum
					fmt.Printf("  %d + %d = %d\n", send, more, sum)
				}
				return true
			}
			used[d] = false
		}
		return false
	}

	fmt.Println("SEND + MORE = MONEY")
	var vals [8]int
	solve(0, vals)
}

func main() {
	solveCryptarithmetic()
	// 9567 + 1085 = 10652
}
```

**Textual Figure:**

```
SEND + MORE = MONEY   Cryptarithmetic CSP

  Variables:  S  E  N  D  M  O  R  Y  (8 variables)
  Domain:     0–9 for each
  Constraints:
    • All digits different
    • S ≠ 0, M ≠ 0  (leading digits)
    • S×1000 + E×100 + N×10 + D
      + M×1000 + O×100 + R×10 + E
      = M×10000 + O×1000 + N×100 + E×10 + Y

  Backtracking search (assign digits one variable at a time):
  ──────────────────────────────────────────────
    S=1  → try, but M can't also be 1 (M=1 ✂)
    S=9  → used[9]=T
      E=0 → but check equation partial...
      E=5 → used[5]=T
        N=6 → used[6]=T
          D=7 → used[7]=T
            M=0 ││ (M can't be 0) ││→ skip
            M=1 → used[1]=T
              O=0 → used[0]=T
                R=8 → used[8]=T
                  Y=2 → check equation:

     S E N D       9 5 6 7
   + M O R E     + 1 0 8 5
   ────────     ────────
   M O N E Y     1 0 6 5 2

   9567 + 1085 = 10652  ✓ SOLUTION FOUND!
```

---

## Example 5: CSP with Forward Checking (Sudoku Enhanced)

```go
package main

import "fmt"

type SudokuCSP struct {
	board   [9][9]byte
	domains [9][9]map[byte]bool
}

func NewSudokuCSP(board [9][9]byte) *SudokuCSP {
	csp := &SudokuCSP{board: board}
	// Initialize domains
	for r := 0; r < 9; r++ {
		for c := 0; c < 9; c++ {
			if board[r][c] == '.' {
				csp.domains[r][c] = map[byte]bool{}
				for d := byte('1'); d <= '9'; d++ {
					if isValidCSP(&board, r, c, d) {
						csp.domains[r][c][d] = true
					}
				}
			}
		}
	}
	return csp
}

func isValidCSP(board *[9][9]byte, row, col int, d byte) bool {
	for i := 0; i < 9; i++ {
		if board[row][i] == d || board[i][col] == d { return false }
		r, c := 3*(row/3)+i/3, 3*(col/3)+i%3
		if board[r][c] == d { return false }
	}
	return true
}

func (csp *SudokuCSP) Solve() bool {
	// MRV: find cell with smallest domain
	minSize, bestR, bestC := 10, -1, -1
	for r := 0; r < 9; r++ {
		for c := 0; c < 9; c++ {
			if csp.board[r][c] == '.' {
				if len(csp.domains[r][c]) < minSize {
					minSize = len(csp.domains[r][c])
					bestR, bestC = r, c
				}
			}
		}
	}

	if bestR == -1 { return true }
	if minSize == 0 { return false } // empty domain = dead end

	// Try each value in domain
	for d := range csp.domains[bestR][bestC] {
		// Forward check: save removed values
		removed := csp.assign(bestR, bestC, d)
		if removed != nil {
			if csp.Solve() { return true }
			csp.unassign(bestR, bestC, d, removed)
		}
	}
	return false
}

func (csp *SudokuCSP) assign(r, c int, d byte) map[[2]int]byte {
	csp.board[r][c] = d
	removed := map[[2]int]byte{}

	// Forward checking: remove d from peers' domains
	for i := 0; i < 9; i++ {
		if csp.board[r][i] == '.' && csp.domains[r][i][d] {
			delete(csp.domains[r][i], d)
			removed[[2]int{r, i}] = d
			if len(csp.domains[r][i]) == 0 { csp.undo(removed); csp.board[r][c] = '.'; return nil }
		}
		if csp.board[i][c] == '.' && csp.domains[i][c][d] {
			delete(csp.domains[i][c], d)
			removed[[2]int{i, c}] = d
			if len(csp.domains[i][c]) == 0 { csp.undo(removed); csp.board[r][c] = '.'; return nil }
		}
		br, bc := 3*(r/3)+i/3, 3*(c/3)+i%3
		if csp.board[br][bc] == '.' && csp.domains[br][bc][d] {
			delete(csp.domains[br][bc], d)
			removed[[2]int{br, bc}] = d
			if len(csp.domains[br][bc]) == 0 { csp.undo(removed); csp.board[r][c] = '.'; return nil }
		}
	}
	return removed
}

func (csp *SudokuCSP) undo(removed map[[2]int]byte) {
	for pos, d := range removed { csp.domains[pos[0]][pos[1]][d] = true }
}

func (csp *SudokuCSP) unassign(r, c int, d byte, removed map[[2]int]byte) {
	csp.board[r][c] = '.'
	csp.undo(removed)
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

	csp := NewSudokuCSP(board)
	if csp.Solve() {
		fmt.Println("Solved with forward checking + MRV:")
		for _, row := range csp.board { fmt.Println(string(row[:])) }
	}
}
```

**Textual Figure:**

```
Forward Checking + MRV for Sudoku

  Initial domains (partial, first row):
  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
  │  5  │  3  │ {4} │{6} │  7  │{8} │{9} │{1} │{2} │
  │fixed│fixed│ dom │dom │fixed│dom │dom │dom │dom │
  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

  MRV Selection (choose variable with smallest domain):
  ─────────────────────────────────────────────
    Cell (0,2): domain={1,2,4}     size=3
    Cell (0,3): domain={2,6}       size=2  ← MRV picks this!
    Cell (0,5): domain={4,6,8}     size=3

  Forward checking after assigning (0,3)=6:
  ─────────────────────────────────────────────
    Remove 6 from:
      • Row 0 peers:   (0,2):{1,2,4}→{1,2,4} (no 6)
      • Col 3 peers:   (3,3):{7,9}→{7,9} (no 6)
      • Box(0,1) peers: (1,3): already=1 (fixed)

    If any domain becomes empty → backtrack immediately!
    (saves exploring entire subtree)

  Benefit: forward checking detects dead ends BEFORE
           recursing deeper, vs basic backtracking which
           only checks at assignment time.
```

---

## Example 6: Magic Square

```go
package main

import "fmt"

func solveMagicSquare(n int) [][]int {
	target := n * (n*n + 1) / 2
	grid := make([][]int, n)
	for i := range grid { grid[i] = make([]int, n) }
	used := make([]bool, n*n+1)

	var solve func(pos int) bool
	solve = func(pos int) bool {
		if pos == n*n {
			// Verify all rows, cols, diagonals sum to target
			return verifyMagic(grid, n, target)
		}

		r, c := pos/n, pos%n
		for num := 1; num <= n*n; num++ {
			if used[num] { continue }

			grid[r][c] = num
			used[num] = true

			// Constraint: check partial row/col sums
			if checkPartial(grid, n, r, c, target) {
				if solve(pos + 1) { return true }
			}

			grid[r][c] = 0
			used[num] = false
		}
		return false
	}

	solve(0)
	return grid
}

func checkPartial(grid [][]int, n, r, c, target int) bool {
	// Check completed row
	if c == n-1 {
		sum := 0
		for j := 0; j < n; j++ { sum += grid[r][j] }
		if sum != target { return false }
	}
	// Check completed column
	if r == n-1 {
		sum := 0
		for i := 0; i < n; i++ { sum += grid[i][c] }
		if sum != target { return false }
	}
	return true
}

func verifyMagic(grid [][]int, n, target int) bool {
	d1, d2 := 0, 0
	for i := 0; i < n; i++ {
		d1 += grid[i][i]
		d2 += grid[i][n-1-i]
	}
	return d1 == target && d2 == target
}

func main() {
	grid := solveMagicSquare(3)
	fmt.Println("3x3 Magic Square (target=15):")
	for _, row := range grid { fmt.Println(row) }
}
```

**Textual Figure:**

```
3×3 Magic Square CSP   target = 3×(9+1)/2 = 15

  Variables: 9 cells, Domain: {1..9}, Constraints: rows/cols/diags = 15

  Backtracking with partial constraint checking:
  ──────────────────────────────────────────────
  pos=0: try 1       pos=0: try 2
  ┌───┬───┬───┐     ┌───┬───┬───┐
  │ 1 │   │   │     │ 2 │   │   │
  ├───┼───┼───┤     ├───┼───┼───┤
  │   │   │   │     │   │   │   │   ...
  ├───┼───┼───┤     ├───┼───┼───┤
  │   │   │   │     │   │   │   │
  └───┴───┴───┘     └───┴───┴───┘
    pos=1: try 2,3...    fill remaining...
    When row complete (pos=2), check sum:
      Row 0: 1+2+3=6 ≠ 15 ✂ PRUNE!

  Solution found:
  ┌───┬───┬───┐
  │ 2 │ 7 │ 6 │  row=15 ✓
  ├───┼───┼───┤
  │ 9 │ 5 │ 1 │  row=15 ✓
  ├───┼───┼───┤
  │ 4 │ 3 │ 8 │  row=15 ✓
  └───┴───┴───┘
   c=15 c=15 c=15
   d1=2+5+8=15 ✓   d2=6+5+4=15 ✓
```

---

## Example 7: Latin Square

```go
package main

import "fmt"

// Latin square: n×n grid where each number 1..n appears exactly once per row and column

func solveLatinSquare(n int) [][]int {
	grid := make([][]int, n)
	for i := range grid { grid[i] = make([]int, n) }

	rowUsed := make([][]bool, n)
	colUsed := make([][]bool, n)
	for i := 0; i < n; i++ {
		rowUsed[i] = make([]bool, n+1)
		colUsed[i] = make([]bool, n+1)
	}

	var solve func(pos int) bool
	solve = func(pos int) bool {
		if pos == n*n { return true }

		r, c := pos/n, pos%n
		for val := 1; val <= n; val++ {
			if rowUsed[r][val] || colUsed[c][val] { continue }

			grid[r][c] = val
			rowUsed[r][val] = true
			colUsed[c][val] = true

			if solve(pos + 1) { return true }

			grid[r][c] = 0
			rowUsed[r][val] = false
			colUsed[c][val] = false
		}
		return false
	}

	solve(0)
	return grid
}

func main() {
	for n := 3; n <= 5; n++ {
		grid := solveLatinSquare(n)
		fmt.Printf("%dx%d Latin Square:\n", n, n)
		for _, row := range grid { fmt.Println(row) }
		fmt.Println()
	}
}
```

**Textual Figure:**

```
3×3 Latin Square CSP
  Variables: 9 cells, Domain: {1,2,3}
  Constraints: each value appears once per row AND once per column

  Backtracking trace:
  ────────────────────────────────────────────────
  pos(0,0)=1    pos(0,1)=2    pos(0,2)=3
  ┌───┬───┬───┐   row constraint: each val once
  │ 1 │ 2 │ 3 │   col constraint: each val once
  ├───┼───┼───┤
  │   │   │   │   pos(1,0): try 1 ││ col 0 has 1 ││→ ││
  └───┴───┴───┘            try 2 ✓ (row OK, col OK)

  pos(1,0)=2    pos(1,1): try 1││col has 2││ try 3 ✓
  ┌───┬───┬───┐   pos(1,2): try 1 ✓
  │ 1 │ 2 │ 3 │
  ├───┼───┼───┤   Row 2: forced by columns
  │ 2 │ 3 │ 1 │   pos(2,0)=3, pos(2,1)=1, pos(2,2)=2
  ├───┼───┼───┤
  │ 3 │ 1 │ 2 │ ✓
  └───┴───┴───┘

  Verify: Each row has {1,2,3} ✓   Each col has {1,2,3} ✓
```

---

## Example 8: Constraint Propagation — Arc Consistency

```go
package main

import "fmt"

// Simplified arc consistency (AC-3 style) for demonstration

type CSP struct {
	n       int
	domains []map[int]bool
	adj     [][]int
}

func NewCSP(n int, adj [][]int) *CSP {
	csp := &CSP{n: n, adj: adj}
	csp.domains = make([]map[int]bool, n)
	for i := 0; i < n; i++ {
		csp.domains[i] = map[int]bool{}
		for c := 1; c <= 3; c++ { // 3 colors
			csp.domains[i][c] = true
		}
	}
	return csp
}

func (csp *CSP) AC3() bool {
	queue := [][2]int{}
	for u := 0; u < csp.n; u++ {
		for _, v := range csp.adj[u] {
			queue = append(queue, [2]int{u, v})
		}
	}

	for len(queue) > 0 {
		arc := queue[0]
		queue = queue[1:]
		u, v := arc[0], arc[1]

		if csp.revise(u, v) {
			if len(csp.domains[u]) == 0 { return false }
			for _, w := range csp.adj[u] {
				if w != v { queue = append(queue, [2]int{w, u}) }
			}
		}
	}
	return true
}

func (csp *CSP) revise(u, v int) bool {
	revised := false
	for val := range csp.domains[u] {
		// Check if any value in v's domain is consistent with val
		hasSupport := false
		for vVal := range csp.domains[v] {
			if vVal != val { hasSupport = true; break }
		}
		if !hasSupport {
			delete(csp.domains[u], val)
			revised = true
		}
	}
	return revised
}

func main() {
	adj := [][]int{
		{1, 2},    // 0
		{0, 2, 3}, // 1
		{0, 1, 3}, // 2
		{1, 2},    // 3
	}

	csp := NewCSP(4, adj)
	fmt.Println("Before AC-3:")
	for i, d := range csp.domains { fmt.Printf("  Var %d: %v\n", i, d) }

	csp.AC3()
	fmt.Println("\nAfter AC-3:")
	for i, d := range csp.domains { fmt.Printf("  Var %d: %v\n", i, d) }
}
```

**Textual Figure:**

```
Graph Coloring with AC-3 (3 colors, 4 vertices)

  Graph:  0───1          Initial domains:
          |\ /|           Var 0: {1, 2, 3}
          | X |           Var 1: {1, 2, 3}
          |/ \|           Var 2: {1, 2, 3}
          2───3           Var 3: {1, 2, 3}

  AC-3 arc queue: (0,1)(0,2)(1,0)(1,2)(1,3)(2,0)(2,1)(2,3)(3,1)(3,2)
  ─────────────────────────────────────────────────────
  Process arc (0,1): revise Var 0 w.r.t. Var 1
    For each val in dom(0), is there a DIFFERENT val in dom(1)?
    val=1: dom(1) has {2,3} ≠ 1 ✓  (has support)
    val=2: dom(1) has {1,3} ≠ 2 ✓
    val=3: dom(1) has {1,2} ≠ 3 ✓
    No change → dom(0) stays {1,2,3}

  Process arc (0,2): same analysis, no change
  ... (all arcs processed, no domain reduction in this case)

  After AC-3:
    Var 0: {1, 2, 3}   (no reduction → need backtracking)
    Var 1: {1, 2, 3}
    Var 2: {1, 2, 3}
    Var 3: {1, 2, 3}

  Note: AC-3 is most effective when initial assignments
        restrict domains, e.g., if Var 0 is pre-assigned
        to 1, then dom(1) and dom(2) lose value 1.
```

---

## Example 9: Job Scheduling CSP

```go
package main

import "fmt"

// Variables: jobs
// Domain: time slots
// Constraints: jobs sharing a resource can't overlap

type Job struct {
	id       int
	duration int
	resource int
}

func scheduleJobs(jobs []Job, slots int) []int {
	n := len(jobs)
	assignment := make([]int, n) // start time for each job
	for i := range assignment { assignment[i] = -1 }

	var solve func(j int) bool
	solve = func(j int) bool {
		if j == n { return true }

		for t := 0; t <= slots-jobs[j].duration; t++ {
			if canSchedule(jobs, assignment, j, t) {
				assignment[j] = t
				if solve(j + 1) { return true }
				assignment[j] = -1
			}
		}
		return false
	}

	if solve(0) { return assignment }
	return nil
}

func canSchedule(jobs []Job, assignment []int, j, t int) bool {
	for i := 0; i < j; i++ {
		if assignment[i] == -1 { continue }
		if jobs[i].resource != jobs[j].resource { continue }

		// Check overlap
		s1, e1 := assignment[i], assignment[i]+jobs[i].duration
		s2, e2 := t, t+jobs[j].duration
		if s1 < e2 && s2 < e1 { return false }
	}
	return true
}

func main() {
	jobs := []Job{
		{0, 2, 0}, // job 0: duration 2, resource 0
		{1, 3, 0}, // job 1: duration 3, resource 0
		{2, 1, 1}, // job 2: duration 1, resource 1
		{3, 2, 1}, // job 3: duration 2, resource 1
		{4, 1, 0}, // job 4: duration 1, resource 0
	}

	result := scheduleJobs(jobs, 6)
	fmt.Println("Job schedule (start times):", result)
	for i, t := range result {
		fmt.Printf("  Job %d: [%d, %d) on resource %d\n",
			i, t, t+jobs[i].duration, jobs[i].resource)
	}
}
```

**Textual Figure:**

```
Job Scheduling CSP   (5 jobs, 6 time slots, 2 resources)

  Jobs: J0(dur=2,R0) J1(dur=3,R0) J2(dur=1,R1) J3(dur=2,R1) J4(dur=1,R0)

  Backtracking: assign start times, check overlap on same resource
  ───────────────────────────────────────────────────────
    J0=0: [0,2) on R0
    J1=0: [0,3) on R0 ││ J0 [0,2) overlaps! ✂
    J1=2: [2,5) on R0 ✓ (no overlap with J0)
    J2=0: [0,1) on R1 ✓ (different resource)
    J3=0: [0,2) on R1 ││ J2 [0,1) overlaps! ││
    J3=1: [1,3) on R1 ✓
    J4=0: [0,1) on R0 ││ J0 [0,2) overlaps! ││
    J4=5: [5,6) on R0 ✓

  Final Schedule:
  Resource 0: │J0│J0│J1│J1│J1│J4│
  Resource 1: │J2│J3│J3│  │  │  │
              0  1  2  3  4  5  6

  Result: J0=[0,2) J1=[2,5) J2=[0,1) J3=[1,3) J4=[5,6)
```

---

## Example 10: CSP Concepts Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Constraint Satisfaction Summary ===")
	fmt.Println()

	techniques := []struct{ technique, description string }{
		{"Backtracking", "Try values; undo on failure"},
		{"Forward checking", "Remove inconsistent values from peers' domains"},
		{"Arc consistency (AC-3)", "Propagate constraints between variable pairs"},
		{"MRV heuristic", "Choose variable with fewest remaining values"},
		{"LCV heuristic", "Try value that rules out fewest for neighbors"},
		{"Constraint propagation", "Infer forced assignments from constraints"},
		{"Domain splitting", "Split large domains for parallel search"},
	}

	for i, t := range techniques {
		fmt.Printf("%d. %-25s %s\n", i+1, t.technique, t.description)
	}

	fmt.Println()
	fmt.Println("Classic CSP problems:")
	fmt.Println("  • N-Queens: columns + diagonals constraints")
	fmt.Println("  • Sudoku:   row + column + box uniqueness")
	fmt.Println("  • Graph coloring: adjacent vertices differ")
	fmt.Println("  • Scheduling: resource + time constraints")
	fmt.Println("  • Cryptarithmetic: digit uniqueness + equation")
}
```

**Textual Figure:**

```
  CSP Solution Techniques Hierarchy
  ═══════════════════════════════════════════════════════════════

                ┌──────────────────────┐
                │  1. BACKTRACKING       │
                │  (base technique)      │
                └──────────┬───────────┘
                           │
              ┌───────────┼───────────┐
              │                         │
  ┌───────────┴───────┐   ┌───────┴───────────┐
  │ 2. FORWARD CHECKING   │   │ 3. ARC CONSISTENCY    │
  │ (prune on assign)    │   │ (AC-3 propagation)    │
  └────────────────────┘   └────────────────────┘

  Enhanced with heuristics:
  ┌────────────────────┬────────────────────┐
  │ MRV (variable order) │ LCV (value order)    │
  │ Pick var with fewest │ Try val that rules   │
  │ remaining values     │ out fewest for       │
  │ → fail-fast          │ neighbors → succeed  │
  └────────────────────┴────────────────────┘

  Classic CSP Problems:             Technique Used:
    N-Queens       ────────────→  Backtracking + col/diag check
    Sudoku         ────────────→  Forward checking + MRV
    Graph Coloring ────────────→  AC-3 + backtracking
    Scheduling     ────────────→  Backtracking + overlap check
    Cryptarithmetic────────────→  Backtracking + arithmetic test
```

---

## Key Takeaways

1. CSPs are defined by variables, domains, and constraints
2. Backtracking = systematic trial with undo; the base CSP solver
3. Forward checking prunes domains immediately after each assignment
4. MRV: always pick the most constrained variable next (fail-fast)
5. Arc consistency (AC-3) propagates constraints to reduce domains before/during search
6. Choosing the right variable ordering and value ordering dramatically reduces search space

> **Next up:** Backtracking vs Brute Force →
