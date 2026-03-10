# Phase 11: Heap / Priority Queue — Max Heap

## Overview

A **max heap** is a complete binary tree where every parent is ≥ its children. The root always holds the **maximum** element.

```
       9
      / \
     7   8
    / \
   3   4

Property: parent ≥ children at every level
```

In Go, there's no built-in max heap — use `container/heap` with reversed `Less()`, or negate values in a min heap.

---

## Example 1: Max Heap from Scratch

```go
package main

import "fmt"

type MaxHeap struct {
	data []int
}

func (h *MaxHeap) Push(val int) {
	h.data = append(h.data, val)
	h.siftUp(len(h.data) - 1)
}

func (h *MaxHeap) Pop() int {
	n := len(h.data)
	max := h.data[0]
	h.data[0] = h.data[n-1]
	h.data = h.data[:n-1]
	if len(h.data) > 0 {
		h.siftDown(0)
	}
	return max
}

func (h *MaxHeap) Peek() int { return h.data[0] }
func (h *MaxHeap) Len() int  { return len(h.data) }

func (h *MaxHeap) siftUp(i int) {
	for i > 0 {
		parent := (i - 1) / 2
		if h.data[i] > h.data[parent] {
			h.data[i], h.data[parent] = h.data[parent], h.data[i]
			i = parent
		} else {
			break
		}
	}
}

func (h *MaxHeap) siftDown(i int) {
	n := len(h.data)
	for {
		largest := i
		left, right := 2*i+1, 2*i+2
		if left < n && h.data[left] > h.data[largest] {
			largest = left
		}
		if right < n && h.data[right] > h.data[largest] {
			largest = right
		}
		if largest == i {
			break
		}
		h.data[i], h.data[largest] = h.data[largest], h.data[i]
		i = largest
	}
}

func main() {
	h := &MaxHeap{}
	for _, v := range []int{5, 3, 8, 1, 2, 7} {
		h.Push(v)
	}
	fmt.Println("Max:", h.Peek()) // 8
	for h.Len() > 0 {
		fmt.Print(h.Pop(), " ") // 8 7 5 3 2 1
	}
	fmt.Println()
}
```

**Textual Figure:**

```
Push sequence: 5, 3, 8, 1, 2, 7

  Push 5:  Push 3:  Push 8:         Push 1:  Push 2:  Push 7:
    5        5         8              8        8        8
            /         / \            / \      / \      / \
           3         5   3          5   3    5   3    5   7
                    (siftUp 8>5)   /        / \      / \  /
                                  1       1   2    1   2 3

  Final max heap:               Array: [8, 5, 7, 1, 2, 3]
         8(0)
        / \
      5(1)  7(2)
     / \   /
    1(3) 2(4) 3(5)

  Pop sequence (extract max each time):
  ┌─────┬─────────┬───────────────────────┐
  │ Pop │ Removed │ Heap after siftDown │
  ├─────┼─────────┼───────────────────────┤
  │  1  │    8    │ [7, 5, 3, 1, 2]     │
  │  2  │    7    │ [5, 2, 3, 1]        │
  │  3  │    5    │ [3, 2, 1]           │
  │  4  │    3    │ [2, 1]              │
  │  5  │    2    │ [1]                 │
  │  6  │    1    │ []                  │
  └─────┴─────────┴───────────────────────┘
  Output: 8 7 5 3 2 1
```

---

## Example 2: Max Heap Using container/heap

```go
package main

import (
	"container/heap"
	"fmt"
)

type MaxIntHeap []int

func (h MaxIntHeap) Len() int           { return len(h) }
func (h MaxIntHeap) Less(i, j int) bool { return h[i] > h[j] } // reversed!
func (h MaxIntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxIntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxIntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func main() {
	h := &MaxIntHeap{5, 3, 8, 1, 2, 7}
	heap.Init(h)
	
	fmt.Println("Max:", (*h)[0]) // 8
	heap.Push(h, 10)
	fmt.Println("Max after push 10:", (*h)[0]) // 10

	for h.Len() > 0 {
		fmt.Print(heap.Pop(h), " ") // 10 8 7 5 3 2 1
	}
	fmt.Println()
}
```

**Textual Figure:**

