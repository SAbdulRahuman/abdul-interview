# Phase 13: Union Find — Weighted Union Find

## Overview

**Weighted Union Find** extends the basic Union-Find by maintaining a **weight/potential** on each edge from child to parent. This allows answering **relative relationship queries** between elements.

| Use Case | What Weight Represents |
|----------|----------------------|
| Relative distances | dist[x] − dist[root] |
| Multiplicative ratios | ratio of x to root |
| Group assignments | Which group (e.g., friend/enemy) |

**Key Idea:** `weight[x]` = relationship from x to `parent[x]`. When compressing paths, combine weights along the path.

---

## Example 1: Basic Weighted Union-Find

```go
package main

import "fmt"

type WeightedUF struct {
	parent []int
	rank   []int
	weight []float64 // weight[x] = ratio from x to parent[x]
}

func NewWUF(n int) *WeightedUF {
	p := make([]int, n)
	r := make([]int, n)
	w := make([]float64, n)
	for i := range p { p[i] = i; w[i] = 1.0 }
	return &WeightedUF{parent: p, rank: r, weight: w}
}

func (uf *WeightedUF) Find(x int) (int, float64) {
	if uf.parent[x] == x { return x, 1.0 }
	root, w := uf.Find(uf.parent[x])
	uf.parent[x] = root
	uf.weight[x] *= w // Combine weights
	return root, uf.weight[x]
}

func (uf *WeightedUF) Union(x, y int, w float64) {
	// w means: x / y = w, i.e., value[x] = w * value[y]
	rootX, wx := uf.Find(x)
	rootY, wy := uf.Find(y)
	if rootX == rootY { return }

	// weight[rootX→rootY] should satisfy: x→rootX→rootY→y path
	// wx * weight[rootX] = w * wy, so weight[rootX→rootY] = w * wy / wx
	if uf.rank[rootX] < uf.rank[rootY] {
		uf.parent[rootX] = rootY
		uf.weight[rootX] = w * wy / wx
	} else if uf.rank[rootX] > uf.rank[rootY] {
		uf.parent[rootY] = rootX
		uf.weight[rootY] = wx / (w * wy)
	} else {
		uf.parent[rootY] = rootX
		uf.weight[rootY] = wx / (w * wy)
		uf.rank[rootX]++
	}
}

func (uf *WeightedUF) Query(x, y int) (float64, bool) {
	rootX, wx := uf.Find(x)
	rootY, wy := uf.Find(y)
	if rootX != rootY { return 0, false }
	return wx / wy, true
}

func main() {
	uf := NewWUF(4)
	uf.Union(0, 1, 2.0) // a/b = 2
	uf.Union(1, 2, 3.0) // b/c = 3

	if ratio, ok := uf.Query(0, 2); ok {
		fmt.Printf("a/c = %.1f\n", ratio) // 6.0
	}
	if _, ok := uf.Query(0, 3); !ok {
		fmt.Println("0 and 3 not connected")
	}
}
```

---

## Example 2: Evaluate Division (LeetCode 399)

```go
package main

import "fmt"

func calcEquation(equations [][]string, values []float64, queries [][]string) []float64 {
	id := map[string]int{}
	idx := 0
	for _, eq := range equations {
		if _, ok := id[eq[0]]; !ok { id[eq[0]] = idx; idx++ }
		if _, ok := id[eq[1]]; !ok { id[eq[1]] = idx; idx++ }
	}

	parent := make([]int, idx)
	weight := make([]float64, idx) // weight[x] = x / parent[x]
	for i := range parent { parent[i] = i; weight[i] = 1.0 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x {
			root := find(parent[x])
			weight[x] *= weight[parent[x]]
			parent[x] = root
		}
		return parent[x]
	}

	union := func(x, y int, w float64) {
		rx, ry := find(x), find(y)
		if rx == ry { return }
		parent[rx] = ry
		weight[rx] = w * weight[y] / weight[x]
	}

	for i, eq := range equations {
		union(id[eq[0]], id[eq[1]], values[i])
	}

	result := make([]float64, len(queries))
	for i, q := range queries {
		idA, okA := id[q[0]]
		idB, okB := id[q[1]]
		if !okA || !okB {
			result[i] = -1.0
			continue
		}
		if find(idA) != find(idB) {
			result[i] = -1.0
		} else {
			result[i] = weight[idA] / weight[idB]
		}
	}
	return result
}

func main() {
	equations := [][]string{{"a","b"},{"b","c"}}
	values := []float64{2.0, 3.0}
	queries := [][]string{{"a","c"},{"b","a"},{"a","e"},{"a","a"},{"x","x"}}
	results := calcEquation(equations, values, queries)
	for i, q := range queries {
		fmt.Printf("%s/%s = %.1f\n", q[0], q[1], results[i])
	}
	// a/c=6.0, b/a=0.5, a/e=-1.0, a/a=1.0, x/x=-1.0
}
```

