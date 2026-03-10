# Phase 13: Union Find — Quick Find & Quick Union

## Overview

**Quick Find** and **Quick Union** are the two foundational implementations of the Union-Find data structure, each optimizing a different operation.

| Implementation | Find | Union | Space | Key Idea |
|---------------|------|-------|-------|----------|
| **Quick Find** | O(1) | O(n) | O(n) | All elements store component ID directly |
| **Quick Union** | O(n) worst | O(n) worst | O(n) | Elements store parent pointer, follow to root |
| **Weighted QU** | O(log n) | O(log n) | O(n) | Attach smaller tree under larger |
| **QU + Path Compression** | O(α(n)) | O(α(n)) | O(n) | Flatten tree during find |

---

## Example 1: Quick Find Implementation

```go
package main

import "fmt"

type QuickFind struct {
	id []int // id[i] = component identifier of i
}

func NewQuickFind(n int) *QuickFind {
	id := make([]int, n)
	for i := range id { id[i] = i }
	return &QuickFind{id: id}
}

func (qf *QuickFind) Find(x int) int {
	return qf.id[x] // O(1) — direct lookup!
}

func (qf *QuickFind) Union(x, y int) {
	idX, idY := qf.id[x], qf.id[y]
	if idX == idY { return }
	// Change ALL elements with id[x]'s value to id[y]'s value → O(n)
	for i := range qf.id {
		if qf.id[i] == idX { qf.id[i] = idY }
	}
}

func (qf *QuickFind) Connected(x, y int) bool {
	return qf.id[x] == qf.id[y]
}

func main() {
	qf := NewQuickFind(6)
	pairs := [][2]int{{0,1},{2,3},{4,5},{0,2}}
	for _, p := range pairs {
		qf.Union(p[0], p[1])
		fmt.Printf("Union(%d,%d): id=%v\n", p[0], p[1], qf.id)
	}
	fmt.Println("Connected(0,3):", qf.Connected(0, 3)) // true
	fmt.Println("Connected(0,4):", qf.Connected(0, 4)) // false
}
```

**Textual Figure:**
```
  Quick Find: id[i] stores the component ID directly

  Initial:  id = [0, 1, 2, 3, 4, 5]

  Union(0,1): Change all id==0 to 1
    id = [1, 1, 2, 3, 4, 5]
    Components: {0,1}  {2}  {3}  {4}  {5}
    ┌─────┐  ┌─┐  ┌─┐  ┌─┐  ┌─┐
    │ 0  1│  │2│  │3│  │4│  │5│   id: all store component root
    └─────┘  └─┘  └─┘  └─┘  └─┘

  Union(2,3): Change all id==2 to 3
    id = [1, 1, 3, 3, 4, 5]
    Components: {0,1}  {2,3}  {4}  {5}

  Union(4,5): Change all id==4 to 5
    id = [1, 1, 3, 3, 5, 5]
    Components: {0,1}  {2,3}  {4,5}

  Union(0,2): id[0]=1, id[2]=3 → Change ALL id==1 to 3
    Scan: id[0]=1→3, id[1]=1→3    ← O(n) scan!
    id = [3, 3, 3, 3, 5, 5]
    Components: {0,1,2,3}  {4,5}
    ┌───────────┐  ┌─────┐
    │ 0  1  2  3│  │ 4  5│
    └───────────┘  └─────┘

  Find(0) = id[0] = 3  → O(1)
  Find(3) = id[3] = 3  → O(1)
  Connected(0,3) → 3 == 3 → true ✓
```

---

## Example 2: Quick Union Implementation

