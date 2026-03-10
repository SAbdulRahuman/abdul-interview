# Phase 23: Design Data Structures — Composite Data Structure Design

## Overview

Composite data structures **combine two or more basic data structures** to achieve operations that no single DS can support efficiently. This is the key pattern behind many "Design X" interview questions.

| Composite | Components | Enables |
|-----------|-----------|---------|
| LRU Cache | HashMap + Doubly Linked List | O(1) get + O(1) evict |
| LFU Cache | HashMap + HashMap + Doubly Linked List | O(1) get + O(1) least-frequent evict |
| Median Finder | Max-Heap + Min-Heap | O(log n) add + O(1) median |
| RandomizedSet | HashMap + Array | O(1) insert, remove, getRandom |
| Time Map | HashMap + Sorted Slice | O(1) set + O(log n) get by time |

---

## Example 1: LRU Cache (HashMap + Doubly Linked List)

```go
package main

import "fmt"

// LeetCode 146: LRU Cache

type LRUNode struct {
	key, val   int
	prev, next *LRUNode
}

type LRUCache struct {
	cap        int
	cache      map[int]*LRUNode
	head, tail *LRUNode
}

func NewLRU(capacity int) *LRUCache {
	head := &LRUNode{}
	tail := &LRUNode{}
	head.next = tail
	tail.prev = head
	return &LRUCache{cap: capacity, cache: make(map[int]*LRUNode), head: head, tail: tail}
}

func (l *LRUCache) remove(node *LRUNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
}

func (l *LRUCache) addToFront(node *LRUNode) {
	node.next = l.head.next
	node.prev = l.head
	l.head.next.prev = node
	l.head.next = node
}

func (l *LRUCache) Get(key int) int {
	if node, ok := l.cache[key]; ok {
		l.remove(node)
		l.addToFront(node)
		return node.val
	}
	return -1
}

func (l *LRUCache) Put(key, value int) {
	if node, ok := l.cache[key]; ok {
		node.val = value
		l.remove(node)
		l.addToFront(node)
		return
	}
	node := &LRUNode{key: key, val: value}
	l.cache[key] = node
	l.addToFront(node)
	if len(l.cache) > l.cap {
		lru := l.tail.prev
		l.remove(lru)
		delete(l.cache, lru.key)
	}
}

func main() {
	cache := NewLRU(3)
	cache.Put(1, 10)
	cache.Put(2, 20)
	cache.Put(3, 30)
	fmt.Println("Get 1:", cache.Get(1)) // 10 (moves to front)
	cache.Put(4, 40)                     // evicts key 2
	fmt.Println("Get 2:", cache.Get(2)) // -1
	fmt.Println("Get 3:", cache.Get(3)) // 30
	fmt.Println("Get 4:", cache.Get(4)) // 40

	fmt.Println("\nHashMap → O(1) lookup")
	fmt.Println("Doubly Linked List → O(1) move/evict")
}
```

**Textual Figure:**
```
LRU Cache (capacity = 3): HashMap + Doubly Linked List

  Structure:
  ┌──────────────────────────────────────────────────────┐
  │  HashMap: key → node pointer (O(1) lookup)           │
  │  DLL:    HEAD ↔ [MRU] ↔ ... ↔ [LRU] ↔ TAIL          │
  └──────────────────────────────────────────────────────┘

  Step-by-step:

  Put(1,10):  HEAD ↔ [1:10] ↔ TAIL
              map: {1:●}

  Put(2,20):  HEAD ↔ [2:20] ↔ [1:10] ↔ TAIL
              map: {1:●, 2:●}

  Put(3,30):  HEAD ↔ [3:30] ↔ [2:20] ↔ [1:10] ↔ TAIL
              map: {1:●, 2:●, 3:●}

  Get(1)=10:  HEAD ↔ [1:10] ↔ [3:30] ↔ [2:20] ↔ TAIL   ← move 1 to front
              map: {1:●, 2:●, 3:●}

  Put(4,40):  HEAD ↔ [4:40] ↔ [1:10] ↔ [3:30] ↔ TAIL   ← evict LRU key=2
              map: {1:●, 3:●, 4:●}       ✗ key 2 removed

  Get(2)=-1   (evicted)
  Get(3)=30:  HEAD ↔ [3:30] ↔ [4:40] ↔ [1:10] ↔ TAIL   ← move 3 to front
  Get(4)=40:  HEAD ↔ [4:40] ↔ [3:30] ↔ [1:10] ↔ TAIL   ← move 4 to front

  ┌───────────────────────────────────┐
  │ Get: map lookup O(1) + move O(1) │
  │ Put: map insert O(1) + evict O(1)│
  └───────────────────────────────────┘
```

---

## Example 2: LFU Cache (HashMap + Frequency Linked Lists)

