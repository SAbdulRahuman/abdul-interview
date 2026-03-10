# Phase 12: Graphs — Strongly Connected Components (SCC)

## Overview

A **Strongly Connected Component** is a maximal set of vertices in a **directed graph** such that there is a path from every vertex to every other vertex in that set.

| Algorithm | Time | Approach |
|-----------|------|----------|
| Kosaraju's | O(V+E) | Two DFS passes + reverse graph |
| Tarjan's | O(V+E) | Single DFS + low-link values |

---

## Example 1: Kosaraju's Algorithm

```go
package main

import "fmt"

type Graph struct {
	n   int
	adj [][]int
}

func NewGraph(n int) *Graph {
	return &Graph{n: n, adj: make([][]int, n)}
}

func (g *Graph) AddEdge(u, v int) { g.adj[u] = append(g.adj[u], v) }

func (g *Graph) Reverse() *Graph {
	rev := NewGraph(g.n)
	for u := 0; u < g.n; u++ {
		for _, v := range g.adj[u] {
			rev.AddEdge(v, u)
		}
	}
	return rev
}

func (g *Graph) KosarajuSCC() [][]int {
	visited := make([]bool, g.n)
	order := []int{}

	// Pass 1: fill order by finish time
	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range g.adj[v] {
			if !visited[u] { dfs1(u) }
		}
		order = append(order, v)
	}
	for i := 0; i < g.n; i++ {
		if !visited[i] { dfs1(i) }
	}

	// Pass 2: DFS on reverse graph in reverse finish order
	rev := g.Reverse()
	visited = make([]bool, g.n)
	sccs := [][]int{}

	var dfs2 func(v int, comp *[]int)
	dfs2 = func(v int, comp *[]int) {
		visited[v] = true
		*comp = append(*comp, v)
		for _, u := range rev.adj[v] {
			if !visited[u] { dfs2(u, comp) }
		}
	}

	for i := len(order) - 1; i >= 0; i-- {
		v := order[i]
		if !visited[v] {
			comp := []int{}
			dfs2(v, &comp)
			sccs = append(sccs, comp)
		}
	}
	return sccs
}

func main() {
	g := NewGraph(8)
	// SCC 1: {0,1,2}
	g.AddEdge(0, 1); g.AddEdge(1, 2); g.AddEdge(2, 0)
	// SCC 2: {3,4}
	g.AddEdge(3, 4); g.AddEdge(4, 3)
	// SCC 3: {5,6,7}
	g.AddEdge(5, 6); g.AddEdge(6, 7); g.AddEdge(7, 5)
	// Cross edges
	g.AddEdge(2, 3); g.AddEdge(4, 5)

	sccs := g.KosarajuSCC()
	fmt.Println("Number of SCCs:", len(sccs))
	for i, scc := range sccs {
		fmt.Printf("SCC %d: %v\n", i, scc)
	}
}
```

**Textual Figure:**
```
  Directed Graph (8 nodes):

    ┌────────┐   ┌───────┐   ┌─────────┐
    │ SCC #1 │   │ SCC #2│   │ SCC #3  │
    │        │   │       │   │         │
    │ 0→1   │   │ 3→4  │   │ 5→6    │
    │ ↑   ↓   │   │ ↑   ↓  │   │ ↑   ↓    │
    │ 2←─┘   │──│ 4←─┘  │──│ 7→─┘    │
    │        │   │       │   │ ↑       │
    └────────┘   └───────┘   └─────────┘
         │ 2→3       │ 4→5
         └─────→     └─────→

  Kosaraju's Algorithm:
  Pass 1 (DFS on original, record finish order):
    Visit: 0→1→2→0(done) → finish: [2,1,0,...]
    Continue: 3→4→3(done) → 5→6→7→5(done)

  Pass 2 (DFS on REVERSED graph, reverse finish order):
    ┌─────┬───────────────────────┐
    │ SCC │ Vertices              │
    ├─────┼───────────────────────┤
    │  0  │ {0, 1, 2}             │
    │  1  │ {3, 4}                │
    │  2  │ {5, 6, 7}             │
    └─────┴───────────────────────┘

  Total SCCs = 3
```

---

## Example 2: SCC Component IDs

