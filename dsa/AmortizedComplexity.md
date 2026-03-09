# Phase 1: Algorithm Complexity — Amortized Complexity

## What is Amortized Complexity?

Amortized analysis computes the **average cost per operation** over a sequence of operations, even when individual operations may be expensive.

Unlike average-case analysis (which uses probability), amortized analysis is **deterministic** — it guarantees the average cost regardless of input.

**Key idea:** Expensive operations happen rarely enough that their cost is "spread out" over many cheap operations.

---

## Three Methods of Amortized Analysis

1. **Aggregate method** — compute total cost, divide by number of operations
2. **Accounting method** — assign "credits" to cheap operations to pay for expensive ones
3. **Potential method** — define a potential function that tracks "stored work"

---

## Example 1: Dynamic Array (Slice) — Amortized O(1) Append

```go
package main

import "fmt"

// Go slices: when full, allocate new array of 2× size and copy
// Single append: usually O(1), occasionally O(n) when resizing
// Amortized: O(1) per append
func demonstrateGrowth(n int) {
    var s []int
    prevCap := 0

    for i := 0; i < n; i++ {
        s = append(s, i)
        if cap(s) != prevCap {
            fmt.Printf("i=%3d → len=%3d, cap=%3d (resized! copied %d elements)\n",
                i, len(s), cap(s), len(s)-1)
            prevCap = cap(s)
        }
    }
}

func main() {
    demonstrateGrowth(50)
    // Watch capacity double: 1 → 2 → 4 → 8 → 16 → 32 → 64
    // Copies: 0 + 1 + 2 + 4 + 8 + 16 + 32 = 63 total copies for 50 appends
    // Average: 63/50 ≈ 1.26 copies per append → O(1) amortized
}
```

**Aggregate analysis:**
- n appends total
- Copies happen at sizes: 1, 2, 4, 8, ..., n
- Total copies: 1 + 2 + 4 + ... + n ≈ 2n
- Amortized cost: 2n / n = O(1)

---

## Example 2: Accounting Method for Dynamic Array

```go
package main

import "fmt"

// Accounting method: charge 3 "coins" per append
// - 1 coin: for the actual insert
// - 2 coins: saved for future resize copy
//
// When resize happens at size k:
//   - Copy k elements (cost k)
//   - Paid by: k/2 elements each saved 2 coins = k coins available
//   - Sufficient to cover the copy!

type AmortizedArray struct {
    data    []int
    credits int
}

func (a *AmortizedArray) Append(val int) {
    a.credits += 3 // charge 3 per operation
    a.credits--     // spend 1 for the insert

    oldCap := cap(a.data)
    a.data = append(a.data, val)
    newCap := cap(a.data)

    if newCap > oldCap && oldCap > 0 {
        // Resize happened — spend credits for copy
        copied := len(a.data) - 1
        a.credits -= copied // not actually subtracting real cost, just accounting
        fmt.Printf("  Resize! copied %d, credits remaining: %d\n", copied, a.credits)
    }
}

func main() {
    a := &AmortizedArray{}
    for i := 0; i < 20; i++ {
        a.Append(i)
    }
    fmt.Printf("Credits never go negative: total credits = %d\n", a.credits)
}
```

---

## Example 3: Stack with Multipop — Amortized O(1)

```go
package main

import "fmt"

// Stack with multipop: pop k elements at once
// Single multipop is O(k) — seems expensive!
// But amortized over n operations, it's O(1) per operation

type Stack struct {
    data []int
    ops  int
}

func (s *Stack) Push(val int) {
    s.data = append(s.data, val)
    s.ops++
}

func (s *Stack) MultiPop(k int) []int {
    s.ops++
    if k > len(s.data) {
        k = len(s.data)
    }
    popped := make([]int, k)
    copy(popped, s.data[len(s.data)-k:])
    s.data = s.data[:len(s.data)-k]
    return popped
}

func main() {
    s := &Stack{}

    // Push 10 elements: 10 operations, each O(1)
    for i := 1; i <= 10; i++ {
        s.Push(i)
    }

    // MultiPop all 10: 1 operation, O(10)
    fmt.Println(s.MultiPop(10)) // [1 2 3 4 5 6 7 8 9 10]

    // Total: 11 operations, total work = 10 + 10 = 20
    // Amortized: 20 / 11 ≈ 1.8 = O(1)
    fmt.Printf("Total ops: %d\n", s.ops)
}

// Key insight: each element can only be popped once after being pushed once
// Total pops across ALL operations ≤ total pushes = n
// So any sequence of n operations does at most 2n work → O(1) amortized
```

