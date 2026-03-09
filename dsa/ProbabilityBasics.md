# Phase 21: Math & Geometry — Probability Basics

## Overview

Probability concepts in algorithms: expected value, random algorithms, and probabilistic analysis. Key for competitive programming, interviews, and algorithm analysis.

| Concept | Formula |
|---------|---------|
| P(A) | favorable / total |
| P(A ∪ B) | P(A) + P(B) - P(A ∩ B) |
| P(A \| B) | P(A ∩ B) / P(B) |
| E[X] | Σ x·P(X=x) |
| Linearity of E | E[X+Y] = E[X] + E[Y] (always) |
| Geometric dist | E[X] = 1/p (expected trials until success) |

---

## Example 1: Basic Probability Simulation

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	trials := 1000000

	// Dice rolling probabilities
	freq := make([]int, 7) // 1-6
	for i := 0; i < trials; i++ {
		die := rand.Intn(6) + 1
		freq[die]++
	}

	fmt.Println("Single die roll frequencies:")
	for i := 1; i <= 6; i++ {
		fmt.Printf("  %d: %.4f (expected: 0.1667)\n", i, float64(freq[i])/float64(trials))
	}

	// Two dice sum
	sumFreq := make([]int, 13)
	for i := 0; i < trials; i++ {
		sum := rand.Intn(6) + 1 + rand.Intn(6) + 1
		sumFreq[sum]++
	}

	fmt.Println("\nTwo dice sum:")
	for s := 2; s <= 12; s++ {
		fmt.Printf("  Sum %2d: %.4f\n", s, float64(sumFreq[s])/float64(trials))
	}
}
```

---

## Example 2: Expected Value Simulation

```go
package main

import (
	"fmt"
	"math/rand"
)

// Expected value of rolling a die until getting a 6
func expectedRollsUntilSix(trials int) float64 {
	totalRolls := 0
	for t := 0; t < trials; t++ {
		rolls := 0
		for {
			rolls++
			if rand.Intn(6)+1 == 6 { break }
		}
		totalRolls += rolls
	}
	return float64(totalRolls) / float64(trials)
}

// Coupon collector: expected draws to get all n types
func couponCollector(n, trials int) float64 {
	totalDraws := 0
	for t := 0; t < trials; t++ {
		seen := make(map[int]bool)
		draws := 0
		for len(seen) < n {
			draws++
			seen[rand.Intn(n)] = true
		}
		totalDraws += draws
	}
	return float64(totalDraws) / float64(trials)
}

func main() {
	trials := 100000

	// Geometric distribution E[X] = 1/p = 6
	sim := expectedRollsUntilSix(trials)
	fmt.Printf("Expected rolls until 6: %.2f (theoretical: 6.00)\n\n", sim)

	// Coupon collector: E = n * H(n)
	for _, n := range []int{5, 10, 20, 50} {
		sim := couponCollector(n, trials)
		// Theoretical: n * (1 + 1/2 + 1/3 + ... + 1/n)
		hn := 0.0
		for i := 1; i <= n; i++ { hn += 1.0 / float64(i) }
		theory := float64(n) * hn
		fmt.Printf("  Coupon collector n=%2d: sim=%.1f, theory=%.1f\n", n, sim, theory)
	}
}
```

---

## Example 3: Linearity of Expectation

```go
package main

import (
	"fmt"
	"math/rand"
)

// Problem: Expected number of fixed points in random permutation
// By linearity: E[X] = Σ P(element i is fixed) = n * (1/n) = 1
func fixedPointsExpected(n, trials int) float64 {
	total := 0
	for t := 0; t < trials; t++ {
		perm := rand.Perm(n)
		count := 0
		for i, v := range perm {
			if i == v { count++ }
		}
		total += count
	}
	return float64(total) / float64(trials)
}

// Problem: Expected inversions in random permutation = n(n-1)/4
func expectedInversions(n, trials int) float64 {
	total := 0
	for t := 0; t < trials; t++ {
		perm := rand.Perm(n)
		inv := 0
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				if perm[i] > perm[j] { inv++ }
			}
		}
		total += inv
	}
	return float64(total) / float64(trials)
}

