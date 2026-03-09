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

---

## Key Takeaways

1. Tarjan's uses disc/low arrays in a single DFS — O(V+E)
2. Bridge: `low[u] > disc[v]` — no back edge from u's subtree bypasses edge (v,u)
3. Articulation Point: `low[u] >= disc[v]` — removing v disconnects u's subtree
4. SCC root: `low[v] == disc[v]` — pop stack to get component members
5. Parallel edges need edge-index tracking (not parent-vertex tracking)

> **Next up:** Eulerian Paths →