```go
package main

import "fmt"

// LeetCode 460: LFU Cache

type LFUNode struct {
	key, val, freq int
	prev, next     *LFUNode
}

type FreqList struct {
	head, tail *LFUNode
	size       int
}

func newFreqList() *FreqList {
	h, t := &LFUNode{}, &LFUNode{}
	h.next = t
	t.prev = h
	return &FreqList{head: h, tail: t}
}

func (fl *FreqList) addFront(node *LFUNode) {
	node.next = fl.head.next
	node.prev = fl.head
	fl.head.next.prev = node
	fl.head.next = node
	fl.size++
}

func (fl *FreqList) remove(node *LFUNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
	fl.size--
}

func (fl *FreqList) removeLast() *LFUNode {
	if fl.size == 0 { return nil }
	node := fl.tail.prev
	fl.remove(node)
	return node
}

type LFUCache struct {
	cap, minFreq int
	cache        map[int]*LFUNode
	freqMap      map[int]*FreqList
}

func NewLFU(capacity int) *LFUCache {
	return &LFUCache{cap: capacity, cache: make(map[int]*LFUNode), freqMap: make(map[int]*FreqList)}
}

func (l *LFUCache) update(node *LFUNode) {
	fl := l.freqMap[node.freq]
	fl.remove(node)
	if node.freq == l.minFreq && fl.size == 0 { l.minFreq++ }
	node.freq++
	if l.freqMap[node.freq] == nil { l.freqMap[node.freq] = newFreqList() }
	l.freqMap[node.freq].addFront(node)
}

func (l *LFUCache) Get(key int) int {
	if node, ok := l.cache[key]; ok { l.update(node); return node.val }
	return -1
}

func (l *LFUCache) Put(key, val int) {
	if l.cap == 0 { return }
	if node, ok := l.cache[key]; ok {
		node.val = val
		l.update(node)
		return
	}
	if len(l.cache) >= l.cap {
		fl := l.freqMap[l.minFreq]
		evicted := fl.removeLast()
		delete(l.cache, evicted.key)
	}
	node := &LFUNode{key: key, val: val, freq: 1}
	l.cache[key] = node
	l.minFreq = 1
	if l.freqMap[1] == nil { l.freqMap[1] = newFreqList() }
	l.freqMap[1].addFront(node)
}

func main() {
	cache := NewLFU(2)
	cache.Put(1, 10)
	cache.Put(2, 20)
	fmt.Println("Get 1:", cache.Get(1)) // 10 (freq 2)
	cache.Put(3, 30)                     // evicts key 2 (freq 1, LRU)
	fmt.Println("Get 2:", cache.Get(2)) // -1
	fmt.Println("Get 3:", cache.Get(3)) // 30

	fmt.Println("\nAll operations O(1)!")
}
```

**Textual Figure:**
```
LFU Cache (capacity = 2): HashMap + Frequency Linked Lists

  Structure:
  ┌──────────────────────────────────────────────────────────┐
  │  cache:   key → node pointer                             │
  │  freqMap: freq → doubly linked list of nodes (LRU order) │
  │  minFreq: tracks current minimum frequency               │
  └──────────────────────────────────────────────────────────┘

  Step-by-step:

  Put(1,10):  cache: {1:●}   minFreq=1
              freqMap: { 1: HEAD ↔ [1:10,f=1] ↔ TAIL }

  Put(2,20):  cache: {1:●, 2:●}   minFreq=1
              freqMap: { 1: HEAD ↔ [2:20,f=1] ↔ [1:10,f=1] ↔ TAIL }

  Get(1)=10:  cache: {1:●, 2:●}   minFreq=1
              freqMap: { 1: HEAD ↔ [2:20,f=1] ↔ TAIL
                         2: HEAD ↔ [1:10,f=2] ↔ TAIL }
              (key 1 moved from freq 1 → freq 2)

  Put(3,30):  Evict LRU from minFreq=1 → evict key 2
              cache: {1:●, 3:●}   minFreq=1
              freqMap: { 1: HEAD ↔ [3:30,f=1] ↔ TAIL
                         2: HEAD ↔ [1:10,f=2] ↔ TAIL }

  Get(2)=-1   (evicted)
  Get(3)=30:  freq 1→2 for key 3
              freqMap: { 2: HEAD ↔ [3:30,f=2] ↔ [1:10,f=2] ↔ TAIL }

  ┌────────────────────────────────────────────────┐
  │ Eviction: remove LRU node from minFreq list    │
  │ All operations O(1) via dual HashMap + DLL     │
  └────────────────────────────────────────────────┘
```

---

## Example 3: Insert Delete GetRandom O(1) (HashMap + Array)

