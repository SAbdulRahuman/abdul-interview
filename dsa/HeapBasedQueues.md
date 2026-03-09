# Phase 7: Queue — Heap-Based Queues

## Overview

A **heap-based queue** uses a binary heap to maintain ordering. Beyond basic priority queues, this covers **indexed priority queues** (update priorities), **double-ended priority queues**, and **specialized heap variants** used in algorithms.

| Variant | Key Feature |
|---------|-------------|
| Indexed PQ | Update & delete by key in O(log n) |
| Double-ended PQ | ExtractMin + ExtractMax |
| D-ary heap | Tunable branching factor |
| Lazy deletion | Mark deleted, clean on extract |

---

## Example 1: Indexed Priority Queue

An indexed PQ lets you **change the priority** of an element by its key.

```go
package main

import "fmt"

type IndexedMinPQ struct {
    n    int   // number of elements
    keys []int // keys[i] = priority of i
    pq   []int // binary heap of indices
    qp   []int // qp[i] = position of i in pq (-1 if absent)
}

func NewIndexedMinPQ(cap int) *IndexedMinPQ {
    qp := make([]int, cap)
    for i := range qp {
        qp[i] = -1
    }
    return &IndexedMinPQ{
        keys: make([]int, cap),
        pq:   make([]int, 1, cap+1), // 1-indexed
        qp:   qp,
    }
}

func (pq *IndexedMinPQ) swim(k int) {
    for k > 1 && pq.keys[pq.pq[k]] < pq.keys[pq.pq[k/2]] {
        pq.swap(k, k/2)
        k /= 2
    }
}

func (pq *IndexedMinPQ) sink(k int) {
    for 2*k <= pq.n {
        j := 2 * k
        if j < pq.n && pq.keys[pq.pq[j+1]] < pq.keys[pq.pq[j]] {
            j++
        }
        if pq.keys[pq.pq[k]] <= pq.keys[pq.pq[j]] {
            break
        }
        pq.swap(k, j)
        k = j
    }
}

func (pq *IndexedMinPQ) swap(i, j int) {
    pq.pq[i], pq.pq[j] = pq.pq[j], pq.pq[i]
    pq.qp[pq.pq[i]] = i
    pq.qp[pq.pq[j]] = j
}

func (pq *IndexedMinPQ) Insert(i, key int) {
    pq.n++
    pq.qp[i] = pq.n
    pq.pq = append(pq.pq, i)
    if len(pq.pq) > pq.n+1 {
        pq.pq[pq.n] = i
    }
    pq.keys[i] = key
    pq.swim(pq.n)
}

func (pq *IndexedMinPQ) DelMin() int {
    min := pq.pq[1]
    pq.swap(1, pq.n)
    pq.n--
    pq.pq = pq.pq[:pq.n+1]
    pq.sink(1)
    pq.qp[min] = -1
    return min
}

func (pq *IndexedMinPQ) DecreaseKey(i, key int) {
    pq.keys[i] = key
    pq.swim(pq.qp[i])
}

func (pq *IndexedMinPQ) IsEmpty() bool { return pq.n == 0 }
func (pq *IndexedMinPQ) Contains(i int) bool { return pq.qp[i] != -1 }

func main() {
    ipq := NewIndexedMinPQ(10)
    ipq.Insert(0, 15)
    ipq.Insert(1, 10)
    ipq.Insert(2, 20)
    ipq.Insert(3, 5)

    fmt.Println("Min index:", ipq.DelMin()) // 3 (key=5)

    ipq.DecreaseKey(2, 1) // change index 2 from 20 to 1
    fmt.Println("Min index:", ipq.DelMin()) // 2 (key=1)
}
```

**Why?** Used in Dijkstra's and Prim's algorithms where you need to update distances efficiently.

---

## Example 2: D-ary Heap (3-ary)

