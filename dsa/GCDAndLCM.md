# Phase 21: Math & Geometry — GCD and LCM

## Overview

**GCD** (Greatest Common Divisor) and **LCM** (Least Common Multiple) are fundamental math operations used in simplifying fractions, timing problems, and number theory.

| Operation | Formula | Go |
|-----------|---------|-----|
| GCD | Euclidean algorithm | `gcd(a, b) = gcd(b, a%b)` |
| LCM | `a * b / gcd(a, b)` | Avoid overflow: `a / gcd(a,b) * b` |

---

## Example 1: GCD — Euclidean Algorithm

```go
package main

import "fmt"

func gcdRecursive(a, b int) int {
	if b == 0 { return a }
	return gcdRecursive(b, a%b)
}

func gcdIterative(a, b int) int {
	for b != 0 {
		a, b = b, a%b
	}
	return a
}

func main() {
	pairs := [][2]int{{12, 8}, {100, 75}, {17, 13}, {48, 36}, {0, 5}}
	for _, p := range pairs {
		fmt.Printf("GCD(%d, %d) = %d (iterative: %d)\n",
			p[0], p[1], gcdRecursive(p[0], p[1]), gcdIterative(p[0], p[1]))
	}
}
```

---

## Example 2: LCM

```go
package main

import "fmt"

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

func lcm(a, b int) int {
	return a / gcd(a, b) * b // divide first to avoid overflow
}

func lcmMultiple(nums []int) int {
	result := nums[0]
	for i := 1; i < len(nums); i++ {
		result = lcm(result, nums[i])
	}
	return result
}

func main() {
	pairs := [][2]int{{4, 6}, {12, 15}, {7, 5}, {100, 75}}
	for _, p := range pairs {
		fmt.Printf("LCM(%d, %d) = %d\n", p[0], p[1], lcm(p[0], p[1]))
	}

	fmt.Println()
	nums := []int{2, 3, 4, 5, 6}
	fmt.Printf("LCM(%v) = %d\n", nums, lcmMultiple(nums))
}
```

---

## Example 3: Extended Euclidean Algorithm

```go
package main

import "fmt"

// Returns gcd, x, y such that a*x + b*y = gcd(a, b)
func extGCD(a, b int) (int, int, int) {
	if a == 0 {
		return b, 0, 1
	}
	g, x1, y1 := extGCD(b%a, a)
	return g, y1 - (b/a)*x1, x1
}

func main() {
	pairs := [][2]int{{35, 15}, {12, 8}, {17, 13}, {99, 78}}
	for _, p := range pairs {
		g, x, y := extGCD(p[0], p[1])
		fmt.Printf("GCD(%d, %d) = %d = %d*(%d) + %d*(%d)\n",
			p[0], p[1], g, p[0], x, p[1], y)
		fmt.Printf("  Verify: %d*%d + %d*%d = %d ✓\n",
			p[0], x, p[1], y, p[0]*x+p[1]*y)
	}
}
```

---

## Example 4: Fraction Simplification

```go
package main

import "fmt"

func gcd(a, b int) int {
	if a < 0 { a = -a }
	if b < 0 { b = -b }
	for b != 0 { a, b = b, a%b }
	return a
}

type Fraction struct{ num, den int }

func simplify(f Fraction) Fraction {
	g := gcd(f.num, f.den)
	f.num /= g
	f.den /= g
	if f.den < 0 { f.num = -f.num; f.den = -f.den }
	return f
}

func addFraction(a, b Fraction) Fraction {
	return simplify(Fraction{
		a.num*b.den + b.num*a.den,
		a.den * b.den,
	})
}

func main() {
	fracs := []Fraction{{6, 8}, {-4, 6}, {15, 25}, {12, -18}}
	for _, f := range fracs {
		s := simplify(f)
		fmt.Printf("  %d/%d → %d/%d\n", f.num, f.den, s.num, s.den)
	}

	fmt.Println()
	a, b := Fraction{1, 3}, Fraction{1, 6}
	sum := addFraction(a, b)
	fmt.Printf("  %d/%d + %d/%d = %d/%d\n", a.num, a.den, b.num, b.den, sum.num, sum.den)
}
```

---

## Example 5: GCD of Array (LeetCode-style)

```go
package main

import "fmt"

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

// Check if we can make all elements equal using GCD
func canMakeEqual(nums []int) int {
	g := nums[0]
	for _, n := range nums[1:] {
		g = gcd(g, n)
	}
	return g
}

// Minimum operations to make all GCDs equal to 1
func hasGCDOne(nums []int) bool {
	g := nums[0]
	for _, n := range nums[1:] {
		g = gcd(g, n)
		if g == 1 { return true }
	}
	return g == 1
}

func main() {
	tests := [][]int{
		{12, 18, 24},
		{5, 10, 15, 25},
		{7, 14, 21},
		{6, 10, 15},
	}
	for _, nums := range tests {
		g := canMakeEqual(nums)
		fmt.Printf("GCD(%v) = %d, has GCD 1: %v\n", nums, g, hasGCDOne(nums))
	}
}
```

