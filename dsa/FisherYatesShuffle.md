# Phase 21: Math & Geometry — Fisher-Yates Shuffle

## Overview

The **Fisher-Yates shuffle** (Knuth shuffle) generates a uniformly random permutation of an array in-place in O(n) time. Each of the n! permutations is equally likely.

| Algorithm | Time | Space | Uniform? |
|-----------|------|-------|----------|
| Fisher-Yates (modern) | O(n) | O(1) | Yes |
| Naive (sort by random) | O(n log n) | O(n) | Yes (if unique keys) |
| Sattolo's | O(n) | O(1) | Cyclic permutations only |

**Key**: Swap element at index i with a random element from [i, n-1].

---

## Example 1: Classic Fisher-Yates Shuffle

```go
package main

import (
	"fmt"
	"math/rand"
)

func shuffle(arr []int) {
	n := len(arr)
	for i := n - 1; i > 0; i-- {
		j := rand.Intn(i + 1) // random index [0, i]
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	arr := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	fmt.Println("Fisher-Yates Shuffle:")
	for trial := 0; trial < 5; trial++ {
		temp := make([]int, len(arr))
		copy(temp, arr)
		shuffle(temp)
		fmt.Printf("  Trial %d: %v\n", trial+1, temp)
	}
}
```

---

## Example 2: Shuffle an Array (LeetCode 384)

```go
package main

import (
	"fmt"
	"math/rand"
)

type Solution struct {
	original []int
	current  []int
}

func Constructor(nums []int) Solution {
	curr := make([]int, len(nums))
	copy(curr, nums)
	return Solution{original: nums, current: curr}
}

func (s *Solution) Reset() []int {
	copy(s.current, s.original)
	return s.current
}

func (s *Solution) Shuffle() []int {
	n := len(s.current)
	for i := n - 1; i > 0; i-- {
		j := rand.Intn(i + 1)
		s.current[i], s.current[j] = s.current[j], s.current[i]
	}
	return s.current
}

func main() {
	sol := Constructor([]int{1, 2, 3, 4, 5})

	fmt.Println("Shuffle:", sol.Shuffle())
	fmt.Println("Shuffle:", sol.Shuffle())
	fmt.Println("Reset:  ", sol.Reset())
	fmt.Println("Shuffle:", sol.Shuffle())
}
```

---

## Example 3: Verify Uniformity

```go
package main

import (
	"fmt"
	"math/rand"
)

func shuffle(arr []int) {
	for i := len(arr) - 1; i > 0; i-- {
		j := rand.Intn(i + 1)
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	n := 4
	trials := 240000 // 4! = 24 permutations
	freq := make(map[string]int)

	for t := 0; t < trials; t++ {
		arr := make([]int, n)
		for i := range arr { arr[i] = i + 1 }
		shuffle(arr)
		freq[fmt.Sprint(arr)]++
	}

	expected := trials / 24 // 4! = 24
	fmt.Printf("n=%d, trials=%d, expected per perm=%d\n\n", n, trials, expected)

	minF, maxF := trials, 0
	for _, cnt := range freq {
		if cnt < minF { minF = cnt }
		if cnt > maxF { maxF = cnt }
	}
	fmt.Printf("Unique permutations: %d (expected 24)\n", len(freq))
	fmt.Printf("Min frequency: %d, Max frequency: %d\n", minF, maxF)
	fmt.Printf("Deviation: %.1f%%\n", 100*float64(maxF-minF)/float64(expected))
}
```

---

## Example 4: Forward (Knuth) vs Backward Fisher-Yates