```go
package main

import "fmt"

type QuickUnion struct {
	parent []int
}

func NewQuickUnion(n int) *QuickUnion {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &QuickUnion{parent: p}
}

func (qu *QuickUnion) Find(x int) int {
	for qu.parent[x] != x { x = qu.parent[x] } // Follow chain → O(n) worst
	return x
}

func (qu *QuickUnion) Union(x, y int) {
	rx, ry := qu.Find(x), qu.Find(y)
	if rx != ry { qu.parent[rx] = ry } // O(1) link (but Find is O(n))
}

func (qu *QuickUnion) Connected(x, y int) bool {
	return qu.Find(x) == qu.Find(y)
}

func main() {
	qu := NewQuickUnion(6)
	pairs := [][2]int{{0,1},{2,3},{4,5},{0,2}}
	for _, p := range pairs {
		qu.Union(p[0], p[1])
		fmt.Printf("Union(%d,%d): parent=%v\n", p[0], p[1], qu.parent)
	}
	fmt.Println("Connected(0,3):", qu.Connected(0, 3)) // true
}
```

**Textual Figure:**
```
  Quick Union: parent[i] stores parent pointer, root points to itself

  Initial: parent = [0, 1, 2, 3, 4, 5]
    ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
    │0│ │1│ │2│ │3│ │4│ │5│  each is its own root
    └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

  Union(0,1): Find(0)=0, Find(1)=1 → parent[0]=1
    parent = [1, 1, 2, 3, 4, 5]
    ┌─┐       ┌─┐     ┌─┐
    │1│       │3│     │5│
    └┬┘       └─┘     └┬┘
     │                  │
    ┌┴┐       ┌─┐     ┌┴┐
    │0│       │2│     │4│
    └─┘       └─┘     └─┘

  Union(2,3): Find(2)=2, Find(3)=3 → parent[2]=3
    parent = [1, 1, 3, 3, 4, 5]
    ┌─┐  ┌─┐  ┌─┐
    │1│  │3│  │5│
    └┬┘  └┬┘  └┬┘
     │    │    │
    ┌┴┐  ┌┴┐  ┌┴┐
    │0│  │2│  │4│
    └─┘  └─┘  └─┘

  Union(4,5): Find(4)=4, Find(5)=5 → parent[4]=5
    (same structure, {4,5} linked)

  Union(0,2): Find(0)=0→1 (root=1), Find(2)=2→3 (root=3) → parent[1]=3
    parent = [1, 3, 3, 3, 5, 5]
        ┌─┐       ┌─┐
        │3│       │5│     Two trees
        └┬┘       └┬┘
      ┌──┴──┐      │
    ┌─┴─┐ ┌─┴─┐  ┌─┴─┐
    │ 1 │ │ 2 │  │ 4 │
    └─┬─┘ └───┘  └───┘
      │
    ┌─┴─┐
    │ 0 │
    └───┘

  Find(0): 0 → 1 → 3 (root) — up to O(n) in worst case!
```

---

## Example 3: Quick Find — Worst Case Analysis

```go
package main

import "fmt"

func main() {
	n := 8
	id := make([]int, n)
	for i := range id { id[i] = i }
	totalWork := 0

	union := func(x, y int) {
		idX, idY := id[x], id[y]
		if idX == idY { return }
		work := 0
		for i := range id {
			if id[i] == idX { id[i] = idY; work++ }
		}
		totalWork += n // Always scans entire array
		fmt.Printf("Union(%d,%d): scanned %d elements, changed %d, id=%v\n", x, y, n, work, id)
	}

	// Worst case: sequential unions
	for i := 0; i < n-1; i++ {
		union(i, i+1)
	}
	fmt.Printf("\nTotal array scans: %d (n-1 unions × n scan = %d)\n", n-1, (n-1)*n)
}
```