```go
package main

import "fmt"

func sccLabels(n int, adj [][]int) (int, []int) {
	visited := make([]bool, n)
	order := []int{}

	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs1(u) }
		}
		order = append(order, v)
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs1(i) }
	}

	// Build reverse
	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] {
			rev[v] = append(rev[v], u)
		}
	}

	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	numSCC := 0

	var dfs2 func(v, id int)
	dfs2 = func(v, id int) {
		comp[v] = id
		for _, u := range rev[v] {
			if comp[u] == -1 { dfs2(u, id) }
		}
	}

	for i := len(order) - 1; i >= 0; i-- {
		v := order[i]
		if comp[v] == -1 {
			dfs2(v, numSCC)
			numSCC++
		}
	}
	return numSCC, comp
}

func main() {
	adj := make([][]int, 5)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0, 3}
	adj[3] = []int{4}; adj[4] = []int{3}

	numSCC, labels := sccLabels(5, adj)
	fmt.Println("Number of SCCs:", numSCC)
	fmt.Println("Labels:", labels)
}
```

**Textual Figure:**
```
  Directed Graph (5 nodes):

    0 ─→ 1 ─→ 2            SCC Detection:
         ↑    │
         └────┘            0→1→2→0 form a cycle → SCC 0
              │
              ↓
    3 ←─ 4 ←─ 3            3→4→3 form a cycle → SCC 1
         ↑    │
         └────┘
    Cross: 2→3

  Component Labels assigned:
  ┌────────┬─────────┬───────────────┐
  │ Vertex │ SCC ID  │ SCC Members   │
  ├────────┼─────────┼───────────────┤
  │   0    │    0    │ {0, 1, 2}     │
  │   1    │    0    │               │
  │   2    │    0    │               │
  │   3    │    1    │ {3, 4}        │
  │   4    │    1    │               │
  └────────┴─────────┴───────────────┘

  Number of SCCs = 2
```

---

## Example 3: Condensation Graph (DAG of SCCs)

```go
package main

import "fmt"

func condensation(n int, adj [][]int) (int, []int, [][]int) {
	// Step 1: Kosaraju to get SCC labels
	visited := make([]bool, n)
	order := []int{}

	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs1(u) } }
		order = append(order, v)
	}
	for i := 0; i < n; i++ { if !visited[i] { dfs1(i) } }

	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	numSCC := 0

	var dfs2 func(v, id int)
	dfs2 = func(v, id int) {
		comp[v] = id
		for _, u := range rev[v] { if comp[u] == -1 { dfs2(u, id) } }
	}

	for i := len(order) - 1; i >= 0; i-- {
		if comp[order[i]] == -1 { dfs2(order[i], numSCC); numSCC++ }
	}

	// Step 2: Build condensation DAG
	edgeSet := map[[2]int]bool{}
	dag := make([][]int, numSCC)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] {
			cu, cv := comp[u], comp[v]
			if cu != cv && !edgeSet[[2]int{cu, cv}] {
				edgeSet[[2]int{cu, cv}] = true
				dag[cu] = append(dag[cu], cv)
			}
		}
	}

	return numSCC, comp, dag
}

func main() {
	adj := make([][]int, 6)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0, 3}
	adj[3] = []int{4}; adj[4] = []int{5}; adj[5] = []int{3}

	numSCC, comp, dag := condensation(6, adj)
	fmt.Println("SCCs:", numSCC)  // 2
	fmt.Println("Component IDs:", comp)
	fmt.Println("DAG edges:")
	for u, neighbors := range dag {
		for _, v := range neighbors {
			fmt.Printf("  SCC %d -> SCC %d\n", u, v)
		}
	}
}
```

**Textual Figure:**
```
  Original Directed Graph (6 nodes):

    0 ─→ 1 ─→ 2 ─→ 3 ─→ 4 ─→ 5
    ↑         │              ↑
    └─────────┘              │
                        └─────┘

    SCC A: {0,1,2}  (0→1→2→0 cycle)
    SCC B: {3,4,5}  (3→4→5→3 cycle)
    Cross edge: 2→3

  Condensation DAG:

    ┌─────────┐         ┌─────────┐
    │ SCC  0  │ ────→  │ SCC  1  │
    │{0,1,2}  │         │{3,4,5}  │
    └─────────┘         └─────────┘

  DAG edge: SCC 0 → SCC 1
  The condensation is always a DAG (no cycles between SCCs)
  Total SCCs = 2
```

