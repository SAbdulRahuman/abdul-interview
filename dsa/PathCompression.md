# Phase 13: Union Find — Path Compression

## Overview

**Path compression** is an optimization for the `Find` operation that makes every node on the find path point directly to the root. This flattens the tree, making subsequent finds nearly O(1).

| Without Path Compression | With Path Compression |
|--------------------------|----------------------|
| O(n) worst case per Find | O(α(n)) amortized |
| Deep trees possible | Trees stay flat |
| Simple parent traversal | Update parents during traversal |

### Variants

| Variant | Description |
|---------|------------|
| **Full path compression** | All nodes → root (recursive) |
| **Path halving** | Every other node → grandparent |
| **Path splitting** | Each node → grandparent |

---

## Example 1: Full Path Compression (Recursive)

```go
package main

import "fmt"

type UnionFind struct {
	parent []int
}

func NewUF(n int) *UnionFind {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &UnionFind{parent: p}
}

func (uf *UnionFind) Find(x int) int {
	if uf.parent[x] != x {
		uf.parent[x] = uf.Find(uf.parent[x]) // Path compression
	}
	return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) {
	uf.parent[uf.Find(x)] = uf.Find(y)
}

func main() {
	uf := NewUF(6)
	// Build chain: 0→1→2→3→4→5
	uf.parent = []int{1, 2, 3, 4, 5, 5}
	fmt.Println("Before compression:", uf.parent)

	root := uf.Find(0) // Triggers full compression
	fmt.Println("Root of 0:", root)
	fmt.Println("After compression:", uf.parent) // All point to 5
}
```

---

## Example 2: Path Compression (Iterative)

```go
package main

import "fmt"

type UF struct {
	parent []int
}

func NewUF(n int) *UF {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &UF{parent: p}
}

func (uf *UF) Find(x int) int {
	root := x
	for uf.parent[root] != root { root = uf.parent[root] }
	// Compress: point all nodes on path to root
	for uf.parent[x] != root {
		next := uf.parent[x]
		uf.parent[x] = root
		x = next
	}
	return root
}

func main() {
	uf := NewUF(6)
	uf.parent = []int{1, 2, 3, 4, 5, 5}
	fmt.Println("Before:", uf.parent)

	uf.Find(0)
	fmt.Println("After:", uf.parent) // [5 5 5 5 5 5]
}
```

---

## Example 3: Path Halving

```go
package main

import "fmt"

type UFHalving struct {
	parent []int
}

func NewUFHalving(n int) *UFHalving {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &UFHalving{parent: p}
}

func (uf *UFHalving) Find(x int) int {
	for uf.parent[x] != x {
		uf.parent[x] = uf.parent[uf.parent[x]] // Skip to grandparent
		x = uf.parent[x]
	}
	return x
}

func main() {
	uf := NewUFHalving(8)
	uf.parent = []int{1, 2, 3, 4, 5, 6, 7, 7}
	fmt.Println("Before:", uf.parent)

	root := uf.Find(0)
	fmt.Println("Root:", root)
	fmt.Println("After halving:", uf.parent)
	// Every other node jumps to grandparent
}
```

---

## Example 4: Path Splitting

```go
package main

import "fmt"

type UFSplitting struct {
	parent []int
}

func NewUFSplitting(n int) *UFSplitting {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &UFSplitting{parent: p}
}

func (uf *UFSplitting) Find(x int) int {
	for uf.parent[x] != x {
		next := uf.parent[x]
		uf.parent[x] = uf.parent[next] // Point to grandparent
		x = next
	}
	return x
}

func main() {
	uf := NewUFSplitting(8)
	uf.parent = []int{1, 2, 3, 4, 5, 6, 7, 7}
	fmt.Println("Before:", uf.parent)

	root := uf.Find(0)
	fmt.Println("Root:", root)
	fmt.Println("After splitting:", uf.parent)
}
```

---

## Example 5: Visualizing Compression Effects

```go
package main

import "fmt"

func findWithTrace(parent []int, x int) int {
	path := []int{x}
	for parent[x] != x {
		x = parent[x]
		path = append(path, x)
	}
	root := x
	fmt.Printf("  Path to root: %v\n", path)

	// Compress
	for _, node := range path {
		parent[node] = root
	}
	return root
}

func main() {
	parent := []int{1, 2, 3, 4, 5, 5}
	fmt.Println("Chain: 0→1→2→3→4→5")
	fmt.Println("Before compression:", parent)

	fmt.Println()
	fmt.Println("Find(0):")
	findWithTrace(parent, 0)
	fmt.Println("After compression:", parent)

	fmt.Println()
	fmt.Println("Find(1) now:")
	findWithTrace(parent, 1)
	fmt.Println("Already compressed:", parent)
}
```

---

## Example 6: Compression Benchmark

