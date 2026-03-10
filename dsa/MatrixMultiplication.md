# Phase 21: Math & Geometry — Matrix Multiplication

## Overview

Matrix multiplication is fundamental for solving linear recurrences, graph path counting, and transformations. Combined with fast exponentiation, it solves recurrences in O(k³ log n).

| Operation | Complexity | Use Case |
|-----------|-----------|----------|
| Standard multiply | O(n³) | General A×B |
| Matrix exponentiation | O(k³ log n) | Linear recurrences |
| Strassen's | O(n^2.807) | Large matrices |
| Chain multiplication | DP O(n³) | Optimal parenthesization |

---

## Example 1: Standard Matrix Multiplication

```go
package main

import "fmt"

type Matrix [][]int

func multiply(a, b Matrix) Matrix {
	m, n, p := len(a), len(a[0]), len(b[0])
	c := make(Matrix, m)
	for i := range c {
		c[i] = make([]int, p)
		for j := 0; j < p; j++ {
			for k := 0; k < n; k++ {
				c[i][j] += a[i][k] * b[k][j]
			}
		}
	}
	return c
}

func printMatrix(name string, m Matrix) {
	fmt.Printf("%s:\n", name)
	for _, row := range m { fmt.Printf("  %v\n", row) }
}

func main() {
	a := Matrix{{1, 2, 3}, {4, 5, 6}}
	b := Matrix{{7, 8}, {9, 10}, {11, 12}}

	printMatrix("A (2x3)", a)
	printMatrix("B (3x2)", b)
	printMatrix("A × B (2x2)", multiply(a, b))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Standard Matrix Multiplication: A(2×3) × B(3×2) │
├─────────────────────────────────────────────────┤
│  A:        B:        C = A×B:                   │
│  ┌1 2 3┐  ┌ 7  8┐   ┌ 58  64┐                  │
│  │4 5 6┘  │ 9 10│   │139 154┘                  │
│           │11 12┘                               │
│                                                 │
│  C[0][0] = 1×7 + 2×9 + 3×11 = 7+18+33 = 58    │
│  C[0][1] = 1×8 + 2×10 + 3×12 = 8+20+36 = 64   │
│  C[1][0] = 4×7 + 5×9 + 6×11 = 28+45+66 = 139  │
│  C[1][1] = 4×8 + 5×10 + 6×12 = 32+50+72 = 154 │
│                                                 │
│  O(m×n×p): m rows of A, n=shared, p cols of B  │
└─────────────────────────────────────────────────┘
```

---

## Example 2: Matrix Exponentiation (Generic)

```go
package main

import "fmt"

const MOD = 1_000_000_007

type Matrix [][]int

func newIdentity(n int) Matrix {
	m := make(Matrix, n)
	for i := range m {
		m[i] = make([]int, n)
		m[i][i] = 1
	}
	return m
}

func matMul(a, b Matrix) Matrix {
	n := len(a)
	c := make(Matrix, n)
	for i := range c {
		c[i] = make([]int, n)
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % MOD
			}
		}
	}
	return c
}

func matPow(m Matrix, exp int) Matrix {
	n := len(m)
	result := newIdentity(n)
	for exp > 0 {
		if exp&1 == 1 { result = matMul(result, m) }
		m = matMul(m, m)
		exp >>= 1
	}
	return result
}

func main() {
	// Fibonacci: [F(n+1), F(n)] = [[1,1],[1,0]]^n * [1, 0]
	fib := Matrix{{1, 1}, {1, 0}}
	for _, n := range []int{5, 10, 50, 1000000} {
		result := matPow(fib, n)
		fmt.Printf("  fib(%d) mod 10^9+7 = %d\n", n, result[0][1])
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Matrix Exponentiation for Fibonacci            │
├─────────────────────────────────────────────────┤
│  [F(n+1)]   [1 1]^n   [1]                       │
│  [F(n)  ] = [1 0]   × [0]                       │
│                                                 │
│  Binary exponentiation of matrix:               │
│  M^10 = M^8 × M^2  (10 = 1010₂)               │
│                                                 │
│  Step: exp=10 (1010)                            │
│    bit 1: result ×= M,  M = M²                  │
│    bit 0: skip,         M = M⁴                  │
│    bit 1: result ×= M⁴, M = M⁸                  │
│    bit 0: skip                                  │
│                                                 │
│  O(k³ log n) — k=2 for Fibonacci               │
│  Computes F(10⁶) mod 10⁹+7 in milliseconds     │
└─────────────────────────────────────────────────┘
```

