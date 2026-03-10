# Phase 12: Graphs — Tarjan's Algorithm

## Overview

**Tarjan's algorithm** performs a single DFS to find:
1. **Strongly Connected Components (SCCs)** in directed graphs
2. **Bridges** (cut edges) in undirected graphs
3. **Articulation Points** (cut vertices) in undirected graphs

It uses **discovery time** (`disc`) and **low-link values** (`low`) to track the earliest reachable ancestor from each subtree.

| Variant | Graph Type | What it Finds |
|---------|-----------|---------------|
| Tarjan's SCC | Directed | Strongly connected components |
| Tarjan's Bridge | Undirected | Bridge edges |
| Tarjan's AP | Undirected | Articulation points |

**Time Complexity:** O(V + E)

---

## Example 1: Tarjan's SCC (Directed Graph)

```go
package main

import "fmt"

func tarjanSCC(n int, adj [][]int) [][]int {
	disc := make([]int, n)
	low := make([]int, n)
	onStack := make([]bool, n)
	visited := make([]bool, n)
	stack := []int{}
	timer := 0
	sccs := [][]int{}

	var dfs func(v int)
	dfs = func(v int) {
		disc[v] = timer; low[v] = timer; timer++
		visited[v] = true
		stack = append(stack, v)
		onStack[v] = true

		for _, u := range adj[v] {
			if !visited[u] {
				dfs(u)
				if low[u] < low[v] { low[v] = low[u] }
			} else if onStack[u] {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}

		// Root of SCC
		if low[v] == disc[v] {
			scc := []int{}
			for {
				u := stack[len(stack)-1]
				stack = stack[:len(stack)-1]
				onStack[u] = false
				scc = append(scc, u)
				if u == v { break }
			}
			sccs = append(sccs, scc)
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i) }
	}
	return sccs
}

func main() {
	adj := make([][]int, 8)
	adj[0] = []int{1}; adj[1] = []int{2}; adj[2] = []int{0, 3}
	adj[3] = []int{4}; adj[4] = []int{5}; adj[5] = []int{3}
	adj[6] = []int{7}; adj[7] = []int{6}

	sccs := tarjanSCC(8, adj)
	fmt.Println("Number of SCCs:", len(sccs))
	for i, scc := range sccs {
		fmt.Printf("SCC %d: %v\n", i, scc)
	}
}
```

**Textual Figure:**

```
Directed graph (8 nodes):

    0 → 1 → 2 → 3 → 4 → 5
    ↑       │       ↑
    └───────┘       └── 5

    6 ⇄ 7

DFS with disc/low/stack:
┌──────┬──────┬─────┬────────┬─────────────────────┐
│ Node │ disc │ low │ stack  │ SCC pop?            │
├──────┼──────┼─────┼────────┼─────────────────────┤
│  0   │  0   │  0  │ [0]    │                     │
│  1   │  1   │  0  │ [0,1]  │                     │
│  2   │  2   │  0  │ [0,1,2]│ back edge 2→0       │
│      │      │     │        │ low[2]→0, low[1]←0  │
│  0   │      │  0  │        │ low==disc → pop SCC  │
│      │      │     │        │ SCC: {2, 1, 0}      │
│  3   │  3   │  3  │ [3]    │                     │
│  4   │  4   │  3  │ [3,4]  │                     │
│  5   │  5   │  3  │ [3,4,5]│ 5→3, low[5]←3      │
│  3   │      │  3  │        │ low==disc → pop SCC  │
│      │      │     │        │ SCC: {5, 4, 3}      │
│  6   │  6   │  6  │ [6]    │                     │
│  7   │  7   │  6  │ [6,7]  │ 7→6, low[7]←6      │
│  6   │      │  6  │        │ low==disc → pop SCC  │
│      │      │     │        │ SCC: {7, 6}         │
└──────┴──────┴─────┴────────┴─────────────────────┘

Result: 3 SCCs
  SCC 0: {2,1,0}    ┌───────┐
  SCC 1: {5,4,3}    │ 0→1→2│ → ┌───────┐
  SCC 2: {7,6}      │  ←──┘ │    │ 3→4→5│    ┌────┐
                    └───────┘    │  ←──┘ │    │ 6⇄7│
                                  └───────┘    └────┘
```

