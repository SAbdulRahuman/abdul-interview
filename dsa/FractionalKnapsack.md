# Phase 17: Greedy Algorithms — Fractional Knapsack

## Overview

The **Fractional Knapsack** problem allows taking fractions of items. Unlike 0/1 knapsack (which needs DP), greedy works here: sort by value/weight ratio and pick the best ratio first.

| Aspect | Detail |
|--------|--------|
| **Strategy** | Sort by value/weight ratio descending |
| **Time** | O(n log n) |
| **Difference from 0/1** | Can take fractions → greedy works |

---

## Example 1: Classic Fractional Knapsack

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct {
	Weight, Value float64
}

func fractionalKnapsack(items []Item, capacity float64) float64 {
	// Sort by value/weight ratio descending
	sort.Slice(items, func(i, j int) bool {
		return items[i].Value/items[i].Weight > items[j].Value/items[j].Weight
	})

	totalValue := 0.0
	remaining := capacity

	for _, item := range items {
		if remaining <= 0 { break }
		if item.Weight <= remaining {
			totalValue += item.Value
			remaining -= item.Weight
		} else {
			// Take fraction
			totalValue += item.Value * (remaining / item.Weight)
			remaining = 0
		}
	}
	return totalValue
}

func main() {
	items := []Item{
		{10, 60}, {20, 100}, {30, 120},
	}
	fmt.Printf("Max value: %.2f\n", fractionalKnapsack(items, 50))
	// 240: take all of item2(100) + all of item3(120) + 2/3 of item1(20) = wait...
	// Actually: ratio 6, 5, 4 → take item1(60), item2(100), 20/30 of item3(80) = 240
}
```

---

## Example 2: With Detailed Breakdown

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

func knapsackDetailed(items []Item, capacity float64) {
	sort.Slice(items, func(i, j int) bool {
		return items[i].Value/items[i].Weight > items[j].Value/items[j].Weight
	})

	totalValue := 0.0
	remaining := capacity

	fmt.Println("Item selection:")
	for _, item := range items {
		ratio := item.Value / item.Weight
		if remaining <= 0 { break }
		if item.Weight <= remaining {
			fmt.Printf("  %s: take ALL (w=%.0f, v=%.0f, ratio=%.2f)\n",
				item.Name, item.Weight, item.Value, ratio)
			totalValue += item.Value
			remaining -= item.Weight
		} else {
			fraction := remaining / item.Weight
			gained := item.Value * fraction
			fmt.Printf("  %s: take %.1f%% (w=%.0f of %.0f, v=%.1f, ratio=%.2f)\n",
				item.Name, fraction*100, remaining, item.Weight, gained, ratio)
			totalValue += gained
			remaining = 0
		}
	}
	fmt.Printf("\nTotal value: %.2f\n", totalValue)
}

func main() {
	items := []Item{
		{"Gold", 10, 60},
		{"Silver", 20, 100},
		{"Bronze", 30, 120},
	}
	knapsackDetailed(items, 50)
}
```

---

## Example 3: Fractional vs 0/1 Knapsack Comparison

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct {
	Weight, Value int
}

func fractional(items []Item, cap int) float64 {
	type R struct{ w, v int; ratio float64 }
	ratios := make([]R, len(items))
	for i, it := range items {
		ratios[i] = R{it.Weight, it.Value, float64(it.Value) / float64(it.Weight)}
	}
	sort.Slice(ratios, func(i, j int) bool { return ratios[i].ratio > ratios[j].ratio })

	total := 0.0
	rem := float64(cap)
	for _, r := range ratios {
		if rem <= 0 { break }
		if float64(r.w) <= rem {
			total += float64(r.v)
			rem -= float64(r.w)
		} else {
			total += r.ratio * rem
			rem = 0
		}
	}
	return total
}

func zeroOneKnapsack(items []Item, cap int) int {
	n := len(items)
	dp := make([]int, cap+1)
	for i := 0; i < n; i++ {
		for w := cap; w >= items[i].Weight; w-- {
			if dp[w-items[i].Weight]+items[i].Value > dp[w] {
				dp[w] = dp[w-items[i].Weight] + items[i].Value
			}
		}
	}
	return dp[cap]
}

func main() {
	items := []Item{{10, 60}, {20, 100}, {30, 120}}
	cap := 50

	fmt.Printf("Fractional: %.2f (greedy O(n log n))\n", fractional(items, cap))
	fmt.Printf("0/1:        %d (DP O(nW))\n", zeroOneKnapsack(items, cap))

	fmt.Println()
	fmt.Println("Fractional ≥ 0/1 always (more flexibility)")
}
```

---

## Example 4: Maximize Profit Loading Truck

```go
package main

