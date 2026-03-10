# Phase 4: Hash Tables — Chaining

## What is Chaining (Separate Chaining)?

**Chaining** is a collision resolution technique where each bucket in the hash table contains a **linked list** (or other collection) of all key-value pairs that hash to that bucket. When multiple keys collide, they are chained together.

**Advantages**: Simple, handles high load factors, easy deletion  
**Disadvantages**: Extra memory for pointers, poor cache locality

---

## Example 1: Basic Chaining with Linked List

```go
package main

import "fmt"

type Node struct {
    Key   string
    Value int
    Next  *Node
}

type ChainMap struct {
    buckets []*Node
    size    int
    cap     int
}

func NewChainMap(cap int) *ChainMap {
    return &ChainMap{
        buckets: make([]*Node, cap),
        cap:     cap,
    }
}

func (m *ChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *ChainMap) Put(key string, value int) {
    idx := m.hash(key)
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            node.Value = value
            return
        }
    }
    m.buckets[idx] = &Node{key, value, m.buckets[idx]}
    m.size++
}

func (m *ChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            return node.Value, true
        }
    }
    return 0, false
}

func (m *ChainMap) Delete(key string) bool {
    idx := m.hash(key)
    if m.buckets[idx] == nil {
        return false
    }
    if m.buckets[idx].Key == key {
        m.buckets[idx] = m.buckets[idx].Next
        m.size--
        return true
    }
    for prev := m.buckets[idx]; prev.Next != nil; prev = prev.Next {
        if prev.Next.Key == key {
            prev.Next = prev.Next.Next
            m.size--
            return true
        }
    }
    return false
}

func main() {
    m := NewChainMap(8)
    m.Put("apple", 1)
    m.Put("banana", 2)
    m.Put("cherry", 3)

    v, ok := m.Get("banana")
    fmt.Printf("Get(banana) = %d, found=%v\n", v, ok) // 2, true

    m.Delete("banana")
    v, ok = m.Get("banana")
    fmt.Printf("After delete: Get(banana) = %d, found=%v\n", v, ok) // 0, false
}
```

**Textual Figure:**

```
Basic Chaining — Insert, Get, Delete

  Input: "apple"=1, "banana"=2, "cherry"=3   capacity=8

  hash("apple")  % 8 → bucket 5
  hash("banana") % 8 → bucket 2
  hash("cherry") % 8 → bucket 5   ← collision with apple!

  Insert "apple":  bucket[5] → prepend → [apple=1]
  Insert "banana": bucket[2] → prepend → [banana=2]
  Insert "cherry": bucket[5] → prepend → [cherry=3 → apple=1]

  Hash Table with chains:
  ┌─────────┬──────────────────────────────────────┐
  │ Bucket  │ Chain (linked list)                  │
  ├─────────┼──────────────────────────────────────┤
  │  [0]    │ nil                                  │
  │  [1]    │ nil                                  │
  │  [2]    │ ┌──────────┐                         │
  │         │ │banana = 2│──→ nil                  │
  │         │ └──────────┘                         │
  │  [3]    │ nil                                  │
  │  [4]    │ nil                                  │
  │  [5]    │ ┌──────────┐    ┌─────────┐          │
  │         │ │cherry = 3│──→ │apple = 1│──→ nil   │
  │         │ └──────────┘    └─────────┘          │
  │  [6]    │ nil                                  │
  │  [7]    │ nil                                  │
  └─────────┴──────────────────────────────────────┘

  Get("banana"): hash=2 → bucket[2] → traverse → found → 2, true

  Delete("banana"):
    bucket[2]: head.Key == "banana" → head = head.Next (nil)
  ┌─────────┬────────────────────┐
  │  [2]    │ nil  (removed!)    │
  └─────────┴────────────────────┘

  Get("banana") after delete: hash=2 → bucket[2] → nil → 0, false
```

---

## Example 2: Chaining with Slice (Array-Based Chains)