```go
package main

import (
	"fmt"
	"math/rand"
)

// LeetCode 380

type RandomizedSet struct {
	arr []int
	idx map[int]int // val → index in arr
}

func NewRandomizedSet() *RandomizedSet {
	return &RandomizedSet{idx: make(map[int]int)}
}

func (rs *RandomizedSet) Insert(val int) bool {
	if _, ok := rs.idx[val]; ok { return false }
	rs.idx[val] = len(rs.arr)
	rs.arr = append(rs.arr, val)
	return true
}

func (rs *RandomizedSet) Remove(val int) bool {
	i, ok := rs.idx[val]
	if !ok { return false }
	// Swap with last, then pop
	last := rs.arr[len(rs.arr)-1]
	rs.arr[i] = last
	rs.idx[last] = i
	rs.arr = rs.arr[:len(rs.arr)-1]
	delete(rs.idx, val)
	return true
}

func (rs *RandomizedSet) GetRandom() int {
	return rs.arr[rand.Intn(len(rs.arr))]
}

func main() {
	rs := NewRandomizedSet()
	rs.Insert(1)
	rs.Insert(2)
	rs.Insert(3)
	fmt.Println("After inserting 1,2,3")

	rs.Remove(2)
	fmt.Println("After removing 2")

	freq := map[int]int{}
	for i := 0; i < 1000; i++ { freq[rs.GetRandom()]++ }
	fmt.Println("Random distribution (1000 samples):", freq)

	fmt.Println("\nKey trick: swap-with-last for O(1) removal from array")
}
```

**Textual Figure:**
```
Insert Delete GetRandom O(1): HashMap + Array

  Structure:
  ┌───────────────────────────────────────────────┐
  │  arr: [values...]        ← random index O(1)  │
  │  idx: val → index in arr ← lookup O(1)        │
  └───────────────────────────────────────────────┘

  Insert(1):  arr: [1]        idx: {1:0}
  Insert(2):  arr: [1,2]      idx: {1:0, 2:1}
  Insert(3):  arr: [1,2,3]    idx: {1:0, 2:1, 3:2}

  Remove(2) — swap-with-last trick:
    Step 1: Find index of 2 → idx[2] = 1
    Step 2: Swap arr[1] with arr[2] (last)

            Before:  arr: [1, 2, 3]    idx: {1:0, 2:1, 3:2}
                          ─────↑─────
                              remove

            Swap:    arr: [1, 3, 3]    idx: {1:0, 3:1}
                              ↑ last moved here

    Step 3: Pop last element
            After:   arr: [1, 3]       idx: {1:0, 3:1}

  GetRandom(): return arr[rand(0..len-1)]  → O(1)

  ┌────────────────────────────────────────────────────┐
  │ Key insight: swap-with-last avoids O(n) array shift│
  │ All three operations: O(1) time                    │
  └────────────────────────────────────────────────────┘
```

---

## Example 4: Find Median from Data Stream (Two Heaps)

```go
package main

import (
	"container/heap"
	"fmt"
)

// LeetCode 295

type MaxHeap []int
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() any          { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any          { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

type MedianFinder struct {
	lo *MaxHeap // left half (max-heap)
	hi *MinHeap // right half (min-heap)
}

func NewMedianFinder() *MedianFinder {
	lo := &MaxHeap{}
	hi := &MinHeap{}
	return &MedianFinder{lo, hi}
}

func (mf *MedianFinder) AddNum(num int) {
	heap.Push(mf.lo, num)
	// Balance: lo.top should be <= hi.top
	if mf.hi.Len() > 0 && (*mf.lo)[0] > (*mf.hi)[0] {
		heap.Push(mf.hi, heap.Pop(mf.lo))
	}
	// Size balance: lo.Len() == hi.Len() or lo.Len() == hi.Len()+1
	if mf.lo.Len() > mf.hi.Len()+1 {
		heap.Push(mf.hi, heap.Pop(mf.lo))
	} else if mf.hi.Len() > mf.lo.Len() {
		heap.Push(mf.lo, heap.Pop(mf.hi))
	}
}

func (mf *MedianFinder) FindMedian() float64 {
	if mf.lo.Len() > mf.hi.Len() { return float64((*mf.lo)[0]) }
	return float64((*mf.lo)[0]+(*mf.hi)[0]) / 2.0
}

func main() {
	mf := NewMedianFinder()
	nums := []int{5, 15, 1, 3, 8, 7, 9}
	for _, n := range nums {
		mf.AddNum(n)
		fmt.Printf("Add %2d → Median = %.1f\n", n, mf.FindMedian())
	}
}
```

**Textual Figure:**
```
Find Median from Data Stream: Two Heaps (MaxHeap + MinHeap)

  Structure:
  ┌─────────────────────────────────────────────────┐
  │  lo (MaxHeap): left half     hi (MinHeap): right│
  │  top = max of left           top = min of right │
  │  Invariant: lo.top ≤ hi.top                     │
  │  Size: |lo| == |hi| or |lo| == |hi|+1           │
  └─────────────────────────────────────────────────┘

  Add   lo (MaxHeap)    hi (MinHeap)    Median
  ───   ─────────────   ─────────────   ──────
   5    [5]             []              5.0
  15    [5]             [15]            10.0
   1    [5,1]           [15]            5.0
   3    [5,3,1]         [15]            5.0
        → rebalance →
        [3,1]           [5,15]          4.0
   8    [5,3,1]         [8,15]          5.0
   7    [5,3,1]         [7,8,15]        6.0
        → rebalance →
        [7,5,3,1]       [8,15]          7.0
   9    [7,5,3,1]       [8,9,15]        7.0
        → final →
        [7,5,3,1]       [8,9,15]        7.0

         lo (max-heap)    hi (min-heap)
            ┌───┐            ┌───┐
     top →  │ 7 │            │ 8 │  ← top
            ├───┤            ├───┤
            │ 5 │            │ 9 │
            ├───┤            ├───┤
            │ 3 │            │15 │
            ├───┤            └───┘
            │ 1 │
            └───┘
     Median = 7.0 (odd count → lo.top)
```