---

## Example 3: Distance-Based Weighted UF

```go
package main

import "fmt"

type DistUF struct {
	parent []int
	dist   []int // dist[x] = distance from x to parent[x]
	size   []int
}

func NewDistUF(n int) *DistUF {
	p := make([]int, n)
	d := make([]int, n)
	s := make([]int, n)
	for i := range p { p[i] = i; s[i] = 1 }
	return &DistUF{parent: p, dist: d, size: s}
}

func (uf *DistUF) Find(x int) int {
	if uf.parent[x] == x { return x }
	root := uf.Find(uf.parent[x])
	uf.dist[x] += uf.dist[uf.parent[x]] // Accumulate distance
	uf.parent[x] = root
	return root
}

func (uf *DistUF) Union(x, y int, w int) {
	// w = dist[x] - dist[y]
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return }
	if uf.size[rx] < uf.size[ry] { rx, ry = ry, rx; x, y = y, x; w = -w }
	uf.parent[ry] = rx
	uf.dist[ry] = uf.dist[x] - uf.dist[y] + w
	uf.size[rx] += uf.size[ry]
}

func (uf *DistUF) Query(x, y int) (int, bool) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry { return 0, false }
	return uf.dist[x] - uf.dist[y], true
}

func main() {
	uf := NewDistUF(5)
	uf.Union(0, 1, 3)  // dist[0] - dist[1] = 3
	uf.Union(1, 2, 5)  // dist[1] - dist[2] = 5

	if d, ok := uf.Query(0, 2); ok {
		fmt.Printf("dist[0] - dist[2] = %d\n", d) // 8
	}
	if d, ok := uf.Query(2, 0); ok {
		fmt.Printf("dist[2] - dist[0] = %d\n", d) // -8
	}
}
```

---

## Example 4: Friend or Enemy (Modular Weighted UF)

```go
package main

import "fmt"

// Two elements in the same group: weight difference mod 2 = 0
// In different groups: weight difference mod 2 = 1

type GroupUF struct {
	parent []int
	rel    []int // 0 = same group as parent, 1 = different group
}

func NewGroupUF(n int) *GroupUF {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &GroupUF{parent: p, rel: r}
}

func (uf *GroupUF) Find(x int) int {
	if uf.parent[x] == x { return x }
	root := uf.Find(uf.parent[x])
	uf.rel[x] = (uf.rel[x] + uf.rel[uf.parent[x]]) % 2
	uf.parent[x] = root
	return root
}

func (uf *GroupUF) Union(x, y int, sameGroup bool) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	relXY := 0
	if !sameGroup { relXY = 1 }

	if rx == ry {
		// Check consistency
		return (uf.rel[x]+uf.rel[y])%2 == relXY%2
	}

	uf.parent[rx] = ry
	uf.rel[rx] = (uf.rel[x] + relXY + uf.rel[y]) % 2
	return true
}

func (uf *GroupUF) SameGroup(x, y int) (bool, bool) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry { return false, false } // unknown
	return (uf.rel[x]+uf.rel[y])%2 == 0, true
}

func main() {
	uf := NewGroupUF(5)
	uf.Union(0, 1, false) // 0 and 1 are enemies
	uf.Union(1, 2, false) // 1 and 2 are enemies
	uf.Union(2, 3, true)  // 2 and 3 are friends

	same, known := uf.SameGroup(0, 2)
	fmt.Printf("0 and 2 same group: %v (known: %v)\n", same, known) // true
	same, known = uf.SameGroup(0, 3)
	fmt.Printf("0 and 3 same group: %v (known: %v)\n", same, known) // true
	same, known = uf.SameGroup(0, 1)
	fmt.Printf("0 and 1 same group: %v (known: %v)\n", same, known) // false
}
```

---

## Example 5: Food Chain Problem (3 Groups, mod 3)