---

## Example 4: Binary Counter — Amortized O(1)

```go
package main

import "fmt"

// Incrementing a binary counter — how many bits flip per increment?
// Worst case: O(k) where k = number of bits (all 1s → all 0s + carry)
// Amortized: O(1) per increment

func increment(bits []int) int {
    flips := 0
    i := 0
    for i < len(bits) && bits[i] == 1 {
        bits[i] = 0 // flip 1 → 0
        flips++
        i++
    }
    if i < len(bits) {
        bits[i] = 1 // flip 0 → 1
        flips++
    }
    return flips
}

func main() {
    bits := make([]int, 8) // 8-bit counter
    totalFlips := 0

    for i := 0; i < 16; i++ {
        flips := increment(bits)
        totalFlips += flips
        fmt.Printf("Count=%2d → bits=%v (flips this step: %d)\n", i+1, bits, flips)
    }

    fmt.Printf("\nTotal flips for 16 increments: %d\n", totalFlips)
    fmt.Printf("Average flips per increment: %.2f\n", float64(totalFlips)/16.0)
    // Total ≈ 2n, Average ≈ 2 → O(1) amortized
}
```

**Why amortized O(1)?**
- Bit 0 flips every increment: n times
- Bit 1 flips every 2 increments: n/2 times
- Bit 2 flips every 4 increments: n/4 times
- Total flips: n + n/2 + n/4 + ... ≤ 2n
- Per increment: 2n / n = 2 = O(1)

---

## Example 5: Hash Map Resizing — Amortized O(1)

```go
package main

import "fmt"

// Simplified hash map with manual resizing
type SimpleHashMap struct {
    buckets  [][]entry
    size     int
    capacity int
}

type entry struct {
    key   string
    value int
}

func NewSimpleHashMap(cap int) *SimpleHashMap {
    return &SimpleHashMap{
        buckets:  make([][]entry, cap),
        capacity: cap,
    }
}

func (m *SimpleHashMap) hash(key string) int {
    h := 0
    for _, ch := range key {
        h = h*31 + int(ch)
    }
    if h < 0 { h = -h }
    return h % m.capacity
}

func (m *SimpleHashMap) Put(key string, value int) {
    // Check load factor — resize if > 0.75
    if float64(m.size)/float64(m.capacity) > 0.75 {
        m.resize()
    }

    idx := m.hash(key)
    for i, e := range m.buckets[idx] {
        if e.key == key {
            m.buckets[idx][i].value = value
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], entry{key, value})
    m.size++
}

func (m *SimpleHashMap) resize() {
    newCap := m.capacity * 2
    newBuckets := make([][]entry, newCap)
    old := m.buckets
    m.buckets = newBuckets
    m.capacity = newCap

    // Rehash all entries — O(n) but happens rarely
    rehashed := 0
    for _, bucket := range old {
        for _, e := range bucket {
            idx := m.hash(e.key)
            m.buckets[idx] = append(m.buckets[idx], e)
            rehashed++
        }
    }
    fmt.Printf("  Resized! capacity: %d, rehashed: %d entries\n", newCap, rehashed)
}

func main() {
    m := NewSimpleHashMap(4)
    for i := 0; i < 20; i++ {
        key := fmt.Sprintf("key%d", i)
        m.Put(key, i)
    }
    fmt.Printf("Final: size=%d, capacity=%d\n", m.size, m.capacity)
}
```

---

## Example 6: Potential Method — Formal Analysis

