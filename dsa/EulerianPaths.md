# Phase 12: Graphs — Eulerian Paths & Circuits

## Overview

| Concept | Definition |
|---------|-----------|
| **Eulerian Path** | A path that visits every **edge** exactly once |
| **Eulerian Circuit** | An Eulerian path that starts and ends at the same vertex |
| **Hamiltonian Path** | Visits every **vertex** exactly once (NP-complete) |

### Existence Conditions

| Graph Type | Eulerian Circuit | Eulerian Path |
|-----------|-----------------|---------------|
| **Undirected** | All vertices have even degree | Exactly 0 or 2 vertices with odd degree |
| **Directed** | in-degree = out-degree for all | At most 1 vertex with out−in=1, at most 1 with in−out=1 |

**Algorithm:** Hierholzer's — O(V + E)

---

## Example 1: Check Eulerian Path/Circuit (Undirected)

```go
package main

import "fmt"

func checkEulerian(n int, adj [][]int) string {
	oddCount := 0
	for i := 0; i < n; i++ {
		if len(adj[i])%2 != 0 {
			oddCount++
		}
	}

	// Check connectivity (ignoring isolated vertices)
	visited := make([]bool, n)
	start := -1
	for i := 0; i < n; i++ {
		if len(adj[i]) > 0 { start = i; break }
	}
	if start == -1 { return "Trivial (no edges)" }

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
	}
	dfs(start)

	for i := 0; i < n; i++ {
		if len(adj[i]) > 0 && !visited[i] { return "Not Eulerian (disconnected)" }
	}

	switch oddCount {
	case 0: return "Eulerian Circuit"
	case 2: return "Eulerian Path (not circuit)"
	default: return "Not Eulerian"
	}
}

func main() {
	// Triangle: all degrees = 2 → circuit
	adj1 := [][]int{{1,2},{0,2},{0,1}}
	fmt.Println(checkEulerian(3, adj1))

	// Path graph: 0-1-2 → vertices 0,2 have odd degree → path
	adj2 := [][]int{{1},{0,2},{1}}
	fmt.Println(checkEulerian(3, adj2))

	// K4: all degrees = 3 → 4 odd → not Eulerian
	adj3 := [][]int{{1,2,3},{0,2,3},{0,1,3},{0,1,2}}
	fmt.Println(checkEulerian(4, adj3))
}
```

---

## Example 2: Check Eulerian Path/Circuit (Directed)

```go
package main

import "fmt"

func checkEulerianDirected(n int, adj [][]int) string {
	inDeg := make([]int, n)
	outDeg := make([]int, n)
	for u := 0; u < n; u++ {
		outDeg[u] = len(adj[u])
		for _, v := range adj[u] { inDeg[v]++ }
	}

	startNodes, endNodes, equal := 0, 0, 0
	for i := 0; i < n; i++ {
		diff := outDeg[i] - inDeg[i]
		switch {
		case diff == 0:  equal++
		case diff == 1:  startNodes++
		case diff == -1: endNodes++
		default:         return "Not Eulerian"
		}
	}

	if startNodes == 0 && endNodes == 0 { return "Eulerian Circuit" }
	if startNodes == 1 && endNodes == 1 { return "Eulerian Path" }
	return "Not Eulerian"
}

func main() {
	adj1 := [][]int{{1},{2},{0}} // 0→1→2→0 → circuit
	fmt.Println(checkEulerianDirected(3, adj1))

	adj2 := [][]int{{1},{2},{}}  // 0→1→2 → path
	fmt.Println(checkEulerianDirected(3, adj2))
}
```

---

## Example 3: Hierholzer's Algorithm (Undirected)

