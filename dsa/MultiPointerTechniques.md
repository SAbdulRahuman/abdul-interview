# Phase 25: Advanced Sliding Window — Multi-Pointer Techniques

## Overview

Multi-pointer extends two-pointer to **three or more pointers**, enabling complex traversal patterns. Key applications include k-way merge, partitioning, and multi-dimensional problems.

### When to Use Multi-Pointer
- **k-Sum**: fix k-2 elements, two-pointer on rest
- **Partitioning**: Dutch National Flag uses 3 pointers
- **k-Way Merge**: one pointer per sorted source
- **Multi-condition windows**: maintain multiple boundaries

---

## Example 1: Dutch National Flag — 3 Pointers (LC 75)

```go
package main

import "fmt"

// Sort array of 0s, 1s, 2s using 3 pointers

func sortColors(nums []int) {
	low, mid, high := 0, 0, len(nums)-1

	for mid <= high {
		switch nums[mid] {
		case 0:
			nums[low], nums[mid] = nums[mid], nums[low]
			low++
			mid++
		case 1:
			mid++
		case 2:
			nums[mid], nums[high] = nums[high], nums[mid]
			high--
			// Don't advance mid — swapped value needs checking
		}
	}
}

func main() {
	tests := [][]int{
		{2, 0, 2, 1, 1, 0},
		{2, 0, 1},
		{0, 0, 2, 1, 2, 1, 0},
	}

	for _, nums := range tests {
		original := append([]int{}, nums...)
		sortColors(nums)
		fmt.Printf("%v → %v\n", original, nums)
	}

	fmt.Println("\n3 pointers: low (0s boundary), mid (scanner), high (2s boundary)")
}
```

**Textual Figure — Dutch National Flag (3 Pointers):**

```
  nums = [2, 0, 2, 1, 1, 0]

  Three pointers: low=0, mid=0, high=5

  ┌───┬───┬───┬───┬───┬───┐
  │ 2 │ 0 │ 2 │ 1 │ 1 │ 0 │  initial
  └───┴───┴───┴───┴───┴───┘
   L,M                   H

  mid=0: nums[0]=2 → swap(mid,high), high--
  [0, 0, 2, 1, 1, 2]    L,M           H

  mid=0: nums[0]=0 → swap(low,mid), low++, mid++
  [0, 0, 2, 1, 1, 2]       L,M        H

  mid=1: nums[1]=0 → swap(low,mid), low++, mid++
  [0, 0, 2, 1, 1, 2]          L,M     H

  mid=2: nums[2]=2 → swap(mid,high), high--
  [0, 0, 1, 1, 2, 2]          L,M  H

  mid=2: nums[2]=1 → mid++
  [0, 0, 1, 1, 2, 2]          L   M,H

  mid=3: nums[3]=1 → mid++
  mid > high → DONE!

  Result: [0, 0, 1, 1, 2, 2]
    ├─ 0s ─┤├─ 1s ─┤├─ 2s ─┤
```

---

## Example 2: Three Way Partition

```go
package main

import "fmt"

// Partition array around a pivot into three sections:
// [< pivot] [== pivot] [> pivot]

func threeWayPartition(arr []int, pivot int) {
	low, mid, high := 0, 0, len(arr)-1

	for mid <= high {
		if arr[mid] < pivot {
			arr[low], arr[mid] = arr[mid], arr[low]
			low++
			mid++
		} else if arr[mid] > pivot {
			arr[mid], arr[high] = arr[high], arr[mid]
			high--
		} else {
			mid++
		}
	}
}

func main() {
	arr := []int{4, 9, 4, 2, 7, 4, 1, 8, 4, 3}
	pivot := 4
	fmt.Printf("Before: %v (pivot=%d)\n", arr, pivot)
	threeWayPartition(arr, pivot)
	fmt.Printf("After:  %v\n", arr)
	fmt.Println("[< 4 | == 4 | > 4]")
}
```