---

## Example 2: Find Bridges (Critical Connections — LeetCode 1192)

```go
package main

import "fmt"

func criticalConnections(n int, connections [][]int) [][]int {
	adj := make([][]int, n)
	for _, c := range connections {
		adj[c[0]] = append(adj[c[0]], c[1])
		adj[c[1]] = append(adj[c[1]], c[0])
	}

	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	bridges := [][]int{}

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++

		for _, u := range adj[v] {
			if u == parent { continue }
			if disc[u] == -1 {
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				// Bridge: no back edge from u's subtree reaches v or above
				if low[u] > disc[v] {
					bridges = append(bridges, []int{v, u})
				}
			} else {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}
	return bridges
}

func main() {
	n := 4
	connections := [][]int{{0,1},{1,2},{2,0},{1,3}}
	bridges := criticalConnections(n, connections)
	fmt.Println("Bridges:", bridges) // [[1 3]]
}
```

**Textual Figure:**

```
Undirected graph: n=4, connections {0-1, 1-2, 2-0, 1-3}

    0 ── 1 ── 3
     \  /
      2

DFS with disc/low (start at 0, parent=-1):
┌──────┬──────┬─────┬───────────────────────────┐
│ Node │ disc │ low │ Action                    │
├──────┼──────┼─────┼───────────────────────────┤
│  0   │  0   │  0  │                           │
│  1   │  1   │  1  │                           │
│  2   │  2   │  0  │ back edge to 0: low←0    │
│  1   │      │  0  │ low[2]=0 < low[1] → 0    │
│  3   │  3   │  3  │ leaf, no back edges      │
└──────┴──────┴─────┴───────────────────────────┘

Bridge check: low[u] > disc[v]?
  Edge (0,1): low[1]=0, disc[0]=0 → 0 > 0? No
  Edge (1,2): low[2]=0, disc[1]=1 → 0 > 1? No
  Edge (1,3): low[3]=3, disc[1]=1 → 3 > 1? Yes → BRIDGE!

    0 ── 1 ┃┃ 3
     \  /   ↑↑
      2    bridge

Result: [[1, 3]]
```

---

## Example 3: Articulation Points (Cut Vertices)

```go
package main

import "fmt"

func articulationPoints(n int, adj [][]int) []int {
	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	isAP := make([]bool, n)

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++
		children := 0

		for _, u := range adj[v] {
			if disc[u] == -1 {
				children++
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }

				// Case 1: Root with 2+ children
				if parent == -1 && children > 1 { isAP[v] = true }
				// Case 2: Non-root where no back edge from u's subtree goes above v
				if parent != -1 && low[u] >= disc[v] { isAP[v] = true }
			} else if u != parent {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}

	result := []int{}
	for i, ap := range isAP {
		if ap { result = append(result, i) }
	}
	return result
}

func main() {
	adj := make([][]int, 5)
	adj[0] = []int{1, 2}
	adj[1] = []int{0, 2}
	adj[2] = []int{0, 1, 3}
	adj[3] = []int{2, 4}
	adj[4] = []int{3}

	aps := articulationPoints(5, adj)
	fmt.Println("Articulation points:", aps) // [2 3]
}
```

**Textual Figure:**

```
Undirected graph (5 nodes):

    0 ── 1
    │  / │
    │ /  │
    2    │
    │    │
    3 ── 4    (wait, adj: 2 connects to 0,1,3)

Actual structure:
    0 ── 1             adj[0]=[1,2]
     \  /              adj[1]=[0,2]
      2 ─── 3 ─── 4    adj[2]=[0,1,3]
                       adj[3]=[2,4]
                       adj[4]=[3]

DFS disc/low:
┌──────┬──────┬─────┬──────────────────────────────┐
│ Node │ disc │ low │ AP check                       │
├──────┼──────┼─────┼──────────────────────────────┤
│  0   │  0   │  0  │ root, only 1 child → no       │
│  1   │  1   │  0  │ back edge to 0                 │
│  2   │  2   │  0  │ low[3]=2 >= disc[2]=2 → AP ✓  │
│  3   │  3   │  3  │ low[4]=4 >= disc[3]=3 → AP ✓  │
│  4   │  4   │  4  │ leaf                           │
└──────┴──────┴─────┴──────────────────────────────┘

AP verification:
  Remove node 2:   0─1     3─4   → disconnected ✓
  Remove node 3:   0─1─2   4     → disconnected ✓

Result: [2, 3]
```