---

## Example 3: Count Paths in Graph (Adjacency Matrix Power)

```go
package main

import "fmt"

type Matrix [][]int

func matMul(a, b Matrix, mod int) Matrix {
	n := len(a)
	c := make(Matrix, n)
	for i := range c {
		c[i] = make([]int, n)
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % mod
			}
		}
	}
	return c
}

func matPow(m Matrix, exp, mod int) Matrix {
	n := len(m)
	result := make(Matrix, n)
	for i := range result {
		result[i] = make([]int, n)
		result[i][i] = 1
	}
	for exp > 0 {
		if exp&1 == 1 { result = matMul(result, m, mod) }
		m = matMul(m, m, mod)
		exp >>= 1
	}
	return result
}

func main() {
	// Graph: 0-1, 0-2, 1-2, 1-3, 2-3
	adj := Matrix{
		{0, 1, 1, 0},
		{1, 0, 1, 1},
		{1, 1, 0, 1},
		{0, 1, 1, 0},
	}

	mod := 1_000_000_007
	for _, k := range []int{1, 2, 3, 5, 10} {
		result := matPow(adj, k, mod)
		fmt.Printf("  Paths of length %d from 0→3: %d\n", k, result[0][3])
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Graph Path Counting: Aᵏ                        │
├─────────────────────────────────────────────────┤
│  Graph:  0 ── 1 ── 3                            │
│          │ ╲ │ ╱                                 │
│          └── 2 ─┘                                 │
│                                                 │
│  Adj matrix A:      A²[0][3] = paths len 2     │
│  ┌0 1 1 0┐           0→1→3 or 0→2→3           │
│  │1 0 1 1│           = 2 paths                   │
│  │1 1 0 1│                                       │
│  │0 1 1 0┘          A³[0][3] = paths len 3      │
│                      0→1→2→3, 0→2→1→3,         │
│                      0→1→0→2(???), etc         │
│                      = 5 paths                   │
│                                                 │
│  Aᵏ[i][j] = # paths of exactly length k          │
└─────────────────────────────────────────────────┘
```

---

## Example 4: Linear Recurrence Solver

```go
package main

import "fmt"

const MOD = 1_000_000_007

type Matrix [][]int

func matMul(a, b Matrix) Matrix {
	n := len(a)
	c := make(Matrix, n)
	for i := range c {
		c[i] = make([]int, n)
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % MOD
			}
		}
	}
	return c
}

func matPow(m Matrix, exp int) Matrix {
	n := len(m)
	result := make(Matrix, n)
	for i := range result { result[i] = make([]int, n); result[i][i] = 1 }
	for exp > 0 {
		if exp&1 == 1 { result = matMul(result, m) }
		m = matMul(m, m); exp >>= 1
	}
	return result
}

// Solve: f(n) = c1*f(n-1) + c2*f(n-2) + ... + ck*f(n-k)
func solveRecurrence(coeffs []int, initial []int, n int) int {
	k := len(coeffs)
	if n < k { return initial[n] }

	// Build transition matrix
	m := make(Matrix, k)
	for i := range m { m[i] = make([]int, k) }
	for j := 0; j < k; j++ { m[0][j] = coeffs[j] % MOD }
	for i := 1; i < k; i++ { m[i][i-1] = 1 }

	result := matPow(m, n-k+1)
	ans := 0
	for j := 0; j < k; j++ {
		ans = (ans + result[0][j]*initial[k-1-j]) % MOD
	}
	return ans
}

func main() {
	// Fibonacci: f(n) = f(n-1) + f(n-2)
	fmt.Println("Fibonacci:")
	for _, n := range []int{5, 10, 100} {
		fmt.Printf("  f(%d) = %d\n", n, solveRecurrence([]int{1, 1}, []int{0, 1}, n))
	}

	// Tribonacci: f(n) = f(n-1) + f(n-2) + f(n-3)
	fmt.Println("Tribonacci:")
	for _, n := range []int{5, 10, 100} {
		fmt.Printf("  f(%d) = %d\n", n, solveRecurrence([]int{1, 1, 1}, []int{0, 0, 1}, n))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Linear Recurrence via Matrix Exponentiation    │
├─────────────────────────────────────────────────┤
│  f(n) = c₁f(n-1) + c₂f(n-2) + ... + cₖf(n-k)  │
│                                                 │
│  Transition matrix M (k×k):                     │
│  ┌c₁ c₂ c₃ ... cₖ┐                              │
│  │1  0  0  ... 0 │                              │
│  │0  1  0  ... 0 │  shift row                   │
│  │:           : │                              │
│  │0  0  ... 1  0 ┘                              │
│                                                 │
│  Tribonacci: f(n)=f(n-1)+f(n-2)+f(n-3)         │
│  M = ┌1 1 1┐   Initial = [0, 0, 1]              │
│      │1 0 0│   M^(n-2) × [1,0,0]^T              │
│      │0 1 0┘                                     │
│                                                 │
│  O(k³ log n) instead of O(n)                    │
└─────────────────────────────────────────────────┘
```