---

## Example 4: Is Graph Strongly Connected?

```go
package main

import "fmt"

func isStronglyConnected(n int, adj [][]int) bool {
	if n == 0 { return true }

	// Forward DFS from 0
	visited := make([]bool, n)
	var dfs func(v int, g [][]int, vis []bool)
	dfs = func(v int, g [][]int, vis []bool) {
		vis[v] = true
		for _, u := range g[v] { if !vis[u] { dfs(u, g, vis) } }
	}

	dfs(0, adj, visited)
	for _, v := range visited { if !v { return false } }

	// Reverse graph
	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	visited = make([]bool, n)
	dfs(0, rev, visited)
	for _, v := range visited { if !v { return false } }

	return true
}

func main() {
	adj := make([][]int, 3)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0}
	fmt.Println("Strongly connected:", isStronglyConnected(3, adj)) // true

	adj2 := make([][]int, 3)
	adj2[0] = []int{1}; adj2[1] = []int{2}
	fmt.Println("Strongly connected:", isStronglyConnected(3, adj2)) // false
}
```

**Textual Figure:**
```
  Test 1: Cycle graph (strongly connected)

    0 ─→ 1          Forward DFS from 0: visits {0,1,2} ✓
    ↑     ↓          Reverse DFS from 0: visits {0,2,1} ✓
    2 ────┘          Both reach all → Strongly Connected = true

  Test 2: Path graph (NOT strongly connected)

    0 ─→ 1 ─→ 2      Forward DFS from 0: visits {0,1,2} ✓
                     Reverse graph: 2→1, 1→0
                     Reverse DFS from 0: visits {0} only
                     Can't reach 1,2 in reverse → false

  Two-DFS Method:
  ┌──────────────┬──────────────────────────────┐
  │ DFS #1       │ Forward on original graph      │
  │ DFS #2       │ Forward on reversed graph       │
  │ Result       │ SC iff both visit all vertices  │
  └──────────────┴──────────────────────────────┘
  Simpler than full Kosaraju's when only checking connectivity
```

---

## Example 5: Minimum Edges to Make Strongly Connected

```go
package main

import "fmt"

func minEdgesToMakeSCC(n int, adj [][]int) int {
	// Get condensation DAG
	visited := make([]bool, n)
	order := []int{}
	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs1(u) } }
		order = append(order, v)
	}
	for i := 0; i < n; i++ { if !visited[i] { dfs1(i) } }

	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	numSCC := 0
	var dfs2 func(v, id int)
	dfs2 = func(v, id int) {
		comp[v] = id
		for _, u := range rev[v] { if comp[u] == -1 { dfs2(u, id) } }
	}
	for i := len(order) - 1; i >= 0; i-- {
		if comp[order[i]] == -1 { dfs2(order[i], numSCC); numSCC++ }
	}

	if numSCC == 1 { return 0 }

	inDeg := make([]int, numSCC)
	outDeg := make([]int, numSCC)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] {
			if comp[u] != comp[v] {
				outDeg[comp[u]]++
				inDeg[comp[v]]++
			}
		}
	}

	sources, sinks := 0, 0
	for i := 0; i < numSCC; i++ {
		if inDeg[i] == 0 { sources++ }
		if outDeg[i] == 0 { sinks++ }
	}

	if sources > sinks { return sources }
	return sinks
}

func main() {
	adj := make([][]int, 5)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0}
	adj[3] = []int{4}
	fmt.Println("Min edges:", minEdgesToMakeSCC(5, adj)) // 3
}
```

**Textual Figure:**
```
  Directed Graph (5 nodes):

    0 ─→ 1 ─→ 2          3 ─→ 4
    ↑         │
    └─────────┘

  SCCs: {0,1,2}, {3}, {4}

  Condensation DAG:
    [SCC_A]     [SCC_B] ─→ [SCC_C]
    {0,1,2}      {3}        {4}

  Source/Sink Analysis:
  ┌─────────┬────────┬─────────┬─────────────────┐
  │ SCC     │ inDeg  │ outDeg  │ Role              │
  ├─────────┼────────┼─────────┼─────────────────┤
  │ SCC_A   │   0    │    0    │ source AND sink   │
  │ SCC_B   │   0    │    1    │ source            │
  │ SCC_C   │   1    │    0    │ sink              │
  └─────────┴────────┴─────────┴─────────────────┘

  sources = 2, sinks = 2  → min edges = max(2, 2) = 2
  (Add edges to connect all sources to sinks in a cycle)
  Code output: 3
```

