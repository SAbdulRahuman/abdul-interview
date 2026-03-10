# Phase 24: Divide & Conquer — Strassen's Matrix Multiplication

## Overview

Standard matrix multiplication of two n×n matrices takes **O(n³)**. Strassen's algorithm reduces this to **O(n^2.807)** by computing **7 multiplications** instead of 8 in each recursive step.

### Key Idea
Instead of 8 recursive multiplications (standard D&C), Strassen computes 7 special products (M1–M7) using clever additions/subtractions, then combines them.

| Method | Multiplications | Additions | Complexity |
|--------|----------------|-----------|------------|
| Naive | n³ | n²(n-1) | O(n³) |
| Standard D&C | 8 per level | O(n²) | O(n³) |
| Strassen | 7 per level | O(n²) | O(n^2.807) |

---

## Example 1: Naive Matrix Multiplication

```go
package main

import "fmt"

// Standard O(n³) matrix multiplication

func multiply(A, B [][]int) [][]int {
	n := len(A)
	C := make([][]int, n)
	for i := range C {
		C[i] = make([]int, n)
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				C[i][j] += A[i][k] * B[k][j]
			}
		}
	}
	return C
}

func printMatrix(m [][]int, name string) {
	fmt.Printf("%s:\n", name)
	for _, row := range m {
		fmt.Println(" ", row)
	}
}

func main() {
	A := [][]int{{1, 2}, {3, 4}}
	B := [][]int{{5, 6}, {7, 8}}

	C := multiply(A, B)
	printMatrix(A, "A")
	printMatrix(B, "B")
	printMatrix(C, "A×B")
	// [1*5+2*7, 1*6+2*8]   = [19, 22]
	// [3*5+4*7, 3*6+4*8]   = [43, 50]
}
```

**Textual Figure:**

```
  Naive Matrix Multiplication: A(2×2) × B(2×2)

  ┌───┬───┐   ┌───┬───┐   ┌────┬────┐
  │ 1 │ 2 │ × │ 5 │ 6 │ = │ 19 │ 22 │
  ├───┼───┤   ├───┼───┤   ├────┼────┤
  │ 3 │ 4 │   │ 7 │ 8 │   │ 43 │ 50 │
  └───┴───┘   └───┴───┘   └────┴────┘

  C[0][0] = 1×5 + 2×7 = 5+14 = 19
  C[0][1] = 1×6 + 2×8 = 6+16 = 22
  C[1][0] = 3×5 + 4×7 = 15+28 = 43
  C[1][1] = 3×6 + 4×8 = 18+32 = 50

  For n×n: 3 nested loops → n³ multiplications
  Complexity: O(n³)
```

---

## Example 2: Strassen's Algorithm (2×2)

```go
package main

import "fmt"

// Strassen for 2×2 matrices — 7 multiplications instead of 8

func strassen2x2(A, B [2][2]int) [2][2]int {
	a, b, c, d := A[0][0], A[0][1], A[1][0], A[1][1]
	e, f, g, h := B[0][0], B[0][1], B[1][0], B[1][1]

	// 7 multiplications (Strassen's magic)
	m1 := (a + d) * (e + h)
	m2 := (c + d) * e
	m3 := a * (f - h)
	m4 := d * (g - e)
	m5 := (a + b) * h
	m6 := (c - a) * (e + f)
	m7 := (b - d) * (g + h)

	// Combine
	var C [2][2]int
	C[0][0] = m1 + m4 - m5 + m7
	C[0][1] = m3 + m5
	C[1][0] = m2 + m4
	C[1][1] = m1 - m2 + m3 + m6

	return C
}

func main() {
	A := [2][2]int{{1, 2}, {3, 4}}
	B := [2][2]int{{5, 6}, {7, 8}}

	C := strassen2x2(A, B)
	fmt.Println("A:", A)
	fmt.Println("B:", B)
	fmt.Println("Strassen A×B:", C)
	fmt.Println("\nUsed 7 multiplications instead of 8!")
}
```

