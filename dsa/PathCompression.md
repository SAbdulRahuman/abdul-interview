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

**Textual Figure:**
```
  Full Path Compression (Recursive): chain 0→1→2→3→4→5

  BEFORE Find(0):                  AFTER Find(0):
  parent = [1, 2, 3, 4, 5, 5]     parent = [5, 5, 5, 5, 5, 5]

    ┌───┐                            ┌───┐
    │ 5 │ ← root                    │ 5 │ ← root
    └─┬─┘                            └─┬─┘
    ┌─┴─┐                       ┌──┬──┼──┬──┬──┐
    │ 4 │                     ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴┐
    └─┬─┘                     │0││1││2││3││4│
    ┌─┴─┐                     └─┘└─┘└─┘└─┘└─┘
    │ 3 │
    └─┬─┘                     Height: 5 → 1
    ┌─┴─┐
    │ 2 │                     Recursive trace:
    └─┬─┘                       Find(0) → Find(1) → Find(2)
    ┌─┴─┐                       → Find(3) → Find(4) → Find(5)=5
    │ 1 │                       Unwind: p[4]=5, p[3]=5,
    └─┬─┘                       p[2]=5, p[1]=5, p[0]=5
    ┌─┴─┐
    │ 0 │
    └───┘
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

**Textual Figure:**
```
  Iterative Path Compression: Two-pass approach

  parent = [1, 2, 3, 4, 5, 5]   chain: 0→1→2→3→4→5

  Pass 1 — Find root:
    x=0 → p[0]=1 → p[1]=2 → p[2]=3 → p[3]=4 → p[4]=5 → p[5]=5 STOP
    root = 5

  Pass 2 — Compress path:
    ┌──────┬────────────┬───────────────┬─────────┐
    │ Node │ Old parent │ Set to root=5 │ Next    │
    ├──────┼────────────┼───────────────┼─────────┤
    │  0   │    1       │ parent[0]=5   │ x=1     │
    │  1   │    2       │ parent[1]=5   │ x=2     │
    │  2   │    3       │ parent[2]=5   │ x=3     │
    │  3   │    4       │ parent[3]=5   │ x=4     │
    │  4   │    5       │ parent[4]=5   │ x=5=root│
    └──────┴────────────┴───────────────┴─────────┘

  Result: parent = [5, 5, 5, 5, 5, 5]
            ┌───┐
            │ 5 │   All nodes point directly to root
            └─┬─┘
       ┌──┬──┼──┬──┐
     ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴┐
     │0││1││2││3││4│
     └─┘└─┘└─┘└─┘└─┘
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