---

## Example 5: Matrix Chain Multiplication (DP)

```go
package main

import (
	"fmt"
	"math"
)

// Minimum cost of multiplying chain of matrices
// dims[i-1] x dims[i] are dimensions of matrix i
func matrixChainOrder(dims []int) (int, string) {
	n := len(dims) - 1
	dp := make([][]int, n)
	split := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		split[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt64 }
		dp[i][i] = 0
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			for k := i; k < j; k++ {
				cost := dp[i][k] + dp[k+1][j] + dims[i]*dims[k+1]*dims[j+1]
				if cost < dp[i][j] {
					dp[i][j] = cost
					split[i][j] = k
				}
			}
		}
	}

	// Reconstruct parenthesization
	var build func(i, j int) string
	build = func(i, j int) string {
		if i == j { return fmt.Sprintf("M%d", i+1) }
		return "(" + build(i, split[i][j]) + " × " + build(split[i][j]+1, j) + ")"
	}

	return dp[0][n-1], build(0, n-1)
}

func main() {
	dims := []int{10, 30, 5, 60}
	cost, order := matrixChainOrder(dims)
	fmt.Printf("Dimensions: %v\n", dims)
	fmt.Printf("Min multiplications: %d\n", cost)
	fmt.Printf("Optimal order: %s\n", order)

	dims2 := []int{40, 20, 30, 10, 30}
	cost2, order2 := matrixChainOrder(dims2)
	fmt.Printf("\nDimensions: %v\n", dims2)
	fmt.Printf("Min multiplications: %d\n", cost2)
	fmt.Printf("Optimal order: %s\n", order2)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Matrix Chain Multiplication DP                 │
├─────────────────────────────────────────────────┤
│  dims = [10, 30, 5, 60]                         │
│  M1(10×30)  M2(30×5)  M3(5×60)                │
│                                                 │
│  Option A: (M1 × M2) × M3                       │
│    M1×M2: 10×30×5 = 1500                       │
│    × M3:  10×5×60 = 3000                       │
│    Total: 4500  ★ optimal                       │
│                                                 │
│  Option B: M1 × (M2 × M3)                       │
│    M2×M3: 30×5×60 = 9000                       │
│    M1×:   10×30×60= 18000                      │
│    Total: 27000  ← 6× worse!                    │
│                                                 │
│  DP: O(n³) for optimal parenthesization         │
└─────────────────────────────────────────────────┘
```

---

## Example 6: Spiral Matrix Traversal (LeetCode 54)

```go
package main

import "fmt"

func spiralOrder(matrix [][]int) []int {
	if len(matrix) == 0 { return nil }

	m, n := len(matrix), len(matrix[0])
	result := make([]int, 0, m*n)
	top, bottom, left, right := 0, m-1, 0, n-1

	for top <= bottom && left <= right {
		for j := left; j <= right; j++ { result = append(result, matrix[top][j]) }
		top++
		for i := top; i <= bottom; i++ { result = append(result, matrix[i][right]) }
		right--
		if top <= bottom {
			for j := right; j >= left; j-- { result = append(result, matrix[bottom][j]) }
			bottom--
		}
		if left <= right {
			for i := bottom; i >= top; i-- { result = append(result, matrix[i][left]) }
			left++
		}
	}
	return result
}

func main() {
	matrices := [][][]int{
		{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}},
		{{1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12}},
	}

	for _, mat := range matrices {
		fmt.Printf("  Matrix: %v\n", mat)
		fmt.Printf("  Spiral: %v\n\n", spiralOrder(mat))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Spiral Matrix Traversal                        │
├─────────────────────────────────────────────────┤
│  ┌───┬───┬───┐                                   │
│  │ 1→│2→│3 │  → right across top               │
│  ├───┬───┬───┤                                   │
│  │ 4 │ 5 │ 6↓│  ↓ down right side               │
│  ├───┬───┬───┤                                   │
│  │ 7 │←8←│9 │  ← left across bottom             │
│  └───┴───┴───┘                                   │
│  ↑ 4 up left side                               │
│  → 5 center                                     │
│                                                 │
│  Order: [1,2,3,6,9,8,7,4,5]                    │
│  Use 4 boundaries: top,bottom,left,right        │
│  Shrink inward after each sweep                 │
└─────────────────────────────────────────────────┘
```