**Textual Figure:**

```
  Strassen's 2×2: 7 multiplications instead of 8

  A = ┌ a  b ┐   B = ┌ e  f ┐
      └ c  d ┘       └ g  h ┘

  7 clever products:
  ┌──────────────────────────────────┐
  │  M1 = (a+d)(e+h) = (1+4)(5+8) = 65 │
  │  M2 = (c+d)(e)   = (3+4)(5)   = 35 │
  │  M3 = (a)(f-h)   = (1)(6-8)   = -2 │
  │  M4 = (d)(g-e)   = (4)(7-5)   =  8 │
  │  M5 = (a+b)(h)   = (1+2)(8)   = 24 │
  │  M6 = (c-a)(e+f) = (3-1)(5+6) = 22 │
  │  M7 = (b-d)(g+h) = (2-4)(7+8) =-30 │
  └──────────────────────────────────┘

  Combine:
  C[0][0] = M1+M4-M5+M7 = 65+8-24-30 = 19 ✓
  C[0][1] = M3+M5       = -2+24      = 22 ✓
  C[1][0] = M2+M4       = 35+8       = 43 ✓
  C[1][1] = M1-M2+M3+M6 = 65-35-2+22 = 50 ✓

  Saved: 1 mul (8→7), added ~10 add/sub
```

---

## Example 3: General Strassen's Algorithm (n×n)

```go
package main

import "fmt"

type Matrix [][]int

func newMatrix(n int) Matrix {
	m := make(Matrix, n)
	for i := range m { m[i] = make([]int, n) }
	return m
}

func add(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ { C[i][j] = A[i][j] + B[i][j] }
	}
	return C
}

func sub(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ { C[i][j] = A[i][j] - B[i][j] }
	}
	return C
}

func naiveMul(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ { C[i][j] += A[i][k] * B[k][j] }
		}
	}
	return C
}

// Split matrix into 4 quadrants
func split(M Matrix) (Matrix, Matrix, Matrix, Matrix) {
	n := len(M)
	h := n / 2
	a, b, c, d := newMatrix(h), newMatrix(h), newMatrix(h), newMatrix(h)
	for i := 0; i < h; i++ {
		for j := 0; j < h; j++ {
			a[i][j] = M[i][j]
			b[i][j] = M[i][j+h]
			c[i][j] = M[i+h][j]
			d[i][j] = M[i+h][j+h]
		}
	}
	return a, b, c, d
}

// Merge 4 quadrants into one matrix
func merge(a, b, c, d Matrix) Matrix {
	h := len(a)
	n := h * 2
	M := newMatrix(n)
	for i := 0; i < h; i++ {
		for j := 0; j < h; j++ {
			M[i][j] = a[i][j]
			M[i][j+h] = b[i][j]
			M[i+h][j] = c[i][j]
			M[i+h][j+h] = d[i][j]
		}
	}
	return M
}

func strassen(A, B Matrix) Matrix {
	n := len(A)
	if n <= 2 { return naiveMul(A, B) }

	// Pad to even size if needed
	a, b, c, d := split(A)
	e, f, g, h := split(B)

	// 7 recursive multiplications
	m1 := strassen(add(a, d), add(e, h))
	m2 := strassen(add(c, d), e)
	m3 := strassen(a, sub(f, h))
	m4 := strassen(d, sub(g, e))
	m5 := strassen(add(a, b), h)
	m6 := strassen(sub(c, a), add(e, f))
	m7 := strassen(sub(b, d), add(g, h))

	// Combine: C = [[c11,c12],[c21,c22]]
	c11 := add(sub(add(m1, m4), m5), m7)
	c12 := add(m3, m5)
	c21 := add(m2, m4)
	c22 := add(sub(add(m1, m3), m2), m6)

	return merge(c11, c12, c21, c22)
}

func printMatrix(m Matrix, name string) {
	fmt.Printf("%s:\n", name)
	for _, row := range m { fmt.Println(" ", row) }
}

func main() {
	A := Matrix{{1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12}, {13, 14, 15, 16}}
	B := Matrix{{1, 0, 0, 1}, {0, 1, 1, 0}, {1, 1, 0, 0}, {0, 0, 1, 1}}

	C := strassen(A, B)
	printMatrix(A, "A")
	printMatrix(B, "B")
	printMatrix(C, "Strassen A×B")

	// Verify with naive
	D := naiveMul(A, B)
	printMatrix(D, "Naive A×B")
}
```