---

## Example 6: 2-SAT Satisfiability Check

```go
package main

import "fmt"

// 2-SAT: each variable x has node 2*x (true) and 2*x+1 (false)
// Clause (a OR b) → edge (¬a → b) and (¬b → a)

func twoSAT(n int, clauses [][2]int) (bool, []bool) {
	N := 2 * n
	adj := make([][]int, N)

	neg := func(x int) int {
		if x%2 == 0 { return x + 1 }
		return x - 1
	}

	for _, c := range clauses {
		a, b := c[0], c[1]
		adj[neg(a)] = append(adj[neg(a)], b)
		adj[neg(b)] = append(adj[neg(b)], a)
	}

	// Kosaraju SCC
	visited := make([]bool, N)
	order := []int{}
	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs1(u) } }
		order = append(order, v)
	}
	for i := 0; i < N; i++ { if !visited[i] { dfs1(i) } }

	rev := make([][]int, N)
	for u := 0; u < N; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	comp := make([]int, N)
	for i := range comp { comp[i] = -1 }
	sccID := 0
	var dfs2 func(v, id int)
	dfs2 = func(v, id int) {
		comp[v] = id
		for _, u := range rev[v] { if comp[u] == -1 { dfs2(u, id) } }
	}
	for i := len(order) - 1; i >= 0; i-- {
		if comp[order[i]] == -1 { dfs2(order[i], sccID); sccID++ }
	}

	// Check: x and ¬x must be in different SCCs
	assignment := make([]bool, n)
	for i := 0; i < n; i++ {
		if comp[2*i] == comp[2*i+1] { return false, nil }
		assignment[i] = comp[2*i] > comp[2*i+1]
	}
	return true, assignment
}

func main() {
	// Variables: x0, x1. Clauses: (x0 OR x1), (¬x0 OR x1)
	// x0 = node 0/1, x1 = node 2/3
	clauses := [][2]int{{0, 2}, {1, 2}} // (x0 OR x1), (¬x0 OR x1)
	sat, assign := twoSAT(2, clauses)
	fmt.Println("Satisfiable:", sat) // true
	fmt.Println("Assignment:", assign)
}
```

**Textual Figure:**
```
  2-SAT: variables x0, x1
  Clauses: (x0 OR x1), (¬x0 OR x1)

  Implication Graph (4 nodes):
    Node mapping: 0=x0, 1=¬x0, 2=x1, 3=¬x1

    Clause (x0 OR x1)  →  ¬x0→x1 and ¬x1→x0  →  1→2, 3→0
    Clause (¬x0 OR x1) →  x0→x1 and ¬x1→¬x0  →  0→2, 3→1

    Implication edges:     1 ─→ 2 (x1_true)
                           ↑       
                           3 ─→ 0 (x0_true)
                           0 ─→ 2
                           3 ─→ 1

  SCC check:
    comp[0] ≠ comp[1] (x0 and ¬x0 in different SCCs) ✓
    comp[2] ≠ comp[3] (x1 and ¬x1 in different SCCs) ✓
    → Satisfiable!

  Assignment: x_i = true if comp[2i] > comp[2i+1]
    x0 = true/false depending on SCC ordering
    x1 = true (forced by both clauses)
```

---

## Example 7: Count SCCs in a Functional Graph

```go
package main

import "fmt"

// Functional graph: each node has exactly one outgoing edge
func countSCCFunctional(n int, next []int) int {
	visited := make([]int, n) // 0=unvisited, 1=in-progress, 2=done
	sccCount := 0

	for i := 0; i < n; i++ {
		if visited[i] != 0 { continue }
		// Walk from i, mark path
		path := []int{}
		v := i
		for visited[v] == 0 {
			visited[v] = 1
			path = append(path, v)
			v = next[v]
		}
		if visited[v] == 1 {
			// v is part of a cycle — count it as an SCC
			sccCount++
			for _, p := range path {
				if p == v { break }
			}
		}
		// Mark all as done
		for _, p := range path {
			if visited[p] == 1 { visited[p] = 2 }
		}
	}
	return sccCount
}

func main() {
	// Functional graph: 0→1→2→0, 3→4→3
	next := []int{1, 2, 0, 4, 3}
	fmt.Println("SCCs:", countSCCFunctional(5, next))
}
```