```go
package main

import "fmt"

type DAryHeap struct {
    data []int
    d    int
}

func NewDAryHeap(d int) *DAryHeap {
    return &DAryHeap{d: d}
}

func (h *DAryHeap) parent(i int) int { return (i - 1) / h.d }
func (h *DAryHeap) child(i, k int) int { return h.d*i + k + 1 }

func (h *DAryHeap) Push(val int) {
    h.data = append(h.data, val)
    h.swimUp(len(h.data) - 1)
}

func (h *DAryHeap) Pop() int {
    n := len(h.data)
    min := h.data[0]
    h.data[0] = h.data[n-1]
    h.data = h.data[:n-1]
    if len(h.data) > 0 {
        h.sinkDown(0)
    }
    return min
}

func (h *DAryHeap) swimUp(i int) {
    for i > 0 && h.data[i] < h.data[h.parent(i)] {
        p := h.parent(i)
        h.data[i], h.data[p] = h.data[p], h.data[i]
        i = p
    }
}

func (h *DAryHeap) sinkDown(i int) {
    for {
        smallest := i
        for k := 0; k < h.d; k++ {
            c := h.child(i, k)
            if c < len(h.data) && h.data[c] < h.data[smallest] {
                smallest = c
            }
        }
        if smallest == i {
            break
        }
        h.data[i], h.data[smallest] = h.data[smallest], h.data[i]
        i = smallest
    }
}

func main() {
    h := NewDAryHeap(3) // 3-ary heap
    for _, v := range []int{10, 4, 15, 20, 0, 8} {
        h.Push(v)
    }

    fmt.Print("3-ary heap extract order: ")
    for len(h.data) > 0 {
        fmt.Printf("%d ", h.Pop())
    }
    fmt.Println() // 0 4 8 10 15 20
}
```

**Why?** D-ary heaps have shallower trees → faster `decrease-key` (O(log_d n)) at the cost of slower `sink` (O(d · log_d n)). Good when decrease-key dominates (Dijkstra on dense graphs).

---

## Example 3: Lazy Deletion Priority Queue

```go
package main

import (
    "container/heap"
    "fmt"
)

type Item struct {
    Val     int
    Deleted bool
}

type LazyMinHeap []*Item
func (h LazyMinHeap) Len() int            { return len(h) }
func (h LazyMinHeap) Less(i, j int) bool  { return h[i].Val < h[j].Val }
func (h LazyMinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *LazyMinHeap) Push(x interface{}) { *h = append(*h, x.(*Item)) }
func (h *LazyMinHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

type LazyPQ struct {
    h    *LazyMinHeap
    refs map[int]*Item // value → item ref for deletion
}

func NewLazyPQ() *LazyPQ {
    h := &LazyMinHeap{}
    heap.Init(h)
    return &LazyPQ{h: h, refs: map[int]*Item{}}
}

func (pq *LazyPQ) Push(val int) {
    item := &Item{Val: val}
    pq.refs[val] = item
    heap.Push(pq.h, item)
}

func (pq *LazyPQ) Delete(val int) {
    if item, ok := pq.refs[val]; ok {
        item.Deleted = true
        delete(pq.refs, val)
    }
}

func (pq *LazyPQ) Pop() (int, bool) {
    for pq.h.Len() > 0 {
        item := heap.Pop(pq.h).(*Item)
        if !item.Deleted {
            delete(pq.refs, item.Val)
            return item.Val, true
        }
    }
    return 0, false
}

func main() {
    pq := NewLazyPQ()
    pq.Push(5)
    pq.Push(3)
    pq.Push(8)
    pq.Push(1)

    pq.Delete(3) // lazily delete 3
    pq.Delete(1) // lazily delete 1

    for {
        val, ok := pq.Pop()
        if !ok {
            break
        }
        fmt.Println("Popped:", val) // 5, 8 (1 and 3 skipped)
    }
}
```

**Why?** When removal from the middle is needed but `O(n)` search is too expensive. Mark as deleted and skip during extraction.

---

## Example 4: Dijkstra's Shortest Path (Heap-Based)

```go
package main

import (
    "container/heap"
    "fmt"
    "math"
)

type Edge struct {
    To, Weight int
}

type Node struct {
    ID   int
    Dist int
}

type MinHeap []*Node
func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].Dist < h[j].Dist }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*Node)) }
func (h *MinHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func dijkstra(graph [][]Edge, src int) []int {
    n := len(graph)
    dist := make([]int, n)
    for i := range dist {
        dist[i] = math.MaxInt64
    }
    dist[src] = 0

    h := &MinHeap{}
    heap.Init(h)
    heap.Push(h, &Node{ID: src, Dist: 0})

    for h.Len() > 0 {
        cur := heap.Pop(h).(*Node)
        if cur.Dist > dist[cur.ID] {
            continue // stale entry
        }
        for _, e := range graph[cur.ID] {
            newDist := dist[cur.ID] + e.Weight
            if newDist < dist[e.To] {
                dist[e.To] = newDist
                heap.Push(h, &Node{ID: e.To, Dist: newDist})
            }
        }
    }
    return dist
}

func main() {
    // Graph: 0→1(4), 0→2(1), 2→1(2), 1→3(1), 2→3(5)
    graph := [][]Edge{
        {{1, 4}, {2, 1}},
        {{3, 1}},
        {{1, 2}, {3, 5}},
        {},
    }

    dist := dijkstra(graph, 0)
    for i, d := range dist {
        fmt.Printf("0 → %d: %d\n", i, d)
    }
    // 0→0:0, 0→1:3, 0→2:1, 0→3:4
}
```

