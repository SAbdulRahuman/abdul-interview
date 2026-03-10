# Phase 14: Backtracking — Permutations

## Overview

A **permutation** is an arrangement of elements in a specific order. For `n` elements, there are `n!` permutations. Backtracking generates permutations by making choices at each position and undoing them to explore alternatives.

| Variant | Count | Notes |
|---------|-------|-------|
| All permutations of `n` distinct | `n!` | Classic swap/used-array approach |
| Permutations with duplicates | `n!/∏(count_i!)` | Sort + skip duplicates |
| k-permutations of n | `n!/(n-k)!` | Stop when path length = k |
| Next permutation | 1 | Lexicographic order algorithm |

---

## Example 1: All Permutations — Swap Approach (LeetCode 46)

```go
package main

import "fmt"

func permute(nums []int) [][]int {
	result := [][]int{}

	var backtrack func(start int)
	backtrack = func(start int) {
		if start == len(nums) {
			perm := make([]int, len(nums))
			copy(perm, nums)
			result = append(result, perm)
			return
		}

		for i := start; i < len(nums); i++ {
			nums[start], nums[i] = nums[i], nums[start] // choose
			backtrack(start + 1)                         // explore
			nums[start], nums[i] = nums[i], nums[start] // unchoose
		}
	}

	backtrack(0)
	return result
}

func main() {
	result := permute([]int{1, 2, 3})
	for _, p := range result {
		fmt.Println(p)
	}
	fmt.Println("Total:", len(result)) // 6
}
```

**Textual Figure:**

```
nums = [1, 2, 3]   Swap approach

  Recursion tree (swap nums[start] with each nums[i] where i ≥ start):
  ───────────────────────────────────────────────────────────────
                         [1, 2, 3]  start=0
              ┌────────────┼────────────┐
         swap(0,0)       swap(0,1)      swap(0,2)
         [1,2,3]         [2,1,3]        [3,2,1]
         start=1         start=1        start=1
       ┌────┴────┐    ┌───┴────┐     ┌───┴────┐
    sw(1,1) sw(1,2)  sw(1,1) sw(1,2)  sw(1,1) sw(1,2)
    [1,2,3] [1,3,2]  [2,1,3] [2,3,1]  [3,2,1] [3,1,2]
    start=2 start=2  start=2 start=2  start=2 start=2
      │       │       │       │       │       │
    LEAF    LEAF     LEAF    LEAF    LEAF    LEAF

  Output: [1,2,3] [1,3,2] [2,1,3] [2,3,1] [3,2,1] [3,1,2]
  Total: 3! = 6 permutations
```

---

## Example 2: All Permutations — Used-Array Approach

```go
package main

import "fmt"

func permuteWithUsed(nums []int) [][]int {
	result := [][]int{}
	used := make([]bool, len(nums))

	var backtrack func(path []int)
	backtrack = func(path []int) {
		if len(path) == len(nums) {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := 0; i < len(nums); i++ {
			if used[i] { continue }

			used[i] = true
			path = append(path, nums[i])
			backtrack(path)
			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack([]int{})
	return result
}

func main() {
	result := permuteWithUsed([]int{1, 2, 3})
	for _, p := range result { fmt.Println(p) }
}
```

**Textual Figure:**

```
nums = [1, 2, 3]   Used-array approach

  Recursion tree (try each unused element at each level):
  ────────────────────────────────────────────────────────────
                         path=[]  used=[F,F,F]
              ┌────────────┼────────────┐
            +1             +2             +3
         used[0]=T      used[1]=T      used[2]=T
       ┌────┴────┐    ┌───┴────┐     ┌───┴────┐
      +2       +3     +1      +3      +1       +2
    (skip 1) (skip 1) (skip 2) (skip 2) (skip 3) (skip 3)
      │        │       │       │       │        │
     +3       +2      +3      +1      +2       +1
      │        │       │       │       │        │
   [1,2,3]  [1,3,2] [2,1,3] [2,3,1] [3,1,2]  [3,2,1]

  Key: used[i]=true prevents element i from being picked again
  At each level, only (n - depth) choices remain
  Total nodes: 1 + 3 + 6 + 6 = 16
```

---

## Example 3: Permutations With Duplicates (LeetCode 47)