```go
package main

import "fmt"

func eulerianPathUndirected(n int, edges [][2]int) []int {
	adj := make([]map[int]int, n) // adj[u][v] = count of edges
	for i := range adj { adj[i] = make(map[int]int) }
	for _, e := range edges {
		adj[e[0]][e[1]]++
		adj[e[1]][e[0]]++
	}

	// Find start vertex
	start := 0
	for i := 0; i < n; i++ {
		deg := 0
		for _, cnt := range adj[i] { deg += cnt }
		if deg%2 == 1 { start = i; break }
	}

	path := []int{}
	stack := []int{start}
	for len(stack) > 0 {
		v := stack[len(stack)-1]
		if len(adj[v]) > 0 {
			// Pick any neighbor
			var u int
			for u = range adj[v] { break }
			adj[v][u]--
			adj[u][v]--
			if adj[v][u] == 0 { delete(adj[v], u) }
			if adj[u][v] == 0 { delete(adj[u], v) }
			stack = append(stack, u)
		} else {
			stack = stack[:len(stack)-1]
			path = append(path, v)
		}
	}

	// Reverse
	for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
		path[i], path[j] = path[j], path[i]
	}
	return path
}

func main() {
	// Square with diagonal: 0-1, 1-2, 2-3, 3-0, 0-2
	edges := [][2]int{{0,1},{1,2},{2,3},{3,0},{0,2}}
	path := eulerianPathUndirected(4, edges)
	fmt.Println("Eulerian path:", path)
}
```

---

## Example 4: Hierholzer's Algorithm (Directed)

```go
package main

import "fmt"

func eulerianPathDirected(n int, adj [][]int) []int {
	inDeg := make([]int, n)
	outDeg := make([]int, n)
	for u := 0; u < n; u++ {
		outDeg[u] = len(adj[u])
		for _, v := range adj[u] { inDeg[v]++ }
	}

	start := 0
	for i := 0; i < n; i++ {
		if outDeg[i]-inDeg[i] == 1 { start = i; break }
	}

	// Copy adj for consumption
	idx := make([]int, n) // next edge index to use
	path := []int{}
	stack := []int{start}

	for len(stack) > 0 {
		v := stack[len(stack)-1]
		if idx[v] < len(adj[v]) {
			u := adj[v][idx[v]]
			idx[v]++
			stack = append(stack, u)
		} else {
			stack = stack[:len(stack)-1]
			path = append(path, v)
		}
	}

	// Reverse
	for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
		path[i], path[j] = path[j], path[i]
	}
	return path
}

func main() {
	adj := [][]int{{1},{2},{0}} // 0→1→2→0
	fmt.Println("Eulerian circuit:", eulerianPathDirected(3, adj))

	adj2 := make([][]int, 5)
	adj2[0] = []int{1}; adj2[1] = []int{2, 3}; adj2[2] = []int{1}
	adj2[3] = []int{4}
	fmt.Println("Eulerian path:", eulerianPathDirected(5, adj2))
}
```

---

## Example 5: Reconstruct Itinerary (LeetCode 332)

```go
package main

import (
	"fmt"
	"sort"
)

func findItinerary(tickets [][]string) []string {
	adj := map[string][]string{}
	for _, t := range tickets {
		adj[t[0]] = append(adj[t[0]], t[1])
	}
	for k := range adj {
		sort.Strings(adj[k])
	}

	idx := map[string]int{}
	result := []string{}

	var dfs func(airport string)
	dfs = func(airport string) {
		for idx[airport] < len(adj[airport]) {
			next := adj[airport][idx[airport]]
			idx[airport]++
			dfs(next)
		}
		result = append(result, airport)
	}

	dfs("JFK")

	// Reverse
	for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
		result[i], result[j] = result[j], result[i]
	}
	return result
}

func main() {
	tickets := [][]string{{"MUC","LHR"},{"JFK","MUC"},{"SFO","SJC"},{"LHR","SFO"}}
	fmt.Println(findItinerary(tickets))
	// [JFK MUC LHR SFO SJC]

	tickets2 := [][]string{{"JFK","SFO"},{"JFK","ATL"},{"SFO","ATL"},{"ATL","JFK"},{"ATL","SFO"}}
	fmt.Println(findItinerary(tickets2))
	// [JFK ATL JFK SFO ATL SFO]
}
```

---

## Example 6: Valid Arrangement of Pairs (LeetCode 2097)