**Textual Figure — Three Way Partition:**

```
  arr = [4, 9, 4, 2, 7, 4, 1, 8, 4, 3]   pivot = 4

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 4 │ 9 │ 4 │ 2 │ 7 │ 4 │ 1 │ 8 │ 4 │ 3 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  L,M                                          H

  Three pointers partition into:
  [< pivot]  [== pivot]  [> pivot]

  Trace (key steps):
  mid=0: arr[0]=4==pivot → mid++
  mid=1: arr[1]=9>pivot  → swap(mid,high), high--
  mid=1: arr[1]=3<pivot  → swap(low,mid), low++, mid++
  ...continue until mid > high...

  Result: [3, 2, 1, 4, 4, 4, 4, 7, 8, 9]
           ├─ <4 ─┤  ├── ==4 ──┤  ├─ >4 ─┤

  Same logic as Dutch National Flag:
  low → boundary of "less than"
  mid → scanner
  high → boundary of "greater than"
```

---

## Example 3: K-Way Merge with Pointers

```go
package main

import (
	"container/heap"
	"fmt"
)

// Merge k sorted arrays — one pointer per array

type Item struct {
	val, arrayIdx, elemIdx int
}
type MinHeap []Item

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mergeKSorted(arrays [][]int) []int {
	h := &MinHeap{}

	// Initialize: one pointer (element) per array
	for i, arr := range arrays {
		if len(arr) > 0 {
			heap.Push(h, Item{arr[0], i, 0})
		}
	}

	var result []int
	for h.Len() > 0 {
		item := heap.Pop(h).(Item)
		result = append(result, item.val)

		// Advance pointer for this array
		nextIdx := item.elemIdx + 1
		if nextIdx < len(arrays[item.arrayIdx]) {
			heap.Push(h, Item{arrays[item.arrayIdx][nextIdx], item.arrayIdx, nextIdx})
		}
	}
	return result
}

func main() {
	arrays := [][]int{
		{1, 4, 7, 10},
		{2, 5, 6, 8},
		{3, 9, 11, 12},
	}

	fmt.Println("Arrays:")
	for _, a := range arrays { fmt.Println(" ", a) }
	fmt.Println("Merged:", mergeKSorted(arrays))
}
```

**Textual Figure — K-Way Merge with Pointers:**

```
  3 sorted arrays, one pointer per array:

  Array 0: [1, 4, 7, 10]     ptr→ 1
  Array 1: [2, 5, 6,  8]     ptr→ 2
  Array 2: [3, 9, 11, 12]    ptr→ 3
                  │
            Min-Heap: [1, 2, 3]

  Step 1: pop 1 (arr0), push 4   heap: [2, 3, 4]
  Step 2: pop 2 (arr1), push 5   heap: [3, 4, 5]
  Step 3: pop 3 (arr2), push 9   heap: [4, 5, 9]
  Step 4: pop 4 (arr0), push 7   heap: [5, 7, 9]
  Step 5: pop 5 (arr1), push 6   heap: [6, 7, 9]
  Step 6: pop 6 (arr1), push 8   heap: [7, 8, 9]
  ...and so on...

  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]

  ┌─────────────────────────────────────┐
  │ One pointer per array + min-heap  │
  │ Always extract global minimum     │
  │ O(N log k) total                  │
  └─────────────────────────────────────┘
```

---

## Example 4: Trapping Rain Water — 2 Pointers (LC 42)

```go
package main

import "fmt"

func trap(height []int) int {
	left, right := 0, len(height)-1
	leftMax, rightMax := 0, 0
	water := 0

	for left < right {
		if height[left] < height[right] {
			if height[left] >= leftMax {
				leftMax = height[left]
			} else {
				water += leftMax - height[left]
			}
			left++
		} else {
			if height[right] >= rightMax {
				rightMax = height[right]
			} else {
				water += rightMax - height[right]
			}
			right--
		}
	}
	return water
}

func main() {
	tests := [][]int{
		{0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1},
		{4, 2, 0, 3, 2, 5},
	}

	for _, h := range tests {
		fmt.Printf("height=%v → water=%d\n", h, trap(h))
	}
}
```