```go
package main

import (
	"fmt"
	"math/rand"
)

// Modern Fisher-Yates (iterate backward)
func shuffleBackward(arr []int) {
	for i := len(arr) - 1; i > 0; i-- {
		j := rand.Intn(i + 1) // [0, i]
		arr[i], arr[j] = arr[j], arr[i]
	}
}

// Knuth variant (iterate forward)
func shuffleForward(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		j := i + rand.Intn(n-i) // [i, n-1]
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	fmt.Println("Both produce uniform shuffles:")

	for _, name := range []string{"Backward", "Forward"} {
		arr := []int{1, 2, 3, 4, 5}
		if name == "Backward" {
			shuffleBackward(arr)
		} else {
			shuffleForward(arr)
		}
		fmt.Printf("  %-10s: %v\n", name, arr)
	}

	// Verify forward version uniformity
	freq := [5][5]int{} // freq[pos][val]
	trials := 100000
	for t := 0; t < trials; t++ {
		arr := []int{0, 1, 2, 3, 4}
		shuffleForward(arr)
		for pos, val := range arr {
			freq[pos][val]++
		}
	}

	fmt.Printf("\nPosition-value frequencies (expected %d each):\n", trials/5)
	for pos := 0; pos < 5; pos++ {
		fmt.Printf("  pos %d: %v\n", pos, freq[pos])
	}
}
```

---