import (
	"fmt"
	"sort"
)

type Box struct {
	Units    int
	Quantity int
}

func maximumUnits(boxes []Box, truckSize int) int {
	// Sort by units per box descending
	sort.Slice(boxes, func(i, j int) bool {
		return boxes[i].Units > boxes[j].Units
	})

	total := 0
	remaining := truckSize
	for _, b := range boxes {
		take := b.Quantity
		if take > remaining { take = remaining }
		total += take * b.Units
		remaining -= take
		if remaining == 0 { break }
	}
	return total
}

func main() {
	boxes := []Box{{1, 3}, {2, 2}, {3, 1}}
	fmt.Println(maximumUnits(boxes, 4)) // 8

	// LC 1710: Maximum Units on a Truck
	boxes2 := []Box{{5, 10}, {2, 5}, {4, 7}, {3, 9}}
	fmt.Println(maximumUnits(boxes2, 10)) // 91
}
```

---

## Example 5: Job Scheduling to Maximize Profit

```go
package main

import (
	"fmt"
	"sort"
)

type Job struct {
	ID       int
	Deadline int
	Profit   int
}

func maxProfitJobs(jobs []Job) (int, []int) {
	sort.Slice(jobs, func(i, j int) bool {
		return jobs[i].Profit > jobs[j].Profit
	})

	maxD := 0
	for _, j := range jobs {
		if j.Deadline > maxD { maxD = j.Deadline }
	}

	slots := make([]int, maxD+1) // 0 means empty
	profit := 0
	scheduled := []int{}

	for _, j := range jobs {
		for t := j.Deadline; t >= 1; t-- {
			if slots[t] == 0 {
				slots[t] = j.ID
				profit += j.Profit
				scheduled = append(scheduled, j.ID)
				break
			}
		}
	}
	return profit, scheduled
}

func main() {
	jobs := []Job{
		{1, 4, 20}, {2, 1, 10}, {3, 1, 40},
		{4, 1, 30}, {5, 3, 50},
	}
	profit, sched := maxProfitJobs(jobs)
	fmt.Println("Max profit:", profit)
	fmt.Println("Jobs:", sched)
}
```

---

## Example 6: Minimize Lateness (Scheduling)

```go
package main

import (
	"fmt"
	"sort"
)

type Task struct {
	Name     string
	Duration int
	Deadline int
}

func minimizeLateness(tasks []Task) int {
	// Sort by deadline (earliest deadline first)
	sort.Slice(tasks, func(i, j int) bool {
		return tasks[i].Deadline < tasks[j].Deadline
	})

	time := 0
	maxLate := 0
	for _, t := range tasks {
		time += t.Duration
		lateness := time - t.Deadline
		if lateness < 0 { lateness = 0 }
		if lateness > maxLate { maxLate = lateness }
		fmt.Printf("  %s: finish=%d, deadline=%d, lateness=%d\n",
			t.Name, time, t.Deadline, lateness)
	}
	return maxLate
}

func main() {
	tasks := []Task{
		{"A", 3, 6}, {"B", 2, 8}, {"C", 1, 9},
		{"D", 4, 9}, {"E", 3, 14}, {"F", 2, 15},
	}
	fmt.Println("Max lateness:", minimizeLateness(tasks))
}
```

---

## Example 7: Greedy Choice Proof for Fractional Knapsack

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Greedy Proof for Fractional Knapsack ===")
	fmt.Println()

	fmt.Println("Claim: Taking items by decreasing value/weight ratio is optimal.")
	fmt.Println()

	fmt.Println("Proof (Exchange Argument):")
	fmt.Println("  1. Let G be greedy solution, O be any optimal solution")
	fmt.Println("  2. If G ≠ O, find first item i where they differ")
	fmt.Println("  3. G takes more of item i (highest remaining ratio)")
	fmt.Println("  4. O takes less of item i, more of some item j with lower ratio")
	fmt.Println("  5. Swap: reduce j in O, increase i in O by same weight")
	fmt.Println("  6. Since ratio(i) ≥ ratio(j), value doesn't decrease")
	fmt.Println("  7. Repeat until O = G")
	fmt.Println("  8. Therefore G is optimal ∎")
	fmt.Println()

	fmt.Println("Why this works for fractional but NOT 0/1:")
	fmt.Println("  Fractional: can take any amount → smooth exchange")
	fmt.Println("  0/1: must take all or nothing → exchange may violate capacity")
}
```

