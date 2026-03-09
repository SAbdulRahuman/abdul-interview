# Phase 21: Math & Geometry — Sieve of Eratosthenes

## Overview

The **Sieve of Eratosthenes** efficiently finds all prime numbers up to a limit n by iteratively marking multiples of each prime as composite. Time: O(n log log n), Space: O(n).

```
Initial:  2 3 4 5 6 7 8 9 10 11 12 13 14 15
Mark 2s:  2 3 _ 5 _ 7 _ 9 __ 11 __ 13 __ 15
Mark 3s:  2 3 _ 5 _ 7 _ _ __ 11 __ 13 __ __
Result:   2 3   5   7       11    13
```

---

## Example 1: Basic Sieve

```go
package main

import "fmt"

func sieveOfEratosthenes(n int) []bool {
	isPrime := make([]bool, n+1)
	for i := 2; i <= n; i++ { isPrime[i] = true }

	for i := 2; i*i <= n; i++ {
		if isPrime[i] {
			for j := i * i; j <= n; j += i {
				isPrime[j] = false
			}
		}
	}
	return isPrime
}

func main() {
	n := 50
	isPrime := sieveOfEratosthenes(n)

	fmt.Printf("Primes up to %d: ", n)
	for i := 2; i <= n; i++ {
		if isPrime[i] { fmt.Printf("%d ", i) }
	}
	fmt.Println()
}
```

---

## Example 2: Count Primes (LeetCode 204)

```go
package main

import "fmt"

func countPrimes(n int) int {
	if n < 2 { return 0 }
	isPrime := make([]bool, n)
	for i := 2; i < n; i++ { isPrime[i] = true }

	for i := 2; i*i < n; i++ {
		if isPrime[i] {
			for j := i * i; j < n; j += i {
				isPrime[j] = false
			}
		}
	}

	count := 0
	for i := 2; i < n; i++ {
		if isPrime[i] { count++ }
	}
	return count
}

func main() {
	for _, n := range []int{10, 100, 1000, 10000, 100000} {
		fmt.Printf("Primes < %6d: %d\n", n, countPrimes(n))
	}
}
```

---

## Example 3: Segmented Sieve (for large ranges)

```go
package main

import (
	"fmt"
	"math"
)

func simpleSieve(limit int) []int {
	isPrime := make([]bool, limit+1)
	for i := 2; i <= limit; i++ { isPrime[i] = true }
	for i := 2; i*i <= limit; i++ {
		if isPrime[i] {
			for j := i * i; j <= limit; j += i { isPrime[j] = false }
		}
	}
	primes := []int{}
	for i := 2; i <= limit; i++ {
		if isPrime[i] { primes = append(primes, i) }
	}
	return primes
}

func segmentedSieve(lo, hi int) []int {
	limit := int(math.Sqrt(float64(hi))) + 1
	smallPrimes := simpleSieve(limit)

	size := hi - lo + 1
	isPrime := make([]bool, size)
	for i := range isPrime { isPrime[i] = true }

	for _, p := range smallPrimes {
		start := ((lo + p - 1) / p) * p
		if start == p { start = p * p }

		for j := start; j <= hi; j += p {
			isPrime[j-lo] = false
		}
	}

	if lo <= 1 && 1-lo >= 0 { isPrime[1-lo] = false }
	if lo == 0 { isPrime[0] = false }

	result := []int{}
	for i, prime := range isPrime {
		if prime && i+lo >= 2 {
			result = append(result, i+lo)
		}
	}
	return result
}

func main() {
	primes := segmentedSieve(100, 150)
	fmt.Printf("Primes in [100, 150]: %v\n", primes)

	primes2 := segmentedSieve(1000000, 1000100)
	fmt.Printf("Primes in [1000000, 1000100]: %v\n", primes2)
}
```

---

## Example 4: Smallest Prime Factor (SPF) Sieve

```go
package main

import "fmt"

func spfSieve(n int) []int {
	spf := make([]int, n+1)
	for i := range spf { spf[i] = i }

	for i := 2; i*i <= n; i++ {
		if spf[i] == i { // i is prime
			for j := i * i; j <= n; j += i {
				if spf[j] == j {
					spf[j] = i
				}
			}
		}
	}
	return spf
}

func factorize(n int, spf []int) []int {
	factors := []int{}
	for n > 1 {
		factors = append(factors, spf[n])
		n /= spf[n]
	}
	return factors
}

func main() {
	spf := spfSieve(100)

	for _, n := range []int{12, 30, 45, 60, 97} {
		fmt.Printf("  %d = %v\n", n, factorize(n, spf))
	}
	fmt.Println()
	fmt.Println("SPF sieve enables O(log n) factorization!")
}
```

---

## Example 5: Euler's Totient Sieve

```go
package main

import "fmt"

// φ(n) = count of numbers 1..n coprime to n
func totientSieve(n int) []int {
	phi := make([]int, n+1)
	for i := range phi { phi[i] = i }

	for i := 2; i <= n; i++ {
		if phi[i] == i { // i is prime
			for j := i; j <= n; j += i {
				phi[j] -= phi[j] / i
			}
		}
	}
	return phi
}

func main() {
	n := 20
	phi := totientSieve(n)
	for i := 1; i <= n; i++ {
		fmt.Printf("  φ(%2d) = %2d\n", i, phi[i])
	}
	fmt.Println()
	fmt.Println("Properties:")
	fmt.Println("  φ(p) = p-1 for prime p")
	fmt.Println("  φ(p^k) = p^k - p^(k-1)")
	fmt.Println("  φ is multiplicative: φ(ab) = φ(a)φ(b) if gcd(a,b)=1")
}
```

---

## Example 6: Prime Gap Analysis