```go
package main

import "fmt"

// Potential method: Φ(Dᵢ) = 2 × size - capacity
// Amortized cost = actual cost + ΔΦ

type PotentialArray struct {
    data []int
    cap  int
}

func (a *PotentialArray) potential() int {
    return 2*len(a.data) - a.cap
}

func (a *PotentialArray) Append(val int) {
    phiBefore := a.potential()
    actualCost := 1 // just the insert

    if len(a.data) == a.cap {
        // Need to resize — actual cost includes copying
        actualCost = len(a.data) + 1 // copy all + insert
        a.cap *= 2
        if a.cap == 0 {
            a.cap = 1
        }
    }

    a.data = append(a.data, val)
    phiAfter := a.potential()
    deltaPhi := phiAfter - phiBefore
    amortizedCost := actualCost + deltaPhi

    fmt.Printf("val=%2d | actual=%2d | ΔΦ=%3d | amortized=%d\n",
        val, actualCost, deltaPhi, amortizedCost)
}

func main() {
    a := &PotentialArray{cap: 0}
    for i := 1; i <= 16; i++ {
        a.Append(i)
    }
    // Amortized cost is always ≤ 3 → O(1)
}
```

---

## Example 7: Splay Tree Access — Amortized O(log n)

```go
package main

import "fmt"

// Splay trees: move accessed element to root via rotations
// Single access can be O(n) if tree is degenerate
// But amortized over m operations: O(log n) per operation

// Simplified demonstration of the concept
type SplayNode struct {
    key         int
    left, right *SplayNode
}

// Simulate access costs
func simulateSplayAccess(n int) {
    fmt.Println("Splay Tree Access Simulation:")
    totalCost := 0

    for i := 0; i < n; i++ {
        // First access to element at depth d costs O(d)
        // But it gets moved to root (depth 0)
        // Subsequent accesses to same element: O(1)
        depth := n - i // worst case: deepest element
        totalCost += depth
    }

    fmt.Printf("n=%d, total cost=%d, amortized=%.1f\n",
        n, totalCost, float64(totalCost)/float64(n))
    // Even though some accesses are expensive, amortized is O(log n)
}

func main() {
    simulateSplayAccess(10)
    simulateSplayAccess(100)
}
```

---

## Example 8: Queue with Two Stacks — Amortized O(1)

```go
package main

import "fmt"

// Implement a queue using two stacks
// Enqueue: push to inStack — O(1) always
// Dequeue: pop from outStack — O(1) if not empty
//          if outStack empty, move all from inStack → O(n) ONCE
// Amortized: each element is moved at most once → O(1)

type QueueTwoStacks struct {
    inStack  []int
    outStack []int
    moves    int
}

func (q *QueueTwoStacks) Enqueue(val int) {
    q.inStack = append(q.inStack, val) // O(1)
}

func (q *QueueTwoStacks) Dequeue() int {
    if len(q.outStack) == 0 {
        // Transfer all elements — O(n) but each element transfers once
        for len(q.inStack) > 0 {
            top := q.inStack[len(q.inStack)-1]
            q.inStack = q.inStack[:len(q.inStack)-1]
            q.outStack = append(q.outStack, top)
            q.moves++
        }
        fmt.Printf("  Transferred %d elements\n", q.moves)
    }
    top := q.outStack[len(q.outStack)-1]
    q.outStack = q.outStack[:len(q.outStack)-1]
    return top
}

func main() {
    q := &QueueTwoStacks{}

    // Enqueue 5 elements
    for i := 1; i <= 5; i++ {
        q.Enqueue(i)
    }

    // Dequeue all — first dequeue triggers transfer
    for i := 0; i < 5; i++ {
        fmt.Println(q.Dequeue())
    }
    // Total work: 5 enqueues + 5 transfers + 5 pops = 15
    // 10 operations → 15/10 = 1.5 = O(1) amortized
}
```

---

## Example 9: Fibonacci Heap — Amortized O(1) Insert

```go
package main

import "fmt"

// Fibonacci Heap — conceptual demonstration
// Insert: O(1) — just add to root list
// Delete Min: O(log n) amortized — consolidation
// Decrease Key: O(1) amortized — cascading cuts

// Simplified simulation showing insert costs
type FibHeapSimple struct {
    roots []int
    min   int
    size  int
}

func (h *FibHeapSimple) Insert(val int) {
    h.roots = append(h.roots, val) // O(1) — just add to list
    if h.size == 0 || val < h.min {
        h.min = val
    }
    h.size++
}

func (h *FibHeapSimple) Min() int {
    return h.min // O(1)
}

func main() {
    h := &FibHeapSimple{}

    values := []int{5, 3, 8, 1, 9, 2, 7}
    for _, v := range values {
        h.Insert(v) // each insert is O(1)
    }

    fmt.Printf("Min: %d, Size: %d\n", h.Min(), h.size) // Min: 1, Size: 7
    fmt.Printf("Roots: %v\n", h.roots)

    // Fibonacci Heap amortized costs:
    fmt.Println("\nFibonacci Heap Operation Costs:")
    fmt.Println("  Insert:       O(1) amortized")
    fmt.Println("  Find Min:     O(1)")
    fmt.Println("  Delete Min:   O(log n) amortized")
    fmt.Println("  Decrease Key: O(1) amortized")
    fmt.Println("  Merge:        O(1)")
}
```

