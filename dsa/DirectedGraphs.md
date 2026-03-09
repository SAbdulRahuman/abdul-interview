# Phase 12: Graphs — Directed Graphs

## Overview

A **directed graph** (digraph) has edges with direction: edge (u → v) means you can go from u to v but not necessarily v to u.

```
0 → 1 → 3
↓       ↑
2 ------┘

adj[0] = [1, 2]
adj[1] = [3]
adj[2] = [3]
adj[3] = []
```

Key properties unique to directed graphs:
- **In-degree / Out-degree** — separate counts
- **Topological order** — possible only if DAG (no cycles)
- **Strongly connected components** — maximal subsets where every vertex reaches every other

---

## Example 1: Build a Directed Graph

```go
package main

import "fmt"

func main() {
	n := 5
	adj := make([][]int, n)

	// Directed edges
	edges := [][2]int{{0,1},{0,2},{1,3},{2,3},{3,4}}
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
	}

	for v, nb := range adj {
		fmt.Printf("%d → %v\n", v, nb)
	}
}
```

---

## Example 2: In-Degree and Out-Degree

```go
package main

import "fmt"

func main() {
	n := 5
	adj := make([][]int, n)
	inDeg := make([]int, n)

	edges := [][2]int{{0,1},{0,2},{1,3},{2,3},{3,4},{4,1}}
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		inDeg[e[1]]++
	}

	for v := 0; v < n; v++ {
		fmt.Printf("Vertex %d: in-degree=%d, out-degree=%d\n", v, inDeg[v], len(adj[v]))
	}
	// Sources (in-degree 0): starting points
	// Sinks (out-degree 0): ending points
}
```

---

## Example 3: Course Schedule — Cycle Detection (LeetCode 207)

```go
package main

import "fmt"

func canFinish(numCourses int, prerequisites [][]int) bool {
	adj := make([][]int, numCourses)
	inDeg := make([]int, numCourses)

	for _, p := range prerequisites {
		adj[p[1]] = append(adj[p[1]], p[0])
		inDeg[p[0]]++
	}

	// Kahn's algorithm (BFS topological sort)
	queue := []int{}
	for i := 0; i < numCourses; i++ {
		if inDeg[i] == 0 { queue = append(queue, i) }
	}

	count := 0
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		count++
		for _, u := range adj[v] {
			inDeg[u]--
			if inDeg[u] == 0 { queue = append(queue, u) }
		}
	}
	return count == numCourses
}

func main() {
	fmt.Println(canFinish(2, [][]int{{1,0}}))       // true
	fmt.Println(canFinish(2, [][]int{{1,0},{0,1}})) // false (cycle)
}
```

---

## Example 4: Course Schedule II — Topological Order (LeetCode 210)

```go
package main

import "fmt"

func findOrder(numCourses int, prerequisites [][]int) []int {
	adj := make([][]int, numCourses)
	inDeg := make([]int, numCourses)

	for _, p := range prerequisites {
		adj[p[1]] = append(adj[p[1]], p[0])
		inDeg[p[0]]++
	}

	queue := []int{}
	for i := 0; i < numCourses; i++ {
		if inDeg[i] == 0 { queue = append(queue, i) }
	}

	order := []int{}
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, u := range adj[v] {
			inDeg[u]--
			if inDeg[u] == 0 { queue = append(queue, u) }
		}
	}

	if len(order) != numCourses { return []int{} }
	return order
}

func main() {
	fmt.Println(findOrder(4, [][]int{{1,0},{2,0},{3,1},{3,2}}))
	// [0 1 2 3] or [0 2 1 3]
}
```

---

## Example 5: DFS Cycle Detection in Directed Graph

```go
package main

import "fmt"

func hasCycleDFS(n int, adj [][]int) bool {
	// 0=unvisited, 1=in current path, 2=done
	state := make([]int, n)

	var dfs func(v int) bool
	dfs = func(v int) bool {
		state[v] = 1 // visiting
		for _, u := range adj[v] {
			if state[u] == 1 { return true }  // back edge = cycle
			if state[u] == 0 && dfs(u) { return true }
		}
		state[v] = 2 // done
		return false
	}

	for i := 0; i < n; i++ {
		if state[i] == 0 && dfs(i) {
			return true
		}
	}
	return false
}

func main() {
	// No cycle
	adj1 := [][]int{{1,2},{3},{3},{}} // 0→1→3, 0→2→3
	fmt.Println("Has cycle?", hasCycleDFS(4, adj1)) // false

	// Has cycle: 0→1→2→0
	adj2 := [][]int{{1},{2},{0},{}}
	fmt.Println("Has cycle?", hasCycleDFS(4, adj2)) // true
}
```

---

## Example 6: Alien Dictionary (LeetCode 269)