**Textual Figure:**
```
  Path Halving: parent[x] = parent[parent[x]], then x = parent[x]
  Skips every other node to grandparent (single pass)

  parent = [1, 2, 3, 4, 5, 6, 7, 7]   chain: 0→1→2→3→4→5→6→7

  Find(0) trace:
  ┌──────┬────────────────────────────┬───────────┬──────────┐
  │ x    │ Action                     │ parent[x] │ next x   │
  ├──────┼────────────────────────────┼───────────┼──────────┤
  │  0   │ p[0]=p[p[0]]=p[1]=2       │  0 → 2   │ x=2      │
  │  2   │ p[2]=p[p[2]]=p[3]=4       │  2 → 4   │ x=4      │
  │  4   │ p[4]=p[p[4]]=p[5]=6       │  4 → 6   │ x=6      │
  │  6   │ p[6]=p[p[6]]=p[7]=7       │  6 → 7   │ x=7=root │
  └──────┴────────────────────────────┴───────────┴──────────┘

  BEFORE:                         AFTER halving:
    ┌─┐                              ┌─┐
    │7│ root                         │7│ root
    └┬┘                              └┬┘
    ┌┴┐                             ┌─┴─┐
    │6│                           ┌─┴┐ ┌┴─┐
    └┬┘                           │4│ │6 │   Nodes 0,2,4,6 jumped
    ┌┴┐                           └┬┘ └──┘   to grandparents
    │5│                           ┌┴┐
    └┬┘                           │2│       Nodes 1,3,5 unchanged
    ┌┴┐                           └┬┘
    │4│                           ┌┴┐
    └┬┘                           │0│
    ┌┴┐                           └─┘
    │3│
    └┬┘                           Height: 7 → 4
    ┌┴┐                           (halved, roughly)
    │2│
    └┬┘
    ┌┴┐
    │1│
    └┬┘
    ┌┴┐
    │0│
    └─┘
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

**Textual Figure:**
```
  Path Splitting: each node points to its grandparent
  parent[x] = parent[next], then x = next (single pass)

  parent = [1, 2, 3, 4, 5, 6, 7, 7]   chain: 0→1→2→3→4→5→6→7

  Find(0) trace:
  ┌──────┬─────────────────────────────┬───────────┬─────────┐
  │ x    │ Action                      │ parent[x] │ next x  │
  ├──────┼─────────────────────────────┼───────────┼─────────┤
  │  0   │ next=1, p[0]=p[1]=2         │  0 → 2   │ x=1     │
  │  1   │ next=2, p[1]=p[2]=3         │  1 → 3   │ x=2     │
  │  2   │ next=3, p[2]=p[3]=4         │  2 → 4   │ x=3     │
  │  3   │ next=4, p[3]=p[4]=5         │  3 → 5   │ x=4     │
  │  4   │ next=5, p[4]=p[5]=6         │  4 → 6   │ x=5     │
  │  5   │ next=6, p[5]=p[6]=7         │  5 → 7   │ x=6     │
  │  6   │ next=7, p[6]=p[7]=7         │  6 → 7   │ x=7=root│
  └──────┴─────────────────────────────┴───────────┴─────────┘

  AFTER splitting: parent = [2, 3, 4, 5, 6, 7, 7, 7]

        ┌─┐
        │7│ root
        └┬┘
      ┌──┴──┐
    ┌─┴┐  ┌┴─┐       Every node now points
    │5│  │6 │       to its grandparent
    └┬┘  └──┘
    ┌┴┐                Difference from halving:
    │4│                path splitting modifies
    └┬┘                ALL nodes on path,
    ┌┴┐                halving modifies every OTHER
    │3│
    └┬┘
    ┌┴┐
    │2│
    └┬┘
    ┌┴┐
    │1│
    └┬┘
    ┌┴┐
    │0│
    └─┘
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

**Textual Figure:**
```
  Visualizing Compression Effects:
  Chain: 0→1→2→3→4→5  parent = [1, 2, 3, 4, 5, 5]

  Find(0) — first call traces full path:
    Path to root: [0, 1, 2, 3, 4, 5]

  BEFORE:              AFTER Find(0):         AFTER Find(1):
    ┌─┐                   ┌─┐                    ┌─┐
    │5│ root              │5│ root               │5│ root
    └┬┘                   └┬┘                    └┬┘
    ┌┴┐              ┌─┬──┼─┬──┐          ┌─┬──┼─┬──┐
    │4│            ┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐     ┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐
    └┬┘            │0││1││2││3││4│     │0││1││2││3││4│
    ┌┴┐            └─┘└─┘└─┘└─┘└─┘     └─┘└─┘└─┘└─┘└─┘
    │3│
    └┬┘           All compressed        Already flat!
    ┌┴┐           parent=[5,5,5,5,5,5]  Find(1)=5 in O(1)
    │2│
    └┬┘           First Find(0): 5 hops (O(n))
    ┌┴┐           Second Find(1): 1 hop  (O(1))
    │1│
    └┬┘           Amortized: expensive first call
    ┌┴┐           pays for all future O(1) calls
    │0│
    └─┘
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

**Textual Figure:**
```
  Benchmark: n=100,000 chain, n Find operations

  WITHOUT compression (each Find traverses full chain):
    Find(0):     0→1→2→...→99999      99999 steps
    Find(1):     1→2→3→...→99999      99998 steps
    Find(2):     2→3→4→...→99999      99997 steps
    ...                              ...
    Find(99999): already root          0 steps
    Total: n(n-1)/2 ≈ 5×10⁹ steps     O(n²)

  WITH compression (Find + flatten):
    Find(0):     99999 steps + compress all          (O(n))
    Find(1):     1 step (already points to root)     (O(1))
    Find(2):     1 step                              (O(1))
    ...          1 step each                         (O(1))
    Find(99999): already root                        (O(1))
    Total: n + (n-1) ≈ 2n steps                      O(n)

  ┌──────────────────────┬──────────────┬──────────────┐
  │                      │ No Compress  │ Compressed   │
  ├──────────────────────┼──────────────┼──────────────┤
  │ Total operations      │ O(n²)        │ O(n)         │
  │ First Find            │ O(n)         │ O(n)         │
  │ Subsequent Finds      │ O(n) each    │ O(1) each    │
  │ Speedup at n=100K     │ ~5×10⁹       │ ~2×10⁵       │
  └──────────────────────┴──────────────┴──────────────┘
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