```
Input: [5, 3, 8, 1, 2, 7]   Less: h[i] > h[j] (reversed for max)

  After heap.Init:              Array: [8, 3, 7, 1, 2, 5]
         8(0)
        / \
      3(1)  7(2)
     / \   /
    1   2  5

  Push 10:
  ┌──────────────────────────────────┐
  │ 10 at idx 6, parent=7 at idx 2│
  │ 10 > 7 → swap                 │
  │ 10 at idx 2, parent=8 at idx 0│
  │ 10 > 8 → swap                 │
  └──────────────────────────────────┘
        10(0)   ← new max!
        / \
      3(1) 8(2)
     / \  / \
    1  2 5   7

  Pop order: 10 → 8 → 7 → 5 → 3 → 2 → 1
```

---

## Example 3: Kth Largest Element (LeetCode 215)

```go
package main

import (
	"container/heap"
	"fmt"
)

// Use min heap of size k — top is the kth largest
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func findKthLargest(nums []int, k int) int {
	h := &MinHeap{}
	for _, n := range nums {
		heap.Push(h, n)
		if h.Len() > k {
			heap.Pop(h)
		}
	}
	return (*h)[0]
}

func main() {
	fmt.Println(findKthLargest([]int{3, 2, 1, 5, 6, 4}, 2))    // 5
	fmt.Println(findKthLargest([]int{3, 2, 3, 1, 2, 4, 5, 5, 6}, 4)) // 4
}
```

**Textual Figure:**

```
Input: [3, 2, 1, 5, 6, 4]    k=2 (find 2nd largest)

Strategy: min heap of size k — root = kth largest

  Process each element:
  ┌──────┬─────┬────────────────────────────┐
  │ Elem │ Act │ Min heap (size ≤ k=2)      │
  ├──────┼─────┼────────────────────────────┤
  │   3  │ push│ [3]                        │
  │   2  │ push│ [2, 3]   heap full (k=2)  │
  │   1  │ skip│ 1 < 2 (top) → skip        │
  │   5  │ pop2│ pop 2, push 5 → [3, 5]   │
  │   6  │ pop3│ pop 3, push 6 → [5, 6]   │
  │   4  │ skip│ 4 < 5 (top) → skip        │
  └──────┴─────┴────────────────────────────┘

  Final min heap:     5
                     /
                    6

  Root = 5 = 2nd largest ✓  (6 is 1st largest)
```

---

## Example 4: Top K Frequent Elements (LeetCode 347)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct {
	val, freq int
}
type MinFreqHeap []Item
func (h MinFreqHeap) Len() int           { return len(h) }
func (h MinFreqHeap) Less(i, j int) bool { return h[i].freq < h[j].freq }
func (h MinFreqHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinFreqHeap) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MinFreqHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func topKFrequent(nums []int, k int) []int {
	freq := map[int]int{}
	for _, n := range nums {
		freq[n]++
	}
	h := &MinFreqHeap{}
	for val, f := range freq {
		heap.Push(h, Item{val, f})
		if h.Len() > k {
			heap.Pop(h)
		}
	}
	result := make([]int, k)
	for i := k - 1; i >= 0; i-- {
		result[i] = heap.Pop(h).(Item).val
	}
	return result
}

func main() {
	fmt.Println(topKFrequent([]int{1, 1, 1, 2, 2, 3}, 2)) // [1 2]
	fmt.Println(topKFrequent([]int{1}, 1))                   // [1]
}
```

**Textual Figure:**

```
Input: [1, 1, 1, 2, 2, 3]    k=2

  Step 1: Count frequencies
  ┌─────┬──────┐
  │ Val │ Freq │
  ├─────┼──────┤
  │  1  │   3  │
  │  2  │   2  │
  │  3  │   1  │
  └─────┴──────┘

  Step 2: Min heap of size k=2 (by freq)
  ┌──────┬────────┬────────────────────────┐
  │ Item │  Freq  │ Heap (by freq)         │
  ├──────┼────────┼────────────────────────┤
  │  1   │   3    │ [(1,3)]                │
  │  2   │   2    │ [(2,2),(1,3)]           │
  │  3   │   1    │ 1 < 2(top) → skip      │
  └──────┴────────┴────────────────────────┘

  Final heap:    (2, freq=2)
                    /
                (1, freq=3)

  Pop both in reverse: [1, 2]  (top k frequent)