**Textual Figure — Trapping Rain Water (Two Pointers):**

```
  height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]

       3 │                       █
       2 │          █ ░ ░ ░ ░ █ █ ░ █
       1 │    █ ░ █ █ ░ █ █ █ █ █ █
       0 │ █  █ █ █ █ █ █ █ █ █ █ █
         └─────────────────────────
           0  1  2  3  4  5  6  7  8  9 10 11
           L→                           ←R
  (░ = trapped water)

  Two pointers converge from both ends:
  Track leftMax and rightMax

  L=0 R=11: h[L]=0 < h[R]=1 → process left
            leftMax=0, water += 0
  L=1 R=11: h[L]=1 < h[R]=1 → process left
            leftMax=1, water += 0
  L=2 R=11: h[L]=0 < h[R]=1 → process left
            water += leftMax(1) - h[2](0) = 1
  ...continue converging...

  Total water trapped = 6

  Key: min(leftMax, rightMax) - height[i]
```

---

## Example 5: Intersection of Three Sorted Arrays (LC 1213)

```go
package main

import "fmt"

// Three pointers — one per array

func intersection3(a, b, c []int) []int {
	i, j, k := 0, 0, 0
	var result []int

	for i < len(a) && j < len(b) && k < len(c) {
		if a[i] == b[j] && b[j] == c[k] {
			result = append(result, a[i])
			i++; j++; k++
		} else {
			// Advance the smallest
			minVal := a[i]
			if b[j] < minVal { minVal = b[j] }
			if c[k] < minVal { minVal = c[k] }

			if a[i] == minVal { i++ }
			if b[j] == minVal { j++ }
			if c[k] == minVal { k++ }
		}
	}
	return result
}

func main() {
	a := []int{1, 2, 3, 4, 5}
	b := []int{1, 2, 5, 7, 9}
	c := []int{1, 3, 4, 5, 8}

	fmt.Printf("a=%v\nb=%v\nc=%v\n", a, b, c)
	fmt.Printf("Intersection: %v\n", intersection3(a, b, c))
}
```

**Textual Figure — Intersection of Three Sorted Arrays:**

```
  a = [1, 2, 3, 4, 5]   ptr i=0
  b = [1, 2, 5, 7, 9]   ptr j=0
  c = [1, 3, 4, 5, 8]   ptr k=0

  Step   i  j  k   a[i] b[j] c[k]  Action
    1    0  0  0    1    1    1    All equal → add 1, i++j++k++
    2    1  1  1    2    2    3    min=2, advance i,j
    3    2  2  1    3    5    3    min=3, advance i,k
    4    3  2  2    4    5    4    min=4, advance i,k
    5    4  2  3    5    5    5    All equal → add 5, i++j++k++
    6    i=5 → out of bounds → DONE

  Result: [1, 5]

  ┌──────────────────────────────────┐
  │ Three pointers, one per array    │
  │ All equal → add to result        │
  │ Otherwise advance the smallest  │
  └──────────────────────────────────┘
```

---

## Example 6: Smallest Range Covering Elements from K Lists (LC 632)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

// One pointer per list + min-heap to track current min