```go
package main

import "fmt"

// N animals, 3 types: A eats B, B eats C, C eats A
// rel[x] = 0: same type as root
//        = 1: eats root's type
//        = 2: eaten by root's type

type FoodChainUF struct {
	parent []int
	rel    []int // relation to parent (mod 3)
}

func NewFoodChainUF(n int) *FoodChainUF {
	p := make([]int, n+1)
	r := make([]int, n+1)
	for i := range p { p[i] = i }
	return &FoodChainUF{parent: p, rel: r}
}

func (uf *FoodChainUF) Find(x int) int {
	if uf.parent[x] == x { return x }
	root := uf.Find(uf.parent[x])
	uf.rel[x] = (uf.rel[x] + uf.rel[uf.parent[x]]) % 3
	uf.parent[x] = root
	return root
}

func (uf *FoodChainUF) SameType(x, y int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry {
		uf.parent[rx] = ry
		uf.rel[rx] = (uf.rel[y] - uf.rel[x] + 3) % 3 // same type → diff = 0
		return true
	}
	return (uf.rel[x]-uf.rel[y]+3)%3 == 0
}

func (uf *FoodChainUF) XeatsY(x, y int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry {
		uf.parent[rx] = ry
		uf.rel[rx] = (uf.rel[y] - uf.rel[x] + 1 + 3) % 3
		return true
	}
	return (uf.rel[x]-uf.rel[y]+3)%3 == 1
}

func main() {
	n := 6
	uf := NewFoodChainUF(n)

	// Valid statements
	fmt.Println(uf.SameType(1, 2)) // true (first info)
	fmt.Println(uf.XeatsY(2, 3))   // true
	fmt.Println(uf.XeatsY(1, 3))   // true (1 same as 2, 2 eats 3)

	// Invalid: 3 eats 1? (1 eats 3's type, so 3 doesn't eat 1)
	fmt.Println(uf.XeatsY(3, 1))   // false — contradiction
}
```

---

## Example 6: Potential Function Approach

```go
package main

import "fmt"

type PotentialUF struct {
	parent    []int
	potential []int // potential[x] relative to root
}

func NewPotentialUF(n int) *PotentialUF {
	p := make([]int, n)
	pot := make([]int, n)
	for i := range p { p[i] = i }
	return &PotentialUF{parent: p, potential: pot}
}

func (uf *PotentialUF) Find(x int) int {
	if uf.parent[x] == x { return x }
	root := uf.Find(uf.parent[x])
	uf.potential[x] += uf.potential[uf.parent[x]]
	uf.parent[x] = root
	return root
}

// Union such that potential[x] - potential[y] = w
func (uf *PotentialUF) Union(x, y, w int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry {
		// Check consistency
		return uf.potential[x]-uf.potential[y] == w
	}
	uf.parent[rx] = ry
	uf.potential[rx] = w + uf.potential[y] - uf.potential[x]
	return true
}

func (uf *PotentialUF) Diff(x, y int) (int, bool) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry { return 0, false }
	return uf.potential[x] - uf.potential[y], true
}

func main() {
	uf := NewPotentialUF(5)
	uf.Union(0, 1, 10) // a - b = 10
	uf.Union(1, 2, 20) // b - c = 20
	uf.Union(3, 4, 5)  // d - e = 5

	if d, ok := uf.Diff(0, 2); ok {
		fmt.Printf("a - c = %d\n", d) // 30
	}
	if _, ok := uf.Diff(0, 3); !ok {
		fmt.Println("0 and 3 not connected")
	}
}
```

---

## Example 7: Checking Consistency of Equations

```go
package main

import "fmt"

func checkConsistency(n int, equations []struct{ a, b, diff int }) (bool, int) {
	parent := make([]int, n)
	pot := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] == x { return x }
		root := find(parent[x])
		pot[x] += pot[parent[x]]
		parent[x] = root
		return root
	}

	falseCount := 0
	for _, eq := range equations {
		rx, ry := find(eq.a), find(eq.b)
		if rx == ry {
			if pot[eq.a]-pot[eq.b] != eq.diff {
				fmt.Printf("Contradiction: %d - %d should be %d, got %d\n",
					eq.a, eq.b, eq.diff, pot[eq.a]-pot[eq.b])
				falseCount++
			}
		} else {
			parent[rx] = ry
			pot[rx] = eq.diff + pot[eq.b] - pot[eq.a]
		}
	}
	return falseCount == 0, falseCount
}

func main() {
	equations := []struct{ a, b, diff int }{
		{0, 1, 5},  // x0 - x1 = 5
		{1, 2, 3},  // x1 - x2 = 3
		{0, 2, 8},  // x0 - x2 = 8 ✓ (5+3=8)
		{0, 2, 7},  // x0 - x2 = 7 ✗ (contradiction)
	}
	ok, falseCount := checkConsistency(3, equations)
	fmt.Printf("Consistent: %v, Contradictions: %d\n", ok, falseCount)
}
```

---

## Example 8: Currency Exchange Rates