```go
package main

import "fmt"

func alienOrder(words []string) string {
	adj := map[byte]map[byte]bool{}
	inDeg := map[byte]int{}

	// Initialize all characters
	for _, w := range words {
		for i := 0; i < len(w); i++ {
			if adj[w[i]] == nil { adj[w[i]] = map[byte]bool{} }
			if _, ok := inDeg[w[i]]; !ok { inDeg[w[i]] = 0 }
		}
	}

	// Build edges from adjacent word pairs
	for i := 0; i < len(words)-1; i++ {
		w1, w2 := words[i], words[i+1]
		// Check invalid: w1 is prefix of w2 but longer
		if len(w1) > len(w2) {
			isPrefix := true
			for j := 0; j < len(w2); j++ {
				if w1[j] != w2[j] { isPrefix = false; break }
			}
			if isPrefix { return "" }
		}
		for j := 0; j < len(w1) && j < len(w2); j++ {
			if w1[j] != w2[j] {
				if !adj[w1[j]][w2[j]] {
					adj[w1[j]][w2[j]] = true
					inDeg[w2[j]]++
				}
				break
			}
		}
	}

	// Topological sort
	queue := []byte{}
	for ch, deg := range inDeg {
		if deg == 0 { queue = append(queue, ch) }
	}

	result := []byte{}
	for len(queue) > 0 {
		ch := queue[0]; queue = queue[1:]
		result = append(result, ch)
		for next := range adj[ch] {
			inDeg[next]--
			if inDeg[next] == 0 { queue = append(queue, next) }
		}
	}

	if len(result) != len(inDeg) { return "" }
	return string(result)
}

func main() {
	fmt.Println(alienOrder([]string{"wrt", "wrf", "er", "ett", "rftt"}))
	// "wertf"
}
```

---

## Example 7: All Paths from Source to Target (LeetCode 797)

```go
package main

import "fmt"

func allPathsSourceTarget(graph [][]int) [][]int {
	target := len(graph) - 1
	var result [][]int

	var dfs func(node int, path []int)
	dfs = func(node int, path []int) {
		path = append(path, node)
		if node == target {
			tmp := make([]int, len(path))
			copy(tmp, path)
			result = append(result, tmp)
			return
		}
		for _, next := range graph[node] {
			dfs(next, path)
		}
	}

	dfs(0, nil)
	return result
}

func main() {
	// DAG: 0→1, 0→2, 1→3, 2→3
	graph := [][]int{{1,2},{3},{3},{}}
	fmt.Println(allPathsSourceTarget(graph)) // [[0 1 3] [0 2 3]]
}
```

---

## Example 8: Longest Path in DAG

```go
package main

import "fmt"

func longestPathDAG(n int, adj [][]int) int {
	inDeg := make([]int, n)
	for _, nb := range adj {
		for _, v := range nb {
			inDeg[v]++
		}
	}

	// Topological sort + DP
	dist := make([]int, n) // longest path ending at each vertex
	queue := []int{}
	for i := 0; i < n; i++ {
		if inDeg[i] == 0 { queue = append(queue, i) }
	}

	maxLen := 0
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		for _, u := range adj[v] {
			if dist[v]+1 > dist[u] {
				dist[u] = dist[v] + 1
			}
			if dist[u] > maxLen { maxLen = dist[u] }
			inDeg[u]--
			if inDeg[u] == 0 { queue = append(queue, u) }
		}
	}
	return maxLen
}

func main() {
	adj := [][]int{{1, 2}, {3}, {3}, {4}, {}}
	fmt.Println("Longest path:", longestPathDAG(5, adj)) // 3 (0→1→3→4 or 0→2→3→4)
}
```

---

## Example 9: Reverse a Directed Graph

```go
package main

import "fmt"

func reverseGraph(n int, adj [][]int) [][]int {
	rev := make([][]int, n)
	for u, nb := range adj {
		for _, v := range nb {
			rev[v] = append(rev[v], u)
		}
	}
	return rev
}

func main() {
	adj := [][]int{{1, 2}, {3}, {3}, {}}
	rev := reverseGraph(4, adj)

	fmt.Println("Original:")
	for v, nb := range adj { fmt.Printf("  %d → %v\n", v, nb) }

	fmt.Println("Reversed:")
	for v, nb := range rev { fmt.Printf("  %d → %v\n", v, nb) }
}
```

**Why reverse?** Used in Kosaraju's SCC algorithm and in problems needing "who can reach me".

---

## Example 10: DAG vs General Directed Graph

```go
package main

import "fmt"

func isDAG(n int, adj [][]int) bool {
	state := make([]int, n)
	var dfs func(v int) bool
	dfs = func(v int) bool {
		state[v] = 1
		for _, u := range adj[v] {
			if state[u] == 1 { return false }
			if state[u] == 0 && !dfs(u) { return false }
		}
		state[v] = 2
		return true
	}
	for i := 0; i < n; i++ {
		if state[i] == 0 && !dfs(i) { return false }
	}
	return true
}

func main() {
	dag := [][]int{{1, 2}, {3}, {3}, {}}
	cycle := [][]int{{1}, {2}, {0}}

	fmt.Println("Is DAG?", isDAG(4, dag))    // true
	fmt.Println("Is DAG?", isDAG(3, cycle))  // false

	fmt.Println("\n=== DAG Properties ===")
	fmt.Println("• Has topological order")
	fmt.Println("• Longest/shortest path in O(V+E)")
	fmt.Println("• DP on DAG is very common")
	fmt.Println("• Course schedule, build systems, task ordering")
}
```

---

## Key Takeaways

1. Directed edges: `adj[u] = append(adj[u], v)` — only one direction
2. In-degree/out-degree are fundamental for topological sort
3. Cycle detection uses 3-state DFS (unvisited, visiting, done)
4. DAGs enable topological sort and efficient DP
5. Reverse graph: useful for SCC and "reachability from target"

> **Next up:** Undirected Graphs →
