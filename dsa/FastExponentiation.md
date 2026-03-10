# Phase 21: Math & Geometry — Fast Exponentiation

## Overview

**Fast exponentiation** (binary exponentiation / exponentiation by squaring) computes `a^n` in O(log n) time by using the binary representation of the exponent.

| Method | Time | Space | Notes |
|--------|------|-------|-------|
| Naive | O(n) | O(1) | Multiply n times |
| Binary Exponentiation | O(log n) | O(1) | Iterative, use bit shifts |
| Matrix Exponentiation | O(k³ log n) | O(k²) | For linear recurrences |
| Recursive | O(log n) | O(log n) | Stack space |

**Key idea**: `a^n = a^(n/2) * a^(n/2)` if n even; `a * a^(n-1)` if n odd.

---

## Example 1: Iterative Binary Exponentiation

```go
package main

import "fmt"

func fastPow(base, exp int) int {
	result := 1
	for exp > 0 {
		if exp&1 == 1 {
			result *= base
		}
		base *= base
		exp >>= 1
	}
	return result
}

func main() {
	tests := [][2]int{{2, 10}, {3, 5}, {5, 3}, {7, 4}, {2, 20}}
	for _, t := range tests {
		fmt.Printf("  %d^%d = %d\n", t[0], t[1], fastPow(t[0], t[1]))
	}

	fmt.Println()
	fmt.Println("How it works for 2^10:")
	fmt.Println("  10 = 1010 in binary")
	fmt.Println("  2^10 = 2^8 * 2^2 = 256 * 4 = 1024")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Binary Exponentiation: 2^10                     │
├────────────────────────────────────────────────┤
│  exp = 10 = 1010 in binary                       │
│                                                  │
│  Iter  exp(bin)  exp&1  base     result           │
│  ────  ───────  ─────  ───────  ──────           │
│  init   1010       -     2        1              │
│    1    1010       0     2²=4     1  (skip)      │
│    2    0101       1     4²=16    1×4=4          │
│    3    0010       0     16²=256  4  (skip)      │
│    4    0001       1     256²     4×256=1024     │
│                                                  │
│  2^10 = 2^(8+2) = 2^8 × 2^2 = 256 × 4 = 1024    │
│  Only 4 multiplies instead of 10!                │
└────────────────────────────────────────────────┘
```

---

## Example 2: Modular Exponentiation

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modPow(base, exp, mod int) int {
	result := 1
	base %= mod
	for exp > 0 {
		if exp&1 == 1 {
			result = result * base % mod
		}
		base = base * base % mod
		exp >>= 1
	}
	return result
}

func main() {
	tests := [][2]int{{2, 100}, {3, 1000000}, {7, 999999999}}
	for _, t := range tests {
		fmt.Printf("  %d^%d mod 10^9+7 = %d\n", t[0], t[1], modPow(t[0], t[1], MOD))
	}

	// Application: Modular inverse
	a := 5
	inv := modPow(a, MOD-2, MOD)
	fmt.Printf("\n  Modular inverse of %d: %d (verify: %d * %d mod M = %d)\n",
		a, inv, a, inv, a*inv%MOD)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Modular Exponentiation                          │
├────────────────────────────────────────────────┤
│  Same as binary exp but multiply mod M at each   │
│  step to prevent overflow                        │
│                                                  │
│  2^100 mod 10^9+7:                               │
│  Iter  exp&1  base (mod M)       result (mod M)  │
│    1     0    2²=4               1              │
│    2     1    4²=16              1×4=4           │
│    3     0    16²=256            4              │
│    4     0    256² mod M         4              │
│    5     1    ... mod M          4×(base) mod M  │
│    ...   ...  keep squaring      keep reducing   │
│                                                  │
│  Modular inverse: a^(M-2) mod M  (Fermat’s)     │
│  5^(10^9+5) mod 10^9+7 = inverse of 5           │
└────────────────────────────────────────────────┘
```

---

## Example 3: Matrix Exponentiation — Fibonacci

```go
package main

import "fmt"

const MOD = 1_000_000_007

type Matrix [2][2]int

func matMul(a, b Matrix, mod int) Matrix {
	var c Matrix
	for i := 0; i < 2; i++ {
		for j := 0; j < 2; j++ {
			for k := 0; k < 2; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % mod
			}
		}
	}
	return c
}