---

## Example 4: Biconnected Components (Edge Groups)

```go
package main

import "fmt"

func biconnectedComponents(n int, adj [][]int) [][][2]int {
	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	edgeStack := [][2]int{}
	components := [][][2]int{}

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++

		for _, u := range adj[v] {
			if disc[u] == -1 {
				edgeStack = append(edgeStack, [2]int{v, u})
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }

				if low[u] >= disc[v] {
					// Pop edges until {v, u}
					comp := [][2]int{}
					for {
						e := edgeStack[len(edgeStack)-1]
						edgeStack = edgeStack[:len(edgeStack)-1]
						comp = append(comp, e)
						if e[0] == v && e[1] == u { break }
					}
					components = append(components, comp)
				}
			} else if u != parent && disc[u] < disc[v] {
				edgeStack = append(edgeStack, [2]int{v, u})
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}
	return components
}

func main() {
	adj := make([][]int, 5)
	adj[0] = []int{1, 2}; adj[1] = []int{0, 2}; adj[2] = []int{0, 1, 3}
	adj[3] = []int{2, 4}; adj[4] = []int{3}

	comps := biconnectedComponents(5, adj)
	fmt.Println("Biconnected components:")
	for i, comp := range comps {
		fmt.Printf("  Component %d: %v\n", i, comp)
	}
}
```

**Textual Figure:**

```
Undirected graph:
    0 ── 1
     \  /
      2 ─── 3 ─── 4
    adj[0]=[1,2], adj[1]=[0,2], adj[2]=[0,1,3],
    adj[3]=[2,4], adj[4]=[3]

Biconnected component detection:
  Edge stack tracks DFS tree + back edges.
  When low[u] >= disc[v], pop edges to form component.

  DFS from 0:
    Visit 0 → push edge (0,1)
    Visit 1 → push edge (1,2)  (skip 0 = parent)
    Visit 2 → back edge (2,0), push it
              low[2] < disc[2], update low
    Return to 1: low[2]=0 < low[1] → update low[1]=0
    Return to 0: low[1]=0 >= disc[0]=0 → pop component!
      Pop: (2,0), (1,2), (0,1) → Component 0: {0,1,2}
    Continue from 2: visit 3 → push (2,3)
    Visit 3 → visit 4, push (3,4)
    Return to 3: low[4]=4 >= disc[3]=3 → pop!
      Pop: (3,4) → Component 1: {3,4}
    Return to 2: low[3]=3 >= disc[2]=2 → pop!
      Pop: (2,3) → Component 2: {2,3}

  Biconnected components:
  ┌───────────┐  ┌───────┐  ┌───────┐
  │ 0─1─2   │  │ 2─3  │  │ 3─4  │
  └───────────┘  └───────┘  └───────┘
    Comp 0          Comp 1      Comp 2
```

---

## Example 5: Low-Link Values Step-by-Step Trace