**Textual Figure:**
```
  Quick Find worst case: n-1 sequential unions on n=8

  ┌────────────┬──────────────────────┬───────┬──────────────────┐
  │ Operation  │ id[] array           │ Scans │ Elements changed │
  ├────────────┼──────────────────────┼───────┼──────────────────┤
  │ Initial    │ [0,1,2,3,4,5,6,7]   │   —   │        —         │
  │ Union(0,1) │ [1,1,2,3,4,5,6,7]   │   8   │        1         │
  │ Union(1,2) │ [2,2,2,3,4,5,6,7]   │   8   │        2         │
  │ Union(2,3) │ [3,3,3,3,4,5,6,7]   │   8   │        3         │
  │ Union(3,4) │ [4,4,4,4,4,5,6,7]   │   8   │        4         │
  │ Union(4,5) │ [5,5,5,5,5,5,6,7]   │   8   │        5         │
  │ Union(5,6) │ [6,6,6,6,6,6,6,7]   │   8   │        6         │
  │ Union(6,7) │ [7,7,7,7,7,7,7,7]   │   8   │        7         │
  └────────────┴──────────────────────┴───────┴──────────────────┘

  Total work: 7 unions × 8 scans = 56 = O(n²)

  Cost per Union:  ████████  O(n) — must scan entire id[] array
  Cost per Find:   █         O(1) — direct array lookup

  For M union operations: O(M × n) total → O(n²) for n unions
```

---

## Example 4: Quick Union — Worst Case (Degenerate Tree)

```go
package main

import "fmt"

func main() {
	n := 8
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	// Naive Quick Union: always attach first root under second
	find := func(x int) int {
		steps := 0
		for parent[x] != x { x = parent[x]; steps++ }
		return x
	}

	// Worst case: sequential unions create a chain
	for i := 0; i < n-1; i++ {
		rx := find(i)
		parent[rx] = i + 1
	}

	fmt.Println("parent:", parent)
	// parent = [1,2,3,4,5,6,7,7] — a chain!

	// Find(0) must traverse entire chain
	x := 0
	steps := 0
	for parent[x] != x { x = parent[x]; steps++ }
	fmt.Printf("Find(0): %d steps to reach root %d\n", steps, x)
	_ = find
}
```

**Textual Figure:**
```
  Quick Union worst case: sequential unions create a chain

  Union(0,1), Union(1,2), Union(2,3), ... Union(6,7):
    parent = [1, 2, 3, 4, 5, 6, 7, 7]

  Resulting degenerate tree (linked list!):
    ┌───┐
    │ 7 │ ← root
    └─┬─┘
      │
    ┌─┴─┐
    │ 6 │       Find(0) path:
    └─┬─┘       0 → 1 → 2 → 3 → 4 → 5 → 6 → 7
      │         7 steps = O(n) !
    ┌─┴─┐
    │ 5 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 4 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 3 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 2 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 1 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 0 │ ← deepest node
    └───┘

  Height = n-1 = 7    → Find is O(n) worst case
  Union is O(n) too   → because it calls Find
  n unions → O(n²) total — same as Quick Find!
```

---

## Example 5: Weighted Quick Union (Union by Size)

```go
package main

import "fmt"

type WeightedQU struct {
	parent []int
	size   []int
}

func NewWeightedQU(n int) *WeightedQU {
	p := make([]int, n)
	s := make([]int, n)
	for i := range p { p[i] = i; s[i] = 1 }
	return &WeightedQU{parent: p, size: s}
}

func (wqu *WeightedQU) Find(x int) int {
	for wqu.parent[x] != x { x = wqu.parent[x] }
	return x
}

func (wqu *WeightedQU) Union(x, y int) {
	rx, ry := wqu.Find(x), wqu.Find(y)
	if rx == ry { return }
	// Attach smaller tree under larger tree
	if wqu.size[rx] < wqu.size[ry] {
		wqu.parent[rx] = ry
		wqu.size[ry] += wqu.size[rx]
	} else {
		wqu.parent[ry] = rx
		wqu.size[rx] += wqu.size[ry]
	}
}

func main() {
	wqu := NewWeightedQU(8)
	pairs := [][2]int{{0,1},{2,3},{4,5},{6,7},{0,2},{4,6},{0,4}}
	for _, p := range pairs {
		wqu.Union(p[0], p[1])
		fmt.Printf("Union(%d,%d): parent=%v size=%v\n", p[0], p[1], wqu.parent, wqu.size)
	}
}
```

