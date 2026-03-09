# Phase 14: Backtracking — Combinations

## Overview

A **combination** is a selection of elements where **order doesn't matter**. For choosing `k` from `n` elements, there are C(n,k) = n!/(k!(n-k)!) combinations. Backtracking generates combinations by always choosing from elements ahead of the current position.

| Pattern | Key Insight |
|---------|-------------|
| Subsets (all sizes) | Every node in the recursion tree is a valid answer |
| k-combinations | Only leaves at depth k are valid answers |
| Combination Sum | Variable depth; stop when sum matches target |
| With duplicates | Sort + skip equal elements at same recursion level |

---

## Example 1: Combinations of k from n (LeetCode 77)

```go
package main

import "fmt"

func combine(n int, k int) [][]int {
	result := [][]int{}

	var backtrack func(start int, path []int)
	backtrack = func(start int, path []int) {
		if len(path) == k {
			result = append(result, append([]int{}, path...))
			return
		}

		// Pruning: need k-len(path) more elements, so stop when not enough remain
		for i := start; i <= n-(k-len(path))+1; i++ {
			path = append(path, i)
			backtrack(i+1, path)
			path = path[:len(path)-1]
		}
	}

	backtrack(1, []int{})
	return result
}

func main() {
	result := combine(4, 2)
	for _, c := range result { fmt.Println(c) }
	fmt.Println("Total:", len(result)) // C(4,2) = 6
}
```

---

## Example 2: All Subsets (LeetCode 78)

```go
package main

import "fmt"

func subsets(nums []int) [][]int {
	result := [][]int{}

	var backtrack func(start int, path []int)
	backtrack = func(start int, path []int) {
		// Every partial path is a valid subset
		result = append(result, append([]int{}, path...))

		for i := start; i < len(nums); i++ {
			path = append(path, nums[i])
			backtrack(i+1, path)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, []int{})
	return result
}

func main() {
	result := subsets([]int{1, 2, 3})
	for _, s := range result { fmt.Println(s) }
	fmt.Println("Total:", len(result)) // 2^3 = 8
}
```

---

## Example 3: Subsets With Duplicates (LeetCode 90)

```go
package main

import (
	"fmt"
	"sort"
)

func subsetsWithDup(nums []int) [][]int {
	sort.Ints(nums)
	result := [][]int{}

	var backtrack func(start int, path []int)
	backtrack = func(start int, path []int) {
		result = append(result, append([]int{}, path...))

		for i := start; i < len(nums); i++ {
			// Skip duplicates at the same level
			if i > start && nums[i] == nums[i-1] { continue }

			path = append(path, nums[i])
			backtrack(i+1, path)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, []int{})
	return result
}

func main() {
	result := subsetsWithDup([]int{1, 2, 2})
	for _, s := range result { fmt.Println(s) }
	// [] [1] [1 2] [1 2 2] [2] [2 2]
}
```

---

## Example 4: Combination Sum (LeetCode 39) — Reuse Allowed

```go
package main

import (
	"fmt"
	"sort"
)

func combinationSum(candidates []int, target int) [][]int {
	sort.Ints(candidates)
	result := [][]int{}

	var backtrack func(start, remaining int, path []int)
	backtrack = func(start, remaining int, path []int) {
		if remaining == 0 {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := start; i < len(candidates); i++ {
			if candidates[i] > remaining { break }
			path = append(path, candidates[i])
			backtrack(i, remaining-candidates[i], path) // i, not i+1 (reuse)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, target, []int{})
	return result
}

func main() {
	fmt.Println(combinationSum([]int{2, 3, 6, 7}, 7))
	// [[2 2 3] [7]]
}
```

---

## Example 5: Combination Sum II — No Reuse (LeetCode 40)

```go
package main

import (
	"fmt"
	"sort"
)

func combinationSum2(candidates []int, target int) [][]int {
	sort.Ints(candidates)
	result := [][]int{}

	var backtrack func(start, remaining int, path []int)
	backtrack = func(start, remaining int, path []int) {
		if remaining == 0 {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := start; i < len(candidates); i++ {
			if candidates[i] > remaining { break }
			if i > start && candidates[i] == candidates[i-1] { continue }

			path = append(path, candidates[i])
			backtrack(i+1, remaining-candidates[i], path) // i+1 (no reuse)
			path = path[:len(path)-1]
		}
	}

	backtrack(0, target, []int{})
	return result
}

func main() {
	fmt.Println(combinationSum2([]int{10, 1, 2, 7, 6, 1, 5}, 8))
	// [[1 1 6] [1 2 5] [1 7] [2 6]]
}
```

---

## Example 6: Combination Sum III — k Numbers Summing to n (LeetCode 216)