```go
package main

import "fmt"

func validArrangement(pairs [][]int) [][]int {
	adj := map[int][]int{}
	inDeg := map[int]int{}
	outDeg := map[int]int{}

	for _, p := range pairs {
		adj[p[0]] = append(adj[p[0]], p[1])
		outDeg[p[0]]++
		inDeg[p[1]]++
	}

	// Find start: out - in == 1 (or any if circuit)
	start := pairs[0][0]
	for v := range outDeg {
		if outDeg[v]-inDeg[v] == 1 {
			start = v
			break
		}
	}

	idx := map[int]int{}
	path := []int{}
	stack := []int{start}

	for len(stack) > 0 {
		v := stack[len(stack)-1]
		if idx[v] < len(adj[v]) {
			u := adj[v][idx[v]]
			idx[v]++
			stack = append(stack, u)
		} else {
			stack = stack[:len(stack)-1]
			path = append(path, v)
		}
	}

	// Reverse
	for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
		path[i], path[j] = path[j], path[i]
	}

	result := make([][]int, len(path)-1)
	for i := 0; i < len(path)-1; i++ {
		result[i] = []int{path[i], path[i+1]}
	}
	return result
}

func main() {
	pairs := [][]int{{5,1},{4,5},{11,9},{9,4}}
	fmt.Println(validArrangement(pairs))
	// [[11 9] [9 4] [4 5] [5 1]]
}
```

---

## Example 7: Eulerian Circuit Verification

```go
package main

import "fmt"

func verifyEulerianCircuit(n int, edges [][2]int, path []int) bool {
	if len(path) != len(edges)+1 { return false }
	if path[0] != path[len(path)-1] { return false }

	// Build edge multiset
	edgeCount := map[[2]int]int{}
	for _, e := range edges {
		a, b := e[0], e[1]
		if a > b { a, b = b, a }
		edgeCount[[2]int{a, b}]++
	}

	// Verify each consecutive pair uses an edge
	for i := 0; i < len(path)-1; i++ {
		a, b := path[i], path[i+1]
		if a > b { a, b = b, a }
		key := [2]int{a, b}
		if edgeCount[key] <= 0 { return false }
		edgeCount[key]--
	}

	// All edges used
	for _, cnt := range edgeCount {
		if cnt != 0 { return false }
	}
	return true
}

func main() {
	edges := [][2]int{{0,1},{1,2},{2,0}}
	path := []int{0, 1, 2, 0}
	fmt.Println("Valid circuit:", verifyEulerianCircuit(3, edges, path)) // true

	badPath := []int{0, 1, 2, 1}
	fmt.Println("Valid circuit:", verifyEulerianCircuit(3, edges, badPath)) // false
}
```

---

## Example 8: Chinese Postman Problem (Simplified)

```go
package main

import "fmt"

// Find minimum cost to traverse all edges (may repeat some)
// If Eulerian: cost = sum of all edge weights
// Otherwise: find minimum weight matching of odd-degree vertices

func chinesePostman(n int, edges [][3]int) int {
	adj := make([][]int, n)
	totalWeight := 0
	degree := make([]int, n)

	for _, e := range edges {
		u, v, w := e[0], e[1], e[2]
		adj[u] = append(adj[u], v)
		adj[v] = append(adj[v], u)
		degree[u]++; degree[v]++
		totalWeight += w
	}

	// Find odd-degree vertices
	oddVertices := []int{}
	for i := 0; i < n; i++ {
		if degree[i]%2 == 1 { oddVertices = append(oddVertices, i) }
	}

	if len(oddVertices) == 0 {
		return totalWeight // Already Eulerian
	}

	// Compute shortest paths between odd-degree vertices (simplified: BFS unweighted)
	fmt.Printf("Need to match %d odd-degree vertices\n", len(oddVertices))
	fmt.Println("Total edge weight:", totalWeight)
	fmt.Println("Add minimum matching weight to get Chinese Postman cost")

	return totalWeight // Simplified — would add matching cost
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,2},{2,3,3},{3,0,4},{0,2,5}}
	cost := chinesePostman(4, edges)
	fmt.Println("Base cost:", cost)
}
```