---

## Example 5: Time-Based Key-Value Store (HashMap + Binary Search)

```go
package main

import (
	"fmt"
	"sort"
)

// LeetCode 981

type TimeEntry struct {
	timestamp int
	value     string
}

type TimeMap struct {
	store map[string][]TimeEntry
}

func NewTimeMap() *TimeMap {
	return &TimeMap{store: make(map[string][]TimeEntry)}
}

func (tm *TimeMap) Set(key, value string, timestamp int) {
	tm.store[key] = append(tm.store[key], TimeEntry{timestamp, value})
}

func (tm *TimeMap) Get(key string, timestamp int) string {
	entries, ok := tm.store[key]
	if !ok { return "" }
	// Binary search: largest timestamp <= given timestamp
	i := sort.Search(len(entries), func(i int) bool { return entries[i].timestamp > timestamp })
	if i == 0 { return "" }
	return entries[i-1].value
}

func main() {
	tm := NewTimeMap()
	tm.Set("foo", "bar", 1)
	tm.Set("foo", "bar2", 4)

	fmt.Println("Get(foo, 1):", tm.Get("foo", 1))  // bar
	fmt.Println("Get(foo, 3):", tm.Get("foo", 3))  // bar
	fmt.Println("Get(foo, 4):", tm.Get("foo", 4))  // bar2
	fmt.Println("Get(foo, 5):", tm.Get("foo", 5))  // bar2

	fmt.Println("\nHashMap + sorted entries + binary search")
}
```

**Textual Figure:**
```
Time-Based Key-Value Store: HashMap + Binary Search

  Structure:
  ┌──────────────────────────────────────────────────────┐
  │  store: map[string][]TimeEntry                       │
  │  Each key → sorted list of (timestamp, value) pairs  │
  └──────────────────────────────────────────────────────┘

  Set("foo","bar",1):   store["foo"] = [(1,"bar")]
  Set("foo","bar2",4):  store["foo"] = [(1,"bar"), (4,"bar2")]

  store["foo"]:
    ┌──────────┬───────────┐
    │ ts=1     │ ts=4      │
    │ val=bar  │ val=bar2  │
    └──────────┴───────────┘
      ↑                 ↑
      sorted by timestamp

  Get("foo", 1) → binary search: largest ts ≤ 1 → "bar"
  Get("foo", 3) → binary search: largest ts ≤ 3 → "bar"  (ts=1)
  Get("foo", 4) → binary search: largest ts ≤ 4 → "bar2" (ts=4)
  Get("foo", 5) → binary search: largest ts ≤ 5 → "bar2" (ts=4)

  Binary search on timeline:
  ──┬────────────────┬──────────────▶ time
    1                4
    "bar"            "bar2"
     ◄── query 1,3 ──►◄── query 4,5 ──►

  ┌────────────────────────────────────┐
  │ Set: O(1) — append to sorted list │
  │ Get: O(log n) — binary search     │
  └────────────────────────────────────┘
```

---

## Example 6: Max Frequency Stack (HashMap + Stack per Frequency)

```go
package main

import "fmt"

// LeetCode 895: Maximum Frequency Stack

type FreqStack struct {
	freq    map[int]int     // val → frequency
	groups  map[int][]int    // freq → stack of values
	maxFreq int
}

func NewFreqStack() *FreqStack {
	return &FreqStack{freq: make(map[int]int), groups: make(map[int][]int)}
}

func (fs *FreqStack) Push(val int) {
	fs.freq[val]++
	f := fs.freq[val]
	fs.groups[f] = append(fs.groups[f], val)
	if f > fs.maxFreq { fs.maxFreq = f }
}

func (fs *FreqStack) Pop() int {
	stack := fs.groups[fs.maxFreq]
	val := stack[len(stack)-1]
	fs.groups[fs.maxFreq] = stack[:len(stack)-1]
	if len(fs.groups[fs.maxFreq]) == 0 { fs.maxFreq-- }
	fs.freq[val]--
	return val
}

func main() {
	fs := NewFreqStack()
	pushes := []int{5, 7, 5, 7, 4, 5}
	for _, v := range pushes { fs.Push(v) }
	fmt.Printf("After pushing %v:\n", pushes)

	for i := 0; i < 4; i++ {
		fmt.Printf("  Pop → %d\n", fs.Pop())
	}
	// Pops: 5 (freq 3), 7 (freq 2), 5 (freq 2), 4 (freq 1)

	fmt.Println("\nfreq map + group stacks = O(1) push/pop")
}
```