---

## Example 5: Prim's MST (Heap-Based)

```go
package main

import (
    "container/heap"
    "fmt"
)

type Edge struct {
    To, Weight int
}

type HeapEdge struct {
    Node, Weight int
}

type MinHeap []*HeapEdge
func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].Weight < h[j].Weight }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*HeapEdge)) }
func (h *MinHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func primMST(graph [][]Edge, n int) int {
    visited := make([]bool, n)
    h := &MinHeap{}
    heap.Init(h)
    heap.Push(h, &HeapEdge{0, 0})

    totalWeight := 0
    edgesUsed := 0

    for h.Len() > 0 && edgesUsed < n {
        e := heap.Pop(h).(*HeapEdge)
        if visited[e.Node] {
            continue
        }
        visited[e.Node] = true
        totalWeight += e.Weight
        edgesUsed++

        for _, adj := range graph[e.Node] {
            if !visited[adj.To] {
                heap.Push(h, &HeapEdge{adj.To, adj.Weight})
            }
        }
    }

    return totalWeight
}

func main() {
    // Graph: 0-1(4), 0-2(1), 1-2(2), 1-3(5), 2-3(8)
    graph := [][]Edge{
        {{1, 4}, {2, 1}},
        {{0, 4}, {2, 2}, {3, 5}},
        {{0, 1}, {1, 2}, {3, 8}},
        {{1, 5}, {2, 8}},
    }

    fmt.Println("MST weight:", primMST(graph, 4)) // 8
}
```

---

## Example 6: Huffman Encoding

```go
package main

import (
    "container/heap"
    "fmt"
    "strings"
)

type HuffNode struct {
    Char  byte
    Freq  int
    Left  *HuffNode
    Right *HuffNode
}

type HuffHeap []*HuffNode
func (h HuffHeap) Len() int            { return len(h) }
func (h HuffHeap) Less(i, j int) bool  { return h[i].Freq < h[j].Freq }
func (h HuffHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *HuffHeap) Push(x interface{}) { *h = append(*h, x.(*HuffNode)) }
func (h *HuffHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func buildHuffmanTree(text string) *HuffNode {
    freq := map[byte]int{}
    for i := 0; i < len(text); i++ {
        freq[text[i]]++
    }

    h := &HuffHeap{}
    heap.Init(h)
    for ch, f := range freq {
        heap.Push(h, &HuffNode{Char: ch, Freq: f})
    }

    for h.Len() > 1 {
        left := heap.Pop(h).(*HuffNode)
        right := heap.Pop(h).(*HuffNode)
        merged := &HuffNode{
            Freq:  left.Freq + right.Freq,
            Left:  left,
            Right: right,
        }
        heap.Push(h, merged)
    }

    return heap.Pop(h).(*HuffNode)
}

func buildCodes(node *HuffNode, prefix string, codes map[byte]string) {
    if node == nil {
        return
    }
    if node.Left == nil && node.Right == nil {
        codes[node.Char] = prefix
        if prefix == "" {
            codes[node.Char] = "0"
        }
        return
    }
    buildCodes(node.Left, prefix+"0", codes)
    buildCodes(node.Right, prefix+"1", codes)
}

func main() {
    text := "abracadabra"
    root := buildHuffmanTree(text)

    codes := map[byte]string{}
    buildCodes(root, "", codes)

    fmt.Println("Huffman codes:")
    for ch, code := range codes {
        fmt.Printf("  '%c' → %s\n", ch, code)
    }

    var encoded strings.Builder
    for i := 0; i < len(text); i++ {
        encoded.WriteString(codes[text[i]])
    }
    fmt.Printf("Encoded: %s (%d bits vs %d bits original)\n",
        encoded.String(), encoded.Len(), len(text)*8)
}
```

---

## Example 7: K Closest Points via Heap (Streaming)

