# Phase 21: Math & Geometry — Combinatorics

## Overview

**Combinatorics** deals with counting, arrangements, and selections. Essential for probability, optimization, and many DSA problems.

| Formula | Value | Meaning |
|---------|-------|---------|
| P(n,r) = n!/(n-r)! | Permutations | Ordered selections |
| C(n,r) = n!/(r!(n-r)!) | Combinations | Unordered selections |
| C(n,r) = C(n-1,r-1) + C(n-1,r) | Pascal's identity | DP relation |
| Stars & bars | C(n+k-1, k-1) | Distribute n items into k bins |

---

## Example 1: Permutations and Combinations

```go
package main

import "fmt"

func factorial(n int) int {
	if n <= 1 { return 1 }
	r := 1
	for i := 2; i <= n; i++ { r *= i }
	return r
}

func P(n, r int) int { return factorial(n) / factorial(n-r) }
func C(n, r int) int { return factorial(n) / (factorial(r) * factorial(n-r)) }

func main() {
	fmt.Println("Permutations P(n,r) — order matters:")
	for _, p := range [][2]int{{5, 2}, {5, 3}, {10, 3}} {
		fmt.Printf("  P(%d,%d) = %d\n", p[0], p[1], P(p[0], p[1]))
	}

	fmt.Println("\nCombinations C(n,r) — order doesn't matter:")
	for _, p := range [][2]int{{5, 2}, {5, 3}, {10, 3}, {10, 5}} {
		fmt.Printf("  C(%d,%d) = %d\n", p[0], p[1], C(p[0], p[1]))
	}
}
```

---

## Example 2: Pascal's Triangle (LeetCode 118/119)

```go
package main

import "fmt"

func pascalTriangle(n int) [][]int {
	tri := make([][]int, n)
	for i := 0; i < n; i++ {
		tri[i] = make([]int, i+1)
		tri[i][0], tri[i][i] = 1, 1
		for j := 1; j < i; j++ {
			tri[i][j] = tri[i-1][j-1] + tri[i-1][j]
		}
	}
	return tri
}

func getRow(n int) []int {
	row := make([]int, n+1)
	row[0] = 1
	for i := 1; i <= n; i++ {
		row[i] = row[i-1] * (n - i + 1) / i
	}
	return row
}

func main() {
	fmt.Println("Pascal's Triangle (5 rows):")
	for i, row := range pascalTriangle(5) {
		fmt.Printf("  Row %d: %v\n", i, row)
	}

	fmt.Println("\nSingle row (O(n) space):")
	fmt.Printf("  Row 10: %v\n", getRow(10))
}
```

---

## Example 3: nCr with Modular Arithmetic (Large n)

```go
package main

import "fmt"

const MOD = 1_000_000_007
const MAXN = 200001

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

func init() {
	fact[0] = 1
	for i := 1; i < MAXN; i++ { fact[i] = fact[i-1] * i % MOD }
	invFact[MAXN-1] = modPow(fact[MAXN-1], MOD-2, MOD)
	for i := MAXN - 2; i >= 0; i-- { invFact[i] = invFact[i+1] * (i + 1) % MOD }
}

func nCr(n, r int) int {
	if r < 0 || r > n { return 0 }
	return fact[n] * invFact[r] % MOD * invFact[n-r] % MOD
}

func main() {
	pairs := [][2]int{{5, 2}, {10, 5}, {100, 50}, {200000, 100000}}
	for _, p := range pairs {
		fmt.Printf("  C(%d, %d) mod 10^9+7 = %d\n", p[0], p[1], nCr(p[0], p[1]))
	}
}
```

---

## Example 4: Catalan Numbers

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

// C(n) = C(2n, n) / (n+1)
func catalan(n int) int {
	if n <= 1 { return 1 }
	// C(2n, n) / (n+1)
	num := 1
	for i := 0; i < n; i++ {
		num = num * ((2*n - i) % MOD) % MOD
	}
	den := 1
	for i := 1; i <= n; i++ {
		den = den * (i % MOD) % MOD
	}
	den = den * ((n + 1) % MOD) % MOD
	return num * modPow(den, MOD-2, MOD) % MOD
}

// DP version (small n)
func catalanDP(n int) []int {
	dp := make([]int, n+1)
	dp[0], dp[1] = 1, 1
	for i := 2; i <= n; i++ {
		for j := 0; j < i; j++ {
			dp[i] += dp[j] * dp[i-1-j]
		}
	}
	return dp
}