func matPow(m Matrix, n, mod int) Matrix {
	result := Matrix{{1, 0}, {0, 1}} // identity
	for n > 0 {
		if n&1 == 1 {
			result = matMul(result, m, mod)
		}
		m = matMul(m, m, mod)
		n >>= 1
	}
	return result
}

func fibonacci(n int) int {
	if n <= 1 { return n }
	// [F(n+1), F(n)] = [[1,1],[1,0]]^n
	base := Matrix{{1, 1}, {1, 0}}
	result := matPow(base, n, MOD)
	return result[0][1]
}

func main() {
	for _, n := range []int{0, 1, 5, 10, 50, 100, 1000000000} {
		fmt.Printf("  fib(%d) mod 10^9+7 = %d\n", n, fibonacci(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Matrix Exponentiation — Fibonacci               │
├────────────────────────────────────────────────┤
│  ┌         ┐ n   ┌             ┐                    │
│  │ F(n+1)  │   = │ 1   1 │ ^ n  * ┌ F(1) ┐          │
│  │ F(n)    │     │ 1   0 │      │ F(0) │          │
│  └         ┘     └       ┘      └      ┘          │
│                                                  │
│  Matrix raised to power n using binary exp:      │
│  M^10 = M^8 × M^2  (same idea, square & mult)   │
│                                                  │
│  n=5: M^5 = M^4 × M^1                           │
│    M^1 = [[1,1],[1,0]]                           │
│    M^2 = [[2,1],[1,1]]                           │
│    M^4 = [[5,3],[3,2]]                           │
│    M^5 = M^4 × M^1 → result[0][1] = F(5) = 5   │
│                                                  │
│  O(8 × log n) = O(log n) for Fibonacci           │
└────────────────────────────────────────────────┘
```

---

## Example 4: Matrix Exponentiation — General Linear Recurrence

```go
package main

import "fmt"

const MOD = 1_000_000_007

type Matrix struct {
	n    int
	data [][]int
}

func newMatrix(n int) Matrix {
	m := Matrix{n: n, data: make([][]int, n)}
	for i := range m.data { m.data[i] = make([]int, n) }
	return m
}

func identity(n int) Matrix {
	m := newMatrix(n)
	for i := 0; i < n; i++ { m.data[i][i] = 1 }
	return m
}

func (a Matrix) mul(b Matrix) Matrix {
	c := newMatrix(a.n)
	for i := 0; i < a.n; i++ {
		for k := 0; k < a.n; k++ {
			if a.data[i][k] == 0 { continue }
			for j := 0; j < a.n; j++ {
				c.data[i][j] = (c.data[i][j] + a.data[i][k]*b.data[k][j]) % MOD
			}
		}
	}
	return c
}

func (m Matrix) pow(exp int) Matrix {
	result := identity(m.n)
	base := m
	for exp > 0 {
		if exp&1 == 1 { result = result.mul(base) }
		base = base.mul(base)
		exp >>= 1
	}
	return result
}

// Tribonacci: T(n) = T(n-1) + T(n-2) + T(n-3)
func tribonacci(n int) int {
	if n <= 0 { return 0 }
	if n <= 2 { return 1 }
	m := newMatrix(3)
	m.data = [][]int{{1, 1, 1}, {1, 0, 0}, {0, 1, 0}}
	result := m.pow(n - 2)
	// [T(n), T(n-1), T(n-2)] = M^(n-2) * [T(2), T(1), T(0)]
	return (result.data[0][0]*1 + result.data[0][1]*1 + result.data[0][2]*0) % MOD
}

func main() {
	fmt.Println("Tribonacci sequence mod 10^9+7:")
	for _, n := range []int{0, 1, 2, 5, 10, 50, 1000000} {
		fmt.Printf("  T(%d) = %d\n", n, tribonacci(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  General Linear Recurrence via Matrix Exp        │
├────────────────────────────────────────────────┤
│  Tribonacci: T(n) = T(n-1) + T(n-2) + T(n-3)    │
│                                                  │
│  ┌          ┐     ┌         ┐ n-2   ┌      ┐     │
│  │ T(n)     │     │ 1  1  1 │       │ T(2) │     │
│  │ T(n-1)   │  =  │ 1  0  0 │     × │ T(1) │     │
│  │ T(n-2)   │     │ 0  1  0 │       │ T(0) │     │
│  └          ┘     └         ┘       └      ┘     │
│                                                  │
│  3×3 matrix raised to power (n-2)                │
│  using binary exponentiation                     │
│                                                  │
│  O(k³ log n) where k = recurrence order          │
│  k=3 for tribonacci → O(27 log n)               │
└────────────────────────────────────────────────┘
```

---

## Example 5: Fast Power for Large Exponents (LeetCode 50 — Pow(x,n))

```go
package main

import "fmt"

func myPow(x float64, n int) float64 {
	if n < 0 {
		x = 1 / x
		n = -n
	}
	result := 1.0
	for n > 0 {
		if n&1 == 1 {
			result *= x
		}
		x *= x
		n >>= 1
	}
	return result
}

func main() {
	tests := []struct {
		x float64
		n int
	}{
		{2.0, 10},
		{2.1, 3},
		{2.0, -2},
		{0.5, -3},
		{1.0, 1000000},
	}
	for _, t := range tests {
		fmt.Printf("  %.2f^%d = %.6f\n", t.x, t.n, myPow(t.x, t.n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Pow(x, n) with Negative Exponents               │
├────────────────────────────────────────────────┤
│  myPow(2.0, -2):                                 │
│    n < 0 → x = 1/x = 0.5, n = 2                 │
│                                                  │
│  exp = 2 = 10 in binary                          │
│  Iter  exp&1   x            result               │
│    1     0     0.5²=0.25    1.0   (skip)         │
│    2     1     0.25²        1.0×0.25 = 0.25      │
│                                                  │
│  2.0^(-2) = 0.25 = 1/4  ✓                        │
│                                                  │
│  Handles: positive, negative, zero exponents     │
│  Key: if n<0, use x=1/x and negate n             │
└────────────────────────────────────────────────┘
```

---

## Example 6: Super Power (LeetCode 372)

```go
package main

import "fmt"

// a^b mod 1337 where b is given as array of digits
func superPow(a int, b []int) int {
	const mod = 1337
	a %= mod

	var pow func(base, exp int) int
	pow = func(base, exp int) int {
		result := 1
		base %= mod
		for exp > 0 {
			if exp&1 == 1 { result = result * base % mod }
			base = base * base % mod
			exp >>= 1
		}
		return result
	}

	result := 1
	for _, digit := range b {
		result = pow(result, 10) * pow(a, digit) % mod
	}
	return result
}

func main() {
	tests := []struct {
		a int
		b []int
	}{
		{2, []int{3}},            // 2^3 = 8
		{2, []int{1, 0}},          // 2^10 = 1024
		{2147483647, []int{2, 0, 0}}, // huge base
	}
	for _, t := range tests {
		fmt.Printf("  %d^%v mod 1337 = %d\n", t.a, t.b, superPow(t.a, t.b))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Super Pow: a^b mod 1337                         │
├────────────────────────────────────────────────┤
│  b = [1, 0] means exponent = 10                  │
│                                                  │
│  Process digit by digit, left to right:          │
│   digit=1: result = 1^10 × a^1 = a mod 1337     │
│   digit=0: result = (a)^10 × a^0                │
│            = a^10 × 1 = a^10 mod 1337            │
│                                                  │
│  Key identity:                                   │
│    a^[d1,d2,...,dk] = (a^[d1,...,dk-1])^10        │
│                       × a^dk                     │
│                                                  │
│  Each pow() call uses binary exponentiation      │
│  Never compute the full exponent number!          │
└────────────────────────────────────────────────┘
```

---

## Example 7: Count Good Numbers (LeetCode 1922)

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		base = base * base % mod
		exp >>= 1
	}
	return result
}

// Count "good" numbers of length n
// Even indices: 0,2,4,6,8 (5 choices)
// Odd indices: 2,3,5,7 (4 choices, primes)
func countGoodNumbers(n int) int {
	evenPos := (n + 1) / 2
	oddPos := n / 2
	return modPow(5, evenPos, MOD) * modPow(4, oddPos, MOD) % MOD
}

func main() {
	for _, n := range []int{1, 4, 50, 1000000000} {
		fmt.Printf("  Good numbers of length %d: %d\n", n, countGoodNumbers(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Count Good Numbers (length n)                   │
├────────────────────────────────────────────────┤
│  Even positions: {0,2,4,6,8} → 5 choices          │
│  Odd  positions: {2,3,5,7}   → 4 choices (primes) │
│                                                  │
│  n=4:  positions 0 1 2 3                         │
│                  E O E O                         │
│                  5 4 5 4                         │
│                                                  │
│  evenPos = (4+1)/2 = 2                           │
│  oddPos  = 4/2     = 2                           │
│  answer  = 5^2 × 4^2 = 25 × 16 = 400             │
│                                                  │
│  Both 5^evenPos and 4^oddPos computed via        │
│  modPow (binary exponentiation mod 10^9+7)       │
└────────────────────────────────────────────────┘
```

---

## Example 8: Exponentiation of Permutations

```go
package main

import "fmt"

// Apply permutation to itself k times (permutation exponentiation)
func permPow(perm []int, k int) []int {
	n := len(perm)
	result := make([]int, n)
	for i := range result { result[i] = i } // identity

	base := make([]int, n)
	copy(base, perm)

	compose := func(a, b []int) []int {
		c := make([]int, n)
		for i := 0; i < n; i++ { c[i] = a[b[i]] }
		return c
	}

	for k > 0 {
		if k&1 == 1 {
			result = compose(base, result)
		}
		base = compose(base, base)
		k >>= 1
	}
	return result
}

func main() {
	perm := []int{1, 2, 3, 4, 0} // rotate left by 1

	for _, k := range []int{1, 2, 3, 5, 10} {
		result := permPow(perm, k)
		fmt.Printf("  perm^%d = %v\n", k, result)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Permutation Exponentiation                      │
├────────────────────────────────────────────────┤
│  perm = [1,2,3,4,0]  (rotate left by 1)          │
│                                                  │
│  perm^1: [1,2,3,4,0]  apply once                 │
│  perm^2: [2,3,4,0,1]  rotate left by 2           │
│  perm^3: [3,4,0,1,2]  rotate left by 3           │
│  perm^5: [0,1,2,3,4]  full cycle → identity!     │
│                                                  │
│  Binary exp on permutations:                     │
│  k=5 = 101₂                                      │
│    compose(base, result) when bit=1              │
│    compose(base, base)   to square               │
│                                                  │
│  O(n log k) instead of O(nk)                     │
└────────────────────────────────────────────────┘
```

---

## Example 9: Binary Lifting (Exponentiation on Functional Graphs)

```go
package main

import "fmt"

// Binary lifting: preprocess ancestors in a tree/functional graph
// Answer "k-th ancestor" queries in O(log k)

const MAXLOG = 20

func buildBinaryLifting(parent []int) [MAXLOG][]int {
	n := len(parent)
	var up [MAXLOG][]int

	up[0] = make([]int, n)
	copy(up[0], parent)

	for j := 1; j < MAXLOG; j++ {
		up[j] = make([]int, n)
		for i := 0; i < n; i++ {
			prev := up[j-1][i]
			if prev == -1 {
				up[j][i] = -1
			} else {
				up[j][i] = up[j-1][prev]
			}
		}
	}
	return up
}

func kthAncestor(up [MAXLOG][]int, node, k int) int {
	for j := 0; j < MAXLOG && node != -1; j++ {
		if k>>j&1 == 1 {
			node = up[j][node]
		}
	}
	return node
}

func main() {
	// Tree: 0 is root
	//     0
	//    / \
	//   1   2
	//  / \
	// 3   4
	// |
	// 5
	parent := []int{-1, 0, 0, 1, 1, 3}
	up := buildBinaryLifting(parent)

	queries := [][2]int{{5, 1}, {5, 2}, {5, 3}, {4, 1}, {4, 2}, {2, 1}}
	for _, q := range queries {
		node, k := q[0], q[1]
		ans := kthAncestor(up, node, k)
		fmt.Printf("  %d-th ancestor of %d = %d\n", k, node, ans)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Binary Lifting — K-th Ancestor                  │
├────────────────────────────────────────────────┤
│  Tree:        0                                  │
│             / \                                  │
│            1   2                                 │
│           / \                                    │
│          3   4                                   │
│          |                                       │
│          5                                       │
│                                                  │
│  up[0][i] = parent[i]      (2⁰ = 1st ancestor)  │
│  up[1][i] = up[0][up[0][i]] (2¹ = 2nd ancestor) │
│  up[j][i] = up[j-1][up[j-1][i]]                 │
│                                                  │
│  Query: 3rd ancestor of 5 (k=3 = 11₂)           │
│    bit 0 (1): up[0][5] = 3                       │
│    bit 1 (1): up[1][3] = up[0][up[0][3]]         │
│                       = up[0][1] = 0             │
│  Answer: 0 (root)                                │
└────────────────────────────────────────────────┘
```

---

## Example 10: Fast Exponentiation Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Fast Exponentiation Patterns ===")
	fmt.Println()

	patterns := []struct{ technique, use, complexity string }{
		{"Binary exponentiation", "a^n (integer)", "O(log n)"},
		{"Modular exponentiation", "a^n mod m (avoid overflow)", "O(log n)"},
		{"Matrix exponentiation", "Linear recurrences (fib, tribo)", "O(k³ log n)"},
		{"Super pow (array exponent)", "Huge exponents as digit array", "O(d log m)"},
		{"Permutation power", "Apply permutation k times", "O(n log k)"},
		{"Binary lifting", "K-th ancestor in tree", "O(n log n) setup, O(log k) query"},
		{"Pow(x, n) float", "LeetCode 50", "O(log n)"},
		{"Euler's theorem", "a^φ(m) ≡ 1 mod m", "Generalized Fermat"},
		{"Repeated squaring", "Core technique for all above", "Square then multiply"},
		{"Tower of exponents", "a^(b^c) mod m — use Euler's", "Recursive reduction"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-28s %-35s %s\n", p.technique, p.use, p.complexity)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Fast Exponentiation — Pattern Map                │
├────────────────────────────────────────────────┤
│  Core: Repeated Squaring                         │
│  result=1, base=a                                │
│  while exp > 0:                                  │
│    if exp&1: result *= base                      │
│    base *= base; exp >>= 1                       │
│                                                  │
│  Variants:                                       │
│  ───────────────────────────────────────────  │
│  Scalar  → O(log n), *= is integer multiply     │
│  Modular → O(log n), mod M at every step         │
│  Matrix  → O(k³ log n), *= is matrix multiply   │
│  Perm    → O(n log k), *= is compose             │
│  Lifting → up[j] = up[j-1] ∘ up[j-1]            │
│  Float   → handle n<0 by x=1/x                  │
│  Digit[] → process exponent digit by digit       │
└────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Core idea**: split exponent by binary digits, square the base each step
2. **Always use modular version** in competitive programming to avoid overflow
3. **Matrix exponentiation** extends fast power to linear recurrences
4. **Binary lifting** is exponentiation applied to ancestor queries in trees
5. **Time**: O(log n) for scalar, O(k³ log n) for k×k matrices

> **Next up:** Combinatorics →
