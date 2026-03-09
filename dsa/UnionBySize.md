# Phase 13: Union Find — Union by Size

## Overview

**Union by size** attaches the smaller set under the root of the larger set. Unlike rank (an upper bound on height), size tracks the **exact number of nodes** in each component.

| Union by Rank | Union by Size |
|---------------|--------------|
| Tracks height upper bound | Tracks exact component size |
| Increment only when equal | Always add sizes |
| Slightly faster in theory | More useful (gives component sizes) |
| Both achieve O(α(n)) with compression | Both achieve O(α(n)) with compression |

---

## Example 1: Basic Union by Size

```go
package main

import "fmt"

type UnionFind struct {
	parent []int
	size   []int
}

func NewUF(n int) *UnionFind {
	p := make([]int, n)
	s := make([]int, n)
	for i := range p { p[i] = i; s[i] = 1 }
	return &UnionFind{parent: p, size: s}
}

func (uf *UnionFind) Find(x int) int {
	if uf.parent[x] != x {
		uf.parent[x] = uf.Find(uf.parent[x])
	}
	return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return false }

	// Attach smaller under larger
	if uf.size[rx] < uf.size[ry] { rx, ry = ry, rx }
	uf.parent[ry] = rx
	uf.size[rx] += uf.size[ry]
	return true
}

func (uf *UnionFind) ComponentSize(x int) int {
	return uf.size[uf.Find(x)]
}

func main() {
	uf := NewUF(8)
	ops := [][2]int{{0,1},{2,3},{4,5},{6,7},{0,2},{4,6},{0,4}}
	for _, op := range ops {
		uf.Union(op[0], op[1])
		fmt.Printf("Union(%d,%d): sizes at roots: ", op[0], op[1])
		for i := 0; i < 8; i++ {
			if uf.Find(i) == i { fmt.Printf("[%d]=%d ", i, uf.size[i]) }
		}
		fmt.Println()
	}
}
```

---

## Example 2: Largest Component Size

```go
package main

import "fmt"

func largestComponent(n int, edges [][2]int) int {
	parent := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	maxSize := 1
	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { continue }
		if size[rx] < size[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
		if size[rx] > maxSize { maxSize = size[rx] }
	}
	return maxSize
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4},{5,6},{6,7},{3,7}}
	fmt.Println("Largest component:", largestComponent(8, edges)) // 4
}
```

---

## Example 3: Size-Based Component Query

```go
package main

import "fmt"

type DSU struct {
	parent []int
	size   []int
	count  int // number of components
}

func NewDSU(n int) *DSU {
	p := make([]int, n)
	s := make([]int, n)
	for i := range p { p[i] = i; s[i] = 1 }
	return &DSU{parent: p, size: s, count: n}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
	rx, ry := d.Find(x), d.Find(y)
	if rx == ry { return false }
	if d.size[rx] < d.size[ry] { rx, ry = ry, rx }
	d.parent[ry] = rx
	d.size[rx] += d.size[ry]
	d.count--
	return true
}

func (d *DSU) GetSize(x int) int { return d.size[d.Find(x)] }
func (d *DSU) Components() int   { return d.count }

func main() {
	dsu := NewDSU(10)
	dsu.Union(0, 1); dsu.Union(2, 3); dsu.Union(4, 5)
	dsu.Union(0, 2); dsu.Union(6, 7)

	fmt.Println("Components:", dsu.Components())       // 5
	fmt.Println("Size of 0's group:", dsu.GetSize(0))  // 4
	fmt.Println("Size of 4's group:", dsu.GetSize(4))  // 2
	fmt.Println("Size of 8's group:", dsu.GetSize(8))  // 1
}
```

---

## Example 4: Making a Large Island (LeetCode 827)

```go
package main

import "fmt"

func largestIsland(grid [][]int) int {
	n := len(grid)
	parent := make([]int, n*n)
	size := make([]int, n*n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(a, b int) {
		ra, rb := find(a), find(b)
		if ra == rb { return }
		if size[ra] < size[rb] { ra, rb = rb, ra }
		parent[rb] = ra
		size[ra] += size[rb]
	}

	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}
	id := func(r, c int) int { return r*n + c }

	// Build islands
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			if grid[i][j] == 0 { continue }
			for _, d := range dirs {
				ni, nj := i+d[0], j+d[1]
				if ni >= 0 && ni < n && nj >= 0 && nj < n && grid[ni][nj] == 1 {
					union(id(i, j), id(ni, nj))
				}
			}
		}
	}

	// Try flipping each 0
	best := 0
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			if find(id(i, j)) == id(i, j) && grid[i][j] == 1 {
				if size[find(id(i,j))] > best { best = size[find(id(i,j))] }
			}
		}
	}

	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			if grid[i][j] == 1 { continue }
			seen := map[int]bool{}
			total := 1
			for _, d := range dirs {
				ni, nj := i+d[0], j+d[1]
				if ni >= 0 && ni < n && nj >= 0 && nj < n && grid[ni][nj] == 1 {
					root := find(id(ni, nj))
					if !seen[root] {
						seen[root] = true
						total += size[root]
					}
				}
			}
			if total > best { best = total }
		}
	}

	if best == 0 { return 1 }
	return best
}

func main() {
	grid := [][]int{{1,0},{0,1}}
	fmt.Println("Largest island:", largestIsland(grid)) // 3

	grid2 := [][]int{{1,1},{1,0}}
	fmt.Println("Largest island:", largestIsland(grid2)) // 4
}
```

---

## Example 5: Size Trace During Merges

