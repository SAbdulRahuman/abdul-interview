# Phase 21: Math & Geometry — Reservoir Sampling

## Overview

**Reservoir sampling** selects k items uniformly at random from a stream of unknown size n. Items arrive one-by-one and you can't store them all.

| Algorithm | Description | Time | Space |
|-----------|-------------|------|-------|
| Algorithm R | Basic reservoir sampling | O(n) | O(k) |
| Algorithm L | Optimized (skip-based) | O(k(1+log(n/k))) | O(k) |
| Weighted | Probability proportional to weight | O(n) | O(k) |

**Key guarantee**: Each element has exactly k/n probability of being in the final sample.

---

## Example 1: Algorithm R — Basic Reservoir Sampling

```go
package main

import (
	"fmt"
	"math/rand"
)

// Select k items uniformly from stream
func reservoirSample(stream []int, k int) []int {
	reservoir := make([]int, k)
	copy(reservoir, stream[:k])

	for i := k; i < len(stream); i++ {
		j := rand.Intn(i + 1) // random index [0, i]
		if j < k {
			reservoir[j] = stream[i]
		}
	}
	return reservoir
}

func main() {
	stream := make([]int, 100)
	for i := range stream { stream[i] = i + 1 }

	fmt.Println("Stream: [1, 2, ..., 100]")
	fmt.Println("Samples of size 5:")
	for trial := 0; trial < 5; trial++ {
		sample := reservoirSample(stream, 5)
		fmt.Printf("  Trial %d: %v\n", trial+1, sample)
	}
}
```

---

## Example 2: Verify Uniform Distribution

```go
package main

import (
	"fmt"
	"math/rand"
)

func reservoirSample(stream []int, k int) []int {
	reservoir := make([]int, k)
	copy(reservoir, stream[:k])
	for i := k; i < len(stream); i++ {
		j := rand.Intn(i + 1)
		if j < k { reservoir[j] = stream[i] }
	}
	return reservoir
}

func main() {
	n := 10
	k := 1
	trials := 100000

	stream := make([]int, n)
	for i := range stream { stream[i] = i }

	freq := make([]int, n)
	for t := 0; t < trials; t++ {
		sample := reservoirSample(stream, k)
		for _, v := range sample { freq[v]++ }
	}

	expected := float64(trials * k) / float64(n)
	fmt.Printf("Stream size=%d, k=%d, trials=%d\n", n, k, trials)
	fmt.Printf("Expected frequency: %.0f\n\n", expected)
	for i, f := range freq {
		bar := ""
		for j := 0; j < f/500; j++ { bar += "█" }
		fmt.Printf("  Element %d: %5d  %s\n", i, f, bar)
	}
}
```

---

## Example 3: Linked List Random Node (LeetCode 382)

```go
package main

import (
	"fmt"
	"math/rand"
)

type ListNode struct {
	Val  int
	Next *ListNode
}

type Solution struct {
	head *ListNode
}

func Constructor(head *ListNode) Solution {
	return Solution{head: head}
}

func (s *Solution) GetRandom() int {
	result := s.head.Val
	node := s.head.Next
	i := 2

	for node != nil {
		if rand.Intn(i) == 0 {
			result = node.Val
		}
		node = node.Next
		i++
	}
	return result
}

func main() {
	// Build list: 1 -> 2 -> 3 -> 4 -> 5
	head := &ListNode{Val: 1}
	curr := head
	for i := 2; i <= 5; i++ {
		curr.Next = &ListNode{Val: i}
		curr = curr.Next
	}

	sol := Constructor(head)

	freq := make(map[int]int)
	for i := 0; i < 10000; i++ {
		freq[sol.GetRandom()]++
	}

	fmt.Println("Linked List Random Node (10000 trials):")
	for v := 1; v <= 5; v++ {
		fmt.Printf("  Value %d: %d times (%.1f%%)\n", v, freq[v], float64(freq[v])/100)
	}
}
```

---

## Example 4: Random Pick Index (LeetCode 398)

```go
package main

import (
	"fmt"
	"math/rand"
)

type PickSolution struct {
	nums []int
}

func NewPickSolution(nums []int) PickSolution {
	return PickSolution{nums: nums}
}

// Pick random index with value == target (reservoir sampling k=1)
func (s *PickSolution) Pick(target int) int {
	result := -1
	count := 0

	for i, v := range s.nums {
		if v == target {
			count++
			if rand.Intn(count) == 0 {
				result = i
			}
		}
	}
	return result
}

func main() {
	nums := []int{1, 2, 3, 3, 3}
	sol := NewPickSolution(nums)

	fmt.Printf("nums = %v\n\n", nums)

	// Pick index of target=3 multiple times
	freq := make(map[int]int)
	for i := 0; i < 9000; i++ {
		freq[sol.Pick(3)]++
	}

	fmt.Println("Pick(3) distribution:")
	for idx, cnt := range freq {
		fmt.Printf("  Index %d: %d times (%.1f%%)\n", idx, cnt, float64(cnt)/90)
	}
}
```