**Textual Figure:**
```
Max Frequency Stack: HashMap + Stack per Frequency

  Structure:
  ┌───────────────────────────────────────────────┐
  │ freq:   val → count                              │
  │ groups: freq → stack of values                    │
  │ maxFreq: current maximum frequency                 │
  └───────────────────────────────────────────────┘

  Push sequence: [5, 7, 5, 7, 4, 5]

  Push  freq map         groups                     maxFreq
  ────  ──────────────  ────────────────────────  ───────
  5     {5:1}             1:[5]                      1
  7     {5:1,7:1}         1:[5,7]                    1
  5     {5:2,7:1}         1:[5,7] 2:[5]              2
  7     {5:2,7:2}         1:[5,7] 2:[5,7]            2
  4     {5:2,7:2,4:1}     1:[5,7,4] 2:[5,7]          2
  5     {5:3,7:2,4:1}     1:[5,7,4] 2:[5,7] 3:[5]    3

  Group stacks state:
    freq=3: │ 5 │  ← maxFreq, pop here first
    freq=2: │ 5 │ 7 │
    freq=1: │ 5 │ 7 │ 4 │

  Pop() = 5  (from freq=3, maxFreq: 3→2)
  Pop() = 7  (from freq=2, maxFreq stays 2)
  Pop() = 5  (from freq=2, maxFreq: 2→1)
  Pop() = 4  (from freq=1, maxFreq stays 1)
```

---

## Example 7: All O(1) Data Structure (HashMap + Doubly Linked List)

```go
package main

import "fmt"

// LeetCode 432: All O(1) — inc, dec, getMaxKey, getMinKey in O(1)

type Bucket struct {
	count      int
	keys       map[string]bool
	prev, next *Bucket
}

type AllOne struct {
	head, tail *Bucket
	keyMap     map[string]*Bucket
}

func NewAllOne() AllOne {
	h, t := &Bucket{keys: map[string]bool{}}, &Bucket{keys: map[string]bool{}}
	h.next = t
	t.prev = h
	return AllOne{head: h, tail: t, keyMap: make(map[string]*Bucket)}
}

func (ao *AllOne) insertAfter(prev *Bucket, count int) *Bucket {
	b := &Bucket{count: count, keys: map[string]bool{}, prev: prev, next: prev.next}
	prev.next.prev = b
	prev.next = b
	return b
}

func (ao *AllOne) removeBucket(b *Bucket) {
	b.prev.next = b.next
	b.next.prev = b.prev
}

func (ao *AllOne) Inc(key string) {
	if bucket, ok := ao.keyMap[key]; ok {
		newCount := bucket.count + 1
		var newBucket *Bucket
		if bucket.next.count == newCount {
			newBucket = bucket.next
		} else {
			newBucket = ao.insertAfter(bucket, newCount)
		}
		newBucket.keys[key] = true
		delete(bucket.keys, key)
		if len(bucket.keys) == 0 { ao.removeBucket(bucket) }
		ao.keyMap[key] = newBucket
	} else {
		var bucket *Bucket
		if ao.head.next.count == 1 {
			bucket = ao.head.next
		} else {
			bucket = ao.insertAfter(ao.head, 1)
		}
		bucket.keys[key] = true
		ao.keyMap[key] = bucket
	}
}

func (ao *AllOne) Dec(key string) {
	bucket := ao.keyMap[key]
	newCount := bucket.count - 1
	delete(bucket.keys, key)

	if newCount > 0 {
		var newBucket *Bucket
		if bucket.prev.count == newCount {
			newBucket = bucket.prev
		} else {
			newBucket = ao.insertAfter(bucket.prev, newCount)
		}
		newBucket.keys[key] = true
		ao.keyMap[key] = newBucket
	} else {
		delete(ao.keyMap, key)
	}

	if len(bucket.keys) == 0 { ao.removeBucket(bucket) }
}

func (ao *AllOne) GetMaxKey() string {
	if ao.tail.prev == ao.head { return "" }
	for k := range ao.tail.prev.keys { return k }
	return ""
}

func (ao *AllOne) GetMinKey() string {
	if ao.head.next == ao.tail { return "" }
	for k := range ao.head.next.keys { return k }
	return ""
}

func main() {
	ao := NewAllOne()
	ao.Inc("a"); ao.Inc("a"); ao.Inc("b")
	fmt.Println("After inc(a), inc(a), inc(b):")
	fmt.Println("  Max:", ao.GetMaxKey()) // a (count 2)
	fmt.Println("  Min:", ao.GetMinKey()) // b (count 1)

	ao.Inc("b"); ao.Inc("b")
	fmt.Println("\nAfter inc(b), inc(b):")
	fmt.Println("  Max:", ao.GetMaxKey()) // b (count 3)
	fmt.Println("  Min:", ao.GetMinKey()) // a (count 2)

	fmt.Println("\nBucket DLL sorted by count + HashMap for O(1)")
}
```