```go
package main

import "fmt"

type Entry struct {
    Key   string
    Value int
}

type SliceChainMap struct {
    buckets [][]Entry
    cap     int
    size    int
}

func NewSliceChainMap(cap int) *SliceChainMap {
    return &SliceChainMap{
        buckets: make([][]Entry, cap),
        cap:     cap,
    }
}

func (m *SliceChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *SliceChainMap) Put(key string, value int) {
    idx := m.hash(key)
    for i := range m.buckets[idx] {
        if m.buckets[idx][i].Key == key {
            m.buckets[idx][i].Value = value
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], Entry{key, value})
    m.size++
}

func (m *SliceChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for _, e := range m.buckets[idx] {
        if e.Key == key {
            return e.Value, true
        }
    }
    return 0, false
}

func (m *SliceChainMap) Delete(key string) bool {
    idx := m.hash(key)
    for i, e := range m.buckets[idx] {
        if e.Key == key {
            // Swap with last and shrink
            last := len(m.buckets[idx]) - 1
            m.buckets[idx][i] = m.buckets[idx][last]
            m.buckets[idx] = m.buckets[idx][:last]
            m.size--
            return true
        }
    }
    return false
}

func main() {
    m := NewSliceChainMap(4)
    for i, w := range []string{"go", "rust", "java", "python", "c", "ruby", "swift", "kotlin"} {
        m.Put(w, i)
    }

    fmt.Println("Bucket contents:")
    for i, bucket := range m.buckets {
        fmt.Printf("  [%d]: ", i)
        for _, e := range bucket {
            fmt.Printf("%s=%d ", e.Key, e.Value)
        }
        fmt.Println()
    }
}
```

**Textual Figure:**

```
Slice-Based Chaining

  Input: "go"=0,"rust"=1,"java"=2,"python"=3,"c"=4,"ruby"=5,"swift"=6,"kotlin"=7
  Table capacity: 4

  Hash assignments (mod 4):
    hash("go")=2,     hash("rust")=0,   hash("java")=2,   hash("python")=1
    hash("c")=3,      hash("ruby")=1,   hash("swift")=2,  hash("kotlin")=3

  Insertions (append to slice):
  ┌─────────┬─────────────────────────────────────────┐
  │ Bucket  │ Slice contents []Entry                │
  ├─────────┼─────────────────────────────────────────┤
  │  [0]    │ [ rust=1 ]                              │
  │  [1]    │ [ python=3, ruby=5 ]                    │
  │  [2]    │ [ go=0, java=2, swift=6 ]               │
  │  [3]    │ [ c=4, kotlin=7 ]                       │
  └─────────┴─────────────────────────────────────────┘

  Memory layout (contiguous within each bucket):
  Bucket[2]:  ┌────────┬────────┬─────────┐
              │ go=0   │ java=2 │ swift=6 │   ← contiguous in memory
              └────────┴────────┴─────────┘
               index 0   index 1  index 2

  Delete uses swap-with-last optimization:
  Delete("java") from bucket[2]:
    1. Find "java" at index 1
    2. Swap with last: bucket[2][1] = bucket[2][2] ("swift=6")
    3. Shrink slice:  bucket[2] = bucket[2][:2]

  Before: [ go=0, java=2, swift=6 ]
  After:  [ go=0, swift=6 ]          ← O(1) deletion!

  Advantage over linked lists: better cache locality within each bucket
```

---

## Example 3: Chaining with Automatic Resizing

```go
package main

import "fmt"

type ResizingChainMap struct {
    buckets [][]*KVPair
    cap     int
    size    int
}

type KVPair struct {
    Key   string
    Value int
}

func NewResizingChainMap(cap int) *ResizingChainMap {
    return &ResizingChainMap{
        buckets: make([][]*KVPair, cap),
        cap:     cap,
    }
}

func (m *ResizingChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *ResizingChainMap) loadFactor() float64 {
    return float64(m.size) / float64(m.cap)
}

func (m *ResizingChainMap) resize() {
    oldBuckets := m.buckets
    m.cap *= 2
    m.buckets = make([][]*KVPair, m.cap)
    m.size = 0

    for _, bucket := range oldBuckets {
        for _, kv := range bucket {
            m.Put(kv.Key, kv.Value)
        }
    }
    fmt.Printf("  [Resized to %d buckets]\n", m.cap)
}

func (m *ResizingChainMap) Put(key string, value int) {
    if m.loadFactor() > 0.75 {
        m.resize()
    }

    idx := m.hash(key)
    for _, kv := range m.buckets[idx] {
        if kv.Key == key {
            kv.Value = value
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], &KVPair{key, value})
    m.size++
}

func (m *ResizingChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for _, kv := range m.buckets[idx] {
        if kv.Key == key {
            return kv.Value, true
        }
    }
    return 0, false
}

func main() {
    m := NewResizingChainMap(4) // start small

    for i := 0; i < 20; i++ {
        key := fmt.Sprintf("key_%d", i)
        m.Put(key, i)
        fmt.Printf("Added %s, size=%d, cap=%d, lf=%.2f\n",
            key, m.size, m.cap, m.loadFactor())
    }
}
```

**Textual Figure:**