**Textual Figure:**
```
  n=8, edges: {0,1},{1,2},{3,4},{5,6},{6,7},{7,3}

  ┌─────────┬────────────┬───────────────────┬───────┐
  │ Edge    │ find(u,v)  │ Action            │ count │
  ├─────────┼────────────┼───────────────────┼───────┤
  │ {0,1}   │ 0 ≠ 1     │ parent[0]=1       │ 8→7  │
  │ {1,2}   │ 1 ≠ 2     │ parent[1]=2       │ 7→6  │
  │ {3,4}   │ 3 ≠ 4     │ parent[3]=4       │ 6→5  │
  │ {5,6}   │ 5 ≠ 6     │ parent[5]=6       │ 5→4  │
  │ {6,7}   │ 6 ≠ 7     │ parent[6]=7       │ 4→3  │
  │ {7,3}   │ 7 ≠ 4     │ parent[7]=4       │ 3→2  │
  └─────────┴────────────┴───────────────────┴───────┘

  Before forced compression:    After find(i) for all i:
    ┌───┐      ┌───┐            ┌───┐     ┌───┐
    │ 2 │      │ 4 │            │ 2 │     │ 4 │
    └─┬─┘      └─┬─┘            └─┬─┘     └─┬─┘
    ┌─┴─┐     ┌──┴──┐         ┌──┴──┐ ┌──┬┴─┬──┐
    │ 1 │     │ 3  7│       ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐
    └─┬─┘     └──┬──┘       │0││1││3││5││6││7│
    ┌─┴─┐     ┌─┴─┐         └─┘└─┘└─┘└─┘└─┘└─┘
    │ 0 │     │ 5 6│
    └───┘     └───┘         parent = [2,2,2,4,4,4,4,4]
                              All flat! 2 components
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

**Textual Figure:**
```
  Kruskal's MST with path compression: n=4, 5 edges

  Sorted edges: {1,2,w=1}, {1,3,w=2}, {0,2,w=3}, {0,1,w=4}, {2,3,w=5}

  ┌────────┬────┬───────────────────────────────┬──────────────────┐
  │ Edge   │ W  │ Action                        │ parent[]         │
  ├────────┼────┼───────────────────────────────┼──────────────────┤
  │ {1,2}  │  1 │ find(1)=1≠find(2)=2 → MST ✓ │ [0,2,2,3]        │
  │ {1,3}  │  2 │ find(1)=2≠find(3)=3 → MST ✓ │ [0,2,3,3]        │
  │        │    │ (compression: p[1]→2→3)      │ p[1] compressed→3│
  │ {0,2}  │  3 │ find(0)=0≠find(2)=3 → MST ✓ │ [3,3,3,3]        │
  │        │    │ 3 MST edges = n-1 → DONE    │                  │
  └────────┴────┴───────────────────────────────┴──────────────────┘

  Final UF forest (fully compressed):
        ┌───┐
        │ 3 │ (root)
        └─┬─┘
     ┌───┼───┐
   ┌─┴─┐┌┴─┐┌┴─┐
   │ 0 ││ 1 ││ 2 │     MST weight = 1+2+3 = 6
   └───┘└───┘└───┘

  MST edges: 1─2, 1─3, 0─2  (weight 6)
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