type Entry struct {
	val, listIdx, elemIdx int
}
type MinHeap []Entry

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(Entry)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func smallestRange(nums [][]int) []int {
	h := &MinHeap{}
	curMax := math.MinInt64

	// Initialize: first element from each list
	for i, list := range nums {
		heap.Push(h, Entry{list[0], i, 0})
		if list[0] > curMax { curMax = list[0] }
	}

	bestRange := []int{(*h)[0].val, curMax}

	for {
		minEntry := heap.Pop(h).(Entry)
		nextIdx := minEntry.elemIdx + 1

		if nextIdx >= len(nums[minEntry.listIdx]) { break }

		nextVal := nums[minEntry.listIdx][nextIdx]
		heap.Push(h, Entry{nextVal, minEntry.listIdx, nextIdx})
		if nextVal > curMax { curMax = nextVal }

		curMin := (*h)[0].val
		if curMax-curMin < bestRange[1]-bestRange[0] {
			bestRange = []int{curMin, curMax}
		}
	}
	return bestRange
}

func main() {
	nums := [][]int{
		{4, 10, 15, 24, 26},
		{0, 9, 12, 20},
		{5, 18, 22, 30},
	}

	fmt.Println("Lists:")
	for _, l := range nums { fmt.Println(" ", l) }
	result := smallestRange(nums)
	fmt.Printf("Smallest range: [%d, %d]\n", result[0], result[1])
}
```

**Textual Figure — Smallest Range Covering K Lists:**

```
  List 0: [4, 10, 15, 24, 26]   ptr→ 4
  List 1: [0,  9, 12, 20]       ptr→ 0
  List 2: [5, 18, 22, 30]       ptr→ 5

  Min-heap tracks current minimum, variable tracks max.

  Step   Heap(min)   Max   Range         Best
    init  [0,4,5]     5    [0,5]=5       [0,5]
    1     pop 0(L1)  → push 9   max=9
          [4,5,9]     9    [4,9]=5       [0,5]
    2     pop 4(L0)  → push 10  max=10
          [5,9,10]   10    [5,10]=5      [0,5]
    3     pop 5(L2)  → push 18  max=18
          [9,10,18]  18    [9,18]=9      [0,5]
    4     pop 9(L1)  → push 12  max=18
          [10,12,18] 18    [10,18]=8     [0,5]
    5     pop 10(L0) → push 15  max=18
          [12,15,18] 18    [12,18]=6     [0,5]
    ...continues...

  Eventually finds [20,24] with range=4 ← best
  Result: [20, 24]
```

---

## Example 7: Move Zeroes — Read/Write Pointers (LC 283)

```go
package main

import "fmt"

// Two pointers: read (scans) and write (places non-zeros)

func moveZeroes(nums []int) {
	write := 0

	// Pass 1: write non-zero elements
	for read := 0; read < len(nums); read++ {
		if nums[read] != 0 {
			nums[write] = nums[read]
			write++
		}
	}

	// Pass 2: fill remaining with zeros
	for write < len(nums) {
		nums[write] = 0
		write++
	}
}

// One pass version with swap
func moveZeroesOnePass(nums []int) {
	write := 0
	for read := 0; read < len(nums); read++ {
		if nums[read] != 0 {
			nums[write], nums[read] = nums[read], nums[write]
			write++
		}
	}
}

func main() {
	a := []int{0, 1, 0, 3, 12}
	b := append([]int{}, a...)
	moveZeroes(a)
	moveZeroesOnePass(b)
	fmt.Printf("Two-pass: %v\n", a)
	fmt.Printf("One-pass: %v\n", b)
}
```

**Textual Figure — Move Zeroes (Read/Write Pointers):**

```
  nums = [0, 1, 0, 3, 12]

  write=0 (W), read scans (R):

  ┌───┬───┬───┬───┬────┐
  │ 0 │ 1 │ 0 │ 3 │ 12 │  R=0: nums[0]=0 → skip
  └───┴───┴───┴───┴────┘
   W,R

  ┌───┬───┬───┬───┬────┐
  │ 1 │ 0 │ 0 │ 3 │ 12 │  R=1: nums[1]=1 → swap(W,R), W++
  └───┴───┴───┴───┴────┘
      W       R

  ┌───┬───┬───┬───┬────┐
  │ 1 │ 0 │ 0 │ 3 │ 12 │  R=2: nums[2]=0 → skip
  └───┴───┴───┴───┴────┘
      W           R

  ┌───┬───┬───┬───┬────┐
  │ 1 │ 3 │ 0 │ 0 │ 12 │  R=3: nums[3]=3 → swap(W,R), W++
  └───┴───┴───┴───┴────┘
          W            R

  ┌───┬───┬────┬───┬───┐
  │ 1 │ 3 │ 12 │ 0 │ 0 │  R=4: nums[4]=12 → swap(W,R), W++
  └───┴───┴────┴───┴───┘
              W             R

  Result: [1, 3, 12, 0, 0]