func main() {
	trials := 100000

	fmt.Println("Linearity of Expectation:")
	fmt.Println("\nFixed points in random permutation:")
	for _, n := range []int{5, 10, 50, 100} {
		e := fixedPointsExpected(n, trials)
		fmt.Printf("  n=%3d: E[fixed] = %.3f (theory: 1.000)\n", n, e)
	}

	fmt.Println("\nExpected inversions:")
	for _, n := range []int{5, 10, 20} {
		e := expectedInversions(n, trials/10)
		theory := float64(n) * float64(n-1) / 4
		fmt.Printf("  n=%2d: E[inv] = %.1f (theory: %.1f)\n", n, e, theory)
	}
}
```

---

## Example 4: Monte Carlo — Estimate Pi

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
)

func estimatePi(samples int) float64 {
	inside := 0
	for i := 0; i < samples; i++ {
		x := rand.Float64()
		y := rand.Float64()
		if x*x+y*y <= 1.0 {
			inside++
		}
	}
	return 4.0 * float64(inside) / float64(samples)
}

func main() {
	fmt.Println("Monte Carlo Pi Estimation:")
	fmt.Printf("  True π = %.10f\n\n", math.Pi)

	for _, n := range []int{1000, 10000, 100000, 1000000, 10000000} {
		est := estimatePi(n)
		err := math.Abs(est-math.Pi) / math.Pi * 100
		fmt.Printf("  n=%8d: π ≈ %.6f  (error: %.4f%%)\n", n, est, err)
	}
}
```

---

## Example 5: Birthday Paradox

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
)

// Probability that at least 2 people share birthday in group of k
func birthdaySimulation(k, trials int) float64 {
	collisions := 0
	for t := 0; t < trials; t++ {
		seen := make(map[int]bool)
		collision := false
		for i := 0; i < k; i++ {
			day := rand.Intn(365)
			if seen[day] { collision = true; break }
			seen[day] = true
		}
		if collision { collisions++ }
	}
	return float64(collisions) / float64(trials)
}

func birthdayTheory(k int) float64 {
	prob := 1.0
	for i := 1; i < k; i++ {
		prob *= (365.0 - float64(i)) / 365.0
	}
	return 1 - prob
}

func main() {
	fmt.Println("Birthday Paradox:")
	fmt.Println("  People  Sim P(collision)  Theory")
	for _, k := range []int{10, 20, 23, 30, 40, 50, 70} {
		sim := birthdaySimulation(k, 100000)
		theory := birthdayTheory(k)
		fmt.Printf("    %3d      %.4f           %.4f\n", k, sim, theory)
	}

	// Expected people for 50% collision ≈ sqrt(π * 365 / 2) ≈ 23.9
	fmt.Printf("\n  50%% threshold: ~23 people (√(π·365/2) = %.1f)\n",
		math.Sqrt(math.Pi*365/2))
}
```

---

## Example 6: Random Walk

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
)

// 1D random walk: expected distance from origin after n steps
func randomWalk1D(steps, trials int) (float64, float64) {
	totalDist := 0.0
	totalDistSq := 0.0
	for t := 0; t < trials; t++ {
		pos := 0
		for s := 0; s < steps; s++ {
			if rand.Intn(2) == 0 { pos++ } else { pos-- }
		}
		d := math.Abs(float64(pos))
		totalDist += d
		totalDistSq += float64(pos * pos)
	}
	return totalDist / float64(trials), totalDistSq / float64(trials)
}

func main() {
	trials := 100000
	fmt.Println("1D Random Walk:")
	fmt.Println("  Steps  E[|X|]   E[X²]  √(n)")

	for _, n := range []int{10, 100, 1000, 10000} {
		avgDist, avgDistSq := randomWalk1D(n, trials)
		fmt.Printf("  %5d  %6.2f   %8.1f  %6.1f\n",
			n, avgDist, avgDistSq, math.Sqrt(float64(n)))
	}

	fmt.Println("\n  E[X²] = n (variance)")
	fmt.Println("  E[|X|] ≈ √(2n/π)")
}
```

---

## Example 7: Randomized Quick Select (Expected O(n))

```go
package main

import (
	"fmt"
	"math/rand"
)

func quickSelect(arr []int, k int) int {
	if len(arr) == 1 { return arr[0] }

	pivot := arr[rand.Intn(len(arr))]
	less, equal, greater := []int{}, []int{}, []int{}

	for _, v := range arr {
		switch {
		case v < pivot:
			less = append(less, v)
		case v == pivot:
			equal = append(equal, v)
		default:
			greater = append(greater, v)
		}
	}

	if k < len(less) {
		return quickSelect(less, k)
	} else if k < len(less)+len(equal) {
		return pivot
	}
	return quickSelect(greater, k-len(less)-len(equal))
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5}

	fmt.Printf("Array: %v\n\n", arr)
	for k := 0; k < len(arr); k++ {
		temp := make([]int, len(arr))
		copy(temp, arr)
		fmt.Printf("  k=%2d: %d-th smallest = %d\n", k, k+1, quickSelect(temp, k))
	}
}
```

