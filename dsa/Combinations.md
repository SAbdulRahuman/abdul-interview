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

**Textual Figure:**

```
n = 4, k = 2   C(4,2) = 6

  Recursion tree (always pick from i+1 onward, stop at depth k):
  ───────────────────────────────────────────────────────
                        []  start=1
         ┌───────┬─────┼─────┬───────┐
        +1      +2       +3       +4✂
       start=2 start=3  start=4  PRUNED
     ┌──┼──┐   ┌─┴─┐    │     (need 1 more
    +2  +3  +4  +3  +4   +4      but 0 remain)
     │   │   │   │   │    │
   [1,2][1,3][1,4][2,3][2,4][3,4]
     ✓    ✓    ✓    ✓    ✓    ✓

  Pruning: i ≤ n-(k-len(path))+1
    At root: i ≤ 4-(2-0)+1 = 3  → try 1,2,3 (not 4)
    At [1]: i ≤ 4-(2-1)+1 = 4   → try 2,3,4 (all ok)

  Result: [1,2] [1,3] [1,4] [2,3] [2,4] [3,4]  (6 total)
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

**Textual Figure:**

```
nums = [1, 2, 3]   All subsets (every node is a valid answer)

  Recursion tree:
  ───────────────────────────────────────────────────────
                     []✓  start=0
           ┌─────────┼─────────┐
         +1✓          +2✓         +3✓
        start=1      start=2     start=3
      ┌───┴───┐     ┌──┴──┐        │
    +2✓     +3✓    +3✓     │       (no more)
   start=2  start=3 start=3 │
     │        │      │     │
    +3✓     (end)   (end)  │
     │                     │
   (end)                   │

  Nodes visited and subsets collected:
    []  [1]  [1,2]  [1,2,3]  [1,3]  [2]  [2,3]  [3]

  Total: 2³ = 8 subsets
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

**Textual Figure:**

```
nums = [1, 2, 2]  (sorted)

  Without duplicate skipping (would produce 8):
  ──────────────────────────────────────
                   []
         ┌────────┼────────┐
        +1        +2₀       +2₁
     ┌──┴──┐   ┌─┴─┐     │
   +2₀   +2₁   +2₁    │    (end)
    │     │     │     │
   +2₁  (end)  (end)  │
  [1,2,2][1,2][1,2]  [2,2][2]← DUP!
                      ^^^DUP of +2₀ path!

  With duplicate skipping (i > start && nums[i]==nums[i-1]):
  ──────────────────────────────────────
                   []✓
         ┌────────┼────────┐
        +1✓       +2₀✓      +2₁ ✂ SKIP
     ┌──┴──┐    ┌─┴─┐    (2₁==2₀ && i>start)
   +2₀✓   +2₁│   +2₁✓    │
    │    SKIP││    │      │
   +2₁✓        (end)    │
    │                    │
  (end)                  │

  Result: [] [1] [1,2] [1,2,2] [2] [2,2]  (6 unique subsets)
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

**Textual Figure:**

```
candidates = [2, 3, 6, 7], target = 7

  Recursion tree (reuse allowed: recurse with i, not i+1):
  ─────────────────────────────────────────────────────────
                       rem=7
         ┌───────┬─────┼─────┬───────┐
        +2       +3       +6      +7
       rem=5    rem=4    rem=1   rem=0 ✓
      ┌──┼──┐   ┌─┴─┐    │     [7] ✓
    +2  +3  +6  +3 +6   +6│+7
   r=3 r=2 <0│  r=1 <0  ││ <0
    │   │ ││  ││ ││  ││││
   +2  +3 │✂  ││ ││  ││││
   r=1 r=-1│  +3│ ││  ││││
    │  ││││  r=-2 ││  all│││
   +2  ✂│││  ││  ││  >rem│││
   r=-1   │  ││  ││  ✂   │││
    ││    │  ││  ││       │││
    ✂     │  ││  ││       │││
   ┌┴─────┘  ││  ││       │││
   +3       ││  ││       │││
   rem=0 ✓  ││  ││       │││
   [2,2,3]  all pruned   all pruned

  Result: [2,2,3] and [7]   (2 solutions)
  Key: +2 recurses with i=0 (reuse), not i+1
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