func main() {
	fmt.Println("Catalan numbers (DP):")
	dp := catalanDP(10)
	for i, v := range dp {
		fmt.Printf("  C(%d) = %d\n", i, v)
	}

	fmt.Println("\nCatalan (mod, large n):")
	for _, n := range []int{100, 1000, 10000} {
		fmt.Printf("  C(%d) mod 10^9+7 = %d\n", n, catalan(n))
	}

	fmt.Println("\nApplications: valid parentheses, BSTs, triangulations, paths")
}
```

---

## Example 5: Stars and Bars (Distribute Items)

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

func init() {
	fact[0] = 1
	for i := 1; i < MAXN; i++ { fact[i] = fact[i-1] * i % MOD }
	invFact[MAXN-1] = modPow(fact[MAXN-1], MOD-2, MOD)
	for i := MAXN - 2; i >= 0; i-- { invFact[i] = invFact[i+1] * (i + 1) % MOD }
}

func nCr(n, r int) int {
	if r < 0 || r > n { return 0 }
	return fact[n] * invFact[r] % MOD * invFact[n-r] % MOD
}

// Ways to put n identical balls into k distinct boxes
// Non-negative solutions: C(n + k - 1, k - 1)
// Positive solutions: C(n - 1, k - 1)
func starsAndBars(n, k int, allowZero bool) int {
	if allowZero {
		return nCr(n+k-1, k-1)
	}
	return nCr(n-1, k-1)
}

func main() {
	fmt.Println("Stars and Bars:")
	
	// Distribute 10 cookies among 3 kids
	fmt.Printf("  10 cookies, 3 kids (allow 0): %d ways\n", starsAndBars(10, 3, true))
	fmt.Printf("  10 cookies, 3 kids (at least 1): %d ways\n", starsAndBars(10, 3, false))

	// x1 + x2 + x3 + x4 = 20, xi >= 0
	fmt.Printf("  Solutions of x1+x2+x3+x4=20: %d\n", starsAndBars(20, 4, true))
}
```

---

## Example 6: Inclusion-Exclusion Principle

```go
package main

import "fmt"

// Count numbers 1..n divisible by at least one of the given primes
func countDivisible(n int, primes []int) int {
	m := len(primes)
	total := 0

	// Iterate subsets using bitmask
	for mask := 1; mask < (1 << m); mask++ {
		product := 1
		bits := 0
		for i := 0; i < m; i++ {
			if mask>>i&1 == 1 {
				product *= primes[i]
				bits++
			}
		}
		count := n / product
		if bits%2 == 1 {
			total += count // odd subset: add
		} else {
			total -= count // even subset: subtract
		}
	}
	return total
}

// Euler's totient using inclusion-exclusion
func eulerTotient(n int) int {
	result := n
	temp := n
	for p := 2; p*p <= temp; p++ {
		if temp%p == 0 {
			for temp%p == 0 { temp /= p }
			result = result / p * (p - 1)
		}
	}
	if temp > 1 { result = result / temp * (temp - 1) }
	return result
}

func main() {
	n := 100
	primes := []int{2, 3, 5}
	fmt.Printf("Numbers 1..%d divisible by %v: %d\n", n, primes, countDivisible(n, primes))
	fmt.Printf("Numbers 1..%d coprime to all: %d\n", n, n-countDivisible(n, primes))

	fmt.Println("\nEuler's Totient φ(n):")
	for _, v := range []int{1, 6, 10, 12, 100} {
		fmt.Printf("  φ(%d) = %d\n", v, eulerTotient(v))
	}
}
```

---

## Example 7: Derangements (No Element in Original Position)

```go
package main

import "fmt"

// D(n) = (n-1) * (D(n-1) + D(n-2))
// Also: D(n) = n! * Σ(-1)^k / k! for k=0..n
func derangements(n int) []int {
	if n == 0 { return []int{1} }
	if n == 1 { return []int{1, 0} }

	d := make([]int, n+1)
	d[0] = 1
	d[1] = 0
	for i := 2; i <= n; i++ {
		d[i] = (i - 1) * (d[i-1] + d[i-2])
	}
	return d
}

func main() {
	d := derangements(10)
	fmt.Println("Derangements D(n):")
	fact := 1
	for i, v := range d {
		if i > 0 { fact *= i }
		ratio := 0.0
		if fact > 0 { ratio = float64(v) / float64(fact) }
		fmt.Printf("  D(%d) = %-10d  (out of %d! = %-10d  ratio = %.6f)\n", i, v, i, fact, ratio)
	}
	fmt.Println("\nRatio converges to 1/e ≈ 0.367879")
}
```

---

## Example 8: Unique Paths (Grid Combinatorics, LeetCode 62)

```go
package main

import "fmt"

// Unique paths from top-left to bottom-right in m x n grid
// Answer = C(m+n-2, m-1)
func uniquePaths(m, n int) int {
	// Choose (m-1) down moves out of (m+n-2) total moves
	total := m + n - 2
	choose := m - 1
	if choose > total-choose { choose = total - choose }

	result := 1
	for i := 0; i < choose; i++ {
		result = result * (total - i) / (i + 1)
	}
	return result
}

// With obstacles (DP)
func uniquePathsWithObstacles(grid [][]int) int {
	m, n := len(grid), len(grid[0])
	dp := make([]int, n)
	if grid[0][0] == 0 { dp[0] = 1 }

	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			if grid[i][j] == 1 {
				dp[j] = 0
			} else if j > 0 {
				dp[j] += dp[j-1]
			}
		}
	}
	return dp[n-1]
}

func main() {
	for _, p := range [][2]int{{3, 7}, {3, 3}, {5, 5}, {10, 10}} {
		fmt.Printf("  Grid %dx%d: %d paths\n", p[0], p[1], uniquePaths(p[0], p[1]))
	}

	grid := [][]int{{0, 0, 0}, {0, 1, 0}, {0, 0, 0}}
	fmt.Printf("\n  Grid with obstacle: %d paths\n", uniquePathsWithObstacles(grid))
}
```