---

## Example 5: Weighted Reservoir Sampling

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sort"
)

type Item struct {
	Value  string
	Weight float64
}

// Weighted sampling: Efraimidis–Spirakis algorithm
// Key for each item: u^(1/w) where u is uniform [0,1)
func weightedReservoir(items []Item, k int) []string {
	type entry struct {
		value string
		key   float64
	}

	entries := make([]entry, 0, len(items))
	for _, item := range items {
		u := rand.Float64()
		key := math.Pow(u, 1.0/item.Weight)
		entries = append(entries, entry{item.Value, key})
	}

	sort.Slice(entries, func(i, j int) bool {
		return entries[i].key > entries[j].key
	})

	result := make([]string, min(k, len(entries)))
	for i := range result { result[i] = entries[i].value }
	return result
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	items := []Item{
		{"Apple", 10},
		{"Banana", 5},
		{"Cherry", 2},
		{"Date", 1},
	}

	freq := make(map[string]int)
	for i := 0; i < 10000; i++ {
		sample := weightedReservoir(items, 1)
		freq[sample[0]]++
	}

	fmt.Println("Weighted sampling (10000 trials, k=1):")
	for _, item := range items {
		fmt.Printf("  %-8s (w=%2.0f): %d times\n", item.Value, item.Weight, freq[item.Value])
	}
}
```

---

## Example 6: Streaming Median with Reservoir

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
)

// Approximate median using reservoir of size k
func streamMedian(stream []int, k int) float64 {
	reservoir := make([]int, k)
	copy(reservoir, stream[:k])

	for i := k; i < len(stream); i++ {
		j := rand.Intn(i + 1)
		if j < k { reservoir[j] = stream[i] }
	}

	sort.Ints(reservoir)
	if k%2 == 0 {
		return float64(reservoir[k/2-1]+reservoir[k/2]) / 2
	}
	return float64(reservoir[k/2])
}

func main() {
	n := 10000
	stream := make([]int, n)
	for i := range stream { stream[i] = i + 1 }

	// Shuffle to simulate random stream
	rand.Shuffle(n, func(i, j int) { stream[i], stream[j] = stream[j], stream[i] })

	trueMedian := float64(n+1) / 2
	fmt.Printf("True median: %.1f\n\n", trueMedian)

	for _, k := range []int{10, 100, 1000} {
		approx := streamMedian(stream, k)
		fmt.Printf("  Reservoir k=%4d: median ≈ %.1f  (error=%.1f%%)\n",
			k, approx, 100*abs(approx-trueMedian)/trueMedian)
	}
}

func abs(x float64) float64 { if x < 0 { return -x }; return x }
```

---

## Example 7: Sample K from Stream (Online Algorithm)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Online reservoir sampling — processes items one at a time
type ReservoirSampler struct {
	reservoir []int
	k         int
	count     int
}

func NewSampler(k int) *ReservoirSampler {
	return &ReservoirSampler{
		reservoir: make([]int, 0, k),
		k:         k,
	}
}

func (s *ReservoirSampler) Add(item int) {
	s.count++
	if len(s.reservoir) < s.k {
		s.reservoir = append(s.reservoir, item)
	} else {
		j := rand.Intn(s.count)
		if j < s.k {
			s.reservoir[j] = item
		}
	}
}

func (s *ReservoirSampler) Sample() []int {
	result := make([]int, len(s.reservoir))
	copy(result, s.reservoir)
	return result
}

func main() {
	sampler := NewSampler(3)

	fmt.Println("Processing stream...")
	for i := 1; i <= 20; i++ {
		sampler.Add(i * 10)
		if i%5 == 0 {
			fmt.Printf("  After %2d items: sample = %v\n", i, sampler.Sample())
		}
	}
}
```

---

## Example 8: Random Point in Non-Overlapping Rectangles (LeetCode 497)

```go
package main

import (
	"fmt"
	"math/rand"
)

type RectSolution struct {
	rects     [][]int
	prefixSum []int
	total     int
}

func NewRectSolution(rects [][]int) RectSolution {
	s := RectSolution{rects: rects}
	s.prefixSum = make([]int, len(rects))
	for i, r := range rects {
		area := (r[2] - r[0] + 1) * (r[3] - r[1] + 1)
		s.total += area
		s.prefixSum[i] = s.total
	}
	return s
}