```go
package main

import "fmt"

func tarjanTrace(n int, adj [][]int) {
	disc := make([]int, n)
	low := make([]int, n)
	visited := make([]bool, n)
	timer := 0

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++
		visited[v] = true
		fmt.Printf("Visit %d: disc=%d, low=%d\n", v, disc[v], low[v])

		for _, u := range adj[v] {
			if !visited[u] {
				dfs(u, v)
				if low[u] < low[v] {
					fmt.Printf("  Update low[%d]: %d → %d (from child %d)\n", v, low[v], low[u], u)
					low[v] = low[u]
				}
			} else if u != parent {
				if disc[u] < low[v] {
					fmt.Printf("  Update low[%d]: %d → %d (back edge to %d)\n", v, low[v], disc[u], u)
					low[v] = disc[u]
				}
			}
		}
		fmt.Printf("Finish %d: disc=%d, low=%d\n", v, disc[v], low[v])
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i, -1) }
	}

	fmt.Println("\nFinal values:")
	for i := 0; i < n; i++ {
		fmt.Printf("  Node %d: disc=%d, low=%d\n", i, disc[i], low[i])
	}
}

func main() {
	adj := make([][]int, 5)
	adj[0] = []int{1, 3}; adj[1] = []int{0, 2}; adj[2] = []int{1, 3}
	adj[3] = []int{0, 2, 4}; adj[4] = []int{3}
	tarjanTrace(5, adj)
}
```

**Textual Figure:**

```
Undirected graph:
    adj[0]=[1,3], adj[1]=[0,2], adj[2]=[1,3],
    adj[3]=[0,2,4], adj[4]=[3]

    0 ── 1
    │    │
    │    2
    │    │
    3 ──┘
    │
    4

DFS trace step-by-step:
  Visit 0: disc=0, low=0
  ├─ Visit 1: disc=1, low=1
  │  ├─ Visit 2: disc=2, low=2
  │  │  └─ neighbor 3: unvisited → recurse
  │  │     Visit 3: disc=3, low=3
  │  │     ├─ neighbor 0: visited, back edge
  │  │     │  Update low[3]: 3 → 0
  │  │     ├─ neighbor 2: parent, skip
  │  │     └─ Visit 4: disc=4, low=4
  │  │        Finish 4: disc=4, low=4
  │  │     Update low[3]: 0 (no change, 4>0)
  │  │     Finish 3: disc=3, low=0
  │  │  Update low[2]: 2 → 0 (from child 3)
  │  │  Finish 2: disc=2, low=0
  │  Update low[1]: 1 → 0 (from child 2)
  │  Finish 1: disc=1, low=0
  Finish 0: disc=0, low=0

Final values:
┌──────┬──────┬─────┐
│ Node │ disc │ low │
├──────┼──────┼─────┤
│  0   │  0   │  0  │
│  1   │  1   │  0  │
│  2   │  2   │  0  │
│  3   │  3   │  0  │
│  4   │  4   │  4  │
└──────┴──────┴─────┘
  Node 4: low=4≠low of others → bridge edge (3,4)
  Nodes 0-3: all have low=0 → same 2-edge-connected block
```

---

## Example 6: Count Bridges in a Graph

```go
package main

import "fmt"

func countBridges(n int, edges [][2]int) int {
	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	count := 0

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++
		for _, u := range adj[v] {
			if disc[u] == -1 {
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				if low[u] > disc[v] { count++ }
			} else if u != parent {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}
	return count
}

func main() {
	edges := [][2]int{{0,1},{1,2},{2,0},{1,3},{3,4},{4,5},{5,3}}
	fmt.Println("Number of bridges:", countBridges(6, edges)) // 1 (edge 1-3)
}
```

**Textual Figure:**

```
Undirected graph (6 nodes):
  edges: {0-1, 1-2, 2-0, 1-3, 3-4, 4-5, 5-3}

    0 ── 1 ─── 3 ── 4
     \  /       │    │
      2         └─ 5 ─┘

DFS disc/low:
┌──────┬──────┬─────┬────────────────────────┐
│ Node │ disc │ low │ Notes                  │
├──────┼──────┼─────┼────────────────────────┤
│  0   │  0   │  0  │                        │
│  1   │  1   │  0  │ cycle via 2→0         │
│  2   │  2   │  0  │ back edge to 0        │
│  3   │  3   │  3  │ cycle via 4→5→3      │
│  4   │  4   │  3  │ 5 has back edge to 3  │
│  5   │  5   │  3  │ back edge to 3        │
└──────┴──────┴─────┴────────────────────────┘

Bridge check (low[u] > disc[v]):
  (1,3): low[3]=3 > disc[1]=1 → BRIDGE ✓
  All other edges: low[u] ≤ disc[v] → not bridges

    ┌───────┐       ┌─────────┐
    │ 0─1─2│─bridge─│ 3─4─5 │
    └───────┘       └─────────┘

Result: 1 bridge (edge 1-3)
```