```
Automatic Resizing — Load Factor Trigger

  Start: capacity=4, threshold=0.75
  Inserting keys key_0 through key_19:

  Phase 1: capacity=4 (resize when size > 3)
  ┌────────┬──────────────────────┐
  │ Bucket │ Contents             │
  ├────────┼──────────────────────┤
  │  [0]   │ key_0                │
  │  [1]   │ key_1                │
  │  [2]   │ key_2                │
  │  [3]   │                      │
  └────────┴──────────────────────┘
  size=3, lf=0.75 → adding key_3 triggers resize!

  RESIZE: 4 → 8 buckets
  ┌──────────────────────────────────┐
  │ Rehash all existing keys into   │
  │ new table with capacity=8       │
  │ hash(key) % 8 ≠ hash(key) % 4  │
  │ Elements redistribute!          │
  └──────────────────────────────────┘

  Phase 2: capacity=8 (resize when size > 6)
  ...add more keys until lf > 0.75 → resize to 16

  Phase 3: capacity=16 (resize when size > 12)
  ...add more keys until lf > 0.75 → resize to 32

  Resize history for 20 keys:
  ┌────────┬──────┬───────┬─────────────────┐
  │  Step  │ Size │  Cap  │ Event           │
  ├────────┼──────┼───────┼─────────────────┤
  │  3→4  │   3  │ 4→8  │ resize (lf=0.75)│
  │  6→7  │   6  │ 8→16 │ resize (lf=0.75)│
  │ 12→13 │  12  │ 16→32│ resize (lf=0.75)│
  └────────┴──────┴───────┴─────────────────┘
  Final: 20 keys in 32 buckets, lf = 0.625

  Total rehash operations: 3 + 6 + 12 = 21 rehashes
  Amortized insert cost: O(1)
```

---

## Example 4: Chaining with Sorted Chains

```go
package main

import "fmt"

type SortedNode struct {
    Key   string
    Value int
    Next  *SortedNode
}

type SortedChainMap struct {
    buckets []*SortedNode
    cap     int
}

func NewSortedChainMap(cap int) *SortedChainMap {
    return &SortedChainMap{
        buckets: make([]*SortedNode, cap),
        cap:     cap,
    }
}

func (m *SortedChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *SortedChainMap) Put(key string, value int) {
    idx := m.hash(key)

    // Empty bucket
    if m.buckets[idx] == nil || m.buckets[idx].Key >= key {
        if m.buckets[idx] != nil && m.buckets[idx].Key == key {
            m.buckets[idx].Value = value
            return
        }
        m.buckets[idx] = &SortedNode{key, value, m.buckets[idx]}
        return
    }

    // Find insertion point
    prev := m.buckets[idx]
    for prev.Next != nil && prev.Next.Key < key {
        prev = prev.Next
    }

    if prev.Next != nil && prev.Next.Key == key {
        prev.Next.Value = value
    } else {
        prev.Next = &SortedNode{key, value, prev.Next}
    }
}

func (m *SortedChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            return node.Value, true
        }
        if node.Key > key {
            break // early termination in sorted chain
        }
    }
    return 0, false
}

func main() {
    m := NewSortedChainMap(4)
    words := []string{"delta", "alpha", "charlie", "bravo", "echo"}
    for i, w := range words {
        m.Put(w, i)
    }

    // Display sorted chains
    for i := 0; i < m.cap; i++ {
        fmt.Printf("Bucket %d: ", i)
        for node := m.buckets[i]; node != nil; node = node.Next {
            fmt.Printf("%s=%d → ", node.Key, node.Value)
        }
        fmt.Println("nil")
    }
}
```

**Textual Figure:**

```
Sorted Chain Insertion

  Input: "delta"=0, "alpha"=1, "charlie"=2, "bravo"=3, "echo"=4
  Table capacity: 4

  Suppose hash values (mod 4):
    hash("delta")=2, hash("alpha")=2, hash("charlie")=2
    hash("bravo")=1, hash("echo")=0

  Insert "delta":   bucket[2] empty → [delta=0]
  Insert "alpha":   bucket[2] → "alpha" < "delta" → insert before
                    → [alpha=1 → delta=0]
  Insert "charlie": bucket[2] → "alpha" < "charlie" < "delta"
                    → insert between
                    → [alpha=1 → charlie=2 → delta=0]
  Insert "bravo":   bucket[1] empty → [bravo=3]
  Insert "echo":    bucket[0] empty → [echo=4]

  Final sorted chains:
  ┌────────┬──────────────────────────────────────────────┐
  │ Bucket │ Sorted Chain                                 │
  ├────────┼──────────────────────────────────────────────┤
  │  [0]   │ ┌────────┐                                   │
  │        │ │echo = 4│──→ nil                            │
  │        │ └────────┘                                   │
  │  [1]   │ ┌─────────┐                                  │
  │        │ │bravo = 3│──→ nil                           │
  │        │ └─────────┘                                  │
  │  [2]   │ ┌─────────┐   ┌───────────┐   ┌─────────┐  │
  │        │ │alpha = 1│─→│charlie = 2│─→│delta = 0│─→nil│
  │        │ └─────────┘   └───────────┘   └─────────┘  │
  │  [3]   │ nil                                          │
  └────────┴──────────────────────────────────────────────┘

  Get("bravo"):  hash=1 → bucket[1] → "bravo" found → 3, true
  Get("fox"):    hash=? → traverse sorted chain
                 → if node.Key > "fox" → STOP early (not found)

  Advantage: Early termination on unsuccessful search
  ┌────────────────────────────────────────────────┐
  │ Unsorted: must traverse entire chain → O(n)    │
  │ Sorted:   stop when key > target → O(n/2) avg │
  └────────────────────────────────────────────────┘
```

