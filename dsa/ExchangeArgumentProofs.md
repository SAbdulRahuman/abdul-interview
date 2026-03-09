# Phase 17: Greedy Algorithms — Exchange Argument Proofs

## Overview

The **Exchange Argument** is the primary technique for proving greedy algorithms are correct. It works by showing that any optimal solution can be transformed into the greedy solution without worsening the result.

| Step | Action |
|------|--------|
| 1 | Assume OPT ≠ GREEDY |
| 2 | Find first point of difference |
| 3 | Exchange OPT's choice with GREEDY's |
| 4 | Show solution doesn't get worse |
| 5 | Repeat → OPT = GREEDY |

---

## Example 1: Activity Selection Exchange Argument

```go
package main

import (
	"fmt"
	"sort"
)

func activitySelection(intervals [][]int) [][]int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})

	selected := [][]int{intervals[0]}
	end := intervals[0][1]

	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] >= end {
			selected = append(selected, intervals[i])
			end = intervals[i][1]
		}
	}
	return selected
}

func main() {
	intervals := [][]int{{1, 4}, {3, 5}, {0, 6}, {5, 7}, {3, 9}, {5, 9}, {6, 10}, {8, 11}}
	result := activitySelection(intervals)

	fmt.Println("Selected:", result)
	fmt.Println()
	fmt.Println("Exchange Argument Proof:")
	fmt.Println("  Let G = greedy, O = any optimal solution")
	fmt.Println("  G picks earliest-finishing activity at each step")
	fmt.Println()
	fmt.Println("  If O's first pick o₁ finishes later than G's g₁:")
	fmt.Println("    Replace o₁ with g₁ in O")
	fmt.Println("    g₁ finishes earlier → doesn't conflict with O's remaining")
	fmt.Println("    New O has same size but matches G at position 1")
	fmt.Println()
	fmt.Println("  Repeat for each position → O = G")
	fmt.Println("  Therefore |G| ≥ |O| → G is optimal ∎")
}
```

---

## Example 2: Scheduling to Minimize Total Completion Time

```go
package main

import (
	"fmt"
	"sort"
)

func minTotalCompletion(jobs []int) (int, []int) {
	sorted := make([]int, len(jobs))
	copy(sorted, jobs)
	sort.Ints(sorted)

	total := 0
	time := 0
	for _, j := range sorted {
		time += j
		total += time
	}
	return total, sorted
}

func main() {
	jobs := []int{3, 1, 2}
	total, order := minTotalCompletion(jobs)
	fmt.Println("Order:", order)
	fmt.Println("Total completion time:", total)
	// Order: [1,2,3], completion: 1+3+6 = 10

	fmt.Println()
	fmt.Println("Exchange Argument:")
	fmt.Println("  If job i before job j, and p_i > p_j (processing time):")
	fmt.Println("    Swap them: job j now finishes p_j earlier")
	fmt.Println("    job i finishes p_j later, but net change = p_j - p_i < 0")
	fmt.Println("    Total completion decreases → original wasn't optimal")
	fmt.Println("  Therefore: sort by processing time (SPT) is optimal ∎")
}
```

---

## Example 3: Minimize Maximum Lateness

```go
package main

import (
	"fmt"
	"sort"
)

type Job struct {
	ID       int
	Duration int
	Deadline int
}

func minimizeLateness(jobs []Job) int {
	// EDD: Earliest Deadline Due
	sort.Slice(jobs, func(i, j int) bool {
		return jobs[i].Deadline < jobs[j].Deadline
	})

	time := 0
	maxLateness := 0
	for _, j := range jobs {
		time += j.Duration
		lateness := time - j.Deadline
		if lateness > maxLateness { maxLateness = lateness }
	}
	return maxLateness
}

func main() {
	jobs := []Job{
		{1, 1, 4}, {2, 3, 6}, {3, 2, 12},
		{4, 4, 8}, {5, 3, 14}, {6, 2, 15},
	}
	fmt.Println("Min max lateness:", minimizeLateness(jobs))

	fmt.Println()
	fmt.Println("Exchange Argument (EDD rule):")
	fmt.Println("  If jobs i,j are adjacent and d_i > d_j (later deadline first):")
	fmt.Println("    Swap them (j before i now)")
	fmt.Println("    j's lateness: same finish, earlier deadline → lateness may increase")
	fmt.Println("    BUT: j now finishes p_i earlier")
	fmt.Println("    i's lateness: finishes at same total time, later deadline → less late")
	fmt.Println("    Net: max of two latenesses doesn't increase")
	fmt.Println("  Therefore: EDD minimizes maximum lateness ∎")
}
```

---

## Example 4: Fractional Knapsack Exchange

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct {
	Name   string
	Weight float64
	Value  float64
}

func fractionalKnapsack(items []Item, cap float64) float64 {
	sort.Slice(items, func(i, j int) bool {
		return items[i].Value/items[i].Weight > items[j].Value/items[j].Weight
	})

	total := 0.0
	rem := cap
	for _, it := range items {
		if rem <= 0 { break }
		take := it.Weight
		if take > rem { take = rem }
		total += it.Value * (take / it.Weight)
		rem -= take
	}
	return total
}