**Textual Figure:**
```
  Weighted Quick Union: always attach smaller tree under larger

  Step-by-step (8 elements):

  Union(0,1): size equal → parent[1]=0, size[0]=2
  Union(2,3): size equal → parent[3]=2, size[2]=2
  Union(4,5): size equal → parent[5]=4, size[4]=2
  Union(6,7): size equal → parent[7]=6, size[6]=2

    ┌───┐  ┌───┐  ┌───┐  ┌───┐
    │ 0 │  │ 2 │  │ 4 │  │ 6 │   4 trees, height=1
    └─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘
    ┌─┴─┐  ┌─┴─┐  ┌─┴─┐  ┌─┴─┐
    │ 1 │  │ 3 │  │ 5 │  │ 7 │
    └───┘  └───┘  └───┘  └───┘

  Union(0,2): size[0]=2 == size[2]=2 → parent[2]=0, size[0]=4
  Union(4,6): size[4]=2 == size[6]=2 → parent[6]=4, size[4]=4

      ┌───┐              ┌───┐
      │ 0 │ (size=4)     │ 4 │ (size=4)
      └─┬─┘              └─┬─┘
     ┌──┴──┐           ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐      ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 2 │      │ 5 │ │ 6 │
   └───┘ └─┬─┘      └───┘ └─┬─┘
          ┌─┴─┐            ┌─┴─┐
          │ 3 │            │ 7 │
          └───┘            └───┘

  Union(0,4): size[0]=4 == size[4]=4 → parent[4]=0, size[0]=8

            ┌───┐
            │ 0 │ (size=8, root)
            └─┬─┘
        ┌────┬┴────┐
      ┌─┴─┐┌┴──┐┌─┴─┐
      │ 1 ││ 2 ││ 4 │
      └───┘└─┬─┘└─┬─┘
           ┌─┴┐┌──┴──┐
           │ 3││ 5  6│
           └──┘└──┬──┘
                ┌─┴─┐
                │ 7 │
                └───┘

  Max height = 3 = log₂(8) ← guaranteed by weighted union!
  Compare: naive Quick Union → height up to 7
```

---

## Example 6: Quick Find vs Quick Union Comparison

```go
package main

import (
	"fmt"
	"time"
)

func benchQuickFind(n int) time.Duration {
	id := make([]int, n)
	for i := range id { id[i] = i }
	start := time.Now()
	for i := 0; i < n-1; i++ {
		old := id[i]
		for j := range id {
			if id[j] == old { id[j] = id[i+1] }
		}
	}
	return time.Since(start)
}

func benchQuickUnion(n int) time.Duration {
	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	start := time.Now()
	for i := 0; i < n-1; i++ {
		x := i
		for parent[x] != x { x = parent[x] }
		parent[x] = i + 1
	}
	return time.Since(start)
}

func main() {
	for _, n := range []int{1000, 5000, 10000} {
		fmt.Printf("n=%5d  QuickFind: %v  QuickUnion: %v\n", n, benchQuickFind(n), benchQuickUnion(n))
	}
}
```

**Textual Figure:**
```
  Quick Find vs Quick Union — Side by Side:
  ═══════════════════════════════════════════

  QUICK FIND:                    QUICK UNION:
  ┌────────────────────┐         ┌────────────────────┐
  │ id = [3,3,3,3,5,5] │         │ par = [1,3,3,3,5,5]│
  └────────────────────┘         └────────────────────┘
  All elements store root         Elements store parent
  directly → flat array           → tree structure

  ┌─────────┬────────────┬─────────────┐
  │         │ Quick Find │ Quick Union │
  ├─────────┼────────────┼─────────────┤
  │ Find    │    O(1)    │   O(n)      │
  │ Union   │    O(n)    │   O(n)      │
  │ n union │    O(n²)   │   O(n²)     │
  │ Storage │   id[n]    │  parent[n]  │
  │ Tree?   │   No tree  │  Tree form  │
  └─────────┴────────────┴─────────────┘

  Quick Find (flat):           Quick Union (tree):
  ┌───────────────────┐          ┌───┐
  │ 0  1  2  3  4  5  │          │ 3 │ root
  │ all id=3  all id=5│          └─┬─┘
  └───────────────────┘         ┌──┴──┐
  Find = O(1): just return     ┌┴┐  ┌┴┐
  id[x]                        │1│  │2│
                               └┬┘  └─┘
  Union = O(n): scan and        │
  update all matching ids      ┌┴┐
                               │0│  Find = O(n): walk up
                               └─┘

  Both are O(n²) for n operations — neither is great alone!
```