**Textual Figure:**

```
candidates = [1, 1, 2, 5, 6, 7, 10] (sorted), target = 8

  Recursion tree (no reuse: i+1, skip dups at same level):
  ───────────────────────────────────────────────────────
                         rem=8
         ┌──────┬────┼─────┬────┬────┐
        +1₀     +1₁│   +2    +5   +6   +7  +10│
       rem=7  SKIP││ rem=6 rem=3 rem=2 rem=1 >8│
        │    (dup)││   │    │     │     │    ││
      ┌─┴─┐       │ ...  +5   +6  +7│   ││
    +1₁  +2       │      r=-2  r=-4 r=-6  ││
    r=6  r=5      │       ✂    ││    ✂    ││
     │    │       │            +6           ││
   +2  +5  +5     │            rem=0 ✓      ││
   r=4 rem=0✓ r=0✓│            [2,6]        ││
    │  [1,2,5] │  │                         ││
   +5     [1,1,6]←┘                         ││
   rem=0✓ via +6                             ││
    │                                        ││
   wait→ +6, r=-1 not valid                  ││
   path: +1₀+1₁+6=8 ✓                       ││
   path: +1₀+7=8 ✓ via +7 branch             ││

  Result: [1,1,6] [1,2,5] [1,7] [2,6]   (4 solutions)
  Key: i+1 (no reuse) + skip if nums[i]==nums[i-1] at same level
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

**Textual Figure:**

```
k = 3, n = 9   (pick 3 digits from 1–9 that sum to 9)

  Recursion tree (start=1, digits 1–9, stop at count=k):
  ──────────────────────────────────────────────────────────
                      rem=9, count=0
         ┌──────┬─────┼─────┬─────┬────┐
        +1      +2      +3     +4    +5...+9│
       rem=8   rem=7   rem=6  rem=5  i>rem │
        │       │       │      │    PRUNE││
      ┌─┴─┐   ┌─┴─┐   ┌─┴─┐  ┌┴┐        ││
    +2  +3   +3  +4   +4 +5  +5 │        ││
    r=6 r=5  r=4 r=3  r=2 r=1 r=0│       ││
     │   │    │   │    │   │  count=2│      ││
    +6  +5   +4  +3   │   │  but k=3 │      ││
    r=0✓ r=0✓ r=0│  r=0✓ ...      need 1   ││
  [1,2,6][1,3,5] │  [2,3,4]      more     ││
                 +5                         ││
                r=-3 ✂ PRUNE                ││

  Result: [1,2,6] [1,3,5] [2,3,4]   (3 solutions)
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

**Textual Figure:**

```
nums = [1, 2, 3]   Bitmask subsets (n=3, 2³=8 masks)

  ┌───────┬─────────┬─────────────┬────────────┐
  │ mask  │ binary  │ bits set    │ subset     │
  ├───────┼─────────┼─────────────┼────────────┤
  │   0   │  000    │ none        │ []         │
  │   1   │  001    │ bit 0       │ [1]        │
  │   2   │  010    │ bit 1       │ [2]        │
  │   3   │  011    │ bits 0,1    │ [1,2]      │
  │   4   │  100    │ bit 2       │ [3]        │
  │   5   │  101    │ bits 0,2    │ [1,3]      │
  │   6   │  110    │ bits 1,2    │ [2,3]      │
  │   7   │  111    │ bits 0,1,2  │ [1,2,3]    │
  └───────┴─────────┴─────────────┴────────────┘

  Check: mask & (1<<i) != 0  → include nums[i]
    mask=5 (101): bit0=1→+1, bit1=0, bit2=1→+3 → [1,3]

  Total: 8 subsets (no recursion needed)
```

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

**Textual Figure:**