```go
package main

import (
	"fmt"
	"math"
)

type CurrencyUF struct {
	parent []int
	rate   []float64 // rate[x] = exchange rate from x to parent[x]
}

func NewCurrencyUF(n int) *CurrencyUF {
	p := make([]int, n)
	r := make([]float64, n)
	for i := range p { p[i] = i; r[i] = 1.0 }
	return &CurrencyUF{parent: p, rate: r}
}

func (uf *CurrencyUF) Find(x int) int {
	if uf.parent[x] == x { return x }
	root := uf.Find(uf.parent[x])
	uf.rate[x] *= uf.rate[uf.parent[x]]
	uf.parent[x] = root
	return root
}

func (uf *CurrencyUF) AddRate(x, y int, r float64) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return }
	uf.parent[rx] = ry
	uf.rate[rx] = r * uf.rate[y] / uf.rate[x]
}

func (uf *CurrencyUF) Convert(x, y int) (float64, bool) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry { return 0, false }
	return uf.rate[x] / uf.rate[y], true
}

func (uf *CurrencyUF) DetectArbitrage(x, y int, knownRate float64) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx != ry { return false }
	computed := uf.rate[x] / uf.rate[y]
	return math.Abs(computed-knownRate) > 1e-9
}

func main() {
	// 0=USD, 1=EUR, 2=GBP, 3=JPY
	uf := NewCurrencyUF(4)
	uf.AddRate(0, 1, 0.85) // 1 USD = 0.85 EUR
	uf.AddRate(1, 2, 0.88) // 1 EUR = 0.88 GBP

	if rate, ok := uf.Convert(0, 2); ok {
		fmt.Printf("1 USD = %.4f GBP\n", rate) // 0.748
	}
	if rate, ok := uf.Convert(2, 0); ok {
		fmt.Printf("1 GBP = %.4f USD\n", rate)
	}
	if _, ok := uf.Convert(0, 3); !ok {
		fmt.Println("USD-JPY rate unknown")
	}
}
```

---

## Example 9: Path Compression with Weights Trace

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Weighted UF Path Compression ===")
	fmt.Println()

	parent := []int{1, 2, 3, 3}
	weight := []float64{2.0, 3.0, 5.0, 1.0}
	// Chain: 0 →(2.0) 1 →(3.0) 2 →(5.0) 3

	fmt.Println("Before compression:")
	fmt.Printf("  0 →(%.1f) 1 →(%.1f) 2 →(%.1f) 3\n", weight[0], weight[1], weight[2])
	fmt.Println("  value[0]/value[3] = 2.0 × 3.0 × 5.0 = 30.0")

	var find func(x int) int
	find = func(x int) int {
		if parent[x] == x { return x }
		root := find(parent[x])
		fmt.Printf("  Compress %d: weight %.1f × %.1f = %.1f\n", x, weight[x], weight[parent[x]], weight[x]*weight[parent[x]])
		weight[x] *= weight[parent[x]]
		parent[x] = root
		return root
	}

	fmt.Println()
	fmt.Println("Find(0):")
	find(0)
	fmt.Println()
	fmt.Println("After compression:")
	fmt.Printf("  parent: %v\n", parent)
	fmt.Printf("  weight: %v\n", weight)
	fmt.Println("  All nodes point directly to root 3")
}
```

---

## Example 10: Weighted UF Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Weighted Union-Find Summary ===")
	fmt.Println()

	topics := []struct{ topic, detail string }{
		{"Core Idea", "Store relationship weights on parent edges"},
		{"Find", "Compress path AND combine weights to root"},
		{"Union", "Set root weight to maintain relationship invariant"},
		{"Query", "Compare weights-to-root of two nodes"},
		{"Ratio queries", "weight[x]/weight[y] gives x/y ratio"},
		{"Distance queries", "weight[x]-weight[y] gives relative distance"},
		{"Modular (2 groups)", "(rel[x]+rel[y])%2 checks same group"},
		{"Modular (3 groups)", "Food chain: (rel[x]-rel[y]+3)%3 checks relation"},
		{"Consistency", "If same root, check if computed matches given"},
		{"LeetCode 399", "Evaluate Division — classic weighted UF problem"},
	}

	for i, t := range topics {
		fmt.Printf("%2d. %-22s %s\n", i+1, t.topic, t.detail)
	}

	fmt.Println()
	fmt.Println("Time: O(α(n)) per operation (same as standard UF)")
	fmt.Println("Space: O(n) extra for weight array")
}
```

---

## Key Takeaways

1. Weighted UF stores relationship values on edges to parents
2. Path compression must multiply/add weights along the compressed path
3. Enables ratio queries (LeetCode 399), distance queries, group classification
4. Modular weights handle k-group problems (friend/enemy, food chain)
5. Same O(α(n)) amortized complexity as standard Union-Find

> **Phase 13 Complete! Next up:** Phase 14 — Backtracking →