---

## Example 10: Amortized vs Average vs Worst Case

```go
package main

import "fmt"

func main() {
    fmt.Println("=== Amortized vs Average vs Worst Case ===")
    fmt.Println()

    comparisons := []struct {
        Operation string
        Worst     string
        Average   string
        Amortized string
    }{
        {"Slice append", "O(n)", "O(1)", "O(1)"},
        {"Hash map insert", "O(n)", "O(1)", "O(1)"},
        {"Stack multipop(k)", "O(k)", "O(k)", "O(1)"},
        {"Binary counter increment", "O(bits)", "O(1)", "O(1)"},
        {"Splay tree access", "O(n)", "O(log n)", "O(log n)"},
        {"Queue (2 stacks) dequeue", "O(n)", "O(1)", "O(1)"},
    }

    fmt.Printf("%-30s %-10s %-10s %-10s\n",
        "Operation", "Worst", "Average", "Amortized")
    fmt.Println("--------------------------------------------------------------")
    for _, c := range comparisons {
        fmt.Printf("%-30s %-10s %-10s %-10s\n",
            c.Operation, c.Worst, c.Average, c.Amortized)
    }

    fmt.Println()
    fmt.Println("Key differences:")
    fmt.Println("• Worst case:  single operation's maximum cost")
    fmt.Println("• Average case: expected cost assuming random inputs")
    fmt.Println("• Amortized:   guaranteed average over ANY sequence of operations")
}
```

---

## Example 11: Union-Find — Amortized O(α(n))

```go
package main

import "fmt"

// Union-Find with path compression + union by rank
// Single operation: O(log n) worst case
// Amortized: O(α(n)) ≈ O(1) — inverse Ackermann, practically constant

type UnionFind struct {
    parent []int
    rank   []int
    ops    int
}

func NewUnionFind(n int) *UnionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    for i := range parent {
        parent[i] = i
    }
    return &UnionFind{parent: parent, rank: rank}
}

func (uf *UnionFind) Find(x int) int {
    uf.ops++
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x]) // path compression
    }
    return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) {
    rx, ry := uf.Find(x), uf.Find(y)
    if rx == ry { return }
    // Union by rank
    if uf.rank[rx] < uf.rank[ry] {
        uf.parent[rx] = ry
    } else if uf.rank[rx] > uf.rank[ry] {
        uf.parent[ry] = rx
    } else {
        uf.parent[ry] = rx
        uf.rank[rx]++
    }
}

func main() {
    n := 10
    uf := NewUnionFind(n)

    uf.Union(0, 1); uf.Union(2, 3); uf.Union(4, 5)
    uf.Union(6, 7); uf.Union(8, 9)
    uf.Union(0, 2); uf.Union(4, 6); uf.Union(0, 4)
    uf.Union(0, 8)

    // All elements now in same set
    for i := 0; i < n; i++ {
        fmt.Printf("Find(%d) = %d\n", i, uf.Find(i))
    }
    fmt.Printf("Total internal ops: %d (amortized ≈ O(1) per operation)\n", uf.ops)
}
```

---

## Key Takeaways

1. **Amortized ≠ average case**: amortized is a GUARANTEE, average depends on probability
2. **Expensive operations are rare**: their cost is "absorbed" by many cheap operations
3. **Dynamic arrays**: O(n) resize happens after O(n) cheap appends → O(1) amortized
4. **Three methods**: aggregate (total/n), accounting (credits), potential (Φ)
5. **Common amortized O(1)**: slice append, hash map insert, stack multipop, queue with two stacks
6. **Union-Find amortized**: O(α(n)) ≈ O(1) — the most practical amortized result

> **Next up:** Best / Average / Worst Case →