---

## Example 6: Water Jug Problem (GCD Application)

```go
package main

import "fmt"

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

// Can we measure exactly target liters with jugs of capacity a and b?
func canMeasure(a, b, target int) bool {
	if target > a+b { return false }
	if a == 0 || b == 0 { return target == a || target == b || target == 0 }
	return target%gcd(a, b) == 0
}

func main() {
	tests := []struct{ a, b, target int }{
		{3, 5, 4},
		{2, 6, 5},
		{1, 1, 12},
		{3, 5, 7},
	}
	for _, t := range tests {
		fmt.Printf("Jugs(%d, %d) → measure %d: %v\n",
			t.a, t.b, t.target, canMeasure(t.a, t.b, t.target))
	}
	fmt.Println()
	fmt.Println("Bézout's identity: ax + by = target solvable iff gcd(a,b) | target")
}
```

---

## Example 7: Ugly Numbers (GCD/LCM Application)

```go
package main

import "fmt"

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

func lcm(a, b int) int { return a / gcd(a, b) * b }

// Nth ugly number (factors only 2, 3, 5)
func nthUglyNumber(n int) int {
	ugly := make([]int, n)
	ugly[0] = 1
	i2, i3, i5 := 0, 0, 0

	for i := 1; i < n; i++ {
		next2 := ugly[i2] * 2
		next3 := ugly[i3] * 3
		next5 := ugly[i5] * 5

		minVal := min(next2, min(next3, next5))
		ugly[i] = minVal

		if minVal == next2 { i2++ }
		if minVal == next3 { i3++ }
		if minVal == next5 { i5++ }
	}
	return ugly[n-1]
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	fmt.Println("First 15 ugly numbers:")
	for i := 1; i <= 15; i++ {
		fmt.Printf("  %d: %d\n", i, nthUglyNumber(i))
	}
}
```

---

## Example 8: Binary GCD (Stein's Algorithm)

```go
package main

import "fmt"

func binaryGCD(a, b int) int {
	if a == 0 { return b }
	if b == 0 { return a }

	// Find common power of 2
	shift := 0
	for (a|b)&1 == 0 {
		a >>= 1
		b >>= 1
		shift++
	}

	// Remove factors of 2 from a
	for a&1 == 0 { a >>= 1 }

	for b != 0 {
		for b&1 == 0 { b >>= 1 }
		if a > b { a, b = b, a }
		b -= a
	}
	return a << shift
}

func main() {
	pairs := [][2]int{{48, 36}, {100, 75}, {17, 13}, {1024, 768}}
	for _, p := range pairs {
		fmt.Printf("BinaryGCD(%d, %d) = %d\n", p[0], p[1], binaryGCD(p[0], p[1]))
	}
	fmt.Println()
	fmt.Println("Stein's algorithm: uses only subtraction and shifts (no division)")
}
```

---

## Example 9: Count Pairs with GCD = K

```go
package main

import "fmt"

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

func countPairsWithGCD(nums []int, k int) int {
	count := 0
	n := len(nums)
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if gcd(nums[i], nums[j]) == k {
				count++
			}
		}
	}
	return count
}

func main() {
	nums := []int{2, 4, 6, 8, 10}
	for k := 1; k <= 5; k++ {
		c := countPairsWithGCD(nums, k)
		if c > 0 {
			fmt.Printf("Pairs with GCD=%d: %d\n", k, c)
		}
	}
}
```

---

## Example 10: GCD/LCM Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== GCD / LCM Patterns ===")
	fmt.Println()

	patterns := []struct{ topic, key, use string }{
		{"Euclidean GCD", "gcd(a,b) = gcd(b, a%b)", "O(log min(a,b))"},
		{"Extended GCD", "ax + by = gcd(a,b)", "Modular inverse, Bézout"},
		{"LCM from GCD", "lcm = a/gcd(a,b)*b", "Divide first to avoid overflow"},
		{"Array GCD", "Fold gcd over array", "Simplify set of numbers"},
		{"Array LCM", "Fold lcm over array", "Synchronization periods"},
		{"Water jug", "target % gcd == 0", "Bézout's identity"},
		{"Fraction simplify", "Divide by gcd", "Rational arithmetic"},
		{"Binary GCD", "Shifts + subtract", "No division needed"},
		{"Coprime check", "gcd(a,b) == 1", "RSA, Euler's totient"},
		{"GCD subarray", "Prefix GCD", "Range GCD queries"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-18s %-30s → %s\n", p.topic, p.key, p.use)
	}
}
```

---

## Key Takeaways

1. Euclidean algorithm: `gcd(a, b) = gcd(b, a%b)`, O(log min(a,b))
2. LCM: always compute as `a / gcd(a,b) * b` to prevent overflow
3. Extended GCD: finds x, y where `ax + by = gcd(a,b)` — needed for modular inverse
4. Water jug / Bézout: target measurable iff divisible by gcd
5. Binary GCD (Stein's): uses only shifts and subtraction — useful on systems without division

> **Next up:** Sieve of Eratosthenes →