---

## Example 5: Multi-Value Chaining (Multimap)

```go
package main

import "fmt"

type MultiMap struct {
    buckets []map[string][]int
    cap     int
}

func NewMultiMap(cap int) *MultiMap {
    m := &MultiMap{
        buckets: make([]map[string][]int, cap),
        cap:     cap,
    }
    for i := range m.buckets {
        m.buckets[i] = make(map[string][]int)
    }
    return m
}

func (m *MultiMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *MultiMap) Add(key string, value int) {
    idx := m.hash(key)
    m.buckets[idx][key] = append(m.buckets[idx][key], value)
}

func (m *MultiMap) GetAll(key string) []int {
    idx := m.hash(key)
    return m.buckets[idx][key]
}

func main() {
    mm := NewMultiMap(8)
    // Student → grades
    mm.Add("Alice", 95)
    mm.Add("Alice", 87)
    mm.Add("Alice", 92)
    mm.Add("Bob", 78)
    mm.Add("Bob", 85)

    fmt.Println("Alice's grades:", mm.GetAll("Alice"))
    fmt.Println("Bob's grades:", mm.GetAll("Bob"))
}
```

**Textual Figure:**

```
Multimap — Multiple Values Per Key

  Insert: Alice→95, Alice→87, Alice→92, Bob→78, Bob→85

  hash("Alice") % 8 = bucket 3 (suppose)
  hash("Bob")   % 8 = bucket 6 (suppose)

  Internal structure (map[string][]int per bucket):
  ┌────────┬──────────────────────────────────────┐
  │ Bucket │ map[string][]int                     │
  ├────────┼──────────────────────────────────────┤
  │  [0]   │ {}                                   │
  │  [1]   │ {}                                   │
  │  [2]   │ {}                                   │
  │  [3]   │ { "Alice": [95, 87, 92] }            │
  │  [4]   │ {}                                   │
  │  [5]   │ {}                                   │
  │  [6]   │ { "Bob": [78, 85] }                  │
  │  [7]   │ {}                                   │
  └────────┴──────────────────────────────────────┘

  Operations:
  Add("Alice", 95) → bucket[3]["Alice"] = append([], 95)       = [95]
  Add("Alice", 87) → bucket[3]["Alice"] = append([95], 87)     = [95, 87]
  Add("Alice", 92) → bucket[3]["Alice"] = append([95,87], 92)  = [95, 87, 92]

  GetAll("Alice") → hash=3 → bucket[3]["Alice"] → [95, 87, 92]
  GetAll("Bob")   → hash=6 → bucket[6]["Bob"]   → [78, 85]

  Use case: Student → Grades mapping
  ┌─────────┬────────────────┬─────────┐
  │ Student │ Grades         │ Average │
  ├─────────┼────────────────┼─────────┤
  │ Alice   │ [95, 87, 92]   │  91.3   │
  │ Bob     │ [78, 85]       │  81.5   │
  └─────────┴────────────────┴─────────┘
```

---

## Example 6: Chain Length Statistics

```go
package main

import (
    "fmt"
    "math"
)

func chainStats(keys []string, tableSize int) {
    hash := func(s string) int {
        h := 0
        for _, c := range s {
            h = (h*31 + int(c))
        }
        if h < 0 {
            h = -h
        }
        return h % tableSize
    }

    chains := make([]int, tableSize)
    for _, k := range keys {
        chains[hash(k)]++
    }

    maxLen := 0
    empty := 0
    totalLen := 0
    for _, l := range chains {
        if l == 0 {
            empty++
        }
        totalLen += l
        if l > maxLen {
            maxLen = l
        }
    }

    avgLen := float64(totalLen) / float64(tableSize-empty)
    variance := 0.0
    for _, l := range chains {
        if l > 0 {
            diff := float64(l) - avgLen
            variance += diff * diff
        }
    }
    variance /= float64(tableSize - empty)

    fmt.Printf("Table size: %d, Keys: %d\n", tableSize, len(keys))
    fmt.Printf("Empty buckets: %d (%.1f%%)\n", empty, 100*float64(empty)/float64(tableSize))
    fmt.Printf("Max chain: %d, Avg chain: %.2f, StdDev: %.2f\n",
        maxLen, avgLen, math.Sqrt(variance))
    fmt.Printf("Load factor: %.2f\n", float64(len(keys))/float64(tableSize))
}

func main() {
    keys := make([]string, 1000)
    for i := range keys {
        keys[i] = fmt.Sprintf("item_%d", i)
    }

    fmt.Println("=== Small Table (high load) ===")
    chainStats(keys, 100)
    fmt.Println("\n=== Medium Table ===")
    chainStats(keys, 500)
    fmt.Println("\n=== Large Table (low load) ===")
    chainStats(keys, 2000)
}
```