---

## Example 7: Rotate Matrix 90° (LeetCode 48)

```go
package main

import "fmt"

func rotate(matrix [][]int) {
	n := len(matrix)

	// Transpose
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
		}
	}

	// Reverse each row
	for i := 0; i < n; i++ {
		for l, r := 0, n-1; l < r; l, r = l+1, r-1 {
			matrix[i][l], matrix[i][r] = matrix[i][r], matrix[i][l]
		}
	}
}

func printMatrix(name string, m [][]int) {
	fmt.Printf("%s:\n", name)
	for _, row := range m { fmt.Printf("  %v\n", row) }
}

func main() {
	m := [][]int{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}
	printMatrix("Original", m)
	rotate(m)
	printMatrix("Rotated 90° CW", m)

	m2 := [][]int{{5, 1, 9, 11}, {2, 4, 8, 10}, {13, 3, 6, 7}, {15, 14, 12, 16}}
	printMatrix("\nOriginal", m2)
	rotate(m2)
	printMatrix("Rotated 90° CW", m2)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Rotate Matrix 90° Clockwise                    │
├─────────────────────────────────────────────────┤
│  Step 1: Transpose (swap across diagonal)       │
│  ┌1 2 3┐      ┌1 4 7┐                           │
│  │4 5 6│  →   │2 5 8│                           │
│  │7 8 9┘      │3 6 9┘                           │
│                                                 │
│  Step 2: Reverse each row                       │
│  ┌1 4 7┐      ┌7 4 1┐                           │
│  │2 5 8│  →   │8 5 2│                           │
│  │3 6 9┘      │9 6 3┘                           │
│                                                 │
│  In-place: O(n²) time, O(1) space               │
│  90° CCW: transpose + reverse columns           │
│  180°: reverse rows then reverse columns        │
└─────────────────────────────────────────────────┘
```

---

## Example 8: Set Matrix Zeroes (LeetCode 73)

```go
package main

import "fmt"

func setZeroes(matrix [][]int) {
	m, n := len(matrix), len(matrix[0])
	firstRowZero, firstColZero := false, false

	// Check first row/col
	for j := 0; j < n; j++ {
		if matrix[0][j] == 0 { firstRowZero = true; break }
	}
	for i := 0; i < m; i++ {
		if matrix[i][0] == 0 { firstColZero = true; break }
	}

	// Use first row/col as markers
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			if matrix[i][j] == 0 {
				matrix[i][0] = 0
				matrix[0][j] = 0
			}
		}
	}

	// Zero out cells
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			if matrix[i][0] == 0 || matrix[0][j] == 0 {
				matrix[i][j] = 0
			}
		}
	}

	// Handle first row/col
	if firstRowZero {
		for j := 0; j < n; j++ { matrix[0][j] = 0 }
	}
	if firstColZero {
		for i := 0; i < m; i++ { matrix[i][0] = 0 }
	}
}

func main() {
	m := [][]int{{1, 1, 1}, {1, 0, 1}, {1, 1, 1}}
	fmt.Println("Before:", m)
	setZeroes(m)
	fmt.Println("After: ", m)

	m2 := [][]int{{0, 1, 2, 0}, {3, 4, 5, 2}, {1, 3, 1, 5}}
	fmt.Println("\nBefore:", m2)
	setZeroes(m2)
	fmt.Println("After: ", m2)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Set Matrix Zeroes: O(1) Space                  │
├─────────────────────────────────────────────────┤
│  Before:        After:                          │
│  ┌1 1 1┐       ┌1 0 1┐                          │
│  │1 0 1│  →    │0 0 0│                          │
│  │1 1 1┘       │1 0 1┘                          │
│                                                 │
│  Strategy: use first row/col as markers         │
│                                                 │
│  1. Record if first row/col have zeros          │
│  2. Scan: if cell=0, mark row[i][0]=0,col[0][j]=0│
│  3. Zero cells per markers (skip first row/col) │
│  4. Handle first row/col last                   │
│                                                 │
│  O(mn) time, O(1) extra space                   │
└─────────────────────────────────────────────────┘
```