---

## Example 9: Multinomial Coefficients (Anagram Counting)

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

func init() {
	fact[0] = 1
	for i := 1; i < MAXN; i++ { fact[i] = fact[i-1] * i % MOD }
	invFact[MAXN-1] = modPow(fact[MAXN-1], MOD-2, MOD)
	for i := MAXN - 2; i >= 0; i-- { invFact[i] = invFact[i+1] * (i + 1) % MOD }
}

// Count distinct permutations of string (with repeats)
// n! / (c1! * c2! * ... * ck!)
func countAnagrams(s string) int {
	freq := make(map[rune]int)
	for _, c := range s { freq[c]++ }

	n := len(s)
	result := fact[n]
	for _, cnt := range freq {
		result = result * invFact[cnt] % MOD
	}
	return result
}

func main() {
	words := []string{"aab", "aaabb", "abcde", "mississippi", "aabbccdd"}
	for _, w := range words {
		fmt.Printf("  \"%s\" → %d distinct arrangements\n", w, countAnagrams(w))
	}
}
```

---

## Example 10: Combinatorial Identities and Patterns

```go
package main

import "fmt"

func C(n, r int) int {
	if r < 0 || r > n { return 0 }
	if r > n-r { r = n - r }
	result := 1
	for i := 0; i < r; i++ {
		result = result * (n - i) / (i + 1)
	}
	return result
}

func main() {
	fmt.Println("=== Combinatorial Identities ===")
	n := 8

	// 1. Sum of row = 2^n
	sum := 0
	for r := 0; r <= n; r++ { sum += C(n, r) }
	fmt.Printf("  ΣC(%d,r) = %d  (should be 2^%d = %d)\n", n, sum, n, 1<<n)

	// 2. Alternating sum = 0
	altSum := 0
	for r := 0; r <= n; r++ {
		if r%2 == 0 { altSum += C(n, r) } else { altSum -= C(n, r) }
	}
	fmt.Printf("  Σ(-1)^r·C(%d,r) = %d  (should be 0)\n", n, altSum)

	// 3. Vandermonde's: C(m+n, r) = Σ C(m,k)·C(n,r-k)
	m, r := 5, 4
	lhs := C(m+n, r)
	rhs := 0
	for k := 0; k <= r; k++ { rhs += C(m, k) * C(n, r-k) }
	fmt.Printf("  Vandermonde: C(%d,%d) = %d = Σ C(%d,k)·C(%d,%d-k) = %d\n",
		m+n, r, lhs, m, n, r, rhs)

	// 4. Hockey stick: C(r,r) + C(r+1,r) + ... + C(n,r) = C(n+1, r+1)
	r = 2
	hockeySum := 0
	for i := r; i <= n; i++ { hockeySum += C(i, r) }
	fmt.Printf("  Hockey stick: Σ C(i,%d) for i=%d..%d = %d = C(%d,%d) = %d\n",
		r, r, n, hockeySum, n+1, r+1, C(n+1, r+1))

	fmt.Println("\n=== Key Patterns ===")
	patterns := []struct{ name, formula string }{
		{"Permutation", "P(n,r) = n!/(n-r)!"},
		{"Combination", "C(n,r) = n!/(r!(n-r)!)"},
		{"Pascal's", "C(n,r) = C(n-1,r-1) + C(n-1,r)"},
		{"Stars & bars", "C(n+k-1, k-1)"},
		{"Catalan", "C(2n,n)/(n+1)"},
		{"Derangement", "D(n) = (n-1)(D(n-1)+D(n-2))"},
		{"Multinomial", "n!/(c1!·c2!·...·ck!)"},
		{"Inc-Exc", "Σ(-1)^|S|·f(∩S)"},
	}
	for _, p := range patterns {
		fmt.Printf("  %-14s %s\n", p.name, p.formula)
	}
}
```

---

## Key Takeaways

1. **Precompute factorials** for O(1) nCr queries with modular arithmetic
2. **Pascal's triangle** gives DP relation: `C(n,r) = C(n-1,r-1) + C(n-1,r)`
3. **Catalan numbers** appear in parentheses, BSTs, triangulations
4. **Inclusion-Exclusion** handles "at least one" counting
5. **Stars and bars** for distributing identical items into distinct groups

> **Next up:** Coordinate Geometry →