```go
package main

import (
	"fmt"
	"time"
)

func benchmarkFind(n int, compress bool) time.Duration {
	parent := make([]int, n)
	// Build worst-case chain
	for i := 0; i < n-1; i++ { parent[i] = i + 1 }
	parent[n-1] = n - 1

	findNoCompress := func(x int) int {
		for parent[x] != x { x = parent[x] }
		return x
	}

	findCompress := func(x int) int {
		root := x
		for parent[root] != root { root = parent[root] }
		for parent[x] != root {
			next := parent[x]; parent[x] = root; x = next
		}
		return root
	}

	start := time.Now()
	for i := 0; i < n; i++ {
		if compress {
			findCompress(i)
		} else {
			findNoCompress(i)
		}
	}
	return time.Since(start)
}

func main() {
	n := 100000
	fmt.Printf("n = %d\n", n)
	fmt.Printf("Without compression: %v\n", benchmarkFind(n, false))
	fmt.Printf("With compression:    %v\n", benchmarkFind(n, true))
}
```

---

## Example 7: Path Compression with Connected Components

```go
package main

import "fmt"

func connectedComponents(n int, edges [][2]int) int {
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	count := n
	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx != ry { parent[rx] = ry; count-- }
	}

	// Show parent array after all compressions
	for i := 0; i < n; i++ { find(i) } // Force full compression
	fmt.Println("Parent array:", parent)

	return count
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4},{5,6},{6,7},{7,3}}
	fmt.Println("Components:", connectedComponents(8, edges)) // 2
}
```

---

## Example 8: Compression in Kruskal's MST

```go
package main

import (
	"fmt"
	"sort"
)

func kruskalMST(n int, edges [][3]int) int {
	sort.Slice(edges, func(i, j int) bool { return edges[i][2] < edges[j][2] })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) } // Path compression
		return parent[x]
	}

	totalWeight := 0
	edgesUsed := 0
	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx != ry {
			parent[rx] = ry
			totalWeight += e[2]
			edgesUsed++
			if edgesUsed == n-1 { break }
		}
	}
	return totalWeight
}

func main() {
	edges := [][3]int{{0,1,4},{0,2,3},{1,2,1},{1,3,2},{2,3,5}}
	fmt.Println("MST weight:", kruskalMST(4, edges)) // 6
}
```

---

## Example 9: Amortized Analysis Demonstration

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Path Compression Amortized Analysis ===")
	fmt.Println()

	n := 16
	parent := make([]int, n)
	for i := 0; i < n-1; i++ { parent[i] = i + 1 }
	parent[n-1] = n - 1

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	// First find: O(n) work but compresses everything
	ops := 0
	originalFind := func(x int) int {
		if parent[x] != x {
			ops++
			parent[x] = find(parent[x])
		}
		return parent[x]
	}

	ops = 0
	originalFind(0)
	fmt.Printf("First Find(0): %d parent updates\n", ops)
	fmt.Println("Parent after:", parent)

	// Second find: O(1) — already compressed
	ops = 0
	originalFind(0)
	fmt.Printf("Second Find(0): %d parent updates\n", ops)

	// All subsequent finds are O(1)
	for i := 0; i < n; i++ {
		ops = 0
		originalFind(i)
		fmt.Printf("Find(%d): %d ops, ", i, ops)
	}
	fmt.Println()
}
```

---

## Example 10: Compression Variants Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Path Compression Variants ===")
	fmt.Println()

	variants := []struct{ name, code, amortized, detail string }{
		{
			"Full Compression (Recursive)",
			"if parent[x] != x { parent[x] = Find(parent[x]) }; return parent[x]",
			"O(α(n))",
			"All nodes → root. Best flattening.",
		},
		{
			"Full Compression (Iterative)",
			"Find root, then re-traverse setting all to root",
			"O(α(n))",
			"Two-pass: find root, then compress.",
		},
		{
			"Path Halving",
			"parent[x] = parent[parent[x]]; x = parent[x]",
			"O(α(n))",
			"Every other node → grandparent. Single pass.",
		},
		{
			"Path Splitting",
			"next = parent[x]; parent[x] = parent[next]; x = next",
			"O(α(n))",
			"Each node → grandparent. Single pass.",
		},
		{
			"No Compression",
			"while parent[x] != x { x = parent[x] }",
			"O(n)",
			"Can degenerate to linked list.",
		},
	}

	for i, v := range variants {
		fmt.Printf("%d. %s\n", i+1, v.name)
		fmt.Printf("   Code:      %s\n", v.code)
		fmt.Printf("   Amortized: %s\n", v.amortized)
		fmt.Printf("   Detail:    %s\n\n", v.detail)
	}

	fmt.Println("Key insight: All compression variants achieve O(α(n))")
	fmt.Println("when combined with union by rank/size.")
	fmt.Println("Full compression is most common in competitive programming.")
}
```

---

## Key Takeaways

1. Path compression makes every node on the find path point directly to root
2. Reduces subsequent Find calls to near O(1)
3. Three variants: full compression, path halving, path splitting — all O(α(n))
4. Always use path compression — no reason not to (no extra space cost)
5. Combined with union by rank/size → O(α(n)) amortized per operation

> **Next up:** Union by Rank →