```

---

## Example 5: Task Scheduler (LeetCode 621)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MaxHeapInt []int
func (h MaxHeapInt) Len() int           { return len(h) }
func (h MaxHeapInt) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeapInt) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeapInt) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeapInt) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func leastInterval(tasks []byte, n int) int {
	freq := make([]int, 26)
	for _, t := range tasks {
		freq[t-'A']++
	}
	h := &MaxHeapInt{}
	for _, f := range freq {
		if f > 0 {
			heap.Push(h, f)
		}
	}
	time := 0
	for h.Len() > 0 {
		cycle := n + 1
		var temp []int
		for cycle > 0 && h.Len() > 0 {
			f := heap.Pop(h).(int)
			if f > 1 {
				temp = append(temp, f-1)
			}
			time++
			cycle--
		}
		for _, f := range temp {
			heap.Push(h, f)
		}
		if h.Len() > 0 {
			time += cycle // idle time
		}
	}
	return time
}

func main() {
	fmt.Println(leastInterval([]byte{'A', 'A', 'A', 'B', 'B', 'B'}, 2))       // 8
	fmt.Println(leastInterval([]byte{'A', 'A', 'A', 'B', 'B', 'B'}, 0))       // 6
	fmt.Println(leastInterval([]byte{'A', 'A', 'A', 'A', 'A', 'A', 'B', 'C', 'D', 'E', 'F', 'G'}, 2)) // 16
}
```

**Textual Figure:**

```
Input: tasks=[A,A,A,B,B,B]  n=2 (cooldown)

  Frequencies: A=3, B=3
  Max heap: [3, 3]  (A and B both freq=3)

  Scheduling with cooldown n=2 (cycle = n+1 = 3):
  ┌───────┬───────┬───────┬───────┬─────────────┐
  │ Time  │ Slot1 │ Slot2 │ Slot3 │ Heap after  │
  ├───────┼───────┼───────┼───────┼─────────────┤
  │ 0-2   │   A   │   B   │ idle  │ [2, 2]      │
  │ 3-5   │   A   │   B   │ idle  │ [1, 1]      │
  │ 6-7   │   A   │   B   │       │ []          │
  └───────┴───────┴───────┴───────┴─────────────┘

  Schedule: A B _ A B _ A B
  Total time: 8

  Key: max heap gives highest-freq task first,
       ensuring we minimize idle slots.
```

---

## Example 6: Reorganize String (LeetCode 767)

```go
package main

import (
	"container/heap"
	"fmt"
	"strings"
)

type CharFreq struct {
	ch   byte
	freq int
}
type MaxCharHeap []CharFreq
func (h MaxCharHeap) Len() int           { return len(h) }
func (h MaxCharHeap) Less(i, j int) bool { return h[i].freq > h[j].freq }
func (h MaxCharHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxCharHeap) Push(x interface{}) { *h = append(*h, x.(CharFreq)) }
func (h *MaxCharHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func reorganizeString(s string) string {
	freq := [26]int{}
	for i := 0; i < len(s); i++ {
		freq[s[i]-'a']++
	}
	h := &MaxCharHeap{}
	for i, f := range freq {
		if f > 0 {
			heap.Push(h, CharFreq{byte('a' + i), f})
		}
	}
	var sb strings.Builder
	var prev CharFreq
	for h.Len() > 0 {
		cur := heap.Pop(h).(CharFreq)
		if prev.freq > 0 {
			heap.Push(h, prev)
		}
		sb.WriteByte(cur.ch)
		cur.freq--
		prev = cur
	}
	if sb.Len() != len(s) {
		return ""
	}
	return sb.String()
}

func main() {
	fmt.Println(reorganizeString("aab"))  // "aba"
	fmt.Println(reorganizeString("aaab")) // ""
}
```

**Textual Figure:**