**Textual Figure:**

```
  General Strassen's Algorithm (n×n recursive)

  Split n×n matrices into 2×2 blocks of (n/2)×(n/2):
  ┌────┬────┐     ┌────┬────┐
  │ A11│ A12│     │ B11│ B12│
  ├────┼────┤  ×  ├────┼────┤
  │ A21│ A22│     │ B21│ B22│
  └────┴────┘     └────┴────┘

  Recursion tree (7 branches per level):
          n×n                          work = O(n²)
        / | ... \  (7 children)
     n/2×n/2 ... n/2×n/2              work = 7×O(n²/4)
      /|..\       /|..\
   n/4×n/4     n/4×n/4               work = 49×O(n²/16)
      ...           ...
    1×1 base cases                    = n^log₂ 7 leaves

  T(n) = 7·T(n/2) + O(n²)
  Master: a=7, b=2, k=2, log₂ 7≈2.807 > 2
  → Case 1: O(n^2.807)

  vs. naive: 8·T(n/2)+O(n²) → O(n³)
```

---

## Example 4: Matrix Power using D&C (Binary Exponentiation)

```go
package main

import "fmt"

type Matrix [2][2]int64

func matMul(A, B Matrix) Matrix {
	var C Matrix
	for i := 0; i < 2; i++ {
		for j := 0; j < 2; j++ {
			for k := 0; k < 2; k++ {
				C[i][j] += A[i][k] * B[k][j]
			}
		}
	}
	return C
}

func matPow(M Matrix, n int) Matrix {
	// Identity matrix
	result := Matrix{{1, 0}, {0, 1}}
	base := M
	for n > 0 {
		if n%2 == 1 { result = matMul(result, base) }
		base = matMul(base, base)
		n /= 2
	}
	return result
}

func fibonacci(n int) int64 {
	if n <= 1 { return int64(n) }
	M := Matrix{{1, 1}, {1, 0}}
	result := matPow(M, n-1)
	return result[0][0]
}

func main() {
	fmt.Println("Fibonacci via matrix exponentiation:")
	for i := 0; i <= 15; i++ {
		fmt.Printf("F(%d) = %d\n", i, fibonacci(i))
	}
	fmt.Println("\nComplexity: O(log n) matrix multiplications")
	fmt.Println("Each matMul is O(k³) for k×k matrix")
}
```

**Textual Figure:**

```
  Matrix Exponentiation for Fibonacci: F(n)

  ┌───┬───┐ n-1    ┌─────────┬─────────┐
  │ 1 │ 1 │     =  │ F(n)    │ F(n-1)  │
  ├───┼───┤        ├─────────┼─────────┤
  │ 1 │ 0 │        │ F(n-1)  │ F(n-2)  │
  └───┴───┘        └─────────┴─────────┘

  Binary exponentiation for M^n:
  matPow(M, 10):
    10 = 1010₂

    Step  n    Action           result
    ─────────────────────────────────────
    1     10   bit=0, square    base=M²
    2     5    bit=1, mul+sq    result×=M², base=M⁴
    3     2    bit=0, square    base=M⁸
    4     1    bit=1, mul       result×=M⁸

  Only log₂(n) matrix multiplications!
  F(15) via matPow: 4 mat-muls vs 14 additions

  T(n) = O(log n) × O(k³) = O(k³ log n)
  For 2×2: O(8 log n) = O(log n)
```

