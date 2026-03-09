# Phase 10: Binary Search — Minimize Maximum / Maximize Minimum Pattern

## Overview

Two of the most common binary search on answer patterns:

| Pattern | Goal | Binary Search |
|---------|------|---------------|
| **Minimize Maximum** | Make the largest value as small as possible | Minimize: `hi = mid` |
| **Maximize Minimum** | Make the smallest value as large as possible | Maximize: `lo = mid` |

Both require a **feasibility check** that is monotonic in the candidate answer.

---

## Example 1: Minimize Maximum — Split Array Largest Sum (LeetCode 410)

```go
package main

import "fmt"

func splitArray(nums []int, k int) int {
	lo, hi := 0, 0
	for _, n := range nums {
		if n > lo { lo = n }
		hi += n
	}
	// Minimize the maximum subarray sum
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canSplitWith(nums, k, mid) {
			hi = mid // can do with this max → try smaller
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canSplitWith(nums []int, k, maxSum int) bool {
	groups, cur := 1, 0
	for _, n := range nums {
		if cur+n > maxSum {
			groups++
			cur = 0
		}
		cur += n
	}
	return groups <= k
}

func main() {
	fmt.Println(splitArray([]int{7, 2, 5, 10, 8}, 2))  // 18
	fmt.Println(splitArray([]int{1, 2, 3, 4, 5}, 2))   // 9
	fmt.Println(splitArray([]int{1, 4, 4}, 3))           // 4
}
```

---

## Example 2: Maximize Minimum — Aggressive Cows

```go
package main

import (
	"fmt"
	"sort"
)

func aggressiveCows(stalls []int, cows int) int {
	sort.Ints(stalls)
	lo, hi := 1, stalls[len(stalls)-1]-stalls[0]
	// Maximize the minimum distance between cows
	for lo < hi {
		mid := lo + (hi-lo+1)/2 // upper mid for maximize
		if canPlaceWith(stalls, cows, mid) {
			lo = mid // feasible → try larger
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func canPlaceWith(stalls []int, cows, minDist int) bool {
	placed, last := 1, stalls[0]
	for i := 1; i < len(stalls); i++ {
		if stalls[i]-last >= minDist {
			placed++
			last = stalls[i]
		}
	}
	return placed >= cows
}

func main() {
	fmt.Println(aggressiveCows([]int{1, 2, 4, 8, 9}, 3))     // 3
	fmt.Println(aggressiveCows([]int{1, 2, 8, 4, 9}, 3))     // 3
	fmt.Println(aggressiveCows([]int{0, 3, 4, 7, 10, 9}, 4)) // 3
}
```

---

## Example 3: Minimize Maximum — Capacity to Ship Packages (LeetCode 1011)

```go
package main

import "fmt"

func shipWithinDays(weights []int, days int) int {
	lo, hi := 0, 0
	for _, w := range weights {
		if w > lo { lo = w }
		hi += w
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		d, cur := 1, 0
		for _, w := range weights {
			if cur+w > mid {
				d++
				cur = 0
			}
			cur += w
		}
		if d <= days {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(shipWithinDays([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, 5)) // 15
	fmt.Println(shipWithinDays([]int{3, 2, 2, 4, 1, 4}, 3))                // 6
}
```

---

## Example 4: Maximize Minimum — Magnetic Force (LeetCode 1552)

```go
package main

import (
	"fmt"
	"sort"
)

func maxDistance(position []int, m int) int {
	sort.Ints(position)
	lo, hi := 1, position[len(position)-1]-position[0]
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		cnt, last := 1, position[0]
		for i := 1; i < len(position); i++ {
			if position[i]-last >= mid {
				cnt++
				last = position[i]
			}
		}
		if cnt >= m {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(maxDistance([]int{1, 2, 3, 4, 7}, 3))                // 3
	fmt.Println(maxDistance([]int{5, 4, 3, 2, 1, 1000000000}, 2))    // 999999999
}
```

---

## Example 5: Minimize Maximum — Allocate Mailboxes (LeetCode 1478 Variant)

```go
package main

import (
	"fmt"
	"sort"
)

// Simplified: minimize the maximum distance any house is from its assigned worker
func minimizeMaxDist(houses []int, workers int) int {
	sort.Ints(houses)
	lo, hi := 0, houses[len(houses)-1]-houses[0]
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canCover(houses, workers, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canCover(houses []int, workers, maxDist int) bool {
	w := 1
	start := houses[0]
	for _, h := range houses {
		if h-start > 2*maxDist { // diameter = 2*radius
			w++
			start = h
		}
	}
	return w <= workers
}

func main() {
	fmt.Println(minimizeMaxDist([]int{1, 4, 8, 10, 20}, 3)) // 3
}
```

---

## Example 6: Minimize Maximum — Minimize Maximum Difference of Pairs (LeetCode 2616)

```go
package main

import (
	"fmt"
	"sort"
)

func minimizeMax(nums []int, p int) int {
	sort.Ints(nums)
	n := len(nums)
	lo, hi := 0, nums[n-1]-nums[0]
	for lo < hi {
		mid := lo + (hi-lo)/2
		// Greedy: count pairs with diff <= mid
		count := 0
		i := 0
		for i < n-1 {
			if nums[i+1]-nums[i] <= mid {
				count++
				i += 2 // use both elements
			} else {
				i++
			}
		}
		if count >= p {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(minimizeMax([]int{10, 1, 2, 7, 1, 3}, 2)) // 1
	fmt.Println(minimizeMax([]int{4, 2, 1, 2}, 1))          // 0
}
```