**Textual Figure:**

```
Chain Length Distribution Analysis

  1000 keys across different table sizes:

  Small Table (size=100, load=10.0):
  ┌─────────┬──────────────────────────────────────────┐
  │Bucket 0│ ████████████ (12 keys)                  │
  │Bucket 1│ ████████ (8 keys)                       │
  │Bucket 2│ ██████████████ (14 keys)                │
  │  ...   │ ...                                      │
  │Bucket99│ ███████ (7 keys)                         │
  └─────────┴──────────────────────────────────────────┘
  Max chain: ~18   Avg: 10.0   StdDev: ~3.2
  Empty buckets: 0 (0%)

  Medium Table (size=500, load=2.0):
  ┌─────────┬──────────────────────────────────────────┐
  │Bucket 0│ ███ (3 keys)                             │
  │Bucket 1│ █ (1 key)                                │
  │Bucket 2│ (empty)                                   │
  │Bucket 3│ ████ (4 keys)                            │
  │  ...   │ ...                                      │
  └─────────┴──────────────────────────────────────────┘
  Max chain: ~7    Avg: 2.0    StdDev: ~1.4
  Empty buckets: ~65 (13%)

  Large Table (size=2000, load=0.5):
  ┌─────────┬──────────────────────────────────────────┐
  │Bucket 0│ █ (1 key)                                │
  │Bucket 1│ (empty)                                   │
  │Bucket 2│ (empty)                                   │
  │Bucket 3│ █ (1 key)                                │
  │  ...   │ mostly empty or single-element chains    │
  └─────────┴──────────────────────────────────────────┘
  Max chain: ~4    Avg: 0.5    StdDev: ~0.7
  Empty buckets: ~1213 (60.7%)

  ┌────────────┬──────┬──────┬──────┬────────┬─────────┐
  │ Table Size │ Load │ Max  │ Avg  │ StdDev │ Empty % │
  ├────────────┼──────┼──────┼──────┼────────┼─────────┤
  │    100     │ 10.0 │  ~18 │ 10.0 │  ~3.2  │   0%    │
  │    500     │  2.0 │   ~7 │  2.0 │  ~1.4  │  13%    │
  │   2000     │  0.5 │   ~4 │  0.5 │  ~0.7  │  61%    │
  └────────────┴──────┴──────┴──────┴────────┴─────────┘
  Lower load factor → shorter chains → faster lookups
```

---

## Example 7: Chaining with BST (O(log n) Worst Case)

```go
package main

import "fmt"

type BSTNode struct {
    Key   string
    Value int
    Left  *BSTNode
    Right *BSTNode
}

func (n *BSTNode) Insert(key string, value int) *BSTNode {
    if n == nil {
        return &BSTNode{Key: key, Value: value}
    }
    if key < n.Key {
        n.Left = n.Left.Insert(key, value)
    } else if key > n.Key {
        n.Right = n.Right.Insert(key, value)
    } else {
        n.Value = value
    }
    return n
}

func (n *BSTNode) Search(key string) (int, bool) {
    if n == nil {
        return 0, false
    }
    if key < n.Key {
        return n.Left.Search(key)
    } else if key > n.Key {
        return n.Right.Search(key)
    }
    return n.Value, true
}

type BSTChainMap struct {
    buckets []*BSTNode
    cap     int
}

func NewBSTChainMap(cap int) *BSTChainMap {
    return &BSTChainMap{
        buckets: make([]*BSTNode, cap),
        cap:     cap,
    }
}

func (m *BSTChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *BSTChainMap) Put(key string, value int) {
    idx := m.hash(key)
    m.buckets[idx] = m.buckets[idx].Insert(key, value)
}

func (m *BSTChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    return m.buckets[idx].Search(key)
}

func main() {
    m := NewBSTChainMap(4)
    words := []string{"go", "rust", "java", "python", "c", "swift", "kotlin", "scala"}
    for i, w := range words {
        m.Put(w, i)
    }

    for _, w := range words {
        v, ok := m.Get(w)
        fmt.Printf("Get(%q) = %d, found=%v\n", w, v, ok)
    }
    // Worst case lookup: O(log n) instead of O(n) with linked list
    // Java's HashMap uses this approach when chains get long
}
```