```
n = 5, k = 3   Gosper's Hack (iterate masks with exactly 3 bits set)

  Trace of mask transitions (binary, 5 bits):
  ────────────────────────────────────────────
  Step  mask(bin)  Subset       Gosper step
  ────────────────────────────────────────────
   1    00111      {1,2,3}     initial: (1<<3)-1
   2    01011      {1,2,4}     c=1, r=8, next
   3    01101      {1,3,4}     c=1, r=8, next
   4    01110      {2,3,4}     c=2, r=8, next
   5    10011      {1,2,5}     c=1, r=16, next
   6    10101      {1,3,5}     c=1, r=16, next
   7    10110      {2,3,5}     c=2, r=16, next
   8    11001      {1,4,5}     c=1, r=16, next
   9    11010      {2,4,5}     c=2, r=16, next
  10    11100      {3,4,5}     c=4, r=16, next
  11    100000... ≥ 2⁵ → STOP

  Algorithm: c = mask & (-mask)     ← lowest set bit
             r = mask + c           ← carry propagation
             mask = ((r^mask)>>2)/c | r

  Result: 10 subsets = C(5,3) ✓
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

**Textual Figure:**

```
nums = [1, 2, 3, 4, 5], counting subsets that sum to target=6

  Recursion tree with sum pruning:
  ──────────────────────────────────────────────────────────
                     start=0, sum=0
         ┌──────┬────┼────┬────┐
        +1     +2     +3    +4   +5
       s=1    s=2    s=3   s=4   s=5
      ┌─┼─┐  ┌─┼─┐  ┌┴┐  ┌┴┐   │
    +2 +3+4  +3+4 +5 +4+5 +5 │  (end)
    s=3 s=4s=5 s=5s=6 s=8 s=7s=8s=9
     │   │  │  │ s=6✓ >6 >6 >6 >6
    +3 +4+5  +4 +5 count! ✂  ✂  ✂  │
    s=6✓ s=7>6 s=7 >6          │
   count! ✂     │              │
    {1,2,3} {2,4}←┘              │
                                 │
  Found subsets summing to 6:    │
    {1,2,3}: 1+2+3 = 6 ✓        │
    {2,4}:   2+4   = 6 ✓        │
    {1,5}:   1+5   = 6 ✓        │

  Result: Target 6 has 3 combinations
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

**Textual Figure:**

```
  Combination Patterns Decision Tree
  ═════════════════════════════════════════════

              ┌─────────────────┐
              │ Order matters?  │
              └────────┬────────┘
             YES│          │NO
              │           │
        PERMUTATION    COMBINATION
              │           │
        ┌─────┴─────┐  │
        │ Use        │  ┌────┴───────────┐
        │ Permutations│  │ Reuse allowed?   │
        └───────────┘  └────┬───────────┘
                         YES│        │NO
                          │         │
                    recurse(i)  recurse(i+1)
                          │         │
                    ┌─────┴─────┐  │
                    │ Duplicates?  │  │
                    └─────┬─────┘  │
                       YES│   NO│   │
                        │     │    │
                    sort+skip  │   sort+skip
                    at level   │   at level

  ┌───────────────┬───────────────┬───────────────┐
  │   ALL SUBSETS  │   BITMASK      │  GOSPER'S HACK │
  │   O(2ⁿ·n)      │   O(2ⁿ·n)      │  O(C(n,k))     │
  │   Record at    │   No recursion │  k-bit masks   │
  │   every node   │   iterate ints  │  only          │
  └───────────────┴───────────────┴───────────────┘
```

---

## Key Takeaways

1. Combinations = subsets where order doesn't matter → always start from `i` or `i+1`
2. Pruning `i <= n-(k-len(path))+1` avoids exploring branches with too few remaining elements
3. Reuse allowed: pass `i` to next level; no reuse: pass `i+1`
4. Bitmask approach: iterate integers 0 to 2^n-1 for all subsets (no recursion)
5. Gosper's hack efficiently enumerates all bitmasks with exactly k bits set

> **Next up:** Constraint Satisfaction →
