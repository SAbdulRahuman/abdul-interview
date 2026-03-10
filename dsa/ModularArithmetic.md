# Phase 21: Math & Geometry — Modular Arithmetic

## Overview

**Modular arithmetic** performs operations within a fixed range [0, m-1]. Essential for avoiding overflow, cryptography, and competitive programming.

| Property | Formula |
|----------|---------|
| Addition | `(a + b) % m` |
| Subtraction | `((a - b) % m + m) % m` |
| Multiplication | `(a * b) % m` |
| Division | `a * modInverse(b) % m` |
| Exponentiation | Fast power: `a^b % m` |
| Inverse | `a^(m-2) % m` (when m is prime) |

---

## Example 1: Basic Modular Operations

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modAdd(a, b int) int { return (a%MOD + b%MOD) % MOD }
func modSub(a, b int) int { return ((a%MOD - b%MOD) + MOD) % MOD }
func modMul(a, b int) int { return (a % MOD) * (b % MOD) % MOD }

func main() {
	a, b := 1_000_000_000, 999_999_999

	fmt.Println("With mod 10^9+7:")
	fmt.Printf("  %d + %d ≡ %d\n", a, b, modAdd(a, b))
	fmt.Printf("  %d - %d ≡ %d\n", a, b, modSub(a, b))
	fmt.Printf("  %d * %d ≡ %d\n", a, b, modMul(a, b))

	fmt.Println()
	fmt.Println("Why 10^9+7?")
	fmt.Println("  • It's prime (needed for modular inverse)")
	fmt.Println("  • Fits in 32-bit signed int")
	fmt.Println("  • (a*b) fits in 64-bit when a,b < 10^9+7")
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Modular Arithmetic Operations                    │
├──────────────────────────────────────────────────┤
│  a = 1,000,000,000    b = 999,999,999             │
│  M = 1,000,000,007 (10⁹+7)                       │
│                                                  │
│  ADD: (a%M + b%M) % M                            │
│    = (999999993 + 999999999) % M                  │
│    = 999999985                                    │
│                                                  │
│  SUB: ((a%M - b%M) + M) % M   ← avoid negative   │
│    = ((999999993 - 999999999) + M) % M            │
│    = 1                                            │
│                                                  │
│  MUL: (a%M) * (b%M) % M                          │
│    = 999999993 * 999999999 % M                    │
│    (uses 64-bit to avoid overflow)                │
└──────────────────────────────────────────────────┘
```

---

## Example 2: Modular Exponentiation (Fast Power)

```go
package main

import "fmt"

func modPow(base, exp, mod int) int {
	result := 1
	base %= mod
	for exp > 0 {
		if exp&1 == 1 {
			result = result * base % mod
		}
		exp >>= 1
		base = base * base % mod
	}
	return result
}

func main() {
	tests := [][3]int{
		{2, 10, 1000},     // 1024 % 1000 = 24
		{3, 100, 1000000007},
		{7, 256, 13},
	}
	for _, t := range tests {
		fmt.Printf("  %d^%d mod %d = %d\n", t[0], t[1], t[2], modPow(t[0], t[1], t[2]))
	}
	fmt.Println()
	fmt.Println("Time complexity: O(log exp)")
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Modular Exponentiation: 2^10 mod 1000            │
├──────────────────────────────────────────────────┤
│  exp=10 = 1010₂                                    │
│                                                  │
│  Step  exp  bit  base       result                │
│  ────  ───  ───  ─────────  ──────                │
│   1    10    0   2²=4       1 (skip)              │
│   2     5    1   4²=16      1×4=4                 │
│   3     2    0   16²=256    4 (skip)              │
│   4     1    1   256²       4×256=1024            │
│                                                  │
│  1024 mod 1000 = 24  ✓                            │
│  Only log₂(10) = 4 multiplies instead of 10      │
└──────────────────────────────────────────────────┘
```

---

## Example 3: Modular Inverse

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		exp >>= 1; base = base * base % mod
	}
	return result
}

// Fermat's little theorem: a^(-1) ≡ a^(p-2) mod p (p prime)
func modInverse(a, mod int) int {
	return modPow(a, mod-2, mod)
}

// Using extended GCD (works for any coprime mod)
func modInverseExtGCD(a, mod int) int {
	g, x, _ := extGCD(a%mod, mod)
	if g != 1 { return -1 } // inverse doesn't exist
	return (x%mod + mod) % mod
}

func extGCD(a, b int) (int, int, int) {
	if a == 0 { return b, 0, 1 }
	g, x1, y1 := extGCD(b%a, a)
	return g, y1 - (b/a)*x1, x1
}

func main() {
	for _, a := range []int{2, 3, 5, 7, 100} {
		inv := modInverse(a, MOD)
		verify := a * inv % MOD
		fmt.Printf("  %d^(-1) mod 10^9+7 = %d  (verify: %d * %d mod M = %d)\n",
			a, inv, a, inv, verify)
	}
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Modular Inverse: a⁻¹ mod p                        │
├──────────────────────────────────────────────────┤
│  Fermat's little theorem (p prime):               │
│    a^(p-1) ≡ 1 (mod p)                            │
│    a^(p-2) ≡ a⁻¹ (mod p)                          │
│                                                  │
│  Example: 5⁻¹ mod 10⁹+7                           │
│    = 5^(10⁹+5) mod 10⁹+7                         │
│    = 400000003                                    │
│    Verify: 5 × 400000003 mod M = 1  ✓             │
│                                                  │
│  Extended GCD (works for any coprime mod):        │
│    Find x,y: a·x + m·y = gcd(a,m) = 1            │
│    Then x mod m = a⁻¹ mod m                       │
└──────────────────────────────────────────────────┘
```

---

## Example 4: Modular Division

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		exp >>= 1; base = base * base % mod
	}
	return result
}