---

## Example 7: Quick Union with Path Compression

```go
package main

import "fmt"

type QUPC struct {
	parent []int
}

func NewQUPC(n int) *QUPC {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &QUPC{parent: p}
}

func (qu *QUPC) Find(x int) int {
	if qu.parent[x] != x {
		qu.parent[x] = qu.Find(qu.parent[x]) // Path compression
	}
	return qu.parent[x]
}

func (qu *QUPC) Union(x, y int) {
	rx, ry := qu.Find(x), qu.Find(y)
	if rx != ry { qu.parent[rx] = ry }
}

func main() {
	qu := NewQUPC(8)
	// Build deep tree first
	qu.parent = []int{1, 2, 3, 4, 5, 6, 7, 7}
	fmt.Println("Before:", qu.parent)

	root := qu.Find(0) // Compresses entire path
	fmt.Println("Root:", root)
	fmt.Println("After:", qu.parent) // All should point to 7
}
```

**Textual Figure:**
```
  Quick Union + Path Compression:

  BEFORE Find(0):                  AFTER Find(0):
  Chain: 0→1→2→3→4→5→6→7          All point directly to root

    ┌───┐                            ┌───┐
    │ 7 │ ← root                     │ 7 │ ← root
    └─┬─┘                            └─┬─┘
      │                          ┌──┬──┼──┬──┬──┬──┐
    ┌─┴─┐                     ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐
    │ 6 │                     │0││1││2││3││4││5││6│
    └─┬─┘                     └─┘└─┘└─┘└─┘└─┘└─┘└─┘
      │
    ┌─┴─┐                     Height: 7 → 1
    │ 5 │                     Next Find(0): O(1) !
    └─┬─┘
      │                        Path compression trace:
    ┌─┴─┐                       Find(0) → Find(1) → Find(2) →
    │ 4 │                       ... → Find(6) → Find(7) = 7
    └─┬─┘                       Unwind: parent[6]=7, parent[5]=7,
      │                         parent[4]=7, parent[3]=7,
    ┌─┴─┐                       parent[2]=7, parent[1]=7,
    │ 3 │                        parent[0]=7
    └─┬─┘
      │                        parent: [1,2,3,4,5,6,7,7]
    ┌─┴─┐                           → [7,7,7,7,7,7,7,7]
    │ 2 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 1 │
    └─┬─┘
      │
    ┌─┴─┐
    │ 0 │
    └───┘
```

---

## Example 8: Connected Components with Quick Find

```go
package main

import "fmt"

func countComponentsQF(n int, edges [][2]int) int {
	id := make([]int, n)
	for i := range id { id[i] = i }

	find := func(x int) int { return id[x] }

	union := func(x, y int) {
		idX, idY := find(x), find(y)
		if idX == idY { return }
		for i := range id {
			if id[i] == idX { id[i] = idY }
		}
	}

	for _, e := range edges {
		union(e[0], e[1])
	}

	roots := map[int]bool{}
	for _, v := range id { roots[v] = true }
	return len(roots)
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4},{5,6},{6,7}}
	fmt.Println("Components:", countComponentsQF(8, edges)) // 3
}
```