```go
package main

import (
	"fmt"
	"sort"
)

func permuteUnique(nums []int) [][]int {
	sort.Ints(nums)
	result := [][]int{}
	used := make([]bool, len(nums))

	var backtrack func(path []int)
	backtrack = func(path []int) {
		if len(path) == len(nums) {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := 0; i < len(nums); i++ {
			if used[i] { continue }
			// Skip duplicates: same value, previous not used = duplicate branch
			if i > 0 && nums[i] == nums[i-1] && !used[i-1] { continue }

			used[i] = true
			path = append(path, nums[i])
			backtrack(path)
			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack([]int{})
	return result
}

func main() {
	result := permuteUnique([]int{1, 1, 2})
	for _, p := range result { fmt.Println(p) }
	// [1 1 2] [1 2 1] [2 1 1]
	fmt.Println("Total:", len(result)) // 3
}
```

**Textual Figure:**

```
nums = [1, 1, 2]  (sorted)   Duplicate handling

  Without duplicate skipping (would produce 6, with repeats):
  ───────────────────────────────────────────────────────
                     []
           ┌────────┼────────┐
          +1₀       +1₁       +2
                   DUP!

  With duplicate skipping:
  ───────────────────────
                     []       i=0: pick 1₀
           ┌────────┼────────┐    i=1: skip 1₁ (nums[1]==nums[0]
          +1₀      │✂ SKIP   +2        && !used[0])
         used[0]=T  │       used[2]=T   i=2: pick 2
        ┌──┴──┐     │      ┌──┴──┐
       +1₁   +2     │     +1₀   +1₁
        │     │     │      │   SKIP││ (same val, used[0]=F)
       +2    +1₁    │     +1₁    │
        │     │     │      │      │
     [1,1,2] [1,2,1] │    [2,1,1]  │
       ✓      ✓     │      ✓      │
                     │            │
                     └──────────┘
                     Pruned duplicate branches

  Result: [1,1,2]  [1,2,1]  [2,1,1]   (3 = 3!/(2!1!))
```

---

## Example 4: k-Permutations (Choose k from n)

```go
package main

import "fmt"

func kPermutations(nums []int, k int) [][]int {
	result := [][]int{}
	used := make([]bool, len(nums))

	var backtrack func(path []int)
	backtrack = func(path []int) {
		if len(path) == k {
			result = append(result, append([]int{}, path...))
			return
		}

		for i := 0; i < len(nums); i++ {
			if used[i] { continue }
			used[i] = true
			path = append(path, nums[i])
			backtrack(path)
			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack([]int{})
	return result
}

func main() {
	result := kPermutations([]int{1, 2, 3, 4}, 2)
	for _, p := range result { fmt.Println(p) }
	fmt.Println("Total:", len(result)) // 12 = 4!/(4-2)!
}
```

**Textual Figure:**

```
nums = [1, 2, 3, 4], k = 2   (choose 2 in order)

  Recursion tree (stop at depth k=2 instead of n):
  ───────────────────────────────────────────────────
                       []  depth=0
          ┌───────┬────┼────┬───────┐
         +1     +2     +3      +4
       depth=1                depth=1
      ┌──┼──┐              ┌──┼──┐
     +2  +3  +4            +1  +2  +3
      │   │   │             │   │   │
   [1,2][1,3][1,4]        [4,1][4,2][4,3]
     ✓    ✓    ✓    ...     ✓    ✓    ✓
     depth=2 → STOP (len(path)==k)

  Full result: [1,2] [1,3] [1,4] [2,1] [2,3] [2,4]
               [3,1] [3,2] [3,4] [4,1] [4,2] [4,3]
  Total: P(4,2) = 4!/(4-2)! = 12
```

---

## Example 5: Next Permutation (LeetCode 31)