## Example 5: Sattolo's Algorithm (Cyclic Permutations)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Sattolo's: produces only cyclic permutations (single cycle)
// Difference: j ranges [0, i-1] instead of [0, i]
func sattolo(arr []int) {
	for i := len(arr) - 1; i > 0; i-- {
		j := rand.Intn(i) // [0, i-1] — NOT [0, i]
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func isSingleCycle(arr []int) bool {
	n := len(arr)
	visited := make([]bool, n)
	pos := 0
	for i := 0; i < n; i++ {
		if visited[pos] { return false }
		visited[pos] = true
		pos = arr[pos]
	}
	return pos == 0
}

func main() {
	fmt.Println("Sattolo's Algorithm (cyclic permutations only):")

	allCyclic := true
	for trial := 0; trial < 10; trial++ {
		arr := []int{0, 1, 2, 3, 4}
		sattolo(arr)
		cyclic := isSingleCycle(arr)
		if !cyclic { allCyclic = false }
		fmt.Printf("  %v — single cycle: %v\n", arr, cyclic)
	}
	fmt.Printf("\nAll cyclic: %v (should be true)\n", allCyclic)
}
```

---

## Example 6: Partial Shuffle (Pick k Random Elements)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Shuffle only first k elements — O(k) instead of O(n)
func partialShuffle(arr []int, k int) []int {
	n := len(arr)
	if k > n { k = n }

	temp := make([]int, n)
	copy(temp, arr)

	for i := 0; i < k; i++ {
		j := i + rand.Intn(n-i)
		temp[i], temp[j] = temp[j], temp[i]
	}
	return temp[:k]
}

func main() {
	arr := []int{10, 20, 30, 40, 50, 60, 70, 80, 90, 100}

	fmt.Println("Partial shuffle (pick k from 10 elements):")
	for _, k := range []int{1, 3, 5, 7} {
		sample := partialShuffle(arr, k)
		fmt.Printf("  k=%d: %v\n", k, sample)
	}
}
```

---

## Example 7: Inside-Out Algorithm (Immutable Input)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Fisher-Yates inside-out: creates shuffled copy without modifying input
func shuffleInsideOut(source []int) []int {
	n := len(source)
	result := make([]int, n)

	for i := 0; i < n; i++ {
		j := rand.Intn(i + 1)
		if j != i {
			result[i] = result[j]
		}
		result[j] = source[i]
	}
	return result
}

func main() {
	source := []int{1, 2, 3, 4, 5, 6, 7, 8}

	fmt.Printf("Source (unchanged): %v\n\n", source)
	for trial := 0; trial < 5; trial++ {
		shuffled := shuffleInsideOut(source)
		fmt.Printf("  Shuffled: %v\n", shuffled)
	}
	fmt.Printf("\nSource after: %v\n", source)
}
```

---

## Example 8: Common Mistake — Biased Shuffle

```go
package main

import (
	"fmt"
	"math/rand"
)

// WRONG: biased shuffle — each element swaps with ANY position
func biasedShuffle(arr []int) {
	n := len(arr)
	for i := 0; i < n; i++ {
		j := rand.Intn(n) // BUG: should be rand.Intn(n-i) + i
		arr[i], arr[j] = arr[j], arr[i]
	}
}

// CORRECT: Fisher-Yates
func correctShuffle(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		j := i + rand.Intn(n-i)
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	n := 3
	trials := 600000

	// Test biased
	biased := make(map[string]int)
	for t := 0; t < trials; t++ {
		arr := []int{1, 2, 3}
		biasedShuffle(arr)
		biased[fmt.Sprint(arr)]++
	}

	// Test correct
	correct := make(map[string]int)
	for t := 0; t < trials; t++ {
		arr := []int{1, 2, 3}
		correctShuffle(arr)
		correct[fmt.Sprint(arr)]++
	}

	expected := trials / 6
	fmt.Printf("Expected per permutation: %d\n\n", expected)

	fmt.Println("BIASED shuffle:")
	for k, v := range biased {
		fmt.Printf("  %-10s: %6d (%.1f%%)\n", k, v, float64(v)*100/float64(trials))
	}

	fmt.Println("\nCORRECT shuffle:")
	for k, v := range correct {
		fmt.Printf("  %-10s: %6d (%.1f%%)\n", k, v, float64(v)*100/float64(trials))
	}
}
```

---

## Example 9: Shuffle String Characters

```go
package main

import (
	"fmt"
	"math/rand"
)

func shuffleString(s string) string {
	runes := []rune(s)
	for i := len(runes) - 1; i > 0; i-- {
		j := rand.Intn(i + 1)
		runes[i], runes[j] = runes[j], runes[i]
	}
	return string(runes)
}

// Generate random password
func randomPassword(length int) string {
	chars := []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%")
	password := make([]rune, length)
	for i := range password {
		password[i] = chars[rand.Intn(len(chars))]
	}
	// Shuffle for extra randomness
	for i := len(password) - 1; i > 0; i-- {
		j := rand.Intn(i + 1)
		password[i], password[j] = password[j], password[i]
	}
	return string(password)
}

func main() {
	word := "algorithm"
	fmt.Printf("Original: %s\n", word)
	for i := 0; i < 5; i++ {
		fmt.Printf("  Shuffle %d: %s\n", i+1, shuffleString(word))
	}

	fmt.Println("\nRandom passwords:")
	for i := 0; i < 5; i++ {
		fmt.Printf("  %s\n", randomPassword(12))
	}
}
```

---

## Example 10: Fisher-Yates Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Fisher-Yates Shuffle Patterns ===")
	fmt.Println()

	patterns := []struct{ variant, key, use string }{
		{"Modern (backward)", "j ∈ [0, i] for i=n-1..1", "Standard in-place shuffle"},
		{"Knuth (forward)", "j ∈ [i, n-1] for i=0..n-2", "Equivalent to modern"},
		{"Inside-out", "Build shuffled copy from source", "Immutable input"},
		{"Partial shuffle", "Only shuffle first k positions", "O(k) random selection"},
		{"Sattolo's", "j ∈ [0, i-1] (exclusive)", "Cyclic permutations only"},
		{"LeetCode 384", "Reset/Shuffle with saved original", "OOP design problem"},
		{"Biased (WRONG)", "j ∈ [0, n-1] for all i", "NOT uniform — common bug"},
		{"Crypto-safe", "Use crypto/rand instead of math/rand", "Security-sensitive apps"},
		{"String shuffle", "Convert to []rune, shuffle, convert back", "Anagram generation"},
		{"Sort by random", "Assign random keys, sort", "O(n log n) alternative"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-20s %-40s %s\n", p.variant, p.key, p.use)
	}

	fmt.Println("\n=== Critical Rules ===")
	fmt.Println("  1. j must range [i, n-1] (or [0, i]) — not [0, n-1]")
	fmt.Println("  2. n! possible outputs for n elements")
	fmt.Println("  3. O(n) time, O(1) extra space (in-place)")
	fmt.Println("  4. Each permutation has exactly 1/n! probability")
}
```

---

## Key Takeaways

1. **Fisher-Yates** is the only O(n) in-place uniform shuffle
2. **Key rule**: swap range must shrink each iteration — [i, n) or [0, i]
3. **Common bug**: swapping with any random index produces biased results
4. **Partial shuffle** for selecting k random items in O(k)
5. **Sattolo's** variant restricts to cyclic permutations only

> **Next up:** Probability Basics →