---

## Example 7: Handle Parallel Edges in Bridge Detection

```go
package main

import "fmt"

func bridgesWithParallelEdges(n int, edges [][2]int) [][2]int {
	type Edge struct{ to, idx int }
	adj := make([][]Edge, n)
	for i, e := range edges {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], i})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], i})
	}

	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	bridges := [][2]int{}

	var dfs func(v, parentEdge int)
	dfs = func(v, parentEdge int) {
		disc[v] = timer; low[v] = timer; timer++
		for _, e := range adj[v] {
			if e.idx == parentEdge { continue } // skip same edge, not same vertex
			if disc[e.to] == -1 {
				dfs(e.to, e.idx)
				if low[e.to] < low[v] { low[v] = low[e.to] }
				if low[e.to] > disc[v] {
					bridges = append(bridges, [2]int{v, e.to})
				}
			} else {
				if disc[e.to] < low[v] { low[v] = disc[e.to] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}
	return bridges
}

func main() {
	// Parallel edge between 0-1: NOT a bridge
	edges := [][2]int{{0,1},{0,1},{1,2}}
	bridges := bridgesWithParallelEdges(3, edges)
	fmt.Println("Bridges:", bridges) // [[1 2]]
}
```

**Textual Figure:**

```
Graph with parallel edges:
  edges: {0-1, 0-1, 1-2} (edge indices: 0, 1, 2)

    0 ═══ 1 ─── 2
    (two parallel edges between 0 and 1)

Key: track by EDGE INDEX, not parent vertex.
  This prevents skipping all edges to parent.

DFS (skip by edge index, not vertex):
┌──────┬──────┬─────┬─────────────────────────────┐
│ Node │ disc │ low │ Notes                       │
├──────┼──────┼─────┼─────────────────────────────┤
│  0   │  0   │  0  │ use edge idx=0 to reach 1   │
│  1   │  1   │  0  │ skip edge idx=0 (same edge) │
│      │      │     │ edge idx=1 to 0: back edge  │
│      │      │     │ low[1]←0 (parallel edge!)   │
│  2   │  2   │  2  │ leaf, no back edges         │
└──────┴──────┴─────┴─────────────────────────────┘

Bridge check:
  (0,1): low[1]=0, disc[0]=0 → 0>0? No (parallel edge saves it)
  (1,2): low[2]=2, disc[1]=1 → 2>1? Yes → BRIDGE!

  Without edge-index tracking, 0-1 would be
  wrongly detected as a bridge.

Result: [[1, 2]]
```

---

## Example 8: 2-Edge-Connected Components

```go
package main

import "fmt"

func twoEdgeConnected(n int, adj [][]int) (int, []int) {
	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	isBridge := map[[2]int]bool{}

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++
		for _, u := range adj[v] {
			if disc[u] == -1 {
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				if low[u] > disc[v] {
					isBridge[[2]int{v, u}] = true
					isBridge[[2]int{u, v}] = true
				}
			} else if u != parent {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}

	// BFS/DFS ignoring bridges to find components
	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	numComp := 0
	for i := 0; i < n; i++ {
		if comp[i] != -1 { continue }
		queue := []int{i}
		comp[i] = numComp
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if comp[u] == -1 && !isBridge[[2]int{v, u}] {
					comp[u] = numComp
					queue = append(queue, u)
				}
			}
		}
		numComp++
	}

	return numComp, comp
}

func main() {
	adj := make([][]int, 6)
	adj[0] = []int{1, 2}; adj[1] = []int{0, 2, 3}; adj[2] = []int{0, 1}
	adj[3] = []int{1, 4, 5}; adj[4] = []int{3, 5}; adj[5] = []int{3, 4}

	numComp, comp := twoEdgeConnected(6, adj)
	fmt.Println("2-edge-connected components:", numComp)
	fmt.Println("Labels:", comp)
}
```