```
Input: "aab"   freqs: a=2, b=1

  Max heap by frequency: [(a,2), (b,1)]

  Build string (never place same char adjacent):
  ┌──────┬────────┬────────┬────────┬──────────┐
  │ Step │ Pop    │ Place  │ prev   │ Result   │
  ├──────┼────────┼────────┼────────┼──────────┤
  │  1   │ (a,2)  │  'a'   │ (a,1)  │ "a"      │
  │  2   │ (b,1)  │  'b'   │ (b,0)  │ "ab"     │
  │      │        │ push   │ (a,1)  │          │
  │  3   │ (a,1)  │  'a'   │ (a,0)  │ "aba"    │
  └──────┴────────┴────────┴────────┴──────────┘
  Output: "aba" ✓

  Input: "aaab" → a=3, b=1
  Cannot avoid adjacent 'a's when a > (n+1)/2
  Output: "" (impossible)
```

---

## Example 7: Ugly Number II (LeetCode 264)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Int64Heap []int64
func (h Int64Heap) Len() int           { return len(h) }
func (h Int64Heap) Less(i, j int) bool { return h[i] < h[j] }
func (h Int64Heap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *Int64Heap) Push(x interface{}) { *h = append(*h, x.(int64)) }
func (h *Int64Heap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func nthUglyNumber(n int) int {
	h := &Int64Heap{}
	heap.Push(h, int64(1))
	seen := map[int64]bool{1: true}
	primes := []int64{2, 3, 5}
	var val int64
	for i := 0; i < n; i++ {
		val = heap.Pop(h).(int64)
		for _, p := range primes {
			next := val * p
			if !seen[next] {
				seen[next] = true
				heap.Push(h, next)
			}
		}
	}
	return int(val)
}

func main() {
	for i := 1; i <= 10; i++ {
		fmt.Printf("Ugly(%d) = %d\n", i, nthUglyNumber(i))
	}
	// 1 2 3 4 5 6 8 9 10 12
}
```

**Textual Figure:**

```
Ugly numbers: only prime factors 2, 3, 5

  Min heap + seen set:
  ┌─────┬───────┬──────────────────────────┐
  │ Pop │  Val  │ Push (val×2, val×3, val×5) │
  ├─────┼───────┼──────────────────────────┤
  │  1  │   1   │ push 2, 3, 5             │
  │  2  │   2   │ push 4, 6, 10            │
  │  3  │   3   │ push 9, 15 (6 seen)      │
  │  4  │   4   │ push 8, 12, 20           │
  │  5  │   5   │ push 25 (10,15 seen)     │
  │  6  │   6   │ push 18, 30 (12 seen)    │
  │  7  │   8   │ push 16, 24, 40          │
  │  8  │   9   │ push 27, 45 (18 seen)    │
  │  9  │  10   │ push 50 (20,30 seen)     │
  │ 10  │  12   │ ...                      │
  └─────┴───────┴──────────────────────────┘

  Ugly sequence: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 ...
```

---

## Example 8: Sort Characters by Frequency (LeetCode 451)

```go
package main

import (
	"container/heap"
	"fmt"
	"strings"
)

type CF struct {
	ch   rune
	freq int
}
type MaxCFHeap []CF
func (h MaxCFHeap) Len() int           { return len(h) }
func (h MaxCFHeap) Less(i, j int) bool { return h[i].freq > h[j].freq }
func (h MaxCFHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxCFHeap) Push(x interface{}) { *h = append(*h, x.(CF)) }
func (h *MaxCFHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func frequencySort(s string) string {
	freq := map[rune]int{}
	for _, c := range s {
		freq[c]++
	}
	h := &MaxCFHeap{}
	for ch, f := range freq {
		heap.Push(h, CF{ch, f})
	}
	var sb strings.Builder
	for h.Len() > 0 {
		item := heap.Pop(h).(CF)
		sb.WriteString(strings.Repeat(string(item.ch), item.freq))
	}
	return sb.String()
}

func main() {
	fmt.Println(frequencySort("tree"))    // "eert" or "eetr"
	fmt.Println(frequencySort("cccaaa"))  // "aaaccc" or "cccaaa"
	fmt.Println(frequencySort("Aabb"))    // "bbAa" or "bbaA"
}
```

**Textual Figure:**

```
Input: "tree"

  Step 1: Frequency count
  ┌─────┬──────┐
  │ Char│ Freq │
  ├─────┼──────┤
  │  t  │   1  │
  │  r  │   1  │
  │  e  │   2  │
  └─────┴──────┘

  Step 2: Max heap by frequency
      (e, 2)
      /    \
   (t, 1) (r, 1)

  Step 3: Pop and repeat char by freq
  Pop (e,2) → "ee"
  Pop (t,1) → "eet"
  Pop (r,1) → "eetr"

  Output: "eetr" (or "eert")
```

---

## Example 9: Maximum Score from K Operations

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type MaxHeapInt []int
func (h MaxHeapInt) Len() int           { return len(h) }
func (h MaxHeapInt) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeapInt) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeapInt) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeapInt) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxKelements(nums []int, k int) int64 {
	h := &MaxHeapInt{}
	for _, n := range nums {
		heap.Push(h, n)
	}
	var score int64
	for i := 0; i < k; i++ {
		top := heap.Pop(h).(int)
		score += int64(top)
		heap.Push(h, int(math.Ceil(float64(top)/3)))
	}
	return score
}

func main() {
	fmt.Println(maxKelements([]int{10, 10, 10, 10, 10}, 5)) // 50
	fmt.Println(maxKelements([]int{1, 10, 3, 3, 3}, 3))      // 17
}
```

**Textual Figure:**

```
Input: [1, 10, 3, 3, 3]    k=3

  Max heap: [10, 3, 3, 1, 3]

  k operations (take max, replace with ceil(max/3)):
  ┌────┬────────┬───────────┬────────────────────┐
  │ Op │  Take  │ Push back │ Score (cumulative) │
  ├────┼────────┼───────────┼────────────────────┤
  │ 1  │   10   │ ceil(10/3)=4│ 10                 │
  │ 2  │    4   │ ceil(4/3)=2 │ 14                 │
  │ 3  │    3   │ ceil(3/3)=1 │ 17                 │
  └────┴────────┴───────────┴────────────────────┘

  Heap after operations:
       3
      / \
     2   3
    / \
   1   1

  Total score: 17
```

---

## Example 10: Min vs Max Heap Comparison

```go
package main

import (
	"container/heap"
	"fmt"
)

// Min Heap
type MinH []int
func (h MinH) Len() int           { return len(h) }
func (h MinH) Less(i, j int) bool { return h[i] < h[j] }
func (h MinH) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// Max Heap via negation
func pushMax(h *MinH, val int) { heap.Push(h, -val) }
func popMax(h *MinH) int       { return -heap.Pop(h).(int) }
func peekMax(h *MinH) int      { return -(*h)[0] }

func main() {
	// Method 1: Reverse Less
	fmt.Println("=== Reverse Less method ===")

	// Method 2: Negate values (simpler, works with min heap)
	fmt.Println("=== Negation method ===")
	h := &MinH{}
	heap.Init(h)
	for _, v := range []int{5, 3, 8, 1, 2, 7} {
		pushMax(h, v)
	}
	fmt.Println("Max:", peekMax(h)) // 8
	for h.Len() > 0 {
		fmt.Print(popMax(h), " ") // 8 7 5 3 2 1
	}
	fmt.Println()
}
```

**Textual Figure:**

```
Min Heap vs Max Heap Comparison:

  Method 1: Reverse Less()          Method 2: Negate values
  ┌──────────────────────┐     ┌──────────────────────┐
  │ Less: h[i] > h[j]    │     │ Less: h[i] < h[j]    │
  │                      │     │ Store -val, negate   │
  │ Min heap:  1         │     │ Min heap: -8         │
  │           / \        │     │          / \         │
  │          3   2       │     │        -5  -7        │
  │                      │     │        / \  /        │
  │ Max heap:  8         │     │      -1 -3 -2        │
  │           / \        │     │                      │
  │          5   7       │     │ peekMax = -(-8) = 8  │
  │         / \  /       │     │ popMax  = -Pop() = 8 │
  │        1  3 2        │     │                      │
  └──────────────────────┘     └──────────────────────┘

  Input: [5, 3, 8, 1, 2, 7]
  Pop order (max): 8 → 7 → 5 → 3 → 2 → 1
```

---

## Key Takeaways

1. Max heap: parent ≥ children; root = maximum element
2. Two ways in Go: reverse `Less()` direction or negate values in a min heap
3. Kth largest → use min heap of size k (more efficient than max heap)
4. Task scheduling / frequency problems naturally use max heaps
5. Go's heap interface: just change `<` to `>` in `Less()` for max heap

> **Next up:** Heapify →