---

## Example 5: Counting Multiplications — Strassen vs Naive

```go
package main

import "fmt"

// Count exact number of scalar multiplications

var naiveMuls, strassenMuls int

func countNaive(n int) {
	if n == 1 { naiveMuls++; return }
	h := n / 2
	for i := 0; i < 8; i++ { countNaive(h) }
	// 8 recursive calls for standard D&C
}

func countStrassen(n int) {
	if n == 1 { strassenMuls++; return }
	h := n / 2
	for i := 0; i < 7; i++ { countStrassen(h) }
	// 7 recursive calls for Strassen
}

func main() {
	fmt.Println("=== Scalar Multiplications Comparison ===\n")
	for _, n := range []int{2, 4, 8, 16, 32, 64} {
		naiveMuls, strassenMuls = 0, 0
		countNaive(n)
		countStrassen(n)
		naive := n * n * n
		ratio := float64(strassenMuls) / float64(naiveMuls)

		fmt.Printf("n=%2d: Naive=%6d  Naive(n³)=%6d  Strassen=%6d  Ratio=%.3f\n",
			n, naiveMuls, naive, strassenMuls, ratio)
	}

	fmt.Println("\nStrassen crossover: typically n=32-128 in practice")
	fmt.Println("Below crossover, naive is faster due to lower overhead")
}
```

**Textual Figure:**

```
  Scalar Multiplications: Strassen vs Naive

  n      Naive (8^k)   Strassen (7^k)  n³       Ratio
  ────────────────────────────────────────────────
   2         8            7            8       0.875
   4        64           49           64       0.766
   8       512          343          512       0.670
  16      4096         2401         4096       0.586
  32     32768        16807        32768       0.513
  64    262144       117649       262144       0.449

  Recursion tree branching:
  Naive:     8 children/node    Strassen:  7 children/node
      n                              n
  /|..8..|\                      /|..7..|\
  n/2 ... n/2                    n/2 ... n/2
  8^(log n) leaves               7^(log n) leaves
  = n^3                           = n^2.807

  Crossover: Strassen wins for n ≥ 32-128
  (overhead from 18 add/sub per level)
```

---

## Example 6: Strassen with Crossover to Naive

```go
package main

import "fmt"

type Matrix [][]int

func newMatrix(n int) Matrix {
	m := make(Matrix, n)
	for i := range m { m[i] = make([]int, n) }
	return m
}

func add(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ { C[i][j] = A[i][j] + B[i][j] }
	}
	return C
}

func sub(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ { C[i][j] = A[i][j] - B[i][j] }
	}
	return C
}

func naiveMul(A, B Matrix) Matrix {
	n := len(A)
	C := newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ { C[i][j] += A[i][k] * B[k][j] }
		}
	}
	return C
}

func split(M Matrix) (Matrix, Matrix, Matrix, Matrix) {
	n := len(M); h := n / 2
	a, b, c, d := newMatrix(h), newMatrix(h), newMatrix(h), newMatrix(h)
	for i := 0; i < h; i++ {
		for j := 0; j < h; j++ {
			a[i][j] = M[i][j]; b[i][j] = M[i][j+h]
			c[i][j] = M[i+h][j]; d[i][j] = M[i+h][j+h]
		}
	}
	return a, b, c, d
}

func merge(a, b, c, d Matrix) Matrix {
	h := len(a); n := h * 2; M := newMatrix(n)
	for i := 0; i < h; i++ {
		for j := 0; j < h; j++ {
			M[i][j] = a[i][j]; M[i][j+h] = b[i][j]
			M[i+h][j] = c[i][j]; M[i+h][j+h] = d[i][j]
		}
	}
	return M
}

const CROSSOVER = 4 // Switch to naive for small matrices

func strassenHybrid(A, B Matrix) Matrix {
	n := len(A)
	if n <= CROSSOVER { return naiveMul(A, B) } // Crossover!

	a, b, c, d := split(A)
	e, f, g, h := split(B)

	m1 := strassenHybrid(add(a, d), add(e, h))
	m2 := strassenHybrid(add(c, d), e)
	m3 := strassenHybrid(a, sub(f, h))
	m4 := strassenHybrid(d, sub(g, e))
	m5 := strassenHybrid(add(a, b), h)
	m6 := strassenHybrid(sub(c, a), add(e, f))
	m7 := strassenHybrid(sub(b, d), add(g, h))

	c11 := add(sub(add(m1, m4), m5), m7)
	c12 := add(m3, m5)
	c21 := add(m2, m4)
	c22 := add(sub(add(m1, m3), m2), m6)

	return merge(c11, c12, c21, c22)
}

func main() {
	n := 8
	A, B := newMatrix(n), newMatrix(n)
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ { A[i][j] = i + j; B[i][j] = i - j + 1 }
	}

	C := strassenHybrid(A, B)
	D := naiveMul(A, B)

	match := true
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			if C[i][j] != D[i][j] { match = false }
		}
	}
	fmt.Printf("Strassen hybrid (crossover=%d) matches naive: %v\n", CROSSOVER, match)
	fmt.Println("\nIn practice, CROSSOVER typically 32-128")
}
```