```go
package main

import (
	"fmt"
	"sort"
)

func nextPermutation(nums []int) {
	n := len(nums)

	// Step 1: Find rightmost element smaller than its successor
	i := n - 2
	for i >= 0 && nums[i] >= nums[i+1] {
		i--
	}

	if i >= 0 {
		// Step 2: Find rightmost element greater than nums[i]
		j := n - 1
		for nums[j] <= nums[i] {
			j--
		}
		// Step 3: Swap
		nums[i], nums[j] = nums[j], nums[i]
	}

	// Step 4: Reverse suffix
	sort.Ints(nums[i+1:])
}

func main() {
	tests := [][]int{
		{1, 2, 3},
		{3, 2, 1},
		{1, 1, 5},
		{1, 3, 2},
	}

	for _, nums := range tests {
		fmt.Printf("%v → ", nums)
		nextPermutation(nums)
		fmt.Println(nums)
	}
}
```

**Textual Figure:**

```
nums = [1, 3, 2]   →  Next Permutation Algorithm

  Step 1: Find rightmost ascent (nums[i] < nums[i+1])
  ──────────────────────────────────────
    [1, 3, 2]
     ↑
     i=0  (nums[0]=1 < nums[1]=3)  ✓
          (scan right-to-left, 3>2 so skip, 1<3 so stop)

  Step 2: Find rightmost element > nums[i]
  ──────────────────────────────────────
    [1, 3, 2]
              ↑
              j=2  (nums[2]=2 > nums[0]=1)  ✓

  Step 3: Swap nums[i] and nums[j]
  ──────────────────────────────────────
    [1, 3, 2]  →  swap(0,2)  →  [2, 3, 1]

  Step 4: Reverse suffix after i
  ──────────────────────────────────────
    [2, 3, 1]  →  reverse [1:]  →  [2, 1, 3]

  Lexicographic sequence for [1,2,3]:
    [1,2,3] → [1,3,2] → [2,1,3] → [2,3,1] → [3,1,2] → [3,2,1]
                         ↑
                    We are here!

  Result: [1, 3, 2] → [2, 1, 3] ✓
```

---

## Example 6: Permutation Rank (Lexicographic Index)

```go
package main

import "fmt"

// Find the 1-based rank of a permutation in lexicographic order
func permutationRank(perm []int) int {
	n := len(perm)
	rank := 0
	factorial := 1
	for i := 2; i < n; i++ { factorial *= i }

	for i := 0; i < n-1; i++ {
		// Count elements smaller than perm[i] in remaining suffix
		count := 0
		for j := i + 1; j < n; j++ {
			if perm[j] < perm[i] { count++ }
		}
		rank += count * factorial
		factorial /= (n - 1 - i)
	}
	return rank + 1
}

func main() {
	perms := [][]int{
		{1, 2, 3}, // rank 1
		{1, 3, 2}, // rank 2
		{2, 1, 3}, // rank 3
		{3, 2, 1}, // rank 6
	}

	for _, p := range perms {
		fmt.Printf("Rank of %v = %d\n", p, permutationRank(p))
	}
}
```

**Textual Figure:**

```
perm = [3, 2, 1]   Finding rank in lexicographic order

  All permutations of [1,2,3] in order:
  ┌──────┬──────────┐
  │ Rank │ Perm     │
  ├──────┼──────────┤
  │  1   │ [1,2,3]  │
  │  2   │ [1,3,2]  │
  │  3   │ [2,1,3]  │
  │  4   │ [2,3,1]  │
  │  5   │ [3,1,2]  │
  │  6   │ [3,2,1]  │ ← target
  └──────┴──────────┘

  Calculation for [3, 2, 1]:
    Position 0: perm[0]=3
      Elements smaller than 3 in suffix [2,1]: count=2
      rank += 2 × 2! = 4

    Position 1: perm[1]=2
      Elements smaller than 2 in suffix [1]: count=1
      rank += 1 × 1! = 1

    rank = 4 + 1 + 1 = 6  ✓
```

---

## Example 7: Kth Permutation (LeetCode 60)

```go
package main

import (
	"fmt"
	"strings"
)

func getPermutation(n int, k int) string {
	// Compute factorials
	factorial := make([]int, n+1)
	factorial[0] = 1
	for i := 1; i <= n; i++ { factorial[i] = factorial[i-1] * i }

	// Available digits
	digits := make([]int, n)
	for i := range digits { digits[i] = i + 1 }

	k-- // convert to 0-based
	result := strings.Builder{}

	for i := n; i >= 1; i-- {
		idx := k / factorial[i-1]
		result.WriteByte(byte('0' + digits[idx]))
		// Remove used digit
		digits = append(digits[:idx], digits[idx+1:]...)
		k %= factorial[i-1]
	}

	return result.String()
}

func main() {
	for k := 1; k <= 6; k++ {
		fmt.Printf("k=%d: %s\n", k, getPermutation(3, k))
	}
	fmt.Println()
	fmt.Println("k=9:", getPermutation(4, 9))
}
```