func main() {
	items := []Item{{"A", 10, 60}, {"B", 20, 100}, {"C", 30, 120}}
	fmt.Printf("Max value: %.0f\n", fractionalKnapsack(items, 50))

	fmt.Println()
	fmt.Println("Exchange Argument:")
	fmt.Println("  Let G select by ratio r_i = v_i/w_i (descending)")
	fmt.Println("  Let O be optimal, differing at item k")
	fmt.Println("  O uses less of item k (high ratio), more of item j (low ratio)")
	fmt.Println()
	fmt.Println("  Exchange: shift δ weight from j to k in O")
	fmt.Println("  Value change: δ × r_k - δ × r_j = δ(r_k - r_j) ≥ 0")
	fmt.Println("  Solution doesn't worsen → repeat until O = G ∎")
}
```

---

## Example 5: Optimal Merge Pattern Exchange

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func optimalMerge(files []int) int {
	h := &IntHeap{}
	for _, f := range files { heap.Push(h, f) }

	cost := 0
	for h.Len() > 1 {
		a, b := heap.Pop(h).(int), heap.Pop(h).(int)
		merged := a + b
		cost += merged
		heap.Push(h, merged)
	}
	return cost
}

func main() {
	files := []int{4, 3, 2, 6}
	fmt.Println("Min merge cost:", optimalMerge(files))

	fmt.Println()
	fmt.Println("Exchange Argument:")
	fmt.Println("  Claim: two smallest files should be merged first")
	fmt.Println()
	fmt.Println("  If OPT merges files a,b first where a > smallest s:")
	fmt.Println("    Swap s into a's position")
	fmt.Println("    First merge cost: s+b < a+b (since s < a)")
	fmt.Println("    s appears in fewer future merges (deeper in tree)")
	fmt.Println("    Total cost doesn't increase → greedy is optimal ∎")
}
```

---

## Example 6: Stable Matching (Gale-Shapley) Exchange

```go
package main

import "fmt"

func stableMatching(menPrefs, womenPrefs [][]int) [][2]int {
	n := len(menPrefs)
	womenPartner := make([]int, n)
	menPartner := make([]int, n)
	free := make([]bool, n)
	proposals := make([]int, n) // next woman to propose to

	for i := range womenPartner { womenPartner[i] = -1 }
	for i := range menPartner { menPartner[i] = -1 }

	womenRank := make([][]int, n)
	for w := 0; w < n; w++ {
		womenRank[w] = make([]int, n)
		for rank, m := range womenPrefs[w] {
			womenRank[w][m] = rank
		}
	}

	freeCount := n
	for freeCount > 0 {
		var m int
		for i := 0; i < n; i++ {
			if menPartner[i] == -1 && !free[i] || menPartner[i] == -1 {
				m = i
				break
			}
		}
		if menPartner[m] != -1 { break }

		w := menPrefs[m][proposals[m]]
		proposals[m]++

		if womenPartner[w] == -1 {
			womenPartner[w] = m
			menPartner[m] = w
			freeCount--
		} else if womenRank[w][m] < womenRank[w][womenPartner[w]] {
			oldM := womenPartner[w]
			menPartner[oldM] = -1
			womenPartner[w] = m
			menPartner[m] = w
		}
	}

	result := make([][2]int, n)
	for m := 0; m < n; m++ {
		result[m] = [2]int{m, menPartner[m]}
	}
	return result
}

func main() {
	menPrefs := [][]int{{0, 1, 2}, {1, 0, 2}, {0, 1, 2}}
	womenPrefs := [][]int{{1, 0, 2}, {0, 1, 2}, {0, 1, 2}}

	matching := stableMatching(menPrefs, womenPrefs)
	for _, m := range matching {
		fmt.Printf("Man %d → Woman %d\n", m[0], m[1])
	}
}
```

---

## Example 7: Huffman Exchange Proof

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Huffman Coding: Exchange Argument ===")
	fmt.Println()

	fmt.Println("Claim: In an optimal code, the two lowest-frequency")
	fmt.Println("symbols should be siblings at maximum depth.")
	fmt.Println()

	fmt.Println("Proof:")
	fmt.Println("  1. Let T be optimal tree, a,b = two lowest freq symbols")
	fmt.Println("  2. Let x,y = two deepest siblings in T")
	fmt.Println()
	fmt.Println("  3. If a ≠ x: swap a and x in T")
	fmt.Println("     Cost change: (freq(a)-freq(x)) × (depth(a)-depth(x))")
	fmt.Println("     Since freq(a) ≤ freq(x) and depth(a) ≤ depth(x)")
	fmt.Println("     Change ≤ 0 → cost doesn't increase")
	fmt.Println()
	fmt.Println("  4. Similarly swap b with y")
	fmt.Println()
	fmt.Println("  5. Now a,b are deepest siblings → Huffman's first step")
	fmt.Println("     Merge a,b → new symbol with freq(a)+freq(b)")
	fmt.Println("     Recurse on smaller problem (optimal substructure)")
	fmt.Println()
	fmt.Println("  Therefore Huffman is optimal ∎")
}
```

---

## Example 8: MST Kruskal Exchange Proof

```go
package main