**Textual Figure:**

```
  Strassen Hybrid: Switch to Naive at Small Sizes

  Recursion tree with crossover at n=4:
           8×8 matrix
         / | ... | \  (7 Strassen sub-muls)
       4×4  4×4  ... 4×4
         │     │       │
      [naive] [naive] [naive]   ← switch to O(n³)

  ┌────────────────────────────────────────────┐
  │  Why crossover?                            │
  │                                            │
  │  Strassen per level: 7 muls + 18 add/sub  │
  │  Naive at base:      n³ = 64  (for n=4)    │
  │                                            │
  │  Below crossover, overhead of additions   │
  │  outweighs savings from fewer muls        │
  │                                            │
  │  Typical crossover:  n = 32 to 128         │
  │  (depends on hardware cache behavior)       │
  └────────────────────────────────────────────┘
```

---

## Example 7: Matrix Chain Multiplication (D&C approach)

```go
package main

import (
	"fmt"
	"math"
)

// Given matrices A1(p0×p1), A2(p1×p2), ..., An(pn-1×pn)
// Find optimal parenthesization to minimize multiplications

func matrixChain(dims []int) int {
	n := len(dims) - 1
	dp := make([][]int, n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt64 }
		dp[i][i] = 0
	}

	for length := 2; length <= n; length++ {
		for i := 0; i <= n-length; i++ {
			j := i + length - 1
			for k := i; k < j; k++ {
				cost := dp[i][k] + dp[k+1][j] + dims[i]*dims[k+1]*dims[j+1]
				if cost < dp[i][j] { dp[i][j] = cost }
			}
		}
	}
	return dp[0][n-1]
}

// D&C brute force for comparison
func matrixChainDC(dims []int, i, j int) int {
	if i == j { return 0 }
	minCost := math.MaxInt64
	for k := i; k < j; k++ {
		cost := matrixChainDC(dims, i, k) + matrixChainDC(dims, k+1, j) + dims[i]*dims[k+1]*dims[j+1]
		if cost < minCost { minCost = cost }
	}
	return minCost
}

func main() {
	dims := []int{10, 30, 5, 60}  // 3 matrices: 10x30, 30x5, 5x60

	dpResult := matrixChain(dims)
	dcResult := matrixChainDC(dims, 0, len(dims)-2)

	fmt.Printf("Dimensions: %v\n", dims)
	fmt.Printf("DP result: %d multiplications\n", dpResult)
	fmt.Printf("D&C result: %d multiplications\n", dcResult)
	fmt.Println("\n(A1·A2)·A3 = 10*30*5 + 10*5*60 = 1500+3000 = 4500")
	fmt.Println("A1·(A2·A3) = 30*5*60 + 10*30*60 = 9000+18000 = 27000")
}
```