```go
package main

import (
    "container/heap"
    "fmt"
    "math"
)

type Point struct {
    X, Y float64
    Dist float64
}

type MaxDistHeap []*Point
func (h MaxDistHeap) Len() int            { return len(h) }
func (h MaxDistHeap) Less(i, j int) bool  { return h[i].Dist > h[j].Dist }
func (h MaxDistHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxDistHeap) Push(x interface{}) { *h = append(*h, x.(*Point)) }
func (h *MaxDistHeap) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

func kClosestStream(k int) (add func(x, y float64), result func() []*Point) {
    h := &MaxDistHeap{}
    heap.Init(h)

    add = func(x, y float64) {
        dist := math.Sqrt(x*x + y*y)
        heap.Push(h, &Point{x, y, dist})
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    result = func() []*Point {
        pts := make([]*Point, h.Len())
        for i := range pts {
            pts[i] = (*h)[i]
        }
        return pts
    }

    return
}

func main() {
    add, result := kClosestStream(3)

    points := [][2]float64{{1, 2}, {3, 4}, {0, 1}, {5, 5}, {-1, 0}, {2, 1}}
    for _, p := range points {
        add(p[0], p[1])
        fmt.Printf("Added (%.0f,%.0f) → closest 3: ", p[0], p[1])
        for _, pt := range result() {
            fmt.Printf("(%.0f,%.0f) ", pt.X, pt.Y)
        }
        fmt.Println()
    }
}
```

---

## Example 8: Double-Ended Priority Queue

```go
package main

import (
    "container/heap"
    "fmt"
)

// Uses two heaps + lazy deletion for O(log n) min AND max extraction
type DEPQ struct {
    minH   *MinH
    maxH   *MaxH
    active map[int]int // value → count
    size   int
}

type MinH []int
func (h MinH) Len() int            { return len(h) }
func (h MinH) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinH) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH) Pop() interface{}   { old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v }

type MaxH []int
func (h MaxH) Len() int            { return len(h) }
func (h MaxH) Less(i, j int) bool  { return h[i] > h[j] }
func (h MaxH) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxH) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH) Pop() interface{}   { old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v }

func NewDEPQ() *DEPQ {
    mn := &MinH{}
    mx := &MaxH{}
    heap.Init(mn)
    heap.Init(mx)
    return &DEPQ{minH: mn, maxH: mx, active: map[int]int{}}
}

func (d *DEPQ) Insert(val int) {
    heap.Push(d.minH, val)
    heap.Push(d.maxH, val)
    d.active[val]++
    d.size++
}

func (d *DEPQ) clean(h interface{ Len() int }, peek func() int, pop func() int) {
    for h.Len() > 0 {
        top := peek()
        if d.active[top] > 0 {
            break
        }
        pop()
    }
}

func (d *DEPQ) ExtractMin() (int, bool) {
    d.clean(d.minH, func() int { return (*d.minH)[0] }, func() int { return heap.Pop(d.minH).(int) })
    if d.minH.Len() == 0 {
        return 0, false
    }
    val := heap.Pop(d.minH).(int)
    d.active[val]--
    if d.active[val] == 0 {
        delete(d.active, val)
    }
    d.size--
    return val, true
}

func (d *DEPQ) ExtractMax() (int, bool) {
    d.clean(d.maxH, func() int { return (*d.maxH)[0] }, func() int { return heap.Pop(d.maxH).(int) })
    if d.maxH.Len() == 0 {
        return 0, false
    }
    val := heap.Pop(d.maxH).(int)
    d.active[val]--
    if d.active[val] == 0 {
        delete(d.active, val)
    }
    d.size--
    return val, true
}

func main() {
    depq := NewDEPQ()
    for _, v := range []int{5, 3, 8, 1, 9, 2, 7} {
        depq.Insert(v)
    }

    min, _ := depq.ExtractMin()
    fmt.Println("Min:", min) // 1

    max, _ := depq.ExtractMax()
    fmt.Println("Max:", max) // 9

    min, _ = depq.ExtractMin()
    fmt.Println("Next min:", min) // 2

    max, _ = depq.ExtractMax()
    fmt.Println("Next max:", max) // 8
}
```

---

## Example 9: Heap Sort

