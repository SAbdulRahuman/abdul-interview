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

---

## Key Takeaways

1. **Core idea**: split exponent by binary digits, square the base each step
2. **Always use modular version** in competitive programming to avoid overflow
3. **Matrix exponentiation** extends fast power to linear recurrences
4. **Binary lifting** is exponentiation applied to ancestor queries in trees
5. **Time**: O(log n) for scalar, O(k³ log n) for k×k matrices

> **Next up:** Combinatorics →