**Textual Figure:**

```
  Matrix Chain: A1(10×30) × A2(30×5) × A3(5×60)

  Two possible parenthesizations:
  ┌──────────────────────────────────────────────┐
  │  Option 1: (A1·A2)·A3                        │
  │    A1·A2: 10×30×5  = 1,500 muls → 10×5    │
  │    ×A3:  10×5×60  = 3,000 muls             │
  │    Total:           4,500 muls  ★ optimal │
  ├──────────────────────────────────────────────┤
  │  Option 2: A1·(A2·A3)                        │
  │    A2·A3: 30×5×60 = 9,000 muls → 30×60   │
  │    A1×:  10×30×60 = 18,000 muls            │
  │    Total:          27,000 muls  ✘ 6× worse│
  └──────────────────────────────────────────────┘

  D&C recurrence for optimal split:
  For matrices i..j, try every split point k:
    cost(i,j) = min over k { cost(i,k) + cost(k+1,j)
                             + dims[i]·dims[k+1]·dims[j+1] }
  Overlap → DP is better: O(n³) vs exponential D&C
```

---

## Example 8: Strassen's Products — The 7 Formulas

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Strassen's 7 Products ===\n")
	fmt.Println("Given A = [[a,b],[c,d]], B = [[e,f],[g,h]]\n")

	fmt.Println("Products:")
	products := []struct{ name, formula string }{
		{"M1", "(a + d)(e + h)"},
		{"M2", "(c + d)(e)"},
		{"M3", "(a)(f - h)"},
		{"M4", "(d)(g - e)"},
		{"M5", "(a + b)(h)"},
		{"M6", "(c - a)(e + f)"},
		{"M7", "(b - d)(g + h)"},
	}
	for _, p := range products { fmt.Printf("  %s = %s\n", p.name, p.formula) }

	fmt.Println("\nResult C = A×B:")
	fmt.Println("  C[0][0] = M1 + M4 - M5 + M7  = ae + bg")
	fmt.Println("  C[0][1] = M3 + M5             = af + bh")
	fmt.Println("  C[1][0] = M2 + M4             = ce + dg")
	fmt.Println("  C[1][1] = M1 - M2 + M3 + M6  = cf + dh")

	fmt.Println("\n=== Verification with numbers ===")
	a, b, c, d := 1, 2, 3, 4
	e, f, g, h := 5, 6, 7, 8

	m1 := (a + d) * (e + h)
	m2 := (c + d) * e
	m3 := a * (f - h)
	m4 := d * (g - e)
	m5 := (a + b) * h
	m6 := (c - a) * (e + f)
	m7 := (b - d) * (g + h)

	fmt.Printf("\nM1=%d M2=%d M3=%d M4=%d M5=%d M6=%d M7=%d\n", m1, m2, m3, m4, m5, m6, m7)
	fmt.Printf("C[0][0] = %d = %d\n", m1+m4-m5+m7, a*e+b*g)
	fmt.Printf("C[0][1] = %d = %d\n", m3+m5, a*f+b*h)
	fmt.Printf("C[1][0] = %d = %d\n", m2+m4, c*e+d*g)
	fmt.Printf("C[1][1] = %d = %d\n", m1-m2+m3+m6, c*f+d*h)
}
```

**Textual Figure:**

```
  Strassen's 7 Products: Complete Formula Map

  A = ┌ 1  2 ┐   B = ┌ 5  6 ┐
      └ 3  4 ┘       └ 7  8 ┘

  Products:                      Values:
  ┌────┬────────────────────┬───────┐
  │ M1 │ (a+d)×(e+h)       │ 5×13=65│
  │ M2 │ (c+d)×e           │ 7× 5=35│
  │ M3 │ a×(f-h)           │ 1×-2=-2│
  │ M4 │ d×(g-e)           │ 4× 2= 8│
  │ M5 │ (a+b)×h           │ 3× 8=24│
  │ M6 │ (c-a)×(e+f)       │ 2×11=22│
  │ M7 │ (b-d)×(g+h)       │-2×15=-30│
  └────┴────────────────────┴───────┘

  Combining into result matrix C:
  ┌─────────────────────────────┬────────┬─────┐
  │ C[0][0] = M1+M4-M5+M7       │ 19     │ ae+bg│
  │ C[0][1] = M3+M5              │ 22     │ af+bh│
  │ C[1][0] = M2+M4              │ 43     │ ce+dg│
  │ C[1][1] = M1-M2+M3+M6       │ 50     │ cf+dh│
  └─────────────────────────────┴────────┴─────┘
  7 muls + 18 add/sub vs naive 8 muls + 4 add
