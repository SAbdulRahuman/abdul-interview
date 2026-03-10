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

**Textual Figure:**
```
Directed Acyclic Graph:
    ┌───┐     ┌───┐
    │ 0 │────→│ 1 │
    └─┬─┘     └─┬─┘
      │         │
      ↓         ↓
    ┌───┐     ┌───┐
    │ 2 │────→│ 3 │
    └───┘     └───┘

Kahn's BFS (process indegree-0 nodes):
┌──────┬──────────────────┬─────────┬───────────────────┐
│ Step │ indegree         │ Queue   │ Process             │
├──────┼──────────────────┼─────────┼───────────────────┤
│ Init │ [0, 1, 1, 2]     │ [0]     │                     │
│  1   │ [0, 0, 0, 2]     │ [1, 2]  │ Pop 0; ──indeg 1,2  │
│  2   │ [0, 0, 0, 1]     │ [2]     │ Pop 1; ──indeg 3    │
│  3   │ [0, 0, 0, 0]     │ [3]     │ Pop 2; ──indeg 3    │
│  4   │ [0, 0, 0, 0]     │ []      │ Pop 3; done         │
└──────┴──────────────────┴─────────┴───────────────────┘

Topological Order: [0, 1, 2, 3]  (or [0, 2, 1, 3])
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

**Textual Figure:**
```
Same DAG:
    ┌───┐     ┌───┐
    │ 0 │────→│ 1 │
    └─┬─┘     └─┬─┘
      │         │
      ↓         ↓
    ┌───┐     ┌───┐
    │ 2 │────→│ 3 │
    └───┘     └───┘

DFS Post-order Approach:
  dfs(0):
    └→ dfs(1):
        └→ dfs(3): no unvisited neighbors
           post-order push: 3
       post-order push: 1
    └→ dfs(2): (3 already visited)
       post-order push: 2
   post-order push: 0

  Stack (post-order): [3, 1, 2, 0]
  Reversed:           [0, 2, 1, 3]  ← topological order

  Verify: 0→1 ✓, 0→2 ✓, 1→3 ✓, 2→3 ✓
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

**Textual Figure:**
```
Test 1: prerequisites [[1,0],[2,0],[3,1],[3,2]]
  Dependency edges: 0→1, 0→2, 1→3, 2→3

      ┌───┐     ┌───┐
      │ 0 │────→│ 1 │
      └─┬─┘     └─┬─┘
        │         │
        ↓         ↓
      ┌───┐     ┌───┐
      │ 2 │────→│ 3 │
      └───┘     └───┘

  Kahn's: indeg=[0,1,1,2] → queue=[0]
    Process 0 → order=[0], queue=[1,2]
    Process 1 → order=[0,1], indeg[3]-- → queue=[2]
    Process 2 → order=[0,1,2], indeg[3]-- → queue=[3]
    Process 3 → order=[0,1,2,3]
  Result: [0, 1, 2, 3]

Test 2: prerequisites [[1,0],[0,1]] → cycle
  0 ⇌ 1 (mutual dependency) → no valid order
  Result: []
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

**Textual Figure:**
```
Test 1: words = ["wrt","wrf","er","ett","rftt"]

  Derive edges from adjacent pairs:
    "wrt" vs "wrf" → t → f  (3rd char differs)
    "wrf" vs "er"  → w → e  (1st char differs)
    "er"  vs "ett" → r → t  (2nd char differs)
    "ett" vs "rftt"→ e → r  (1st char differs)

  Dependency graph:
    w → e → r → t → f

  Kahn's: indeg: w=0, e=1, r=1, t=1, f=1
    queue=[w] → process w → queue=[e]
    process e → queue=[r] → process r → queue=[t]
    process t → queue=[f] → process f → done

  Result: "wertf"

Test 3: ["z","x","z"] → z→x, x→z → cycle → ""
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

