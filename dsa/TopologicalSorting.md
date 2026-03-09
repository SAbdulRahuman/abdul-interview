# Phase 12: Graphs — Topological Sorting

## Overview

**Topological sort** produces a linear ordering of vertices in a **DAG** (Directed Acyclic Graph) such that for every edge u→v, u appears before v.

Two approaches:
- **Kahn's Algorithm (BFS)** — process nodes with in-degree 0
- **DFS Post-order** — reverse of post-order traversal

Both run in **O(V + E)**.

---

## Example 1: Kahn's BFS Topological Sort

```go
package main

import "fmt"

func topoSortKahns(n int, adj [][]int) []int {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, u := range adj[v] { indegree[u]++ }
	}

	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	order := []int{}
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, u := range adj[v] {
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}

	if len(order) != n { return nil } // cycle detected
	return order
}

func main() {
	// 0→1→3, 0→2→3
	adj := [][]int{{1,2},{3},{3},{}}
	fmt.Println(topoSortKahns(4, adj)) // [0 1 2 3] or [0 2 1 3]
}
```

---

## Example 2: DFS-Based Topological Sort

```go
package main

import "fmt"

func topoSortDFS(n int, adj [][]int) []int {
	visited := make([]bool, n)
	stack := []int{}

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
		stack = append(stack, v) // post-order
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i) }
	}

	// Reverse for topological order
	for i, j := 0, len(stack)-1; i < j; i, j = i+1, j-1 {
		stack[i], stack[j] = stack[j], stack[i]
	}
	return stack
}

func main() {
	adj := [][]int{{1,2},{3},{3},{}}
	fmt.Println(topoSortDFS(4, adj)) // [0 2 1 3] or [0 1 2 3]
}
```

---

## Example 3: Course Schedule II (LeetCode 210)

```go
package main

import "fmt"

func findOrder(numCourses int, prerequisites [][]int) []int {
	adj := make([][]int, numCourses)
	indegree := make([]int, numCourses)
	for _, p := range prerequisites {
		adj[p[1]] = append(adj[p[1]], p[0])
		indegree[p[0]]++
	}

	queue := []int{}
	for i := 0; i < numCourses; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	order := []int{}
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, u := range adj[v] {
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}

	if len(order) != numCourses { return []int{} }
	return order
}

func main() {
	fmt.Println(findOrder(4, [][]int{{1,0},{2,0},{3,1},{3,2}})) // [0 1 2 3] or [0 2 1 3]
	fmt.Println(findOrder(2, [][]int{{1,0},{0,1}}))              // [] (cycle)
}
```

---

## Example 4: Alien Dictionary (LeetCode 269)

```go
package main

import "fmt"

func alienOrder(words []string) string {
	adj := map[byte][]byte{}
	indegree := map[byte]int{}

	// Initialize all chars
	for _, w := range words {
		for i := 0; i < len(w); i++ {
			indegree[w[i]] = indegree[w[i]] // ensure presence
		}
	}

	// Build edges from adjacent word pairs
	for i := 0; i < len(words)-1; i++ {
		w1, w2 := words[i], words[i+1]
		minLen := len(w1)
		if len(w2) < minLen { minLen = len(w2) }

		if len(w1) > len(w2) {
			prefix := true
			for j := 0; j < minLen; j++ {
				if w1[j] != w2[j] { prefix = false; break }
			}
			if prefix { return "" } // invalid: "abc" before "ab"
		}

		for j := 0; j < minLen; j++ {
			if w1[j] != w2[j] {
				adj[w1[j]] = append(adj[w1[j]], w2[j])
				indegree[w2[j]]++
				break
			}
		}
	}

	queue := []byte{}
	for ch, deg := range indegree {
		if deg == 0 { queue = append(queue, ch) }
	}

	result := []byte{}
	for len(queue) > 0 {
		ch := queue[0]; queue = queue[1:]
		result = append(result, ch)
		for _, next := range adj[ch] {
			indegree[next]--
			if indegree[next] == 0 { queue = append(queue, next) }
		}
	}

	if len(result) != len(indegree) { return "" }
	return string(result)
}

func main() {
	fmt.Println(alienOrder([]string{"wrt","wrf","er","ett","rftt"})) // "wertf"
	fmt.Println(alienOrder([]string{"z","x"}))                       // "zx"
	fmt.Println(alienOrder([]string{"z","x","z"}))                   // "" (cycle)
}
```

---

## Example 5: Longest Path in DAG

```go
package main

import "fmt"

func longestPathDAG(n int, adj [][]int) int {
	visited := make([]bool, n)
	topoStack := []int{}

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
		topoStack = append(topoStack, v)
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i) }
	}

	dist := make([]int, n)
	maxDist := 0
	for i := len(topoStack) - 1; i >= 0; i-- {
		v := topoStack[i]
		for _, u := range adj[v] {
			if dist[v]+1 > dist[u] {
				dist[u] = dist[v] + 1
			}
		}
		if dist[v] > maxDist { maxDist = dist[v] }
	}
	return maxDist
}

func main() {
	// DAG: 0→1→3→5, 0→2→3→4
	adj := [][]int{{1,2},{3},{3},{4,5},{},{}}
	fmt.Println("Longest path:", longestPathDAG(6, adj)) // 3
}
```

---

## Example 6: Parallel Job Scheduling (Minimum Time)