**Textual Figure:**

```
n = 3, k = 5   (find 5th permutation)

  digits = [1, 2, 3],  k = 4 (0-based)

  Step 1: pick digit for position 0
  ─────────────────────────────────────
    factorial[2] = 2! = 2
    idx = 4 / 2 = 2  → pick digits[2] = 3
    k   = 4 % 2 = 0
    digits = [1, 2]   (removed 3)

  Step 2: pick digit for position 1
  ─────────────────────────────────────
    factorial[1] = 1! = 1
    idx = 0 / 1 = 0  → pick digits[0] = 1
    k   = 0 % 1 = 0
    digits = [2]      (removed 1)

  Step 3: pick digit for position 2
  ─────────────────────────────────────
    idx = 0  → pick digits[0] = 2

  Result: "312"  (5th permutation of [1,2,3])

  Verification:  1:"123" 2:"132" 3:"213" 4:"231" 5:"312" ✓
```

---

## Example 8: String Permutations

```go
package main

import (
	"fmt"
	"sort"
)

func permuteString(s string) []string {
	chars := []byte(s)
	sort.Slice(chars, func(i, j int) bool { return chars[i] < chars[j] })

	result := []string{}
	used := make([]bool, len(chars))

	var backtrack func(path []byte)
	backtrack = func(path []byte) {
		if len(path) == len(chars) {
			result = append(result, string(path))
			return
		}

		for i := 0; i < len(chars); i++ {
			if used[i] { continue }
			if i > 0 && chars[i] == chars[i-1] && !used[i-1] { continue }

			used[i] = true
			path = append(path, chars[i])
			backtrack(path)
			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack([]byte{})
	return result
}

func main() {
	fmt.Println("Permutations of 'abc':")
	for _, p := range permuteString("abc") { fmt.Println(" ", p) }

	fmt.Println("\nPermutations of 'aab':")
	for _, p := range permuteString("aab") { fmt.Println(" ", p) }
}
```

**Textual Figure:**

```
s = "aab"   (sorted: "aab")

  Recursion tree with duplicate skipping:
  ───────────────────────────────────────────────────
                       path=""
           ┌─────────┼─────────┐
         +'a'₀       +'a'₁      +'b'
          │        SKIP││         │
         'a'     (dup of a₀)     'b'
       ┌──┴──┐                  ┌──┴──┐
     +'a'₁  +'b'              +'a'₀ +'a'₁
       │      │                 │    SKIP││
      'aa'   'ab'              'ba'
       │      │                 │
     +'b'    +'a'₁             +'a'₁
       │      │                 │
    "aab"   "aba"             "baa"
      ✓       ✓                 ✓

  Result: "aab" "aba" "baa"  (3 = 3!/2!)
  Without skipping: would produce 6 with duplicates
```

---

## Example 9: Permutations as a Sequence (Iterative via Heap's Algorithm)

```go
package main

import "fmt"

func heapPermutations(a []int) [][]int {
	result := [][]int{}
	n := len(a)
	c := make([]int, n) // control array

	perm := make([]int, n)
	copy(perm, a)
	result = append(result, perm)

	i := 0
	for i < n {
		if c[i] < i {
			if i%2 == 0 {
				a[0], a[i] = a[i], a[0]
			} else {
				a[c[i]], a[i] = a[i], a[c[i]]
			}
			perm = make([]int, n)
			copy(perm, a)
			result = append(result, perm)
			c[i]++
			i = 0
		} else {
			c[i] = 0
			i++
		}
	}
	return result
}

func main() {
	result := heapPermutations([]int{1, 2, 3})
	fmt.Println("Heap's algorithm permutations:")
	for _, p := range result { fmt.Println(p) }
	fmt.Println("Total:", len(result))
}
```

**Textual Figure:**