```go
package main

import "fmt"

func combinationSum3(k int, n int) [][]int {
	result := [][]int{}

	var backtrack func(start, remaining, count int, path []int)
	backtrack = func(start, remaining, count int, path []int) {
		if count == k {
			if remaining == 0 {
				result = append(result, append([]int{}, path...))
			}
			return
		}

		for i := start; i <= 9; i++ {
			if i > remaining { break }

			path = append(path, i)
			backtrack(i+1, remaining-i, count+1, path)
			path = path[:len(path)-1]
		}
	}

	backtrack(1, n, 0, []int{})
	return result
}

func main() {
	fmt.Println(combinationSum3(3, 7))  // [[1 2 4]]
	fmt.Println(combinationSum3(3, 9))  // [[1 2 6] [1 3 5] [2 3 4]]
}
```

---

## Example 7: Subsets Using Bitmask (Iterative)

```go
package main

import "fmt"

func subsetsBitmask(nums []int) [][]int {
	n := len(nums)
	total := 1 << n
	result := make([][]int, 0, total)

	for mask := 0; mask < total; mask++ {
		subset := []int{}
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				subset = append(subset, nums[i])
			}
		}
		result = append(result, subset)
	}
	return result
}

func main() {
	result := subsetsBitmask([]int{1, 2, 3})
	for _, s := range result { fmt.Println(s) }
	fmt.Println("Total:", len(result))
}
```

**Why?** Each integer from 0 to 2^n-1 represents a unique subset via its bits. No recursion needed.

---

## Example 8: Subsets of Specific Size — Gosper's Hack

```go
package main

import "fmt"

// Gosper's hack: iterate through all bitmasks with exactly k bits set
func combinationsGosper(n, k int) [][]int {
	result := [][]int{}
	if k == 0 {
		return append(result, []int{})
	}

	mask := (1 << k) - 1 // smallest mask with k bits
	limit := 1 << n

	for mask < limit {
		// Decode mask to indices
		subset := []int{}
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				subset = append(subset, i+1)
			}
		}
		result = append(result, subset)

		// Gosper's hack: next mask with same number of bits
		c := mask & (-mask)       // lowest set bit
		r := mask + c             // carry into next group
		mask = (((r ^ mask) >> 2) / c) | r
	}
	return result
}

func main() {
	result := combinationsGosper(5, 3)
	for _, c := range result { fmt.Println(c) }
	fmt.Println("Total:", len(result)) // C(5,3) = 10
}
```

---

## Example 9: Count Combinations With Target Sum

```go
package main

import "fmt"

func countCombinations(nums []int, target int) int {
	count := 0

	var backtrack func(start, sum int)
	backtrack = func(start, sum int) {
		if sum == target {
			count++
			return
		}
		if sum > target { return }

		for i := start; i < len(nums); i++ {
			backtrack(i+1, sum+nums[i])
		}
	}

	backtrack(0, 0)
	return count
}

func main() {
	nums := []int{1, 2, 3, 4, 5}

	for t := 1; t <= 15; t++ {
		c := countCombinations(nums, t)
		if c > 0 {
			fmt.Printf("Target %2d: %d combinations\n", t, c)
		}
	}
}
```

---

## Example 10: Combinations Concepts Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Combinations Summary ===")
	fmt.Println()

	patterns := []struct{ problem, key, time string }{
		{"C(n,k)", "start from i+1, stop at depth k", "O(C(n,k)·k)"},
		{"All subsets", "record at every node", "O(2^n·n)"},
		{"Subsets w/ dups", "sort + skip same at level", "O(2^n·n)"},
		{"Combo sum (reuse)", "recurse with i (not i+1)", "O(2^t)"},
		{"Combo sum (no reuse)", "recurse with i+1", "O(2^n)"},
		{"Combo sum III", "digits 1-9, k picks, sum=n", "O(C(9,k))"},
		{"Bitmask subsets", "iterate 0..2^n-1", "O(2^n·n)"},
		{"Gosper's hack", "next bitmask with k bits", "O(C(n,k))"},
	}

	fmt.Printf("%-22s %-35s %s\n", "Problem", "Key Technique", "Time")
	fmt.Println(string(make([]byte, 72)))
	for _, p := range patterns {
		fmt.Printf("%-22s %-35s %s\n", p.problem, p.key, p.time)
	}

	fmt.Println()
	fmt.Println("Core patterns:")
	fmt.Println("  1. Always iterate from 'start' to avoid duplicates")
	fmt.Println("  2. Reuse allowed → recurse with i")
	fmt.Println("  3. No reuse → recurse with i+1")
	fmt.Println("  4. Duplicates → sort + skip if nums[i]==nums[i-1] at same level")
}
```

---

## Key Takeaways

1. Combinations = subsets where order doesn't matter → always start from `i` or `i+1`
2. Pruning `i <= n-(k-len(path))+1` avoids exploring branches with too few remaining elements
3. Reuse allowed: pass `i` to next level; no reuse: pass `i+1`
4. Bitmask approach: iterate integers 0 to 2^n-1 for all subsets (no recursion)
5. Gosper's hack efficiently enumerates all bitmasks with exactly k bits set

> **Next up:** Constraint Satisfaction →