**Textual Figure:**
```
All O(1) Data Structure: Bucket DLL + HashMap

  Structure:
  ┌────────────────────────────────────────────────────┐
  │ keyMap: string → *Bucket                            │
  │ Bucket DLL: sorted by count, each has a set of keys │
  │ HEAD ↔ [count=min] ↔ ... ↔ [count=max] ↔ TAIL       │
  └────────────────────────────────────────────────────┘

  Step-by-step:

  Inc(a), Inc(a), Inc(b):
    keyMap: {a: bucket(2), b: bucket(1)}
    HEAD ↔ [count=1, keys={b}] ↔ [count=2, keys={a}] ↔ TAIL
    Min="b" (count=1)    Max="a" (count=2)

  Inc(b), Inc(b):
    keyMap: {a: bucket(2), b: bucket(3)}
    HEAD ↔ [count=2, keys={a}] ↔ [count=3, keys={b}] ↔ TAIL
    Min="a" (count=2)    Max="b" (count=3)

  Bucket DLL visualization:
    HEAD ↔ ┌───────────┐ ↔ ┌───────────┐ ↔ TAIL
            │ count = 2 │   │ count = 3 │
            │ keys={a}  │   │ keys={b}  │
            └───────────┘   └───────────┘
            ↑ getMinKey      ↑ getMaxKey

  ┌──────────────────────────────────────────┐
  │ Inc/Dec: O(1) — move key between buckets │
  │ GetMin/Max: O(1) — head.next / tail.prev │
  └──────────────────────────────────────────┘
```

---

## Example 8: Design Twitter (HashMap + Merge k Sorted)

```go
package main

import (
	"container/heap"
	"fmt"
)

// LeetCode 355

type Tweet struct {
	id, time int
}

type Twitter struct {
	timer   int
	tweets  map[int][]Tweet
	follows map[int]map[int]bool
}

func NewTwitter() *Twitter {
	return &Twitter{tweets: make(map[int][]Tweet), follows: make(map[int]map[int]bool)}
}

func (t *Twitter) PostTweet(userId, tweetId int) {
	t.timer++
	t.tweets[userId] = append(t.tweets[userId], Tweet{tweetId, t.timer})
}

func (t *Twitter) Follow(followerId, followeeId int) {
	if t.follows[followerId] == nil { t.follows[followerId] = make(map[int]bool) }
	t.follows[followerId][followeeId] = true
}

func (t *Twitter) Unfollow(followerId, followeeId int) {
	if t.follows[followerId] != nil { delete(t.follows[followerId], followeeId) }
}

type TweetHeap []struct { tweet Tweet; userId, idx int }
func (h TweetHeap) Len() int            { return len(h) }
func (h TweetHeap) Less(i, j int) bool  { return h[i].tweet.time > h[j].tweet.time }
func (h TweetHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *TweetHeap) Push(x any)         { *h = append(*h, x.(struct{ tweet Tweet; userId, idx int })) }
func (h *TweetHeap) Pop() any           { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

func (t *Twitter) GetNewsFeed(userId int) []int {
	users := map[int]bool{userId: true}
	for f := range t.follows[userId] { users[f] = true }

	h := &TweetHeap{}
	for uid := range users {
		if tweets := t.tweets[uid]; len(tweets) > 0 {
			idx := len(tweets) - 1
			heap.Push(h, struct{ tweet Tweet; userId, idx int }{tweets[idx], uid, idx})
		}
	}

	var result []int
	for h.Len() > 0 && len(result) < 10 {
		top := heap.Pop(h).(struct{ tweet Tweet; userId, idx int })
		result = append(result, top.tweet.id)
		if top.idx > 0 {
			top.idx--
			heap.Push(h, struct{ tweet Tweet; userId, idx int }{t.tweets[top.userId][top.idx], top.userId, top.idx})
		}
	}
	return result
}

func main() {
	tw := NewTwitter()
	tw.PostTweet(1, 101)
	tw.PostTweet(2, 201)
	tw.Follow(1, 2)
	tw.PostTweet(2, 202)
	tw.PostTweet(1, 102)

	fmt.Println("User 1 feed:", tw.GetNewsFeed(1))
	// [102, 202, 201, 101] — latest 10 from self + followees

	tw.Unfollow(1, 2)
	fmt.Println("After unfollow:", tw.GetNewsFeed(1))
	// [102, 101]
}
```

**Textual Figure:**
```
Design Twitter: HashMap + Merge k Sorted Lists

  Structure:
  ┌───────────────────────────────────────────────────┐
  │ tweets:  userId → [(tweetId, time), ...]            │
  │ follows: userId → set of followeeIds                │
  │ timer:   global auto-incrementing timestamp          │
  └───────────────────────────────────────────────────┘

  PostTweet(1,101)  tweets[1]: [(101,t=1)]
  PostTweet(2,201)  tweets[2]: [(201,t=2)]
  Follow(1,2)       follows[1]: {2}
  PostTweet(2,202)  tweets[2]: [(201,t=2),(202,t=3)]
  PostTweet(1,102)  tweets[1]: [(101,t=1),(102,t=4)]

  GetNewsFeed(1):  merge tweets from {user 1, user 2}
    User 1: [(101,t=1), (102,t=4)]
    User 2: [(201,t=2), (202,t=3)]

    Max-Heap merge (by time desc):
    ┌───────────────────────────────┐
    │ Pop t=4: tweet 102 (user 1) │
    │ Pop t=3: tweet 202 (user 2) │
    │ Pop t=2: tweet 201 (user 2) │
    │ Pop t=1: tweet 101 (user 1) │
    └───────────────────────────────┘
    Result: [102, 202, 201, 101]

  Unfollow(1,2) → follows[1]: {}
  GetNewsFeed(1): only user 1's tweets → [102, 101]
```