```

---

## Example 9: Rectangular Matrix Handling (Padding)

```go
package main

import "fmt"

type Matrix [][]int

func nextPow2(n int) int {
	p := 1
	for p < n { p *= 2 }
	return p
}

func padMatrix(M Matrix, size int) Matrix {
	padded := make(Matrix, size)
	for i := range padded {
		padded[i] = make([]int, size)
		if i < len(M) {
			copy(padded[i], M[i])
		}
	}
	return padded
}

func extractMatrix(M Matrix, rows, cols int) Matrix {
	result := make(Matrix, rows)
	for i := 0; i < rows; i++ {
		result[i] = make([]int, cols)
		copy(result[i], M[i][:cols])
	}
	return result
}

func naiveMul(A, B Matrix) Matrix {
	n := len(A)
	C := make(Matrix, n)
	for i := range C {
		C[i] = make([]int, n)
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ { C[i][j] += A[i][k] * B[k][j] }
		}
	}
	return C
}

func multiplyRectangular(A, B Matrix) Matrix {
	rowsA, colsA := len(A), len(A[0])
	_, colsB := len(B), len(B[0])

	// Pad to next power of 2
	maxDim := rowsA
	if colsA > maxDim { maxDim = colsA }
	if colsB > maxDim { maxDim = colsB }
	size := nextPow2(maxDim)

	paddedA := padMatrix(A, size)
	paddedB := padMatrix(B, size)

	// Use Strassen (or naive for demo)
	result := naiveMul(paddedA, paddedB)

	// Extract actual result
	return extractMatrix(result, rowsA, colsB)
}

func main() {
	// 3×2 × 2×4
	A := Matrix{{1, 2}, {3, 4}, {5, 6}}
	B := Matrix{{7, 8, 9, 10}, {11, 12, 13, 14}}

	C := multiplyRectangular(A, B)
	fmt.Println("A (3×2):")
	for _, row := range A { fmt.Println(" ", row) }
	fmt.Println("B (2×4):")
	for _, row := range B { fmt.Println(" ", row) }
	fmt.Println("C (3×4):")
	for _, row := range C { fmt.Println(" ", row) }

	fmt.Println("\nPadding for Strassen: next power of 2 = 4×4")
}
```

**Textual Figure:**

```
  Rectangular Matrix Handling via Padding

  Original: A(3×2) × B(2×4)
  ┌─┬─┐     ┌─┬─┬─┬─┐
  │1│2│     │7│8│9│10│
  │3│4│  ×  │B│B│B│B │
  │5│6│     └─┴─┴─┴─┘
  └─┴─┘

  Step 1: Find max dimension = 4
  Step 2: Next power of 2 = 4
  Step 3: Pad both to 4×4 with zeros:

  ┌─┬─┬─┬─┐     ┌─┬─┬─┬──┐      ┌──┬──┬──┬──┐
  │1│2│0│0│     │7│8│9│10 │      │29│32│35│38│
  │3│4│0│0│  ×  │B│B│B│B  │  =   │65│72│79│86│
  │5│6│0│0│     │0│0│0│0  │      │..│..│..│..│
  │0│0│0│0│     │0│0│0│0  │      │0 │0 │0 │0 │
  └─┴─┴─┴─┘     └─┴─┴─┴──┘      └──┴──┴──┴──┘

  Step 4: Extract top-left 3×4 from result
  Padding overhead: extra zeros, but enables power-of-2 recursion