func (s *RectSolution) Pick() [2]int {
	target := rand.Intn(s.total)

	// Binary search for rectangle
	lo, hi := 0, len(s.prefixSum)-1
	for lo < hi {
		mid := (lo + hi) / 2
		if s.prefixSum[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}

	r := s.rects[lo]
	x := r[0] + rand.Intn(r[2]-r[0]+1)
	y := r[1] + rand.Intn(r[3]-r[1]+1)
	return [2]int{x, y}
}

func main() {
	rects := [][]int{{1, 1, 3, 3}, {5, 5, 6, 6}}
	sol := NewRectSolution(rects)

	freq := make(map[[2]int]int)
	for i := 0; i < 10000; i++ {
		p := sol.Pick()
		freq[p]++
	}

	fmt.Println("Random points from rectangles:")
	fmt.Printf("  Rect1 (1,1)-(3,3): area=%d\n", 9)
	fmt.Printf("  Rect2 (5,5)-(6,6): area=%d\n\n", 4)

	r1, r2 := 0, 0
	for p, cnt := range freq {
		if p[0] <= 3 { r1 += cnt } else { r2 += cnt }
	}
	fmt.Printf("  Rect1 hits: %d (%.1f%%, expected 69.2%%)\n", r1, float64(r1)/100)
	fmt.Printf("  Rect2 hits: %d (%.1f%%, expected 30.8%%)\n", r2, float64(r2)/100)
}
```

---

## Example 9: Random Sampling Without Replacement

```go
package main

import (
	"fmt"
	"math/rand"
)

// Selection sampling: pick k unique items from [0, n)
func sampleWithoutReplacement(n, k int) []int {
	selected := make(map[int]bool)
	result := make([]int, 0, k)

	if k > n/2 {
		// Use complement method
		exclude := make(map[int]bool)
		for len(exclude) < n-k {
			exclude[rand.Intn(n)] = true
		}
		for i := 0; i < n; i++ {
			if !exclude[i] { result = append(result, i) }
		}
		return result
	}

	for len(selected) < k {
		v := rand.Intn(n)
		if !selected[v] {
			selected[v] = true
			result = append(result, v)
		}
	}
	return result
}

// Using Fisher-Yates partial shuffle
func sampleFY(n, k int) []int {
	arr := make([]int, n)
	for i := range arr { arr[i] = i }
	for i := 0; i < k; i++ {
		j := i + rand.Intn(n-i)
		arr[i], arr[j] = arr[j], arr[i]
	}
	return arr[:k]
}

func main() {
	fmt.Println("Sample 5 from [0, 20):")
	fmt.Printf("  Hash method:  %v\n", sampleWithoutReplacement(20, 5))
	fmt.Printf("  Fisher-Yates: %v\n", sampleFY(20, 5))

	fmt.Println("\nSample 8 from [0, 10):")
	fmt.Printf("  Complement:   %v\n", sampleWithoutReplacement(10, 8))
}
```

---

## Example 10: Reservoir Sampling Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Reservoir Sampling Patterns ===")
	fmt.Println()

	patterns := []struct{ technique, use, complexity string }{
		{"Algorithm R", "k items from unknown-size stream", "O(n) time, O(k) space"},
		{"k=1 reservoir", "Random element from stream", "O(n) time, O(1) space"},
		{"Linked list random", "LeetCode 382", "O(n) per call"},
		{"Random pick index", "LeetCode 398 — reservoir by target", "O(n) per call"},
		{"Weighted reservoir", "Efraimidis–Spirakis algorithm", "O(n) time"},
		{"Online sampler", "Process one item at a time", "O(1) per item"},
		{"Random in rectangles", "LeetCode 497 — weighted areas", "O(log n) per pick"},
		{"Without replacement", "k unique from n (hash/FY)", "O(k) time"},
		{"Streaming approx", "Approximate statistics on stream", "O(k) space"},
		{"Verification", "Frequency analysis over many trials", "Probability = k/n"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-24s %-38s %s\n", p.technique, p.use, p.complexity)
	}
}
```

---

## Key Takeaways

1. **Reservoir sampling** guarantees each element has exactly k/n probability
2. **Algorithm R**: keep first k, then each i-th element replaces random position with prob k/i
3. **Online streaming**: process one item at a time without knowing total count
4. **Weighted version** uses key = u^(1/w) and keeps top-k keys
5. **Verification**: run many trials and check frequency matches expected probability

> **Next up:** Fisher-Yates Shuffle →