**Textual Figure:**
```
DAG (6 vertices): 0→1, 0→2, 1→3, 2→3, 3→4, 3→5

    ┌───┐         ┌───┐
    │ 0 │────────→│ 1 │
    └─┬─┘         └─┬─┘
      │             │
      ↓             ↓
    ┌───┐         ┌───┐
    │ 2 │────────→│ 3 │
    └───┘         └─┬─┘
               ┌───┘└───┐
               ↓       ↓
             ┌───┐  ┌───┐
             │ 4 │  │ 5 │
             └───┘  └───┘

Longest Path Computation (topo order + DP):
  Topo stack (post-order reversed): [0, 2, 1, 3, 5, 4]

  dist[v] = max length path ending at v:
    dist[0]=0
    dist[2]=0+1=1, dist[1]=0+1=1
    dist[3]=max(dist[1]+1, dist[2]+1)=2
    dist[4]=dist[3]+1=3, dist[5]=dist[3]+1=3

  Longest path: 0→1→3→4 = 3 edges

Result: 3
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

**Textual Figure:**
```
Jobs DAG with durations:
  Job 0 (dur=3) → Job 1 (dur=2) → Job 3 (dur=1)
  Job 0 (dur=3) → Job 2 (dur=4) → Job 3 (dur=1)

    ┌───────┐       ┌───────┐
    │ 0 (3) │──────→│ 1 (2) │
    └───┬───┘       └───┬───┘
        │               │
        ↓               ↓
    ┌───────┐       ┌───────┐
    │ 2 (4) │──────→│ 3 (1) │
    └───────┘       └───────┘

Earliest completion times:
  Job 0: starts at 0, finishes at 3
  Job 1: starts at 3 (after 0), finishes at 3+2=5
  Job 2: starts at 3 (after 0), finishes at 3+4=7
  Job 3: starts at max(5,7)=7, finishes at 7+1=8

  Critical path: 0 → 2 → 3  (total = 3+4+1 = 8)

Result: 8
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

**Textual Figure:**
```
DAG: 0→1, 0→2, 1→3, 2→3
    ┌───┐     ┌───┐
    │ 0 │────→│ 1 │
    └─┬─┘     └─┬─┘
      │         │
      ↓         ↓
    ┌───┐     ┌───┐
    │ 2 │────→│ 3 │
    └───┘     └───┘

Backtracking generates ALL valid orderings:
  indeg = [0,1,1,2]

  Pick 0 (indeg=0):
    Pick 1 (indeg=0 after 0 processed):
      Pick 2 (indeg=0 after 1 processed):
        Pick 3 → [0, 1, 2, 3] ✓
    Pick 2 (indeg=0 after 0 processed):
      Pick 1 (indeg=0 after 2 processed):
        Pick 3 → [0, 2, 1, 3] ✓

All topological orderings:
  [0, 1, 2, 3]
  [0, 2, 1, 3]
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

**Textual Figure:**
```
Dependency Graph (6 projects):
  Edges: a→d, f→b, b→d, f→a, d→c

    ┌───┐     ┌───┐     ┌───┐     ┌───┐
    │ f │────→│ a │────→│ d │────→│ c │
    └─┬─┘     └───┘     └───┘     └───┘
      │                  ↑
      ↓               ┌──┘
    ┌───┐         ┌───┐
    │ b │────────→│   │  (b→d)
    └───┘         └───┘
    ┌───┐
    │ e │  (no dependencies, independent)
    └───┘

Kahn's BFS:
  indeg: e=0, f=0, a=1, b=1, d=2, c=1
  queue=[e, f] (multiple sources)
  Process e → Process f → unlock a, b
  Process a, b → unlock d
  Process d → unlock c

Result: [e, f, a, b, d, c] (one valid ordering)
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

**Textual Figure:**
```
Test 1: n=3, relations=[[1,3],[2,3]]
  Dependency: 1→3, 2→3

    ┌───┐
    │ 1 │───┐
    └───┘   │    ┌───┐
              └───→│ 3 │
    ┌───┐   ┌───→└───┘
    │ 2 │───┘
    └───┘

  Level-by-level BFS (each level = 1 semester):
    Semester 1: [1, 2]  (indeg=0)
    Semester 2: [3]     (indeg becomes 0)
  Result: 2 semesters

Test 2: relations=[[1,2],[2,3],[3,1]] → cycle
    1 → 2 → 3 → 1  (all indeg ≥ 1, no start)
  Result: -1
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

**Textual Figure:**
```
Weighted DAG (4 vertices):
    ┌───┐ ──w=3── ┌───┐
    │ 0 │────────→│ 1 │
    └─┬─┘         └─┬─┘
      │  w=2       │ w=4
      ↓            ↓
    ┌───┐         ┌───┐
    │ 2 │────────→│ 3 │
    └───┘ ──w=5── └───┘

Critical Path Calculation:
  earliest[0] = 0                        (source)
  earliest[1] = earliest[0] + 3 = 3      (via 0→1)
  earliest[2] = earliest[0] + 2 = 2      (via 0→2)
  earliest[3] = max(3+4, 2+5) = max(7,7) = 7

  Two critical paths (both length 7):
    0 ──3──→ 1 ──4──→ 3   (total=7)
    0 ──2──→ 2 ──5──→ 3   (total=7)

Result: earliest=[0, 3, 2, 7], critical path length=7
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