---

## Example 8: Minimum Cost to Fill Bag

```go
package main

import (
	"fmt"
	"sort"
)

type Package struct {
	Weight int
	Cost   int
}

func minCostFillBag(packages []Package, target int) float64 {
	// Sort by cost/weight ratio ascending (cheapest per unit first)
	sort.Slice(packages, func(i, j int) bool {
		ri := float64(packages[i].Cost) / float64(packages[i].Weight)
		rj := float64(packages[j].Cost) / float64(packages[j].Weight)
		return ri < rj
	})

	totalCost := 0.0
	remaining := float64(target)

	for _, p := range packages {
		if remaining <= 0 { break }
		if float64(p.Weight) <= remaining {
			totalCost += float64(p.Cost)
			remaining -= float64(p.Weight)
		} else {
			ratio := float64(p.Cost) / float64(p.Weight)
			totalCost += ratio * remaining
			remaining = 0
		}
	}

	if remaining > 0 { return -1 } // can't fill
	return totalCost
}

func main() {
	packages := []Package{
		{5, 10}, {10, 15}, {20, 25},
	}
	fmt.Printf("Min cost to fill 30: %.2f\n", minCostFillBag(packages, 30))
}
```

---

## Example 9: Activity Selection by Value/Time Ratio

```go
package main

import (
	"fmt"
	"sort"
)

type Task struct {
	Name     string
	Time     int
	Value    int
}

func maxValueTasks(tasks []Task, totalTime int) (int, []string) {
	sort.Slice(tasks, func(i, j int) bool {
		ri := float64(tasks[i].Value) / float64(tasks[i].Time)
		rj := float64(tasks[j].Value) / float64(tasks[j].Time)
		return ri > rj
	})

	value := 0
	remaining := totalTime
	selected := []string{}

	for _, t := range tasks {
		if remaining <= 0 { break }
		if t.Time <= remaining {
			value += t.Value
			remaining -= t.Time
			selected = append(selected, t.Name)
		}
	}
	return value, selected
}

func main() {
	tasks := []Task{
		{"Code", 4, 100}, {"Test", 2, 40}, {"Design", 3, 80},
		{"Deploy", 1, 30}, {"Review", 2, 50},
	}
	val, sel := maxValueTasks(tasks, 8)
	fmt.Printf("Max value: %d, Tasks: %v\n", val, sel)
}
```

---

## Example 10: Fractional Knapsack Variants Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Fractional Knapsack — Variants & Summary ===")
	fmt.Println()

	fmt.Println("Core idea:")
	fmt.Println("  Sort by value/weight ratio → take greedily")
	fmt.Println("  Works because fractions are allowed")
	fmt.Println()

	fmt.Println("Variants:")
	fmt.Println("  1. Classic: maximize value within weight limit")
	fmt.Println("  2. Minimize cost: sort by cost/weight ascending")
	fmt.Println("  3. Load truck: sort by value descending")
	fmt.Println("  4. Task scheduling: value/time ratio")
	fmt.Println("  5. Resource allocation: efficiency ratio")
	fmt.Println()

	fmt.Println("Comparison:")
	fmt.Println("  ┌────────────────────┬───────────────┬──────────────┐")
	fmt.Println("  │ Problem            │ Algorithm     │ Time         │")
	fmt.Println("  ├────────────────────┼───────────────┼──────────────┤")
	fmt.Println("  │ Fractional Knapsack│ Greedy        │ O(n log n)   │")
	fmt.Println("  │ 0/1 Knapsack       │ DP            │ O(nW)        │")
	fmt.Println("  │ Unbounded Knapsack │ DP            │ O(nW)        │")
	fmt.Println("  └────────────────────┴───────────────┴──────────────┘")
	fmt.Println()

	fmt.Println("Key insight: Fractional ≥ 0/1 ≥ Bounded")
	fmt.Println("More flexibility → higher optimal value")
}
```

---

## Key Takeaways

1. Sort by value/weight ratio descending — take greedily
2. Works because fractions are allowed (exchange argument proof)
3. O(n log n) time — much faster than DP for 0/1 knapsack
4. Fractional knapsack value ≥ 0/1 knapsack value always
5. Same pattern applies: minimize cost, maximize tasks, etc.

> **Next up:** Greedy vs DP →