func modDiv(a, b, mod int) int {
	return a % mod * modPow(b, mod-2, mod) % mod
}

func main() {
	// nCr mod p
	fmt.Println("Combinations mod 10^9+7:")
	for _, pair := range [][2]int{{10, 3}, {20, 5}, {100, 50}} {
		n, r := pair[0], pair[1]

		// Compute n! / (r! * (n-r)!)
		num := 1
		for i := 0; i < r; i++ {
			num = num * ((n - i) % MOD) % MOD
		}
		den := 1
		for i := 1; i <= r; i++ {
			den = den * (i % MOD) % MOD
		}
		result := modDiv(num, den, MOD)
		fmt.Printf("  C(%d, %d) = %d\n", n, r, result)
	}
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Modular Division: (a/b) mod p                    │
├──────────────────────────────────────────────────┤
│  Can't divide mod M directly → multiply by b⁻¹   │
│  (a/b) mod p = a × b^(p-2) mod p                  │
│                                                  │
│  C(10,3) mod M:                                  │
│    num = 10×9×8 = 720                             │
│    den = 1×2×3 = 6                                │
│    result = 720 × 6^(M-2) mod M                   │
│           = 720 × modInverse(6) mod M             │
│           = 120  ✓                                │
└──────────────────────────────────────────────────┘
```

---

## Example 5: Factorial and Inverse Factorial (Precompute)

```go
package main

import "fmt"

const MOD = 1_000_000_007
const MAXN = 100001

var fact [MAXN]int
var invFact [MAXN]int

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		exp >>= 1; base = base * base % mod
	}
	return result
}

func precompute() {
	fact[0] = 1
	for i := 1; i < MAXN; i++ {
		fact[i] = fact[i-1] * i % MOD
	}
	invFact[MAXN-1] = modPow(fact[MAXN-1], MOD-2, MOD)
	for i := MAXN - 2; i >= 0; i-- {
		invFact[i] = invFact[i+1] * (i + 1) % MOD
	}
}

func nCr(n, r int) int {
	if r < 0 || r > n { return 0 }
	return fact[n] * invFact[r] % MOD * invFact[n-r] % MOD
}