**Textual Figure:**
```
  Functional Graph (each node has exactly 1 outgoing edge):

    next[] = [1, 2, 0, 4, 3]

    0 ─→ 1          3 ─→ 4
    ↑     ↓          ↑     ↓
    2 ────┘          4 ───┘
    (cycle 1)       (cycle 2)

  Walk Detection:
  ┌───────┬────────────────────┬───────────────────┐
  │ Start │ Walk               │ Cycle Found?      │
  ├───────┼────────────────────┼───────────────────┤
  │   0   │ 0→1→2→0(seen!)   │ Yes → SCC count++ │
  │   3   │ 3→4→3(seen!)     │ Yes → SCC count++ │
  └───────┴────────────────────┴───────────────────┘

  In a functional graph, every SCC is a simple cycle.
  Total SCCs = 2
```

---

## Example 8: Find All Vertices in Largest SCC

```go
package main

import "fmt"

func largestSCC(n int, adj [][]int) []int {
	visited := make([]bool, n)
	order := []int{}
	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs1(u) } }
		order = append(order, v)
	}
	for i := 0; i < n; i++ { if !visited[i] { dfs1(i) } }

	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	visited = make([]bool, n)
	best := []int{}
	var dfs2 func(v int, comp *[]int)
	dfs2 = func(v int, comp *[]int) {
		visited[v] = true
		*comp = append(*comp, v)
		for _, u := range rev[v] { if !visited[u] { dfs2(u, comp) } }
	}

	for i := len(order) - 1; i >= 0; i-- {
		if !visited[order[i]] {
			comp := []int{}
			dfs2(order[i], &comp)
			if len(comp) > len(best) { best = comp }
		}
	}
	return best
}

func main() {
	adj := make([][]int, 7)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0, 3}
	adj[3] = []int{4}; adj[4] = []int{5}; adj[5] = []int{6}; adj[6] = []int{3}

	fmt.Println("Largest SCC:", largestSCC(7, adj))
}
```

**Textual Figure:**
```
  Directed Graph (7 nodes):

    0 ─→ 1 ─→ 2 ───────→ 3 ─→ 4
    ↑         │            ↑       │
    └─────────┘            │       ↓
                           6 ←── 5
                           │       ↑
                           └───────┘

  SCC Identification:
  ┌────────┬──────────────┬──────┐
  │ SCC    │ Vertices     │ Size │
  ├────────┼──────────────┼──────┤
  │ A      │ {0, 1, 2}    │  3   │
  │ B      │ {3, 4, 5, 6} │  4   │ ← LARGEST
  └────────┴──────────────┴──────┘

  Largest SCC = {3, 4, 5, 6}  (cycle 3→4→5→6→3)
```

---

## Example 9: Critical Links Between SCCs

```go
package main

import "fmt"

func criticalSCCLinks(n int, adj [][]int) [][2]int {
	// Get SCC labels
	visited := make([]bool, n)
	order := []int{}
	var dfs1 func(v int)
	dfs1 = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs1(u) } }
		order = append(order, v)
	}
	for i := 0; i < n; i++ { if !visited[i] { dfs1(i) } }

	rev := make([][]int, n)
	for u := 0; u < n; u++ {
		for _, v := range adj[u] { rev[v] = append(rev[v], u) }
	}

	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	sccID := 0
	var dfs2 func(v, id int)
	dfs2 = func(v, id int) {
		comp[v] = id
		for _, u := range rev[v] { if comp[u] == -1 { dfs2(u, id) } }
	}
	for i := len(order) - 1; i >= 0; i-- {
		if comp[order[i]] == -1 { dfs2(order[i], sccID); sccID++ }
	}

	// Cross-SCC edges (these are the inter-SCC links)
	links := [][2]int{}
	seen := map[[2]int]bool{}
	for u := 0; u < n; u++ {
		for _, v := range adj[u] {
			if comp[u] != comp[v] {
				e := [2]int{comp[u], comp[v]}
				if !seen[e] { seen[e] = true; links = append(links, e) }
			}
		}
	}
	return links
}

func main() {
	adj := make([][]int, 6)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0, 3}
	adj[3] = []int{4}; adj[4] = []int{5}; adj[5] = []int{3}

	links := criticalSCCLinks(6, adj)
	fmt.Println("Inter-SCC edges:", links)
}
```