```
a = [1, 2, 3]   Heap's Algorithm (iterative, one swap per permutation)

  Trace of swaps (c = control array, initially [0,0,0]):
  ──────────────────────────────────────────────────────
  Step  i  c[i]<i?  Swap           Array     Output
  ──────────────────────────────────────────────────────
  init  -    -       -             [1,2,3]   [1,2,3]
   1    0   0<0? N   -              -         -
   2    1   0<1? Y   swap(0,1)     [2,1,3]   [2,1,3]
   3    0   0<0? N   -              -         -
   4    1   1<1? N   -              -         -
   5    2   0<2? Y   swap(0,2)     [3,1,2]   [3,1,2]
   6    0   0<0? N   -              -         -
   7    1   0<1? Y   swap(0,1)     [1,3,2]   [1,3,2]
   8    0   0<0? N   -              -         -
   9    1   1<1? N   -              -         -
  10    2   1<2? Y   swap(1,2)     [1,2,3]→  wait...
         → swap(c[2],2)=swap(1,2) [2,3,1]   [2,3,1]
  11    0   ...Y     swap(0,1)     [3,2,1]   [3,2,1]

  All 6 permutations generated with exactly 1 swap each!
```

---

## Example 10: Permutations Concepts Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Permutations Summary ===")
	fmt.Println()

	methods := []struct{ method, complexity, bestFor string }{
		{"Swap-based", "O(n·n!)", "In-place, distinct elements"},
		{"Used-array", "O(n·n!)", "Flexible, handles duplicates"},
		{"Heap's algorithm", "O(n!)", "Iterative, minimal swaps"},
		{"Next permutation", "O(n)", "Generate one by one in order"},
		{"Kth permutation", "O(n²)", "Direct access by rank"},
		{"Factoradic rank", "O(n²)", "Index ↔ permutation mapping"},
	}

	fmt.Printf("%-20s %-12s %s\n", "Method", "Time", "Best For")
	fmt.Println("------------------------------------------------")
	for _, m := range methods {
		fmt.Printf("%-20s %-12s %s\n", m.method, m.complexity, m.bestFor)
	}

	fmt.Println()
	fmt.Println("Duplicate handling:")
	fmt.Println("  1. Sort the array")
	fmt.Println("  2. Skip if nums[i] == nums[i-1] and !used[i-1]")
	fmt.Println()
	fmt.Println("Interview tips:")
	fmt.Println("  - Swap approach for distinct elements")
	fmt.Println("  - Used-array approach when order matters or duplicates exist")
	fmt.Println("  - Next permutation for lexicographic sequence")
}
```

**Textual Figure:**

```
  Permutation Methods Comparison
  ═════════════════════════════════════════════════════════════

  ┌─────────────────┐     ┌─────────────────┐
  │  Swap Approach   │     │ Used-Array       │
  │  (in-place)      │     │ (build path)     │
  │  O(n·n!)         │     │ O(n·n!)          │
  │  Distinct only   │     │ Handles dups     │
  └────────┬────────┘     └────────┬────────┘
           │                       │
           └─────────┬─────────┘
                     │
  ┌─────────────────┴─────────────────┐
  │        GENERATE ALL n!              │
  └───────────────────────────────────┘

  ┌─────────────────┐     ┌─────────────────┐
  │ Heap's (iter)    │     │ Next Permutation │
  │ O(n!)            │     │ O(n) per step    │
  │ 1 swap each      │     │ Lexicographic    │
  └─────────────────┘     └─────────────────┘

  ┌─────────────────┐     ┌─────────────────┐
  │ Kth Permutation  │     │ Factoradic Rank │
  │ O(n²)            │     │ O(n²)            │
  │ Direct access    │     │ Index ↔ perm    │
  └─────────────────┘     └─────────────────┘
```

---

## Key Takeaways

1. **Swap approach**: O(n!) in-place — swap element at `start` with each remaining position
2. **Used-array approach**: O(n·n!) — build path, track used elements with boolean array
3. **Duplicate handling**: sort first, then `if i > 0 && nums[i] == nums[i-1] && !used[i-1]` skip
4. **Next permutation**: O(n) — find rightmost ascent, swap with ceiling, reverse suffix
5. **Kth permutation**: O(n²) — use factoradic number system to directly compute the answer

> **Next up:** Combinations →