---

## Example 9: Toeplitz Matrix Check (LeetCode 766)

```go
package main

import "fmt"

func isToeplitz(matrix [][]int) bool {
	m, n := len(matrix), len(matrix[0])
	for i := 1; i < m; i++ {
		for j := 1; j < n; j++ {
			if matrix[i][j] != matrix[i-1][j-1] {
				return false
			}
		}
	}
	return true
}

func main() {
	tests := [][][]int{
		{{1, 2, 3, 4}, {5, 1, 2, 3}, {9, 5, 1, 2}},
		{{1, 2}, {2, 2}},
	}

	for _, m := range tests {
		fmt.Printf("  %v → Toeplitz: %v\n", m, isToeplitz(m))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Toeplitz Matrix Check                          │
├─────────────────────────────────────────────────┤
│  A Toeplitz matrix: constant along diagonals    │
│                                                 │
│  ┌───┬───┬───┬───┐                                │
│  │ 1 │ 2 │ 3 │ 4 │  diag₀: [1,1,1]              │
│  ├───┼───┼───┼───┤  diag₁: [2,2,2]              │
│  │ 5 │ 1 │ 2 │ 3 │  diag₂: [3,3]                │
│  ├───┼───┼───┼───┤  diag₋₁:[5,5]                │
│  │ 9 │ 5 │ 1 │ 2 │  diag₋₂:[9]                  │
│  └───┴───┴───┴───┘  ✓ Toeplitz!                 │
│                                                 │
│  Check: m[i][j] == m[i-1][j-1] for all i,j>0   │
│  O(mn) time, O(1) space                         │
└─────────────────────────────────────────────────┘
```

---

## Example 10: Matrix Operations Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Matrix Operations Patterns ===")
	fmt.Println()

	patterns := []struct{ technique, use, complexity string }{
		{"Standard multiply", "A(m×n) * B(n×p)", "O(m*n*p)"},
		{"Matrix exponentiation", "Linear recurrences → O(k³ log n)", "O(k³ log n)"},
		{"Matrix chain order", "Optimal parenthesization DP", "O(n³)"},
		{"Spiral traversal", "Visit matrix in spiral order", "O(m*n)"},
		{"Rotate 90°", "Transpose + reverse rows", "O(n²)"},
		{"Set zeroes", "Use first row/col as markers", "O(m*n), O(1) space"},
		{"Transpose", "Swap (i,j) ↔ (j,i)", "O(n²)"},
		{"Path counting", "Adj^k gives k-length paths", "O(n³ log k)"},
		{"Toeplitz check", "Constant diagonals", "O(m*n)"},
		{"Strassen's", "O(n^2.807) large matrix mult", "Divide and conquer"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-24s %-40s %s\n", p.technique, p.use, p.complexity)
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Matrix Operations Toolkit                      │
├─────────────────────────────────────────────────┤
│  Multiply A×B     → O(mnp)    three nested loops│
│  Matrix exponent   → O(k³logn) binary square    │
│  Chain order DP    → O(n³)    optimal parens    │
│  Spiral traversal  → O(mn)    4 boundaries      │
│  Rotate 90°        → O(n²)    transpose+reverse  │
│  Set zeroes        → O(mn)    marker row/col    │
│  Transpose         → O(n²)    swap (i,j)↔(j,i)  │
│  Path counting     → O(n³logk) adj matrix power │
│  Toeplitz check    → O(mn)    diagonal constant  │
│  Strassen          → O(n^2.8)  D&C huge matrices│
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Matrix exponentiation** solves linear recurrences in O(k³ log n)
2. **Adjacency matrix power** counts paths of length k in graphs
3. **Matrix chain DP** finds optimal multiplication order in O(n³)
4. **In-place rotation**: transpose + reverse — O(n²) time, O(1) space
5. **Use first row/col as markers** for O(1) space matrix zeroing

> **Next up:** Reservoir Sampling →