---

## Example 9: Snapshot Array (HashMap per Index)

```go
package main

import (
	"fmt"
	"sort"
)

// LeetCode 1146

type SnapshotArray struct {
	data   [][]struct{ snap, val int }
	snapId int
}

func NewSnapshotArray(length int) *SnapshotArray {
	data := make([][]struct{ snap, val int }, length)
	for i := range data {
		data[i] = []struct{ snap, val int }{{0, 0}}
	}
	return &SnapshotArray{data: data}
}

func (sa *SnapshotArray) Set(index, val int) {
	history := sa.data[index]
	if history[len(history)-1].snap == sa.snapId {
		history[len(history)-1].val = val
	} else {
		sa.data[index] = append(history, struct{ snap, val int }{sa.snapId, val})
	}
}

func (sa *SnapshotArray) Snap() int {
	id := sa.snapId
	sa.snapId++
	return id
}

func (sa *SnapshotArray) Get(index, snapId int) int {
	history := sa.data[index]
	i := sort.Search(len(history), func(i int) bool { return history[i].snap > snapId })
	return history[i-1].val
}

func main() {
	sa := NewSnapshotArray(3)
	sa.Set(0, 5)
	snap0 := sa.Snap()  // snap 0
	sa.Set(0, 6)
	snap1 := sa.Snap()  // snap 1
	sa.Set(0, 7)

	fmt.Printf("Get(0, snap%d) = %d\n", snap0, sa.Get(0, snap0)) // 5
	fmt.Printf("Get(0, snap%d) = %d\n", snap1, sa.Get(0, snap1)) // 6
	fmt.Printf("Get(0, current is 7 but no snap yet)\n")

	fmt.Println("\nOnly stores changes → memory efficient")
}
```

**Textual Figure:**
```
Snapshot Array: Per-Index History + Binary Search

  Structure:
  ┌────────────────────────────────────────────────────────┐
  │ data[i] = sorted list of (snapId, value) entries    │
  │ Only records changes → memory efficient              │
  └────────────────────────────────────────────────────────┘

  Operations trace (length=3):

  Set(0, 5):       data[0] = [(snap=0, val=5)]
  Snap() → snap 0
  Set(0, 6):       data[0] = [(snap=0, val=5), (snap=1, val=6)]
  Snap() → snap 1
  Set(0, 7):       data[0] = [(snap=0, val=5), (snap=1, val=6), (snap=2, val=7)]

  data[0] history:
    ┌────────────┬────────────┬────────────┐
    │ snap=0     │ snap=1     │ snap=2     │
    │ val=5      │ val=6      │ val=7      │
    └────────────┴────────────┴────────────┘

  Get(0, snap0) → binary search: largest snap ≤ 0 → val=5
  Get(0, snap1) → binary search: largest snap ≤ 1 → val=6
  data[1], data[2] remain [(snap=0, val=0)] (unchanged)

  ┌────────────────────────────────────────────────┐
  │ Set: O(1) append    Get: O(log S) binary search│
  │ Snap: O(1)          S = number of snapshots     │
  └────────────────────────────────────────────────┘
```

---

## Example 10: Design HashMap (Open Addressing)

```go
package main

import "fmt"

// Simple open-addressing hashmap from scratch

const SIZE = 1024

type Entry struct {
	key, val int
	used     bool
	deleted  bool
}

type MyHashMap struct {
	table [SIZE]Entry
}

func (hm *MyHashMap) hash(key int) int { return key % SIZE }

func (hm *MyHashMap) Put(key, val int) {
	h := hm.hash(key)
	for i := 0; i < SIZE; i++ {
		idx := (h + i) % SIZE
		if !hm.table[idx].used || hm.table[idx].deleted || hm.table[idx].key == key {
			hm.table[idx] = Entry{key, val, true, false}
			return
		}
	}
}

func (hm *MyHashMap) Get(key int) int {
	h := hm.hash(key)
	for i := 0; i < SIZE; i++ {
		idx := (h + i) % SIZE
		if !hm.table[idx].used { return -1 }
		if !hm.table[idx].deleted && hm.table[idx].key == key { return hm.table[idx].val }
	}
	return -1
}

func (hm *MyHashMap) Remove(key int) {
	h := hm.hash(key)
	for i := 0; i < SIZE; i++ {
		idx := (h + i) % SIZE
		if !hm.table[idx].used { return }
		if !hm.table[idx].deleted && hm.table[idx].key == key {
			hm.table[idx].deleted = true
			return
		}
	}
}

func main() {
	hm := &MyHashMap{}
	hm.Put(1, 100)
	hm.Put(2, 200)
	fmt.Println("Get 1:", hm.Get(1)) // 100
	fmt.Println("Get 3:", hm.Get(3)) // -1

	hm.Remove(1)
	fmt.Println("After remove 1, Get 1:", hm.Get(1)) // -1

	// Test collision
	hm.Put(1025, 999) // hash collision with key 1
	fmt.Println("Get 1025:", hm.Get(1025)) // 999

	fmt.Println("\nLinear probing: simple but watch load factor")
}
```