---

## Example 9: de Bruijn Sequence Construction

```go
package main

import (
	"fmt"
	"strings"
)

// de Bruijn sequence: shortest string containing all k-length substrings over alphabet
// Uses Eulerian circuit on de Bruijn graph

func deBruijnSequence(k int, alphabet string) string {
	n := len(alphabet)
	if k == 1 { return alphabet }

	// Nodes: all (k-1)-length strings
	// Edge from s[0..k-2] to s[1..k-1] labeled s[k-1]
	adj := map[string][]struct{ to string; ch byte }{}

	// Generate all k-length strings
	var generate func(prefix string)
	generate = func(prefix string) {
		if len(prefix) == k {
			from := prefix[:k-1]
			to := prefix[1:]
			adj[from] = append(adj[from], struct{ to string; ch byte }{to, prefix[k-1]})
			return
		}
		for i := 0; i < n; i++ {
			generate(prefix + string(alphabet[i]))
		}
	}
	generate("")

	// Hierholzer's on directed graph
	start := strings.Repeat(string(alphabet[0]), k-1)
	idx := map[string]int{}
	path := []string{}
	stack := []string{start}

	for len(stack) > 0 {
		v := stack[len(stack)-1]
		if idx[v] < len(adj[v]) {
			e := adj[v][idx[v]]
			idx[v]++
			stack = append(stack, e.to)
		} else {
			stack = stack[:len(stack)-1]
			path = append(path, v)
		}
	}

	// Build sequence
	result := path[len(path)-1]
	for i := len(path) - 2; i >= 0; i-- {
		result += string(path[i][len(path[i])-1])
	}
	return result
}

func main() {
	seq := deBruijnSequence(2, "01")
	fmt.Println("de Bruijn(2, {0,1}):", seq) // e.g., "00110"
	fmt.Println("Length:", len(seq))

	seq3 := deBruijnSequence(2, "abc")
	fmt.Println("de Bruijn(2, {a,b,c}):", seq3)
}
```

---

## Example 10: Eulerian Concepts Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Eulerian Path/Circuit Summary ===")
	fmt.Println()

	concepts := []struct{ topic, detail string }{
		{"Eulerian Path", "Visit every EDGE exactly once"},
		{"Eulerian Circuit", "Eulerian path that returns to start"},
		{"Hamiltonian Path", "Visit every VERTEX exactly once (NP-hard)"},
		{"Undirected Circuit", "All vertices have even degree"},
		{"Undirected Path", "Exactly 0 or 2 odd-degree vertices"},
		{"Directed Circuit", "in-degree = out-degree for all"},
		{"Directed Path", "At most 1 with out-in=1, 1 with in-out=1"},
		{"Algorithm", "Hierholzer's — O(V+E), DFS with edge removal"},
		{"Applications", "Circuit design, DNA assembly, de Bruijn sequences"},
		{"Key Insight", "Post-order collection → reverse gives Euler path"},
	}

	for i, c := range concepts {
		fmt.Printf("%2d. %-22s %s\n", i+1, c.topic, c.detail)
	}

	fmt.Println()
	fmt.Println("Hierholzer's template:")
	fmt.Println("  1. Find start vertex (odd-degree or any)")
	fmt.Println("  2. DFS: consume edges, push to stack")
	fmt.Println("  3. When stuck, pop to result (post-order)")
	fmt.Println("  4. Reverse result → Eulerian path")
}
```

---

## Key Takeaways

1. Eulerian = every **edge** once; Hamiltonian = every **vertex** once
2. Existence checks: count odd-degree (undirected) or compare in/out-degree (directed)
3. Hierholzer's algorithm finds the path/circuit in O(V+E)
4. LeetCode classics: 332 (Reconstruct Itinerary), 2097 (Valid Arrangement)
5. Post-order DFS + reverse = correct Eulerian ordering

> **Phase 12 Complete! Next up:** Phase 13 — Union Find →