---

## Example 8: Probabilistic Counting (HyperLogLog Idea)

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
	"math/bits"
	"math/rand"
)

// Simplified Flajolet-Martin: estimate distinct elements
// Idea: track max leading zeros in hash → 2^max ≈ distinct count
func estimateDistinct(stream []string) float64 {
	maxLeadingZeros := 0

	for _, s := range stream {
		h := fnv.New32a()
		h.Write([]byte(s))
		hash := h.Sum32()
		lz := bits.LeadingZeros32(hash)
		if lz > maxLeadingZeros { maxLeadingZeros = lz }
	}

	return math.Pow(2, float64(maxLeadingZeros)) * 0.77351 // correction factor
}

func main() {
	// Generate stream with known distinct count
	distinct := 1000
	stream := make([]string, 10000)
	for i := range stream {
		stream[i] = fmt.Sprintf("item_%d", rand.Intn(distinct))
	}

	est := estimateDistinct(stream)
	fmt.Printf("Stream size: %d\n", len(stream))
	fmt.Printf("True distinct: %d\n", distinct)
	fmt.Printf("Estimated:     %.0f\n", est)
	fmt.Printf("Note: Real HyperLogLog uses multiple registers for accuracy\n")
}
```

---

## Example 9: Randomized Algorithms Analysis

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"time"
)

// Compare deterministic vs randomized pivot for QuickSort
func quickSortRandom(arr []int) {
	if len(arr) <= 1 { return }
	pi := rand.Intn(len(arr))
	arr[pi], arr[len(arr)-1] = arr[len(arr)-1], arr[pi]
	pivot := arr[len(arr)-1]
	i := 0
	for j := 0; j < len(arr)-1; j++ {
		if arr[j] <= pivot {
			arr[i], arr[j] = arr[j], arr[i]
			i++
		}
	}
	arr[i], arr[len(arr)-1] = arr[len(arr)-1], arr[i]
	quickSortRandom(arr[:i])
	quickSortRandom(arr[i+1:])
}

func main() {
	n := 100000
	fmt.Printf("Sorting %d elements:\n\n", n)

	// Sorted input (worst case for naive quicksort)
	sorted := make([]int, n)
	for i := range sorted { sorted[i] = i }

	// Random pivot
	arr := make([]int, n)
	copy(arr, sorted)
	start := time.Now()
	quickSortRandom(arr)
	randTime := time.Since(start)

	// Verify
	isSorted := sort.IntsAreSorted(arr)
	fmt.Printf("  Randomized pivot on sorted input: %v (correct: %v)\n", randTime, isSorted)

	// Random input
	random := make([]int, n)
	for i := range random { random[i] = rand.Intn(n) }
	copy(arr, random)
	start = time.Now()
	quickSortRandom(arr)
	randTime = time.Since(start)
	fmt.Printf("  Randomized pivot on random input:  %v\n", randTime)

	fmt.Println("\n  Randomized quicksort: O(n log n) expected for ALL inputs")
}
```

---

## Example 10: Probability Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Probability Patterns in Algorithms ===")
	fmt.Println()

	patterns := []struct{ concept, formula, application string }{
		{"Expected value", "E[X] = Σ x·P(x)", "Algorithm analysis"},
		{"Linearity of E", "E[X+Y] = E[X]+E[Y]", "Always true, even dependent"},
		{"Geometric dist", "E = 1/p", "Expected trials until success"},
		{"Coupon collector", "E = n·H(n) ≈ n·ln(n)", "Collect all types"},
		{"Birthday paradox", "~√(πn/2) for 50%", "Hash collision analysis"},
		{"Random walk", "E[X²] = n", "Diffusion, stock prices"},
		{"Monte Carlo", "Sample → estimate", "Integration, optimization"},
		{"Randomized pivot", "O(n log n) expected", "QuickSort, QuickSelect"},
		{"HyperLogLog", "Leading zeros → estimate", "Cardinality estimation"},
		{"Reservoir sampling", "k/n probability each", "Stream sampling"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-20s %-28s %s\n", p.concept, p.formula, p.application)
	}
}
```

---

## Key Takeaways

1. **Linearity of expectation** works even for dependent variables — extremely powerful
2. **Coupon collector**: n·H(n) expected draws to see all n types
3. **Birthday paradox**: collisions happen much sooner than intuition suggests — √(n)
4. **Monte Carlo**: sample randomly to estimate quantities (π, integrals)
5. **Randomized algorithms** (QuickSort, QuickSelect) avoid worst cases with high probability

> **Phase 21 Complete! Next up:** Phase 22 — Segment Tree & Fenwick Tree →