```go
package main

import "fmt"

func main() {
	n := 8
	parent := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx == ry {
			fmt.Printf("  Already connected\n")
			return
		}
		fmt.Printf("  Root %d (size %d) vs Root %d (size %d)", rx, size[rx], ry, size[ry])
		if size[rx] < size[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
		fmt.Printf(" → attach %d under %d, new size = %d\n", ry, rx, size[rx])
	}

	ops := [][2]int{{0,1},{2,3},{0,3},{4,5},{4,6},{4,7},{0,4}}
	for _, op := range ops {
		fmt.Printf("Union(%d, %d):\n", op[0], op[1])
		union(op[0], op[1])
	}
}
```

---

## Example 6: Component Size Histogram

```go
package main

import "fmt"

func componentHistogram(n int, edges [][2]int) map[int]int {
	parent := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { continue }
		if size[rx] < size[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
	}

	hist := map[int]int{} // componentSize → count
	for i := 0; i < n; i++ {
		if find(i) == i { hist[size[i]]++ }
	}
	return hist
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4},{5,6},{7,8},{8,9},{7,9}}
	hist := componentHistogram(10, edges)
	for sz, cnt := range hist {
		fmt.Printf("Size %d: %d component(s)\n", sz, cnt)
	}
}
```

---

## Example 7: Process Queries Online

```go
package main

import "fmt"

func main() {
	n := 6
	parent := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx == ry { return }
		if size[rx] < size[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
	}

	type Query struct {
		qType string // "union" or "size"
		a, b  int
	}

	queries := []Query{
		{"union", 0, 1}, {"size", 0, 0}, {"union", 2, 3},
		{"union", 0, 2}, {"size", 3, 0}, {"union", 4, 5},
		{"size", 4, 0},
	}

	for _, q := range queries {
		switch q.qType {
		case "union":
			union(q.a, q.b)
			fmt.Printf("Union(%d, %d)\n", q.a, q.b)
		case "size":
			fmt.Printf("Size of %d's component: %d\n", q.a, size[find(q.a)])
		}
	}
}
```

---

## Example 8: Minimum Cost to Connect All Points (with Size)

```go
package main

import (
	"fmt"
	"sort"
)

func minCostConnectPoints(points [][]int) int {
	n := len(points)
	abs := func(x int) int { if x < 0 { return -x }; return x }

	type Edge struct{ u, v, w int }
	edges := []Edge{}
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			w := abs(points[i][0]-points[j][0]) + abs(points[i][1]-points[j][1])
			edges = append(edges, Edge{i, j, w})
		}
	}
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	cost := 0
	for _, e := range edges {
		rx, ry := find(e.u), find(e.v)
		if rx == ry { continue }
		if size[rx] < size[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
		cost += e.w
	}
	return cost
}

func main() {
	points := [][]int{{0,0},{2,2},{3,10},{5,2},{7,0}}
	fmt.Println("Min cost:", minCostConnectPoints(points)) // 20
}
```

---

## Example 9: Size Property Proof

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union by Size: Properties ===")
	fmt.Println()

	fmt.Println("CLAIM: With union by size, tree height ≤ log₂(n)")
	fmt.Println()
	fmt.Println("PROOF:")
	fmt.Println("  - A node's depth increases by 1 only when its tree")
	fmt.Println("    is merged under a LARGER tree")
	fmt.Println("  - Each time depth increases, the component size at least doubles")
	fmt.Println("  - Max doublings from 1 to n: log₂(n)")
	fmt.Println()

	fmt.Println("SIZE vs RANK:")
	fmt.Printf("%-20s %-25s %-25s\n", "Feature", "Union by Size", "Union by Rank")
	fmt.Println("------------------------------------------------------------")
	data := []struct{ f, s, r string }{
		{"Tracks", "Exact node count", "Height upper bound"},
		{"Update rule", "Add sizes", "Increment if equal"},
		{"Useful info", "Component sizes available", "Only structure info"},
		{"Height bound", "O(log n)", "O(log n)"},
		{"With compression", "O(α(n))", "O(α(n))"},
	}
	for _, d := range data {
		fmt.Printf("%-20s %-25s %-25s\n", d.f, d.s, d.r)
	}
}
```

---

## Example 10: When to Use Size vs Rank

```go
package main

import "fmt"

func main() {
	fmt.Println("=== When to Use Union by Size vs Rank ===")
	fmt.Println()

	scenarios := []struct{ scenario, choice, reason string }{
		{"Need component sizes", "Size", "Directly available from size array"},
		{"Making a Large Island (LC 827)", "Size", "Need neighbor component sizes"},
		{"Counting components only", "Either", "Both work equally well"},
		{"Kruskal's MST", "Either", "Only need connectivity check"},
		{"LargestComponentSize problems", "Size", "Maximum is max(size[root])"},
		{"Competitive programming", "Rank", "Slightly simpler to code correctly"},
		{"Minimal memory", "Rank", "Rank array uses less space (small values)"},
		{"Online sizing queries", "Size", "O(1) component size lookup"},
		{"Academic/theoretical", "Rank", "Tighter formal analysis"},
		{"Default choice", "Size", "More practical — gives you extra info for free"},
	}

	for i, s := range scenarios {
		fmt.Printf("%2d. %-35s → %-6s (%s)\n", i+1, s.scenario, s.choice, s.reason)
	}
}
```

---

## Key Takeaways

1. Union by size: attach smaller component under larger → O(log n) height
2. Size tracks exact node count — more useful than rank in practice
3. A node's depth increases only when merged under a larger tree
4. Combined with path compression → O(α(n)) amortized per operation
5. Prefer size when you need component size queries (LC 827, 200, etc.)

> **Next up:** Weighted Union Find →