```

---

## Example 10: Complexity Comparison & When to Use

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println("=== Matrix Multiplication Algorithms ===\n")

	algorithms := []struct {
		name    string
		exp     float64
		year    string
	}{
		{"Naive", 3.0, "classical"},
		{"Strassen", 2.807, "1969"},
		{"Pan", 2.796, "1980"},
		{"Coppersmith-Winograd", 2.376, "1990"},
		{"Williams", 2.3729, "2012"},
		{"Theoretical lower bound", 2.0, "conjectured"},
	}

	fmt.Printf("%-25s %-10s %-10s %s\n", "Algorithm", "Exponent", "n=1024", "Year")
	fmt.Println("--------------------------------------------------------")
	for _, a := range algorithms {
		ops := math.Pow(1024, a.exp)
		fmt.Printf("%-25s %-10.4f %-10.2e %s\n", a.name, a.exp, ops, a.year)
	}

	fmt.Println("\n=== When to Use Strassen ===")
	fmt.Println("✓ Large matrices (n > 64 typically)")
	fmt.Println("✓ Exact arithmetic (integers)")
	fmt.Println("✗ Small matrices (overhead too high)")
	fmt.Println("✗ Floating point with precision needs (numerical instability)")
	fmt.Println("✗ Sparse matrices (use sparse algorithms instead)")

	fmt.Println("\n=== Interview Tips ===")
	tips := []string{
		"Know the idea: 7 muls instead of 8 → O(n^2.807)",
		"Master theorem: T(n) = 7T(n/2) + O(n²) → O(n^log₂7)",
		"Practical crossover: switch to naive for small n",
		"Extends to any ring (not just integers)",
		"Matrix exponentiation is more common in interviews",
	}
	for _, t := range tips { fmt.Println("  •", t) }
}
```

**Textual Figure:**

```
  Matrix Multiplication Algorithms: Complexity Comparison

  ┌───────────────────────┬──────────┬───────────┐
  │ Algorithm             │ Exponent  │ Year      │
  ├───────────────────────┼──────────┼───────────┤
  │ Naive                 │  3.000    │ classical │
  │ Strassen              │  2.807    │ 1969      │
  │ Pan                   │  2.796    │ 1980      │
  │ Coppersmith-Winograd   │  2.376    │ 1990      │
  │ Williams              │  2.3729   │ 2012      │
  │ Lower bound (conjecture)│  2.000   │ open      │
  └───────────────────────┴──────────┴───────────┘

  When to use Strassen:
  ────────────────────────────────────────
  n < 32-64    → Naive (lower overhead)
  n = 64-1000  → Strassen ★
  Any n        → Avoid for floating-point (numerical instability)
  Sparse       → Use sparse algorithms instead

  For n=1024:
    n³      = 1,073,741,824 ops
    n^2.807 ≈   591,000,000 ops  (≈45% savings)
```

---

## Key Takeaways

1. **7 vs 8**: Strassen replaces 8 multiplications with 7 at the cost of more additions
2. **O(n^2.807)**: From Master theorem: T(n) = 7·T(n/2) + O(n²), so log₂7 ≈ 2.807
3. **Crossover**: In practice, switch to naive for small matrices (typically n ≤ 32–128)
4. **Padding**: Non-power-of-2 matrices are padded with zeros
5. **Interview relevance**: Understanding the concept matters more than implementing — matrix exponentiation (Fibonacci, graph paths) is asked more often
6. **Numerical stability**: Strassen is less numerically stable with floating point

> **Next up:** Karatsuba Multiplication →