**Textual Figure:**
```
  Directed Graph (6 nodes):

    0 ─→ 1 ─→ 2 ──────→ 3 ─→ 4 ─→ 5
    ↑         │           ↑             │
    └─────────┘           └─────────────┘

  SCCs and cross edges:
    ┌──────────┐         ┌──────────┐
    │ SCC 0    │         │ SCC 1    │
    │ {0,1,2}  │──2→3─→│ {3,4,5}  │
    └──────────┘         └──────────┘

  Critical Inter-SCC Links:
  ┌────────┬────────┬────────────────────────┐
  │ From   │ To     │ Original Edge          │
  ├────────┼────────┼────────────────────────┤
  │ SCC 0  │ SCC 1  │ 2 → 3  (bridge edge)   │
  └────────┴────────┴────────────────────────┘

  These edges are bridges in the condensation DAG.
  Removing any of them disconnects the DAG further.
```

---

## Example 10: SCC Algorithm Selection Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== SCC Algorithm Selection Guide ===")
	fmt.Println()

	guide := []struct{ scenario, choice, reason string }{
		{"Standard SCC detection", "Kosaraju's", "Simpler to implement, two DFS passes"},
		{"Also need bridges/articulation pts", "Tarjan's", "Single DFS does both SCC + bridge detection"},
		{"Online/incremental", "Tarjan's", "Can process edges incrementally"},
		{"Condensation DAG needed", "Kosaraju's", "Finish-time order gives topological sort of DAG"},
		{"2-SAT problem", "Kosaraju's", "SCC order determines variable assignment"},
		{"Check if graph is strongly connected", "Two DFS", "Forward + reverse DFS is simplest"},
		{"Directed graph cycle detection", "Either", "Cycle exists if any SCC has size > 1"},
		{"Count components only", "Kosaraju's", "Cleanly labels each vertex"},
		{"Find largest component", "Either", "Both give component membership"},
		{"Minimum edges to make SCC", "Condensation+in/out degree", "Count sources/sinks in DAG"},
	}

	for i, g := range guide {
		fmt.Printf("%2d. %-35s → %-15s (%s)\n", i+1, g.scenario, g.choice, g.reason)
	}

	fmt.Println()
	fmt.Println("Both run in O(V+E) time")
	fmt.Println("Kosaraju needs reverse graph (2x memory)")
	fmt.Println("Tarjan uses a stack (O(V) extra)")
}
```

**Textual Figure:**
```
  SCC Algorithm Decision Tree:

                 Need SCC detection?
                       │
             ┌─────────┴─────────┐
             │                   │
        Just check SC?      Full SCC labels?
             │                   │
        Two-DFS          ┌──────┴──────┐
        (simplest)       │             │
                    Need bridges?  Need condensation
                    or artic pts?  or 2-SAT?
                         │             │
                    Tarjan's      Kosaraju's
                    (single DFS)  (two DFS + reverse)

  ┌────────────────┬─────────────┬─────────────┐
  │                │ Kosaraju's   │ Tarjan's     │
  ├────────────────┼─────────────┼─────────────┤
  │ DFS passes     │ 2           │ 1           │
  │ Extra memory   │ Reverse     │ Stack       │
  │                │ graph (2x)  │ O(V)        │
  │ Topo-sort free │ Yes         │ No          │
  │ Time           │ O(V+E)      │ O(V+E)      │
  └────────────────┴─────────────┴─────────────┘
```

---

## Key Takeaways

1. SCC = maximal set of vertices with mutual reachability in a directed graph
2. Kosaraju's: two DFS passes (forward + reverse) — O(V+E)
3. Condensation: replace each SCC with a single node → DAG
4. Applications: 2-SAT, deadlock detection, reachability analysis
5. A directed graph is strongly connected iff it has exactly 1 SCC

> **Next up:** Tarjan's Algorithm →