```

---

## Example 8: Partition Labels (LC 763)

```go
package main

import "fmt"

// Track last occurrence of each character → greedy multi-pointer

func partitionLabels(s string) []int {
	// Find last index of each character
	lastIdx := [26]int{}
	for i := 0; i < len(s); i++ {
		lastIdx[s[i]-'a'] = i
	}

	var result []int
	start, end := 0, 0

	for i := 0; i < len(s); i++ {
		if lastIdx[s[i]-'a'] > end {
			end = lastIdx[s[i]-'a']
		}
		if i == end {
			result = append(result, end-start+1)
			start = end + 1
		}
	}
	return result
}

func main() {
	tests := []string{
		"ababcbacadefegdehijhklij",
		"eccbbbbdec",
	}

	for _, s := range tests {
		fmt.Printf("%q → %v\n", s, partitionLabels(s))
	}
}
```

**Textual Figure — Partition Labels (Greedy Multi-Pointer):**

```
  s = "ababcbacadefegdehijhklij"

  Last occurrence of each char:
  a:8  b:5  c:7  d:14  e:15  f:11  g:13  h:19  i:22  j:23  k:20  l:21

  Scan with start and end pointers:

  i=0: 'a' last=8    end=8     ├─────────────────┤
  i=1: 'b' last=5    end=8     │ababcbaca         │
  i=2: 'a' last=8    end=8     │                   │
  i=5: 'b' last=5    end=8     │                   │
  i=7: 'c' last=7    end=8     │                   │
  i=8: 'a' last=8    i==end!   ├─── size=9 ──────┤

  i=9: 'd' last=14   end=14    ├─────────────────┤
  i=10:'e' last=15   end=15    │defegde           │
  i=11:'f' last=11   end=15    │                   │
  i=15:'e' last=15   i==end!   ├─── size=7 ──────┤

  i=16:'h' last=19   end=19    ├─────────────────┤
  i=22:'i' last=22   end=23    │hijhklij          │
  i=23:'j' last=23   i==end!   ├─── size=8 ──────┤

  Result: [9, 7, 8]
```

---

## Example 9: Sort Array by Parity — Partition Variant (LC 905)

```go
package main

import "fmt"

// Two pointers: even from left, odd from right

func sortArrayByParity(nums []int) []int {
	left, right := 0, len(nums)-1

	for left < right {
		if nums[left]%2 == 0 {
			left++
		} else if nums[right]%2 == 1 {
			right--
		} else {
			nums[left], nums[right] = nums[right], nums[left]
			left++
			right--
		}
	}
	return nums
}

// Sort by Parity II (LC 922): even indices get even, odd get odd
func sortArrayByParityII(nums []int) []int {
	even, odd := 0, 1

	for even < len(nums) && odd < len(nums) {
		for even < len(nums) && nums[even]%2 == 0 { even += 2 }
		for odd < len(nums) && nums[odd]%2 == 1 { odd += 2 }

		if even < len(nums) && odd < len(nums) {
			nums[even], nums[odd] = nums[odd], nums[even]
			even += 2
			odd += 2
		}
	}
	return nums
}