```go
package main

import "fmt"

func minCompletionTime(n int, adj [][]int, duration []int) int {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, u := range adj[v] { indegree[u]++ }
	}

	earliest := make([]int, n)
	copy(earliest, duration)

	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		for _, u := range adj[v] {
			if earliest[v]+duration[u] > earliest[u] {
				earliest[u] = earliest[v] + duration[u]
			}
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}

	maxTime := 0
	for _, t := range earliest {
		if t > maxTime { maxTime = t }
	}
	return maxTime
}

func main() {
	// Job 0(3)→Job 1(2)→Job 3(1), Job 0(3)→Job 2(4)→Job 3(1)
	adj := [][]int{{1,2},{3},{3},{}}
	duration := []int{3, 2, 4, 1}
	fmt.Println("Min time:", minCompletionTime(4, adj, duration)) // 8
}
```

---

## Example 7: All Topological Orderings (Backtracking)

```go
package main

import "fmt"

func allTopoSorts(n int, adj [][]int) [][]int {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, u := range adj[v] { indegree[u]++ }
	}

	result := [][]int{}
	visited := make([]bool, n)
	current := []int{}

	var backtrack func()
	backtrack = func() {
		if len(current) == n {
			tmp := make([]int, n)
			copy(tmp, current)
			result = append(result, tmp)
			return
		}
		for i := 0; i < n; i++ {
			if !visited[i] && indegree[i] == 0 {
				visited[i] = true
				current = append(current, i)
				for _, u := range adj[i] { indegree[u]-- }

				backtrack()

				visited[i] = false
				current = current[:len(current)-1]
				for _, u := range adj[i] { indegree[u]++ }
			}
		}
	}

	backtrack()
	return result
}

func main() {
	adj := [][]int{{1,2},{3},{3},{}}
	for _, order := range allTopoSorts(4, adj) {
		fmt.Println(order)
	}
	// [0 1 2 3]
	// [0 2 1 3]
}
```

---

## Example 8: Build Order with Multiple Sources

```go
package main

import "fmt"

func buildOrder(projects []string, deps [][]string) []string {
	idx := map[string]int{}
	for i, p := range projects { idx[p] = i }

	n := len(projects)
	adj := make([][]int, n)
	indegree := make([]int, n)

	for _, d := range deps {
		u, v := idx[d[0]], idx[d[1]]
		adj[u] = append(adj[u], v)
		indegree[v]++
	}

	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	result := []string{}
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		result = append(result, projects[v])
		for _, u := range adj[v] {
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}

	if len(result) != n { return nil }
	return result
}

func main() {
	projects := []string{"a","b","c","d","e","f"}
	deps := [][]string{{"a","d"},{"f","b"},{"b","d"},{"f","a"},{"d","c"}}
	fmt.Println(buildOrder(projects, deps))
	// e.g., [e f a b d c]
}
```

---

## Example 9: Minimum Semesters to Graduate (LeetCode 1136)

```go
package main

import "fmt"

func minimumSemesters(n int, relations [][]int) int {
	adj := make([][]int, n+1)
	indegree := make([]int, n+1)
	for _, r := range relations {
		adj[r[0]] = append(adj[r[0]], r[1])
		indegree[r[1]]++
	}

	queue := []int{}
	for i := 1; i <= n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	semesters, taken := 0, 0
	for len(queue) > 0 {
		semesters++
		size := len(queue)
		for i := 0; i < size; i++ {
			v := queue[0]; queue = queue[1:]
			taken++
			for _, u := range adj[v] {
				indegree[u]--
				if indegree[u] == 0 { queue = append(queue, u) }
			}
		}
	}

	if taken != n { return -1 } // cycle
	return semesters
}

func main() {
	fmt.Println(minimumSemesters(3, [][]int{{1,3},{2,3}}))       // 2
	fmt.Println(minimumSemesters(3, [][]int{{1,2},{2,3},{3,1}})) // -1 (cycle)
}
```

---

## Example 10: Topological Sort with Weighted Edges (Critical Path)

```go
package main

import "fmt"

type Edge struct{ to, weight int }

func criticalPath(n int, adj [][]Edge) ([]int, int) {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, e := range adj[v] { indegree[e.to]++ }
	}

	earliest := make([]int, n)
	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	order := []int{}
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, e := range adj[v] {
			if earliest[v]+e.weight > earliest[e.to] {
				earliest[e.to] = earliest[v] + e.weight
			}
			indegree[e.to]--
			if indegree[e.to] == 0 { queue = append(queue, e.to) }
		}
	}

	maxTime := 0
	for _, t := range earliest {
		if t > maxTime { maxTime = t }
	}
	return earliest, maxTime
}

func main() {
	adj := [][]Edge{
		{{1, 3}, {2, 2}},
		{{3, 4}},
		{{3, 5}},
		{},
	}
	earliest, total := criticalPath(4, adj)
	fmt.Println("Earliest:", earliest) // [0 3 2 7]
	fmt.Println("Critical path length:", total) // 7
}
```

---

## Summary Table

| Method | Approach | Cycle Detection | Extra Info |
|--------|----------|----------------|------------|
| Kahn's BFS | In-degree queue | Yes (len < n) | Level = parallel schedule |
| DFS post-order | Reverse finish | Needs 3-color | Natural for recursion |
| Backtracking | All orderings | N/A | Exponential |

## Key Takeaways

1. Topological sort only works on DAGs — if cycle exists, no valid ordering
2. Kahn's BFS: process in-degree 0 nodes first; level count = minimum parallel steps
3. DFS approach: reverse post-order = topological order
4. Applications: course scheduling, build systems, dependency resolution, critical path
5. Multiple valid orderings may exist — Kahn's gives one, backtracking gives all

> **Next up:** Dijkstra's Algorithm →