**Textual Figure:**
```
  Quick Find: Connected Components on 8 nodes

  Edges: {0,1}, {1,2}, {3,4}, {5,6}, {6,7}

  ┌────────────┬────────────────────────┬─────────────────┐
  │ Edge       │ id[] array             │ Changes         │
  ├────────────┼────────────────────────┼─────────────────┤
  │ Initial    │ [0,1,2,3,4,5,6,7]     │ —               │
  │ {0,1}      │ [1,1,2,3,4,5,6,7]     │ id[0]: 0→1      │
  │ {1,2}      │ [2,2,2,3,4,5,6,7]     │ id[0,1]: 1→2    │
  │ {3,4}      │ [2,2,2,4,4,5,6,7]     │ id[3]: 3→4      │
  │ {5,6}      │ [2,2,2,4,4,6,6,7]     │ id[5]: 5→6      │
  │ {6,7}      │ [2,2,2,4,4,7,7,7]     │ id[5,6]: 6→7    │
  └────────────┴────────────────────────┴─────────────────┘

  Final: id = [2, 2, 2, 4, 4, 7, 7, 7]

  Component 1 (id=2): {0, 1, 2}
  Component 2 (id=4): {3, 4}
  Component 3 (id=7): {5, 6, 7}

  ┌─────────┐   ┌─────┐   ┌─────────┐
  │ 0  1  2 │   │ 3  4│   │ 5  6  7 │
  │  id=2   │   │id=4 │   │  id=7   │
  └─────────┘   └─────┘   └─────────┘

  Result: 3 components
```

---

## Example 9: Evolution from Naive to Optimal

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union-Find Evolution ===")
	fmt.Println()

	n := 8
	// Stage 1: Quick Find
	id := make([]int, n)
	for i := range id { id[i] = i }
	fmt.Println("Quick Find:  id[]  =", id, "  Find=O(1), Union=O(n)")

	// Stage 2: Quick Union
	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	fmt.Println("Quick Union: par[] =", parent, "  Find=O(n), Union=O(n)")

	// Stage 3: Weighted Quick Union
	size := make([]int, n)
	for i := range size { size[i] = 1 }
	fmt.Println("Weighted QU: size[]=", size, "  Find=O(log n), Union=O(log n)")

	// Stage 4: + Path Compression
	fmt.Println("WQU+PathCompress:            Find=O(α(n)), Union=O(α(n))")

	fmt.Println()
	fmt.Println("α(n) ≈ inverse Ackermann ≈ O(1) for all practical n")
}
```

**Textual Figure:**
```
  Union-Find Evolution — Performance Progression:
  ═════════════════════════════════════════════════

  ┌────────────────────────┬────────┬────────┬──────────────┐
  │ Algorithm              │ Find   │ Union  │ n operations │
  ├────────────────────────┼────────┼────────┼──────────────┤
  │ 1. Quick Find          │ O(1)   │ O(n)   │ O(n²)        │
  │ 2. Quick Union         │ O(n)   │ O(n)   │ O(n²)        │
  │ 3. Weighted QU         │O(log n)│O(log n)│ O(n log n)   │
  │ 4. WQU + Path Compress │ O(α(n))│ O(α(n))│ O(n α(n))    │
  └────────────────────────┴────────┴────────┴──────────────┘

  Visual comparison for n=16, n=1000000:

  Quick Find:       ██████████████████████████████████ O(n²)
  Quick Union:      ██████████████████████████████████ O(n²)
  Weighted QU:      █████████████                      O(n log n)
  WQU + Compress:   █                                  O(n α(n)) ≈ O(n)

  Tree shape comparison:

  Quick Union:          Weighted QU:         WQU+Compress:
  (worst case)          (balanced)           (flat)
    ┌─┐                   ┌─┐                 ┌─┐
    │7│                   │0│               ┌─┤0├─┐
    └┬┘                 ┌─┴─┴─┐          ┌──┼──┼──┼──┐
    ┌┴┐               ┌─┴┐ ┌┴─┐       ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴─┐
    │6│               │1 │ │4 │       │1││2││3││4││5 │
    └┬┘               └┬─┘ └┬─┘       └─┘└─┘└─┘└─┘└──┘
    ┌┴┐              ┌┴┐┌┴┐┌┴┐┌┴┐
    │5│              │2││3││5││6│     Height = 1
    └┬┘              └─┘└─┘└─┘└┬┘
    ┌┴┐                       ┌┴┐
    │4│              Height    │7│
    ...              = 3       └─┘
    │0│
    └─┘
    Height = 7