**Textual Figure:**

```
BST-Based Chaining — O(log n) Worst Case

  Input: "go"=0,"rust"=1,"java"=2,"python"=3,
         "c"=4,"swift"=5,"kotlin"=6,"scala"=7
  Table capacity: 4

  Hash assignments (mod 4):
    bucket 0: "rust"=1, "scala"=7
    bucket 1: "python"=3
    bucket 2: "go"=0, "java"=2, "swift"=5, "kotlin"=6
    bucket 3: "c"=4

  Linked List (traditional) for bucket[2]:
    go → java → swift → kotlin → nil
    Search "kotlin": 4 hops → O(n)

  BST (improved) for bucket[2]:
              go
               \
              java
                \
               swift
               /
            kotlin

    Search "kotlin":
      go → "kotlin">"go" → right
      java → "kotlin">"java" → right
      swift → "kotlin"<"swift" → left
      kotlin → found!  (3 comparisons)

  BST for bucket[0]:
         rust
            \
           scala

  All buckets:
  ┌────────┬───────────────────────────────────────┐
  │ Bucket │ Structure                             │
  ├────────┼───────────────────────────────────────┤
  │  [0]   │ BST: rust → scala  (2 nodes)          │
  │  [1]   │ BST: python        (1 node)           │
  │  [2]   │ BST: go,java,kotlin,swift (4 nodes)   │
  │  [3]   │ BST: c             (1 node)           │
  └────────┴───────────────────────────────────────┘

  Comparison:
  ┌─────────────────┬─────────────┬───────────┐
  │ Chain Type      │ Worst Case  │ Used In   │
  ├─────────────────┼─────────────┼───────────┤
  │ Linked List     │ O(n)        │ simple    │
  │ BST             │ O(log n)    │ Java 8+   │
  └─────────────────┴─────────────┴───────────┘
  Java's HashMap converts chains to red-black trees when length > 8
```

---

## Example 8: Thread-Safe Chaining (Mutex Per Bucket)

```go
package main

import (
    "fmt"
    "sync"
)

type SafeEntry struct {
    Key   string
    Value int
}

type ConcurrentChainMap struct {
    buckets [][]SafeEntry
    locks   []sync.RWMutex
    cap     int
}

func NewConcurrentChainMap(cap int) *ConcurrentChainMap {
    return &ConcurrentChainMap{
        buckets: make([][]SafeEntry, cap),
        locks:   make([]sync.RWMutex, cap),
        cap:     cap,
    }
}

func (m *ConcurrentChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *ConcurrentChainMap) Put(key string, value int) {
    idx := m.hash(key)
    m.locks[idx].Lock()
    defer m.locks[idx].Unlock()

    for i := range m.buckets[idx] {
        if m.buckets[idx][i].Key == key {
            m.buckets[idx][i].Value = value
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], SafeEntry{key, value})
}

func (m *ConcurrentChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    m.locks[idx].RLock()
    defer m.locks[idx].RUnlock()

    for _, e := range m.buckets[idx] {
        if e.Key == key {
            return e.Value, true
        }
    }
    return 0, false
}

func main() {
    m := NewConcurrentChainMap(16)
    var wg sync.WaitGroup

    // Concurrent writes
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            m.Put(fmt.Sprintf("key_%d", i), i)
        }(i)
    }
    wg.Wait()

    // Verify
    found := 0
    for i := 0; i < 100; i++ {
        if _, ok := m.Get(fmt.Sprintf("key_%d", i)); ok {
            found++
        }
    }
    fmt.Printf("Inserted 100 keys concurrently, found %d\n", found)
}
```

**Textual Figure:**