import "fmt"

type Edge struct {
	U, V, W int
}

type DSU struct {
	parent, rank []int
}

func NewDSU(n int) *DSU {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &DSU{p, make([]int, n)}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
	px, py := d.Find(x), d.Find(y)
	if px == py { return false }
	if d.rank[px] < d.rank[py] { px, py = py, px }
	d.parent[py] = px
	if d.rank[px] == d.rank[py] { d.rank[px]++ }
	return true
}

func main() {
	fmt.Println("=== Kruskal's MST: Exchange/Cut Argument ===")
	fmt.Println()
	fmt.Println("Claim: Adding cheapest edge that doesn't form cycle is optimal")
	fmt.Println()
	fmt.Println("Proof (Cut Property):")
	fmt.Println("  For any cut (S, V-S), the minimum weight crossing edge")
	fmt.Println("  must be in SOME MST.")
	fmt.Println()
	fmt.Println("  Suppose MST T doesn't include min crossing edge e=(u,v).")
	fmt.Println("  T has a path u→v. This path must cross the cut at some edge f.")
	fmt.Println("  Replace f with e: T' = T - f + e")
	fmt.Println("  w(e) ≤ w(f) → w(T') ≤ w(T)")
	fmt.Println("  T' is still spanning and connected → T' is also MST")
	fmt.Println()
	fmt.Println("  Kruskal processes edges in weight order.")
	fmt.Println("  Each added edge is the min crossing edge for some cut ∎")
}
```

---

## Example 9: Common Exchange Argument Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Exchange Argument Templates ===")
	fmt.Println()

	fmt.Println("Template 1: Adjacent Swap")
	fmt.Println("  If two adjacent elements are in wrong order,")
	fmt.Println("  swapping improves or maintains the solution.")
	fmt.Println("  Used for: scheduling, sorting-based greedy")
	fmt.Println()

	fmt.Println("Template 2: Global Swap")
	fmt.Println("  Replace any element in OPT with greedy's choice.")
	fmt.Println("  Show the replacement doesn't worsen the solution.")
	fmt.Println("  Used for: activity selection, knapsack")
	fmt.Println()

	fmt.Println("Template 3: Structural Exchange")
	fmt.Println("  Modify the structure (tree, graph) of OPT")
	fmt.Println("  to match greedy's structure.")
	fmt.Println("  Used for: Huffman, MST")
	fmt.Println()

	fmt.Println("Template 4: Inductive Exchange")
	fmt.Println("  Show greedy is optimal at step 1.")
	fmt.Println("  Assume optimal for k steps, prove for k+1.")
	fmt.Println("  Used for: general greedy algorithms")
}
```

---

## Example 10: Practice Framework

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Exchange Argument: Step-by-Step Guide ===")
	fmt.Println()

	fmt.Println("Step 1: Define the greedy algorithm clearly")
	fmt.Println("  • What choice does it make at each step?")
	fmt.Println("  • What's the sorting/selection criterion?")
	fmt.Println()

	fmt.Println("Step 2: Assume an optimal solution O ≠ G")
	fmt.Println("  • Let k be the first position where O and G differ")
	fmt.Println()

	fmt.Println("Step 3: Perform the exchange")
	fmt.Println("  • Replace O's choice at position k with G's choice")
	fmt.Println("  • Call the new solution O'")
	fmt.Println()

	fmt.Println("Step 4: Show O' is feasible")
	fmt.Println("  • Verify O' satisfies all constraints")
	fmt.Println()

	fmt.Println("Step 5: Show O' is at least as good as O")
	fmt.Println("  • value(O') ≥ value(O)")
	fmt.Println("  • O' agrees with G at one more position")
	fmt.Println()

	fmt.Println("Step 6: Conclude by induction")
	fmt.Println("  • Repeat the exchange until O' = G")
	fmt.Println("  • Since each exchange doesn't worsen, G is optimal ∎")
	fmt.Println()

	fmt.Println("Common mistakes:")
	fmt.Println("  • Forgetting to check feasibility after exchange")
	fmt.Println("  • Not handling ties properly")
	fmt.Println("  • Assuming exchange works without proving it")
}
```

---

## Key Takeaways

1. Exchange argument: transform OPT into GREEDY without worsening
2. Three templates: adjacent swap, global swap, structural exchange
3. Must verify both feasibility and optimality after exchange
4. Core idea: if greedy's choice is at least as good, prefer it
5. Practice: activity selection, Huffman, scheduling, MST are canonical examples

> **Phase 17 Complete!** Next up: Phase 18 — Intervals →