```

---

## Example 10: Full Quick Union with Rank + Path Compression

```go
package main

import "fmt"

type OptimalUF struct {
	parent []int
	rank   []int
}

func NewOptimalUF(n int) *OptimalUF {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &OptimalUF{parent: p, rank: r}
}

func (uf *OptimalUF) Find(x int) int {
	if uf.parent[x] != x {
		uf.parent[x] = uf.Find(uf.parent[x]) // Path compression
	}
	return uf.parent[x]
}

func (uf *OptimalUF) Union(x, y int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return false }
	if uf.rank[rx] < uf.rank[ry] { rx, ry = ry, rx }
	uf.parent[ry] = rx
	if uf.rank[rx] == uf.rank[ry] { uf.rank[rx]++ }
	return true
}

func main() {
	uf := NewOptimalUF(10)
	ops := [][2]int{{0,1},{2,3},{4,5},{6,7},{8,9},{0,2},{4,6},{8,0},{4,8}}
	for _, op := range ops {
		uf.Union(op[0], op[1])
	}
	// Force path compression on all
	for i := 0; i < 10; i++ { uf.Find(i) }
	fmt.Println("parent:", uf.parent)
	fmt.Println("rank:  ", uf.rank)
}
```

**Textual Figure:**
```
  Optimal Union-Find: Union by Rank + Path Compression

  Operations: {0,1},{2,3},{4,5},{6,7},{8,9},{0,2},{4,6},{8,0},{4,8}

  After pair unions:          After group unions:
  ┌───┐┌───┐┌───┐┌───┐┌───┐      ┌───┐         ┌───┐
  │ 0 ││ 2 ││ 4 ││ 6 ││ 8 │      │ 0 │ (r=2)   │ 4 │ (r=2)
  └─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘      └─┬─┘         └─┬─┘
  ┌─┴┐ ┌─┴┐ ┌─┴┐ ┌─┴┐ ┌─┴┐      ┌──┴──┐     ┌──┴──┐
  │ 1│ │ 3│ │ 5│ │ 7│ │ 9│      │  2  │     │  6  │
  └──┘ └──┘ └──┘ └──┘ └──┘    ┌─┴┐ ┌─┴┐  ┌─┴┐ ┌─┴┐
                               │ 1│ │ 3│  │ 5│ │ 7│
  After Union(8,0):            └──┘ └──┘  └──┘ └──┘
      ┌───┐
      │ 0 │ (r=2)                ┌───┐
      └─┬─┘                     │ 8 │ (r=1)
    ┌──┬┴──┐                    └─┬─┘
  ┌─┴┐│ ┌─┴┐                   ┌─┴─┐
  │ 1││ │ 2 │                  │ 9 │
  └──┘│ └─┬─┘                  └───┘
    ┌─┴┐ ┌┴─┐
    │ 8│ │ 3│
    └┬─┘ └──┘
    ┌┴─┐
    │ 9│
    └──┘

  After Union(4,8) and path compression (Find all):
              ┌───┐
              │ 0 │  (root, rank=3)
              └─┬─┘
    ┌──┬──┬──┬─┬┴┬──┬──┬──┐
  ┌─┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴┐┌┴─┐┌┴─┐
  │1││2││3││4││5││6││7││8 ││9 │
  └─┘└─┘└─┘└─┘└─┘└─┘└─┘└──┘└──┘

  All 10 nodes point directly to root 0!
  parent = [0,0,0,0,0,0,0,0,0,0]
  rank   = [3,0,1,0,2,0,1,0,1,0]

  Note: rank stays at 3 even though actual height = 1
  (rank is never decreased — it's an upper bound)
```

---

## Key Takeaways

1. Quick Find: O(1) find, O(n) union — fast lookup, slow merge
2. Quick Union: O(n) find, O(n) union — tree-based, but can degenerate
3. Weighted Quick Union: O(log n) both — keeps trees balanced by size
4. Path compression flattens trees during find → near O(1) amortized
5. Rank + path compression together → O(α(n)) per operation, theoretically optimal

> **Next up:** Union by Size →