**Textual Figure:**
```
  Amortized Analysis: n=16 chain, compressed by Find(0)

  BEFORE Find(0): chain 0→1→2→...→15
    ┌──┐
    │15│ ← root
    └┬─┘
    ┌┴─┐
    │14│
    └┬─┘
     ...          Height = 15
    ┌┴┐
    │1│
    └┬┘
    ┌┴┐
    │0│
    └─┘

  First Find(0): 15 parent updates (O(n))
    parent = [15,15,15,15,15,15,15,15,15,15,15,15,15,15,15,15]

  AFTER Find(0): completely flat
             ┌──┐
             │15│ root
             └┬─┘
    ┌─┬─┬─┬─┬┼┬─┬─┬─┬─┬─┬─┬─┬─┐
    0 1 2 3 4 5 6 7 8 9 ...14
    All point directly to 15

  ┌───────────┬────────────────┬─────────────────┐
  │ Call      │ Parent updates │ Cost            │
  ├───────────┼────────────────┼─────────────────┤
  │ Find(0)   │      15       │ O(n) expensive  │
  │ Find(0)   │       0       │ O(1) free       │
  │ Find(1)   │       0       │ O(1) free       │
  │ ...       │       0       │ O(1) free       │
  │ Find(14)  │       0       │ O(1) free       │
  └───────────┴────────────────┴─────────────────┘

  Amortized cost: 15 total updates / 16 finds = O(1) per find
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

**Textual Figure:**
```
  Compression Variants Comparison on chain 0→1→2→3→4→5→6→7:

  1. Full Compression        2. Path Halving       3. Path Splitting
     (All → root)              (skip nodes)          (all → grandparent)

  BEFORE (all same):            BEFORE:               BEFORE:
    ┌─┐                          ┌─┐                    ┌─┐
    │7│ root                     │7│                    │7│
    └┬┘                          └┬┘                    └┬┘
    ...                          ...                    ...
    ┌┴┐                          ┌┴┐                    ┌┴┐
    │0│                          │0│                    │0│
    └─┘                          └─┘                    └─┘

  AFTER Find(0):              AFTER Find(0):         AFTER Find(0):
      ┌─┐                        ┌─┐                    ┌─┐
      │7│                        │7│                    │7│
      └┬┘                        └┬┘                    └┬┘
  ┌┬┬┬┼┬┬┐                   ┌─┴─┐                ┌──┴──┐
  0123456                     │4 6│              ┌─┴─┐ ┌┴─┐
                              └─┬─┘              │5  │ │6 │
  Height: 1                   ┌┴┐               └─┬─┘ └──┘
  All nodes → root            │2│               ┌┴┐
  Best flattening             └┬┘               │4│
                              ┌┴┐               └┬┘
                              │0│               ┌┴┐
                              └─┘               │3│
                              nodes 1,3,5        └┬┘
                              unchanged          ...
                              Height: 4          Height: halved

  ┌─────────────────────────┬────────────┬──────────┬───────────────┐
  │ Variant                 │ Amortized  │ Passes   │ Nodes modified │
  ├─────────────────────────┼────────────┼──────────┼───────────────┤
  │ Full (recursive)        │ O(α(n))   │ 2 (rec)  │ All on path    │
  │ Full (iterative)        │ O(α(n))   │ 2        │ All on path    │
  │ Path halving            │ O(α(n))   │ 1        │ Every other    │
  │ Path splitting          │ O(α(n))   │ 1        │ All on path    │
  │ No compression          │ O(n)      │ 1        │ None           │
  └─────────────────────────┴────────────┴──────────┴───────────────┘
```

---

## Key Takeaways

1. Path compression makes every node on the find path point directly to root
2. Reduces subsequent Find calls to near O(1)
3. Three variants: full compression, path halving, path splitting — all O(α(n))
4. Always use path compression — no reason not to (no extra space cost)
5. Combined with union by rank/size → O(α(n)) amortized per operation

> **Next up:** Union by Rank →