**Textual Figure:**

```
Undirected graph (6 nodes):
    adj[0]=[1,2], adj[1]=[0,2,3], adj[2]=[0,1],
    adj[3]=[1,4,5], adj[4]=[3,5], adj[5]=[3,4]

    0 ── 1 ─── 3 ── 4
     \  /       │    │
      2         └─ 5 ─┘

Step 1: Find bridges using Tarjan’s
  Bridge: edge (1,3) → low[3]=3 > disc[1]=1

    0 ── 1 ┃┃ 3 ── 4
     \  /  ↑↑  │    │
      2  bridge └─ 5 ─┘

Step 2: BFS ignoring bridges to find components

  Component 0: BFS from 0, skip bridge (1,3)
    Visit: 0 → 1 → 2       → comp = {0, 1, 2}

  Component 1: BFS from 3, skip bridge (1,3)
    Visit: 3 → 4 → 5       → comp = {3, 4, 5}

  ┌─────────┐          ┌─────────┐
  │ 0─1─2  │──bridge──│ 3─4─5  │
  │ comp=0  │          │ comp=1  │
  └─────────┘          └─────────┘

Result: 2 components, labels=[0,0,0,1,1,1]
```

---

## Example 9: Find All Bridges and Articulation Points Together

```go
package main

import "fmt"

func findBridgesAndAPs(n int, adj [][]int) ([][2]int, []int) {
	disc := make([]int, n)
	low := make([]int, n)
	for i := range disc { disc[i] = -1 }
	timer := 0
	bridges := [][2]int{}
	isAP := make([]bool, n)

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		disc[v] = timer; low[v] = timer; timer++
		children := 0

		for _, u := range adj[v] {
			if disc[u] == -1 {
				children++
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				if low[u] > disc[v] { bridges = append(bridges, [2]int{v, u}) }
				if parent == -1 && children > 1 { isAP[v] = true }
				if parent != -1 && low[u] >= disc[v] { isAP[v] = true }
			} else if u != parent {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if disc[i] == -1 { dfs(i, -1) }
	}

	aps := []int{}
	for i, ap := range isAP {
		if ap { aps = append(aps, i) }
	}
	return bridges, aps
}

func main() {
	adj := make([][]int, 7)
	adj[0] = []int{1, 2}; adj[1] = []int{0, 2}; adj[2] = []int{0, 1, 3}
	adj[3] = []int{2, 4, 5}; adj[4] = []int{3, 5}; adj[5] = []int{3, 4, 6}
	adj[6] = []int{5}

	bridges, aps := findBridgesAndAPs(7, adj)
	fmt.Println("Bridges:", bridges)
	fmt.Println("Articulation points:", aps)
}
```

**Textual Figure:**

```
Undirected graph (7 nodes):
    adj[0]=[1,2], adj[1]=[0,2], adj[2]=[0,1,3],
    adj[3]=[2,4,5], adj[4]=[3,5], adj[5]=[3,4,6],
    adj[6]=[5]

    0 ── 1
     \  /
      2 ─── 3 ── 4
              │    │
              └─ 5 ─┘
                 │
                 6

DFS disc/low:
┌──────┬──────┬─────┬─────────────────────────────┐
│ Node │ disc │ low │ Result                      │
├──────┼──────┼─────┼─────────────────────────────┤
│  0   │  0   │  0  │                             │
│  1   │  1   │  0  │ back edge via 2→0           │
│  2   │  2   │  0  │ AP: low[3]=2 >= disc[2]=2   │
│  3   │  3   │  3  │ cycle 3→4→5→3             │
│  4   │  4   │  3  │                             │
│  5   │  5   │  3  │ AP: low[6]=6 >= disc[5]=5   │
│  6   │  6   │  6  │ leaf                        │
└──────┴──────┴─────┴─────────────────────────────┘

Bridges (low[u] > disc[v]):
  (2,3): low[3]=3 > disc[2]=2 → BRIDGE
  (5,6): low[6]=6 > disc[5]=5 → BRIDGE

Articulation Points (low[u] >= disc[v]):
  Node 2: low[3]=3 >= disc[2]=2 → AP ✓
  Node 5: low[6]=6 >= disc[5]=5 → AP ✓

Result: Bridges=[(2,3),(5,6)], APs=[2,5]
```