func main() {
	a := []int{3, 1, 2, 4}
	fmt.Printf("By parity: %v → %v\n", []int{3, 1, 2, 4}, sortArrayByParity(a))

	b := []int{4, 2, 5, 7}
	fmt.Printf("By parity II: %v → %v\n", []int{4, 2, 5, 7}, sortArrayByParityII(b))
}
```

**Textual Figure — Sort Array by Parity (Two-Pointer Partition):**

```
  nums = [3, 1, 2, 4]

  Left pointer: even from left
  Right pointer: odd from right

  ┌───┬───┬───┬───┐
  │ 3 │ 1 │ 2 │ 4 │   L=0(odd) R=3(even)
  └───┴───┴───┴───┘
    L→          ←R

  L=0: odd, R=3: even → swap!
  ┌───┬───┬───┬───┐
  │ 4 │ 1 │ 2 │ 3 │   L++, R--
  └───┴───┴───┴───┘
       L→  ←R

  L=1: odd, R=2: even → swap!
  ┌───┬───┬───┬───┐
  │ 4 │ 2 │ 1 │ 3 │   L++, R--
  └───┴───┴───┴───┘
       R  L          L > R → DONE

  Result: [4, 2, 1, 3]  (evens left, odds right)
```

---

## Example 10: Multi-Pointer Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Multi-Pointer Patterns Summary ===\n")

	fmt.Println("--- Pattern Categories ---")
	patterns := []struct{ name, pointers, example string }{
		{"Opposite ends", "left→ ←right", "Two Sum, Container Water"},
		{"Same direction", "slow→ fast→", "Remove dupls, Floyd's cycle"},
		{"3-way partition", "low→ mid→ ←high", "Dutch National Flag"},
		{"Read/Write", "read→, write→", "Move zeros, compress"},
		{"K-way merge", "p1, p2, ..., pk", "Merge k sorted lists"},
		{"Multi-condition", "start, end, boundary", "Partition labels"},
	}
	for _, p := range patterns {
		fmt.Printf("  %-18s [%-17s] %s\n", p.name, p.pointers, p.example)
	}

	fmt.Println("\n--- Choosing the Right Approach ---")
	tips := []struct{ scenario, technique string }{
		{"Find pair with target sum", "Opposite ends on sorted"},
		{"Partition into 2 groups", "Two pointers (swap)"},
		{"Partition into 3 groups", "Three pointers (DNF)"},
		{"Merge sorted sources", "One pointer per source + heap"},
		{"Remove/compress in-place", "Read/write pointers"},
		{"Cycle in sequence", "Fast/slow (Floyd)"},
		{"K-sum for any k", "Fix k-2, two-pointer on rest"},
	}
	for _, t := range tips {
		fmt.Printf("  %-35s → %s\n", t.scenario, t.technique)
	}

	fmt.Println("\n--- Complexity ---")
	fmt.Println("  Two pointer: O(n)")
	fmt.Println("  K-way merge: O(N log k) with heap")
	fmt.Println("  K-sum: O(n^(k-1)) with sorting")

	fmt.Println("\n--- All 25 Phases Complete! ---")
	fmt.Println("  You've covered the full DSA preparation roadmap.")
	fmt.Println("  Review, practice, and good luck with interviews!")
}
```

---

## Key Takeaways

1. **Dutch National Flag** (3-pointer): the classic partitioning pattern — low/mid/high
2. **K-way merge**: one pointer per source + min-heap for efficient extraction
3. **Read/Write pointers**: scan with read, write only valid elements → in-place transform
4. **Trapping rain water**: converge from both ends, track max heights
5. **Multi-pointer is just an extension** of two-pointer — identify how many boundaries you need
6. **Practice the fundamental patterns** — most multi-pointer problems reduce to a known template

---

## 🎉 All 25 Phases Complete!

Congratulations! You've covered the full DSA preparation roadmap:
- **217+ concept files** across 25 phases
- **2100+ Go examples** with runnable code
- From algorithm complexity basics to advanced techniques

**Review strategy**: revisit 2-3 files per day, implement from memory, and practice on LeetCode.