---

## Example 7: Maximize Minimum — Equal Piles After Operations

```go
package main

import "fmt"

// Maximize the minimum pile size after removing at most k total elements
func maxMinPile(piles []int, k int) int {
	lo, hi := 0, 0
	for _, p := range piles {
		if p > hi { hi = p }
	}
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		// Check: can we make all piles >= mid using at most k additions?
		needed := 0
		for _, p := range piles {
			if p < mid {
				needed += mid - p
			}
		}
		if needed <= k {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(maxMinPile([]int{1, 2, 3, 4, 5}, 3))   // 3
	fmt.Println(maxMinPile([]int{1, 10, 1, 10, 1}, 10)) // 5
}
```

---

## Example 8: Minimize Maximum — Wood Cutting (Classic)

```go
package main

import "fmt"

// Cut wood pieces to get at least k pieces of length mid
// Maximize the length of each piece
func woodCutting(logs []int, k int) int {
	lo, hi := 0, 0
	for _, l := range logs {
		if l > hi { hi = l }
	}
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		pieces := 0
		for _, l := range logs {
			pieces += l / mid
		}
		if pieces >= k {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(woodCutting([]int{232, 124, 456}, 7))  // 114
	fmt.Println(woodCutting([]int{20, 15, 10, 17}, 7)) // 8
}
```

---

## Example 9: Minimize Maximum — Divide Chocolate (LeetCode 1231)

```go
package main

import "fmt"

// Maximize the minimum total sweetness (you get the smallest piece)
func maximizeSweetness(sweetness []int, k int) int {
	lo, hi := 0, 0
	for _, s := range sweetness {
		if s < lo || lo == 0 { lo = s }
		hi += s
	}
	lo = 0
	hi = hi / (k + 1)

	for lo < hi {
		mid := lo + (hi-lo+1)/2
		pieces, cur := 0, 0
		for _, s := range sweetness {
			cur += s
			if cur >= mid {
				pieces++
				cur = 0
			}
		}
		if pieces >= k+1 {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(maximizeSweetness([]int{1, 2, 3, 4, 5, 6, 7, 8, 9}, 5)) // 6
	fmt.Println(maximizeSweetness([]int{5, 6, 7, 8, 9, 1, 2, 3, 4}, 8)) // 1
}
```

---

## Example 10: Pattern Recognition Guide

```go
package main

import "fmt"

/*
 Pattern recognition for Min-Max / Max-Min binary search:
 
 1. "Minimize the maximum ___"
    → Binary search: hi = mid when feasible
    → Answer range: [max single element, total sum]
    → Example: Split array, ship packages, painter's partition
 
 2. "Maximize the minimum ___"
    → Binary search: lo = mid when feasible (use upper mid!)
    → Answer range: [0 or 1, max possible gap]
    → Example: Aggressive cows, magnetic force, cutting ribbons
 
 3. How to identify?
    → Keywords: "minimize the largest", "maximize the smallest"
    → Partitioning/distributing items
    → Placing items with spacing constraints
*/

// Unified Template
func solve(problemType string, lo, hi int, feasible func(int) bool) int {
	switch problemType {
	case "minimize":
		for lo < hi {
			mid := lo + (hi-lo)/2
			if feasible(mid) {
				hi = mid
			} else {
				lo = mid + 1
			}
		}
	case "maximize":
		for lo < hi {
			mid := lo + (hi-lo+1)/2
			if feasible(mid) {
				lo = mid
			} else {
				hi = mid - 1
			}
		}
	}
	return lo
}

func main() {
	// Minimize: smallest x where x*x >= 100
	ans := solve("minimize", 0, 100, func(x int) bool {
		return x*x >= 100
	})
	fmt.Println("Minimize:", ans) // 10

	// Maximize: largest x where x*x <= 100
	ans = solve("maximize", 0, 100, func(x int) bool {
		return x*x <= 100
	})
	fmt.Println("Maximize:", ans) // 10
}
```

---

## Summary Table

| Problem | Type | Range | Feasibility |
|---------|------|-------|-------------|
| Split Array (LC 410) | Min-Max | `[max, sum]` | ≤ k groups with sum ≤ mid |
| Ship Packages (LC 1011) | Min-Max | `[max, sum]` | ≤ d days with cap mid |
| Aggressive Cows | Max-Min | `[1, range]` | Place c cows with gap ≥ mid |
| Magnetic Force (LC 1552) | Max-Min | `[1, range]` | Place m balls with gap ≥ mid |
| Koko Bananas (LC 875) | Minimize | `[1, max]` | Eat all in ≤ h hours at speed mid |
| Wood Cutting | Maximize | `[0, max]` | Get ≥ k pieces of length mid |

## Key Takeaways

1. **Minimize maximum** → `hi = mid` when feasible → find smallest feasible value
2. **Maximize minimum** → `lo = mid` when feasible (upper mid!) → find largest feasible value
3. Both always have bounded answer space and monotonic feasibility
4. The feasibility check is greedy — O(n) per check, O(n log range) total
5. When confused, ask: "If I increase the answer, does feasibility get easier or harder?"

> **Phase 10 Complete!** Next up: Phase 11 — Heap / Priority Queue →