```
Per-Bucket Locking for Concurrent Access

  ConcurrentChainMap with 16 buckets, each with its own RWMutex

  ┌─────────┬──────────┬────────────────────┐
  │ Bucket  │ RWMutex  │ Chain / Data       │
  ├─────────┼──────────┼────────────────────┤
  │  [0]    │ Lock_0   │ [key_5, key_32]    │
  │  [1]    │ Lock_1   │ [key_1, key_17]    │
  │  [2]    │ Lock_2   │ [key_66]           │
  │   ...   │  ...     │  ...               │
  │  [15]   │ Lock_15  │ [key_15, key_47]   │
  └─────────┴──────────┴────────────────────┘

  Concurrent operations on DIFFERENT buckets (parallel!):
  ┌────────────────────────────────────────────────┐
  │ Goroutine A: Put("key_5")  → Lock bucket 0  │─→ no conflict
  │ Goroutine B: Put("key_1")  → Lock bucket 1  │─→ no conflict
  │ Goroutine C: Get("key_66") → RLock bucket 2 │─→ no conflict
  └────────────────────────────────────────────────┘
  All three execute in parallel! ✓

  Concurrent operations on SAME bucket (serialized):
  ┌────────────────────────────────────────────────┐
  │ Goroutine A: Put("key_5")  → Lock bucket 0  │─→ acquired
  │ Goroutine B: Put("key_32") → Lock bucket 0  │─→ WAITS
  │ Goroutine C: Get("key_5")  → RLock bucket 0 │─→ WAITS
  └────────────────────────────────────────────────┘
  Only one writer at a time per bucket

  Multiple readers on same bucket (RWMutex allows):
  ┌────────────────────────────────────────────────┐
  │ Goroutine A: Get("key_5")  → RLock bucket 0 │─→ reads ✓
  │ Goroutine B: Get("key_32") → RLock bucket 0 │─→ reads ✓
  │ Both readers execute in parallel!            │
  └────────────────────────────────────────────────┘

  Throughput: 16 buckets = up to 16× parallel writes
  vs single global lock = 1× throughput
```

---

## Example 9: Chaining with Move-to-Front Heuristic

```go
package main

import "fmt"

type MTFNode struct {
    Key   string
    Value int
    Next  *MTFNode
}

type MTFChainMap struct {
    buckets []*MTFNode
    cap     int
}

func NewMTFChainMap(cap int) *MTFChainMap {
    return &MTFChainMap{
        buckets: make([]*MTFNode, cap),
        cap:     cap,
    }
}

func (m *MTFChainMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *MTFChainMap) Get(key string) (int, bool) {
    idx := m.hash(key)

    // Check head
    if m.buckets[idx] == nil {
        return 0, false
    }
    if m.buckets[idx].Key == key {
        return m.buckets[idx].Value, true
    }

    // Search and move to front
    prev := m.buckets[idx]
    for prev.Next != nil {
        if prev.Next.Key == key {
            // Move to front
            found := prev.Next
            prev.Next = found.Next
            found.Next = m.buckets[idx]
            m.buckets[idx] = found
            return found.Value, true
        }
        prev = prev.Next
    }
    return 0, false
}

func (m *MTFChainMap) Put(key string, value int) {
    idx := m.hash(key)
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            node.Value = value
            return
        }
    }
    m.buckets[idx] = &MTFNode{key, value, m.buckets[idx]}
}

func main() {
    m := NewMTFChainMap(4)
    words := []string{"a", "b", "c", "d", "e", "f"}
    for i, w := range words {
        m.Put(w, i)
    }

    // Accessing "e" moves it to front of its chain
    m.Get("e")
    m.Get("e") // now O(1) since it's at front

    fmt.Println("Move-to-front: frequently accessed keys become O(1)")

    // Show chain for demonstration
    for i := 0; i < m.cap; i++ {
        fmt.Printf("Bucket %d: ", i)
        for n := m.buckets[i]; n != nil; n = n.Next {
            fmt.Printf("%s ", n.Key)
        }
        fmt.Println()
    }
}
```

**Textual Figure:**

```
Move-to-Front Heuristic

  Input: "a"=0, "b"=1, "c"=2, "d"=3, "e"=4, "f"=5
  Table capacity: 4

  Suppose hash (mod 4):
    hash("a")=1, hash("b")=2, hash("c")=1
    hash("d")=3, hash("e")=1, hash("f")=2

  After inserts (prepend to head):
  ┌────────┬───────────────────────────────────┐
  │ Bucket │ Chain (most recent → oldest)      │
  ├────────┼───────────────────────────────────┤
  │  [0]   │ nil                               │
  │  [1]   │ e=4 → c=2 → a=0 → nil            │
  │  [2]   │ f=5 → b=1 → nil                  │
  │  [3]   │ d=3 → nil                         │
  └────────┴───────────────────────────────────┘

  Get("a") — search bucket[1]:
    e=4 (no) → c=2 (no) → a=0 (FOUND!)
    Move-to-front: detach "a" and prepend to head

  Before: e=4 → c=2 → a=0 → nil
                          ↑ found here (3 hops)
  After:  a=0 → e=4 → c=2 → nil
          ↑ now at front!

  Get("a") again — bucket[1]:
    a=0 (FOUND at head!) → O(1) access!

  Temporal Locality Benefit:
  ┌────────────┬──────────────┬─────────────────────┐
  │  Access    │ Without MTF  │ With MTF            │
  ├────────────┼──────────────┼─────────────────────┤
  │ Get("a")   │ 3 hops       │ 3 hops (first time) │
  │ Get("a")   │ 3 hops       │ 1 hop  (at front!)  │
  │ Get("a")   │ 3 hops       │ 1 hop  (stays front)│
  └────────────┴──────────────┴─────────────────────┘
  Frequently accessed keys "bubble up" to front → amortized O(1)
```