```go
package main

import "fmt"

func heapSort(arr []int) {
    n := len(arr)

    // Build max-heap
    for i := n/2 - 1; i >= 0; i-- {
        siftDown(arr, n, i)
    }

    // Extract elements
    for i := n - 1; i > 0; i-- {
        arr[0], arr[i] = arr[i], arr[0]
        siftDown(arr, i, 0)
    }
}

func siftDown(arr []int, n, i int) {
    for {
        largest := i
        left := 2*i + 1
        right := 2*i + 2

        if left < n && arr[left] > arr[largest] {
            largest = left
        }
        if right < n && arr[right] > arr[largest] {
            largest = right
        }
        if largest == i {
            break
        }
        arr[i], arr[largest] = arr[largest], arr[i]
        i = largest
    }
}

func main() {
    arr := []int{12, 11, 13, 5, 6, 7}
    fmt.Println("Before:", arr)
    heapSort(arr)
    fmt.Println("After: ", arr) // [5 6 7 11 12 13]

    // Already sorted
    arr2 := []int{1, 2, 3, 4, 5}
    heapSort(arr2)
    fmt.Println("Sorted: ", arr2)

    // Reverse sorted
    arr3 := []int{5, 4, 3, 2, 1}
    heapSort(arr3)
    fmt.Println("Reverse:", arr3)
}
```

---

## Example 10: Running Median with Insertions and Deletions

```go
package main

import (
    "container/heap"
    "fmt"
)

type MaxH2 []int
func (h MaxH2) Len() int            { return len(h) }
func (h MaxH2) Less(i, j int) bool  { return h[i] > h[j] }
func (h MaxH2) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxH2) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxH2) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

type MinH2 []int
func (h MinH2) Len() int            { return len(h) }
func (h MinH2) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinH2) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinH2) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinH2) Pop() interface{} {
    old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v
}

type MedianTracker struct {
    lo *MaxH2 // max-heap
    hi *MinH2 // min-heap
}

func NewMedianTracker() *MedianTracker {
    lo := &MaxH2{}
    hi := &MinH2{}
    heap.Init(lo)
    heap.Init(hi)
    return &MedianTracker{lo, hi}
}

func (mt *MedianTracker) Add(num int) {
    if mt.lo.Len() == 0 || num <= (*mt.lo)[0] {
        heap.Push(mt.lo, num)
    } else {
        heap.Push(mt.hi, num)
    }
    mt.balance()
}

func (mt *MedianTracker) balance() {
    for mt.lo.Len() > mt.hi.Len()+1 {
        heap.Push(mt.hi, heap.Pop(mt.lo))
    }
    for mt.hi.Len() > mt.lo.Len() {
        heap.Push(mt.lo, heap.Pop(mt.hi))
    }
}

func (mt *MedianTracker) Median() float64 {
    if mt.lo.Len() > mt.hi.Len() {
        return float64((*mt.lo)[0])
    }
    return float64((*mt.lo)[0]+(*mt.hi)[0]) / 2.0
}

func main() {
    mt := NewMedianTracker()
    stream := []int{5, 15, 1, 3, 8, 7, 9, 10, 6, 2}

    for _, num := range stream {
        mt.Add(num)
        fmt.Printf("Add %2d → median=%.1f (lo=%d, hi=%d)\n",
            num, mt.Median(), mt.lo.Len(), mt.hi.Len())
    }
}
```

---

## Heap Variant Comparison

| Variant | Insert | ExtractMin | DecreaseKey | Use Case |
|---------|--------|-----------|-------------|----------|
| Binary heap | O(log n) | O(log n) | O(n) find + O(log n) | General purpose |
| Indexed PQ | O(log n) | O(log n) | O(log n) | Dijkstra, Prim |
| D-ary heap | O(log_d n) | O(d·log_d n) | O(log_d n) | Dense graphs |
| Fibonacci heap | O(1) amortized | O(log n) | O(1) amortized | Theory-optimal |

## Key Takeaways

1. **Indexed PQ** enables O(log n) decrease-key — essential for efficient Dijkstra/Prim
2. **D-ary heaps** trade sift-down speed for faster inserts — tune `d` to your workload
3. **Lazy deletion** avoids expensive removal — just mark and skip during extraction
4. **Two heaps** (max+min) solve median and double-ended priority queue problems
5. **Heap sort** is in-place with O(n log n) worst case — no extra memory needed
6. Go's `container/heap` is a min-heap by default; flip `Less` for max-heap
7. For Dijkstra with `container/heap`, use **stale entry check** instead of indexed PQ

> **Next up:** Monotonic Queue →