func main() {
	precompute()

	fmt.Println("Precomputed nCr (O(1) per query):")
	pairs := [][2]int{{5, 2}, {10, 3}, {20, 10}, {100, 50}, {1000, 500}}
	for _, p := range pairs {
		fmt.Printf("  C(%d, %d) = %d\n", p[0], p[1], nCr(p[0], p[1]))
	}
}
```

**Textual Figure:**
```
┌───────────────────────────────────────────────────┐
│  Precomputed Factorial / Inv Factorial             │
├───────────────────────────────────────────────────┤
│  Step 1: fact[i] = fact[i-1] × i mod M             │
│    fact[0]=1  fact[1]=1  fact[2]=2  fact[3]=6     │
│    fact[4]=24 fact[5]=120 ...                     │
│                                                   │
│  Step 2: invFact[N] = fact[N]^(M-2) mod M         │
│    (single modPow call — Fermat's theorem)        │
│                                                   │
│  Step 3: invFact[i] = invFact[i+1]×(i+1) mod M   │
│    Fill backwards in O(n)                         │
│                                                   │
│  Query: nCr(n,r) = fact[n]×invFact[r]×invFact[n-r]│
│  O(n) setup, O(1) per query ✔                     │
└───────────────────────────────────────────────────┘
```

---

## Example 6: Chinese Remainder Theorem

```go
package main

import "fmt"

func extGCD(a, b int) (int, int, int) {
	if a == 0 { return b, 0, 1 }
	g, x1, y1 := extGCD(b%a, a)
	return g, y1 - (b/a)*x1, x1
}

// Solve: x ≡ a[i] (mod m[i]) for all i
func crt(remainders, moduli []int) (int, int) {
	curR, curM := remainders[0], moduli[0]

	for i := 1; i < len(remainders); i++ {
		r2, m2 := remainders[i], moduli[i]
		g, p, _ := extGCD(curM, m2)

		if (r2-curR)%g != 0 { return -1, -1 } // no solution

		lcm := curM / g * m2
		diff := (r2 - curR) / g
		curR = (curR + curM*((diff*p)%((m2/g)))) % lcm
		if curR < 0 { curR += lcm }
		curM = lcm
	}
	return curR, curM
}

func main() {
	// x ≡ 2 (mod 3), x ≡ 3 (mod 5), x ≡ 2 (mod 7)
	r := []int{2, 3, 2}
	m := []int{3, 5, 7}

	x, mod := crt(r, m)
	fmt.Printf("Solution: x ≡ %d (mod %d)\n", x, mod)
	fmt.Printf("Verify: %d%%3=%d, %d%%5=%d, %d%%7=%d\n",
		x, x%3, x, x%5, x, x%7)
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Chinese Remainder Theorem                       │
├─────────────────────────────────────────────────┤
│  Solve:  x ≡ 2 (mod 3)                           │
│         x ≡ 3 (mod 5)                            │
│         x ≡ 2 (mod 7)                            │
│                                                 │
│  Step 1: Combine first two:                     │
│    x ≡ 2 (mod 3) ∧ x ≡ 3 (mod 5)                │
│    lcm(3,5)=15, find x: 8 ≡ 2(mod3), 8≡3(mod5) │
│    → x ≡ 8 (mod 15)                              │
│                                                 │
│  Step 2: Combine with third:                    │
│    x ≡ 8 (mod 15) ∧ x ≡ 2 (mod 7)               │
│    lcm(15,7)=105                                │
│    → x ≡ 23 (mod 105)                            │
│                                                 │
│  Verify: 23%3=2✓  23%5=3✓  23%7=2✓            │
└─────────────────────────────────────────────────┘
```

---

## Example 7: Matrix Exponentiation (Fibonacci mod M)

```go
package main

import "fmt"

const MOD = 1_000_000_007

type Matrix [2][2]int

func matMul(a, b Matrix) Matrix {
	var c Matrix
	for i := 0; i < 2; i++ {
		for j := 0; j < 2; j++ {
			for k := 0; k < 2; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % MOD
			}
		}
	}
	return c
}

func matPow(m Matrix, n int) Matrix {
	result := Matrix{{1, 0}, {0, 1}} // identity
	for n > 0 {
		if n&1 == 1 { result = matMul(result, m) }
		m = matMul(m, m)
		n >>= 1
	}
	return result
}

func fibMod(n int) int {
	if n <= 1 { return n }
	m := Matrix{{1, 1}, {1, 0}}
	result := matPow(m, n-1)
	return result[0][0]
}

func main() {
	for _, n := range []int{10, 50, 100, 1000, 1000000} {
		fmt.Printf("  fib(%d) mod 10^9+7 = %d\n", n, fibMod(n))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Matrix Exponentiation for Fibonacci              │
├─────────────────────────────────────────────────┤
│  [F(n+1)]   [1  1]^n   [F(1)]                    │
│  [F(n)  ] = [1  0]   × [F(0)]                    │
│                                                 │
│  M^10 via binary exp (10=1010₂):                │
│    M¹ → M² → (skip M⁴) → M⁸                   │
│    result = M⁸ × M² = M¹⁰                       │
│                                                 │
│  fib(10⁶): only ~20 matrix multiplies!          │
│  O(8 × log n) = O(log n) per query              │
└─────────────────────────────────────────────────┘
```

---

## Example 8: Modular Arithmetic for Large Numbers

```go
package main

import "fmt"

const MOD = 1_000_000_007

// Count paths in grid (n x m) — answer is C(n+m-2, n-1)
func gridPaths(n, m int) int {
	// Use modular nCr
	total := n + m - 2
	choose := n - 1

	num := 1
	den := 1
	for i := 0; i < choose; i++ {
		num = num * ((total - i) % MOD) % MOD
		den = den * ((i + 1) % MOD) % MOD
	}

	// Modular division
	return num * modPow(den, MOD-2, MOD) % MOD
}

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		exp >>= 1; base = base * base % mod
	}
	return result
}

func main() {
	tests := [][2]int{{3, 3}, {5, 5}, {10, 10}, {100, 100}}
	for _, t := range tests {
		fmt.Printf("  Grid %dx%d paths: %d\n", t[0], t[1], gridPaths(t[0], t[1]))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Grid Paths with Modular Arithmetic             │
├─────────────────────────────────────────────────┤
│  Grid n×m → C(n+m-2, n-1) paths                  │
│                                                 │
│  3×3:  C(4,2) = 6                               │
│  5×5:  C(8,4) = 70                              │
│  100×100: C(198,99) mod 10⁹+7                   │
│     num = 198×197×...×100 (mod M at each step)  │
│     den = 99!     (mod M)                       │
│     result = num × den^(M-2) mod M              │
└─────────────────────────────────────────────────┘
```

---

## Example 9: Sum of Geometric Series Mod

```go
package main

import "fmt"

const MOD = 1_000_000_007

func modPow(base, exp, mod int) int {
	result := 1; base %= mod
	for exp > 0 {
		if exp&1 == 1 { result = result * base % mod }
		exp >>= 1; base = base * base % mod
	}
	return result
}

// Sum = 1 + r + r^2 + ... + r^(n-1) = (r^n - 1) / (r - 1)
func geometricSumMod(r, n, mod int) int {
	if r == 1 { return n % mod }
	num := (modPow(r, n, mod) - 1 + mod) % mod
	den := modPow(r-1, mod-2, mod)
	return num * den % mod
}

func main() {
	for _, r := range []int{2, 3, 5} {
		for _, n := range []int{10, 100, 1000} {
			fmt.Printf("  Sum(r=%d, n=%d) mod 10^9+7 = %d\n", r, n, geometricSumMod(r, n, MOD))
		}
	}
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Geometric Series Sum mod M                       │
├──────────────────────────────────────────────────┤
│  S = 1 + r + r² + ... + r^(n-1)                   │
│    = (r^n - 1) / (r - 1)                         │
│                                                  │
│  Sum(r=2, n=10) mod M:                           │
│    = (2¹⁰ - 1) / (2 - 1)                         │
│    = (1024 - 1) / 1 = 1023                       │
│                                                  │
│  Sum(r=3, n=10) mod M:                           │
│    num = (3¹⁰ - 1) mod M = 59048                 │
│    den = (3 - 1)⁻¹ mod M = 2⁻¹ mod M             │
│    S = 59048 × modInverse(2) mod M = 29524       │
└──────────────────────────────────────────────────┘
```

---

## Example 10: Modular Arithmetic Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Modular Arithmetic Patterns ===")
	fmt.Println()

	patterns := []struct{ operation, formula, note string }{
		{"(a+b) mod m", "(a%m + b%m) % m", "Safe addition"},
		{"(a-b) mod m", "((a%m - b%m) + m) % m", "Handle negative"},
		{"(a*b) mod m", "(a%m) * (b%m) % m", "Check overflow"},
		{"(a/b) mod p", "a * b^(p-2) mod p", "Fermat's (p prime)"},
		{"a^b mod m", "Binary exponentiation", "O(log b)"},
		{"n! mod p", "Precompute array", "O(n) setup, O(1) query"},
		{"nCr mod p", "fact[n]*invFact[r]*invFact[n-r]", "O(1) with precomp"},
		{"Fibonacci mod m", "Matrix exponentiation", "O(log n)"},
		{"CRT", "Combine modular equations", "When moduli coprime"},
		{"Sum of series mod", "(r^n-1)/(r-1) mod p", "Geometric series"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-18s %-35s → %s\n", p.operation, p.formula, p.note)
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Modular Arithmetic Decision Tree                │
├─────────────────────────────────────────────────┤
│  Need  a+b mod M?  → (a%M + b%M) % M            │
│  Need  a-b mod M?  → ((a%M - b%M) + M) % M     │
│  Need  a*b mod M?  → (a%M) * (b%M) % M         │
│  Need  a/b mod M?  ─┬─ M prime?                 │
│                     ├─ Yes: a × b^(M-2) % M    │
│                     └─ No:  Extended GCD        │
│  Need  a^b mod M?  → Binary exponentiation      │
│  Need  nCr mod M?  → Precompute fact[]/invFact[]│
│  Need  fib mod M?  → Matrix exponentiation      │
│  Multiple mods?    → Chinese Remainder Theorem  │
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **MOD = 10^9+7**: prime, fits int32, product fits int64
2. **Fast power**: O(log n) via binary exponentiation
3. **Modular inverse**: `a^(p-2) mod p` (Fermat) or extended GCD
4. **Precompute factorials**: O(n) setup → O(1) nCr queries
5. **CRT**: combine multiple modular equations into one

> **Next up:** Fast Exponentiation →