---

## Example 10: Chaining Performance with Different Load Factors

```go
package main

import (
    "fmt"
    "time"
)

func benchChaining(numKeys, tableSize int) time.Duration {
    type node struct {
        key  string
        next *node
    }

    buckets := make([]*node, tableSize)
    hash := func(s string) int {
        h := 0
        for _, c := range s {
            h = (h*31 + int(c))
        }
        if h < 0 {
            h = -h
        }
        return h % tableSize
    }

    keys := make([]string, numKeys)
    for i := range keys {
        keys[i] = fmt.Sprintf("k%d", i)
    }

    start := time.Now()

    // Insert all
    for _, k := range keys {
        idx := hash(k)
        buckets[idx] = &node{key: k, next: buckets[idx]}
    }

    // Lookup all
    for _, k := range keys {
        idx := hash(k)
        for n := buckets[idx]; n != nil; n = n.next {
            if n.key == k {
                break
            }
        }
    }

    return time.Since(start)
}

func main() {
    numKeys := 50000
    fmt.Printf("Keys: %d\n\n", numKeys)
    fmt.Printf("%-12s %-10s %s\n", "Load Factor", "Table Size", "Time")
    fmt.Println("---------------------------------------")

    for _, lf := range []float64{0.5, 1.0, 2.0, 5.0, 10.0} {
        tableSize := int(float64(numKeys) / lf)
        if tableSize < 1 {
            tableSize = 1
        }
        t := benchChaining(numKeys, tableSize)
        fmt.Printf("%-12.1f %-10d %v\n", lf, tableSize, t)
    }
    // Chaining gracefully handles load factors > 1
    // Performance degrades linearly, not catastrophically
}
```

**Textual Figure:**

```
Chaining Performance vs Load Factor

  50,000 keys, varying table sizes:

  ┌─────────────┬────────────┬──────────────────────────────┐
  │ Load Factor │ Table Size │ Avg Chain Length              │
  ├─────────────┼────────────┼──────────────────────────────┤
  │    0.5      │  100,000   │ █ 0.5  (mostly empty)        │
  │    1.0      │   50,000   │ ██ 1.0 (ideal balance)       │
  │    2.0      │   25,000   │ ████ 2.0                     │
  │    5.0      │   10,000   │ ██████████ 5.0               │
  │   10.0      │    5,000   │ ████████████████████ 10.0    │
  └─────────────┴────────────┴──────────────────────────────┘

  Lookup cost visualization:
  Load 0.5:  bucket → [k1] → done                  (1 comparison avg)
  Load 1.0:  bucket → [k1] → done                  (1 comparison avg)
  Load 2.0:  bucket → [k1 → k2] → done             (2 comparisons avg)
  Load 5.0:  bucket → [k1 → k2 → k3 → k4 → k5]    (5 comparisons avg)
  Load 10.0: bucket → [k1 → k2 → ... → k10]        (10 comparisons avg)

  Time growth (relative):
  Time
   5× │                                          ×
      │
   4× │
      │
   3× │                              ×
      │
   2× │                  ×
      │
   1× │     ×     ×
      │
   0  ┼──────────────────────────────────────
      0.5   1.0   2.0    5.0         10.0
                Load Factor

  Key insight: Chaining degrades LINEARLY with load factor
  (unlike open addressing which degrades EXPONENTIALLY)
  → Chaining gracefully handles load factors > 1
```

---

## Chaining Complexity

| Operation | Average    | Worst Case |
|-----------|-----------|------------|
| Insert    | O(1)      | O(1)*      |
| Search    | O(1 + α)  | O(n)       |
| Delete    | O(1 + α)  | O(n)       |

*α = load factor = n / table_size*

*Insert is O(1) for prepending to head of chain

## Key Takeaways

1. **Simple implementation** — linked list per bucket
2. **Load factor can exceed 1** — unlike open addressing
3. **Easy deletion** — no tombstones needed
4. **Poor cache locality** — pointer chasing hurts performance
5. **BST chains** (Java HashMap) — O(log n) worst case for long chains
6. **Move-to-front** — self-adjusting for frequently accessed keys
7. **Concurrent access** — per-bucket locking enables parallelism

> **Next up:** Load Factor →