```go
package main

import "fmt"

func sieve(n int) []int {
	isPrime := make([]bool, n+1)
	for i := 2; i <= n; i++ { isPrime[i] = true }
	for i := 2; i*i <= n; i++ {
		if isPrime[i] {
			for j := i * i; j <= n; j += i { isPrime[j] = false }
		}
	}
	primes := []int{}
	for i := 2; i <= n; i++ {
		if isPrime[i] { primes = append(primes, i) }
	}
	return primes
}

func main() {
	primes := sieve(1000)

	maxGap, maxI := 0, 0
	fmt.Println("Notable prime gaps up to 1000:")
	for i := 1; i < len(primes); i++ {
		gap := primes[i] - primes[i-1]
		if gap > maxGap {
			maxGap = gap
			maxI = i
			fmt.Printf("  New max gap: %d (between %d and %d)\n",
				gap, primes[i-1], primes[i])
		}
	}
	fmt.Printf("\nLargest gap: %d (between %d and %d)\n",
		maxGap, primes[maxI-1], primes[maxI])
}
```

---

## Example 7: Goldbach's Conjecture Verifier

```go
package main

import "fmt"

func sieve(n int) []bool {
	isPrime := make([]bool, n+1)
	for i := 2; i <= n; i++ { isPrime[i] = true }
	for i := 2; i*i <= n; i++ {
		if isPrime[i] {
			for j := i * i; j <= n; j += i { isPrime[j] = false }
		}
	}
	return isPrime
}

// Every even number > 2 = sum of two primes (unproven but verified)
func goldbachPair(n int, isPrime []bool) (int, int) {
	for i := 2; i <= n/2; i++ {
		if isPrime[i] && isPrime[n-i] {
			return i, n - i
		}
	}
	return -1, -1
}

func main() {
	isPrime := sieve(1000)

	fmt.Println("Goldbach pairs for even numbers:")
	for n := 4; n <= 50; n += 2 {
		a, b := goldbachPair(n, isPrime)
		fmt.Printf("  %2d = %d + %d\n", n, a, b)
	}
}
```

---

## Example 8: Sum of Divisors Sieve

```go
package main

import "fmt"

func sumOfDivisors(n int) []int {
	sigma := make([]int, n+1)
	for i := 1; i <= n; i++ {
		for j := i; j <= n; j += i {
			sigma[j] += i
		}
	}
	return sigma
}

func main() {
	n := 30
	sigma := sumOfDivisors(n)

	// Perfect numbers: σ(n) = 2n
	fmt.Println("Perfect numbers (σ(n) = 2n):")
	for i := 2; i <= n; i++ {
		if sigma[i] == 2*i {
			fmt.Printf("  %d (divisors sum to %d)\n", i, sigma[i]-i)
		}
	}

	// Abundant numbers: σ(n) > 2n
	fmt.Println("\nAbundant numbers (σ(n) > 2n):")
	for i := 2; i <= n; i++ {
		if sigma[i] > 2*i {
			fmt.Printf("  %d ", i)
		}
	}
	fmt.Println()
}
```

---

## Example 9: Sieve for Mobius Function

```go
package main

import "fmt"

// μ(n): 1 if n is squarefree with even prime factors,
//       -1 if squarefree with odd prime factors,
//        0 if has squared prime factor
func mobiusSieve(n int) []int {
	mu := make([]int, n+1)
	for i := range mu { mu[i] = 1 }
	isPrime := make([]bool, n+1)
	for i := 2; i <= n; i++ { isPrime[i] = true }

	for i := 2; i <= n; i++ {
		if isPrime[i] {
			for j := i; j <= n; j += i {
				if j != i { isPrime[j] = false }
				mu[j] *= -1
			}
			sq := i * i
			for j := sq; j <= n; j += sq {
				mu[j] = 0
			}
		}
	}
	return mu
}

func main() {
	n := 20
	mu := mobiusSieve(n)
	for i := 1; i <= n; i++ {
		symbol := "0"
		if mu[i] == 1 { symbol = "+1" }
		if mu[i] == -1 { symbol = "-1" }
		fmt.Printf("  μ(%2d) = %s\n", i, symbol)
	}
}
```

---

## Example 10: Sieve Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Sieve Patterns ===")
	fmt.Println()

	sieves := []struct{ name, complexity, use string }{
		{"Basic sieve", "O(n log log n), O(n)", "Find all primes ≤ n"},
		{"Segmented sieve", "O(n log log n), O(√n)", "Primes in [lo, hi]"},
		{"SPF sieve", "O(n log log n)", "O(log n) factorization"},
		{"Euler totient sieve", "O(n log log n)", "φ(n) for all n"},
		{"Sum of divisors", "O(n log n)", "σ(n), perfect numbers"},
		{"Möbius sieve", "O(n log log n)", "Inclusion-exclusion"},
		{"Linear sieve", "O(n)", "Each composite marked once"},
		{"Bitwise sieve", "O(n log log n), O(n/8)", "Space optimized"},
	}

	for _, s := range sieves {
		fmt.Printf("  %-22s %-25s → %s\n", s.name, s.complexity, s.use)
	}

	fmt.Println()
	fmt.Println("Optimization tips:")
	fmt.Println("  • Start marking from i² (smaller multiples already marked)")
	fmt.Println("  • Only check odd numbers (skip evens after 2)")
	fmt.Println("  • Use bitset instead of bool array for 8x space savings")
}
```

---

## Key Takeaways

1. Basic sieve: O(n log log n) — start marking from i², only for primes
2. SPF sieve: precompute smallest prime factor for O(log n) factorization
3. Segmented sieve: handle large ranges with O(√n) memory
4. Euler's totient sieve: φ(n) for all n simultaneously
5. Sieve pattern generalizes: divisor count, divisor sum, Möbius function

> **Next up:** Modular Arithmetic →