**Textual Figure:**
```
Design HashMap: Open Addressing with Linear Probing

  Structure:
  ┌───────────────────────────────────────────┐
  │ table[SIZE] of Entry{key, val, used, deleted} │
  │ hash(key) = key % SIZE                        │
  └───────────────────────────────────────────┘

  Put(1,100):   hash(1)=1     table[1] = {1, 100, used}
  Put(2,200):   hash(2)=2     table[2] = {2, 200, used}

  table (SIZE=1024, showing slots 0-5):
    idx:  │  0  │  1       │  2       │  3  │  4  │  5  │
    ─────┼────┼─────────┼─────────┼────┼────┼────┤
    val:  │    │ key=1    │ key=2    │    │    │    │
          │    │ val=100  │ val=200  │    │    │    │

  Remove(1): table[1].deleted = true  (tombstone)
    idx:  │  0  │  1       │  2       │  3  │
    ─────┼────┼─────────┼─────────┼────┤
    val:  │    │ ✗ DEL   │ key=2    │    │

  Put(1025,999): hash(1025)=1 → slot 1 is deleted → reuse!
    idx:  │  0  │  1         │  2       │  3  │
    ─────┼────┼───────────┼─────────┼────┤
    val:  │    │ key=1025   │ key=2    │    │
          │    │ val=999    │ val=200  │    │

  Linear probing on collision:
    hash(k) → slot occupied? → try slot+1 → slot+2 → ...
```

---

## Example 11: Design Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Composite Data Structure Design Patterns ===\n")

	patterns := []struct {
		problem, components, trick string
	}{
		{"LRU Cache", "HashMap + DLL", "Move-to-front on access"},
		{"LFU Cache", "HashMap + freq buckets + DLL", "Maintain minFreq"},
		{"RandomizedSet", "HashMap + Array", "Swap-with-last for removal"},
		{"Median Finder", "MaxHeap + MinHeap", "Balance heap sizes"},
		{"TimeMap", "HashMap + sorted list", "Binary search by timestamp"},
		{"Max Freq Stack", "freq map + group stacks", "pop from maxFreq group"},
		{"All O(1)", "key map + count bucket DLL", "Buckets sorted by count"},
		{"Twitter Feed", "follow graph + merge k sorted", "Heap-based merge"},
		{"Snapshot Array", "per-index history + binary search", "Store only changes"},
		{"HashMap", "Array + probing/chaining", "Hash function + collision resolution"},
	}

	for _, p := range patterns {
		fmt.Printf("%-18s: %s → %s\n", p.problem, p.components, p.trick)
	}

	fmt.Println("\n--- Design Approach ---")
	fmt.Println("1. List required operations + their time constraints")
	fmt.Println("2. No single DS satisfies all? → Combine!")
	fmt.Println("3. Map provides O(1) lookup; List/Tree provides ordering")
	fmt.Println("4. Keep both structures synchronized")
}
```

**Textual Figure:**
```
Composite Data Structure Design: Pattern Summary

  ┌────────────────┬──────────────────────────┬──────────────────────────┐
  │ Problem        │ Components               │ Key Trick                │
  ├────────────────┼──────────────────────────┼──────────────────────────┤
  │ LRU Cache      │ HashMap + DLL            │ Move-to-front on access  │
  │ LFU Cache      │ HashMap + freq DLLs      │ Maintain minFreq         │
  │ RandomizedSet  │ HashMap + Array           │ Swap-with-last removal   │
  │ Median Finder  │ MaxHeap + MinHeap         │ Balance heap sizes       │
  │ TimeMap        │ HashMap + sorted list     │ Binary search by time    │
  │ FreqStack      │ freq map + group stacks   │ Pop from maxFreq group   │
  │ All O(1)       │ key map + bucket DLL      │ Buckets sorted by count  │
  │ Twitter Feed   │ follow graph + heap merge │ Merge k sorted lists     │
  │ Snapshot Array │ per-index history + bsrch │ Store only changes       │
  │ HashMap        │ Array + probing/chaining  │ Hash + collision resolve │
  └────────────────┴──────────────────────────┴──────────────────────────┘

  Design Approach:
    1. List required ops + time constraints
           │
    2. No single DS works? → Combine!
           │
    3. Map = O(1) lookup  ─┬─  List/Tree = ordering
                           │
    4. Keep BOTH in sync on every insert/delete
```

---

## Key Takeaways

1. **HashMap + Linked List** = most common composite for caches (LRU, LFU, All O(1))
2. **HashMap + Array** = O(1) random access + O(1) random selection
3. **Two Heaps** = dynamic median / partitioning
4. **HashMap + Sorted Structure** = time-versioned or snapshot access
5. **Frequency maps + grouped stacks** = frequency-ordered operations
6. **Always keep both structures in sync** during insert/delete
7. **Design approach**: identify required operations, pick DS for each, combine

> **Next up:** Space-Time Tradeoffs →