---

## Example 10: Tarjan's vs Kosaraju's Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Tarjan's vs Kosaraju's SCC Comparison ===")
	fmt.Println()

	rows := []struct{ aspect, tarjan, kosaraju string }{
		{"DFS passes", "1", "2"},
		{"Extra structure", "Stack + low-link", "Reverse graph"},
		{"Memory", "O(V) stack", "O(V+E) reverse graph"},
		{"Bridges/APs", "Yes (extends naturally)", "No"},
		{"Implementation", "More complex", "Simpler"},
		{"Online processing", "Possible", "Not practical"},
		{"Output order", "Reverse topological", "Topological of DAG"},
		{"Time complexity", "O(V+E)", "O(V+E)"},
		{"Space complexity", "O(V)", "O(V+E)"},
		{"Best for", "Bridges + SCC together", "Just SCC detection"},
	}

	fmt.Printf("%-20s %-28s %-28s\n", "Aspect", "Tarjan's", "Kosaraju's")
	fmt.Println("-----------------------------------------------------------------------")
	for _, r := range rows {
		fmt.Printf("%-20s %-28s %-28s\n", r.aspect, r.tarjan, r.kosaraju)
	}

	fmt.Println()
	fmt.Println("Key insight:")
	fmt.Println("  low[v] = min discovery time reachable from subtree of v")
	fmt.Println("  Bridge:  low[u] > disc[v]  (no back edge up)")
	fmt.Println("  AP:      low[u] >= disc[v] (subtree can't escape)")
	fmt.Println("  SCC root: low[v] == disc[v] (head of component)")
}
```

**Textual Figure:**

```
Tarjan’s vs Kosaraju’s — Visual Comparison:

Same directed graph:
    0 → 1 → 2       Tarjan’s: 1 DFS pass
    ↑       │       Kosaraju’s: 2 DFS passes
    └───────┘

Tarjan’s (single DFS + stack):
  DFS: 0→1→2→(back to 0)
  Stack: [0, 1, 2]
  At node 0: low[0]==disc[0] → pop SCC: {2,1,0}

Kosaraju’s (two DFS + reverse graph):
  Pass 1: DFS to get finish order: [2, 1, 0]
  Reverse graph: 1→0, 2→1, 0→2
  Pass 2: DFS in reverse finish order on reversed graph
    Start at 0: visits 0→2→1 → SCC: {0,2,1}

┌────────────────────┬───────────────┬───────────────┐
│ Aspect             │ Tarjan’s       │ Kosaraju’s     │
├────────────────────┼───────────────┼───────────────┤
│ DFS passes         │ 1             │ 2             │
│ Extra structure     │ Stack+low-link│ Reverse graph │
│ Space              │ O(V)          │ O(V+E)        │
│ Bridges/APs        │ Yes           │ No            │
│ Complexity         │ O(V+E)        │ O(V+E)        │
└────────────────────┴───────────────┴───────────────┘

low-link key insight:
  low[v] = min disc[t] reachable from subtree(v)
  Bridge:  low[u] > disc[v]    (no bypass)
  AP:      low[u] >= disc[v]   (subtree trapped)
  SCC:     low[v] == disc[v]   (component root)
```

---

## Key Takeaways

1. Tarjan's uses disc/low arrays in a single DFS — O(V+E)
2. Bridge: `low[u] > disc[v]` — no back edge from u's subtree bypasses edge (v,u)
3. Articulation Point: `low[u] >= disc[v]` — removing v disconnects u's subtree
4. SCC root: `low[v] == disc[v]` — pop stack to get component members
5. Parallel edges need edge-index tracking (not parent-vertex tracking)

> **Next up:** Eulerian Paths →
