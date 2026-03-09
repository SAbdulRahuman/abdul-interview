# Phase 4: Hash Tables — Collision Resolution

## What is Collision Resolution?

A **collision** occurs when two different keys hash to the same bucket. Collision resolution strategies determine how to handle this. The two main approaches are:

1. **Chaining** (separate chaining) — each bucket holds a linked list
2. **Open addressing** — find another empty slot in the table

---

## Example 1: Collision Demonstration

```go
package main

import "fmt"

func simpleHash(key int, size int) int {
    return key % size
}

func main() {
    tableSize := 7
    keys := []int{10, 17, 24, 31, 5, 12, 19}

    for _, k := range keys {
        fmt.Printf("hash(%2d) = %d\n", k, simpleHash(k, tableSize))
    }
    // 10, 17, 24, 31 all hash to 3 → collision!
    // This demonstrates why collision resolution is critical
}
```

---

## Example 2: Separate Chaining (Linked List)

```go
package main

import "fmt"

type Node struct {
    Key   string
    Value int
    Next  *Node
}

type ChainingMap struct {
    buckets []*Node
    size    int
}

func NewChainingMap(capacity int) *ChainingMap {
    return &ChainingMap{
        buckets: make([]*Node, capacity),
    }
}

func (m *ChainingMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c)) % len(m.buckets)
    }
    return h
}

func (m *ChainingMap) Put(key string, value int) {
    idx := m.hash(key)
    // Check if key exists
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            node.Value = value
            return
        }
    }
    // Prepend new node
    m.buckets[idx] = &Node{Key: key, Value: value, Next: m.buckets[idx]}
    m.size++
}

func (m *ChainingMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for node := m.buckets[idx]; node != nil; node = node.Next {
        if node.Key == key {
            return node.Value, true
        }
    }
    return 0, false
}

func (m *ChainingMap) Delete(key string) bool {
    idx := m.hash(key)
    if m.buckets[idx] == nil {
        return false
    }
    // Head deletion
    if m.buckets[idx].Key == key {
        m.buckets[idx] = m.buckets[idx].Next
        m.size--
        return true
    }
    // Search chain
    for prev := m.buckets[idx]; prev.Next != nil; prev = prev.Next {
        if prev.Next.Key == key {
            prev.Next = prev.Next.Next
            m.size--
            return true
        }
    }
    return false
}

func (m *ChainingMap) Display() {
    for i, node := range m.buckets {
        fmt.Printf("Bucket %d: ", i)
        for ; node != nil; node = node.Next {
            fmt.Printf("[%s:%d] → ", node.Key, node.Value)
        }
        fmt.Println("nil")
    }
}

func main() {
    m := NewChainingMap(7)
    m.Put("apple", 1)
    m.Put("banana", 2)
    m.Put("cherry", 3)
    m.Put("date", 4)
    m.Put("elderberry", 5)
    m.Put("fig", 6)

    m.Display()

    v, ok := m.Get("cherry")
    fmt.Printf("\nGet(cherry) = %d, found=%v\n", v, ok)

    m.Delete("banana")
    fmt.Println("\nAfter deleting banana:")
    m.Display()
}
```

---

## Example 3: Linear Probing (Open Addressing)

```go
package main

import "fmt"

type LinearMap struct {
    keys   []string
    values []int
    used   []bool
    cap    int
    size   int
}

func NewLinearMap(capacity int) *LinearMap {
    return &LinearMap{
        keys:   make([]string, capacity),
        values: make([]int, capacity),
        used:   make([]bool, capacity),
        cap:    capacity,
    }
}

func (m *LinearMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *LinearMap) Put(key string, value int) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if !m.used[probe] {
            m.keys[probe] = key
            m.values[probe] = value
            m.used[probe] = true
            m.size++
            return
        }
        if m.keys[probe] == key {
            m.values[probe] = value // update
            return
        }
    }
    fmt.Println("Table is full!")
}

func (m *LinearMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if !m.used[probe] {
            return 0, false
        }
        if m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    return 0, false
}

func (m *LinearMap) Display() {
    for i := 0; i < m.cap; i++ {
        if m.used[i] {
            fmt.Printf("  [%d] %s: %d\n", i, m.keys[i], m.values[i])
        } else {
            fmt.Printf("  [%d] empty\n", i)
        }
    }
}

func main() {
    m := NewLinearMap(11)
    words := []string{"cat", "dog", "rat", "bat", "hat", "mat", "sat"}
    for i, w := range words {
        m.Put(w, i+1)
    }
    m.Display()

    for _, w := range []string{"dog", "hat", "fox"} {
        v, ok := m.Get(w)
        fmt.Printf("Get(%q) = %d, found=%v\n", w, v, ok)
    }
}
```

---

## Example 4: Quadratic Probing

```go
package main

import "fmt"

type QuadraticMap struct {
    keys   []string
    values []int
    used   []bool
    cap    int
    size   int
}

func NewQuadraticMap(capacity int) *QuadraticMap {
    return &QuadraticMap{
        keys:   make([]string, capacity),
        values: make([]int, capacity),
        used:   make([]bool, capacity),
        cap:    capacity,
    }
}

func (m *QuadraticMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *QuadraticMap) Put(key string, value int) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i*i) % m.cap // quadratic: +1, +4, +9, +16...
        if !m.used[probe] {
            m.keys[probe] = key
            m.values[probe] = value
            m.used[probe] = true
            m.size++
            return
        }
        if m.keys[probe] == key {
            m.values[probe] = value
            return
        }
    }
}

func (m *QuadraticMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i*i) % m.cap
        if !m.used[probe] {
            return 0, false
        }
        if m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    return 0, false
}

func main() {
    m := NewQuadraticMap(11)
    // Insert keys that would cluster with linear probing
    for _, key := range []string{"abc", "bca", "cab", "bac", "acb"} {
        m.Put(key, 1)
    }

    for i := 0; i < m.cap; i++ {
        if m.used[i] {
            fmt.Printf("[%d] %s\n", i, m.keys[i])
        } else {
            fmt.Printf("[%d] empty\n", i)
        }
    }
}
```

---

## Example 5: Double Hashing

```go
package main

import "fmt"

type DoubleHashMap struct {
    keys   []string
    values []int
    used   []bool
    cap    int
    size   int
}

func NewDoubleHashMap(capacity int) *DoubleHashMap {
    return &DoubleHashMap{
        keys:   make([]string, capacity),
        values: make([]int, capacity),
        used:   make([]bool, capacity),
        cap:    capacity,
    }
}

func (m *DoubleHashMap) hash1(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *DoubleHashMap) hash2(key string) int {
    h := 0
    for _, c := range key {
        h = (h*37 + int(c))
    }
    if h < 0 {
        h = -h
    }
    // Must be non-zero and coprime to capacity
    return 1 + (h % (m.cap - 1))
}

func (m *DoubleHashMap) Put(key string, value int) {
    h1 := m.hash1(key)
    h2 := m.hash2(key)

    for i := 0; i < m.cap; i++ {
        probe := (h1 + i*h2) % m.cap
        if !m.used[probe] {
            m.keys[probe] = key
            m.values[probe] = value
            m.used[probe] = true
            m.size++
            return
        }
        if m.keys[probe] == key {
            m.values[probe] = value
            return
        }
    }
}

func (m *DoubleHashMap) Get(key string) (int, bool) {
    h1 := m.hash1(key)
    h2 := m.hash2(key)

    for i := 0; i < m.cap; i++ {
        probe := (h1 + i*h2) % m.cap
        if !m.used[probe] {
            return 0, false
        }
        if m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    return 0, false
}

func main() {
    m := NewDoubleHashMap(13)
    words := []string{"alpha", "beta", "gamma", "delta", "epsilon"}
    for i, w := range words {
        m.Put(w, i)
    }

    for i := 0; i < m.cap; i++ {
        if m.used[i] {
            fmt.Printf("[%2d] %s = %d\n", i, m.keys[i], m.values[i])
        }
    }
}
```

---

## Example 6: Robin Hood Hashing

```go
package main

import "fmt"

type RHEntry struct {
    Key      string
    Value    int
    Occupied bool
    PSL      int // Probe Sequence Length (distance from ideal position)
}

type RobinHoodMap struct {
    table []RHEntry
    cap   int
    size  int
}

func NewRobinHoodMap(capacity int) *RobinHoodMap {
    return &RobinHoodMap{
        table: make([]RHEntry, capacity),
        cap:   capacity,
    }
}

func (m *RobinHoodMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *RobinHoodMap) Put(key string, value int) {
    entry := RHEntry{Key: key, Value: value, Occupied: true, PSL: 0}
    idx := m.hash(key)

    for {
        probe := (idx + entry.PSL) % m.cap

        if !m.table[probe].Occupied {
            m.table[probe] = entry
            m.size++
            return
        }

        if m.table[probe].Key == key {
            m.table[probe].Value = value
            return
        }

        // Robin Hood: steal from the rich (lower PSL)
        if entry.PSL > m.table[probe].PSL {
            entry, m.table[probe] = m.table[probe], entry
            idx = m.hash(entry.Key)
            entry.PSL = 0
            // Recalculate PSL for displaced entry
            for i := 0; ; i++ {
                if (idx+i)%m.cap == (idx+entry.PSL)%m.cap {
                    break
                }
                entry.PSL++
            }
        }
        entry.PSL++
    }
}

func (m *RobinHoodMap) Display() {
    for i, e := range m.table {
        if e.Occupied {
            fmt.Printf("[%d] %s=%d (PSL:%d)\n", i, e.Key, e.Value, e.PSL)
        } else {
            fmt.Printf("[%d] empty\n", i)
        }
    }
}

func main() {
    m := NewRobinHoodMap(11)
    for _, w := range []string{"cat", "dog", "rat", "bat", "hat"} {
        m.Put(w, 1)
    }
    m.Display()
    // Robin Hood hashing reduces variance in probe lengths
}
```

---

## Example 7: Cuckoo Hashing

```go
package main

import "fmt"

type CuckooMap struct {
    table1 []string
    table2 []string
    vals1  []int
    vals2  []int
    used1  []bool
    used2  []bool
    cap    int
}

func NewCuckooMap(capacity int) *CuckooMap {
    return &CuckooMap{
        table1: make([]string, capacity),
        table2: make([]string, capacity),
        vals1:  make([]int, capacity),
        vals2:  make([]int, capacity),
        used1:  make([]bool, capacity),
        used2:  make([]bool, capacity),
        cap:    capacity,
    }
}

func (m *CuckooMap) h1(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *CuckooMap) h2(key string) int {
    h := 0
    for _, c := range key {
        h = (h*37 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *CuckooMap) Put(key string, value int) bool {
    maxLoops := m.cap

    for i := 0; i < maxLoops; i++ {
        idx1 := m.h1(key)
        if !m.used1[idx1] {
            m.table1[idx1] = key
            m.vals1[idx1] = value
            m.used1[idx1] = true
            return true
        }
        // Evict from table1
        key, m.table1[idx1] = m.table1[idx1], key
        value, m.vals1[idx1] = m.vals1[idx1], value

        idx2 := m.h2(key)
        if !m.used2[idx2] {
            m.table2[idx2] = key
            m.vals2[idx2] = value
            m.used2[idx2] = true
            return true
        }
        // Evict from table2
        key, m.table2[idx2] = m.table2[idx2], key
        value, m.vals2[idx2] = m.vals2[idx2], value
    }

    fmt.Println("Cycle detected — need rehashing!")
    return false
}

func (m *CuckooMap) Get(key string) (int, bool) {
    idx1 := m.h1(key)
    if m.used1[idx1] && m.table1[idx1] == key {
        return m.vals1[idx1], true
    }
    idx2 := m.h2(key)
    if m.used2[idx2] && m.table2[idx2] == key {
        return m.vals2[idx2], true
    }
    return 0, false
}

func main() {
    m := NewCuckooMap(7)
    words := []string{"go", "rust", "java", "python"}
    for i, w := range words {
        m.Put(w, i)
    }

    for _, w := range []string{"go", "rust", "java", "python", "c++"} {
        v, ok := m.Get(w)
        fmt.Printf("Get(%q) = %d, found=%v\n", w, v, ok)
    }
    // Cuckoo: O(1) worst-case lookup!
}
```

---

## Example 8: Deletion with Tombstones (Open Addressing)

```go
package main

import "fmt"

const (
    stateEmpty     = 0
    stateOccupied  = 1
    stateTombstone = 2
)

type TombstoneMap struct {
    keys   []string
    values []int
    state  []int
    cap    int
    size   int
}

func NewTombstoneMap(capacity int) *TombstoneMap {
    return &TombstoneMap{
        keys:   make([]string, capacity),
        values: make([]int, capacity),
        state:  make([]int, capacity),
        cap:    capacity,
    }
}

func (m *TombstoneMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *TombstoneMap) Put(key string, value int) {
    idx := m.hash(key)
    tombstoneIdx := -1

    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        switch m.state[probe] {
        case stateEmpty:
            if tombstoneIdx != -1 {
                probe = tombstoneIdx // reuse tombstone slot
            }
            m.keys[probe] = key
            m.values[probe] = value
            m.state[probe] = stateOccupied
            m.size++
            return
        case stateTombstone:
            if tombstoneIdx == -1 {
                tombstoneIdx = probe
            }
        case stateOccupied:
            if m.keys[probe] == key {
                m.values[probe] = value
                return
            }
        }
    }
}

func (m *TombstoneMap) Delete(key string) bool {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if m.state[probe] == stateEmpty {
            return false
        }
        if m.state[probe] == stateOccupied && m.keys[probe] == key {
            m.state[probe] = stateTombstone // mark as tombstone
            m.size--
            return true
        }
    }
    return false
}

func (m *TombstoneMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if m.state[probe] == stateEmpty {
            return 0, false
        }
        if m.state[probe] == stateOccupied && m.keys[probe] == key {
            return m.values[probe], true
        }
        // Skip tombstones
    }
    return 0, false
}

func (m *TombstoneMap) Display() {
    for i := 0; i < m.cap; i++ {
        switch m.state[i] {
        case stateEmpty:
            fmt.Printf("[%d] empty\n", i)
        case stateTombstone:
            fmt.Printf("[%d] TOMBSTONE\n", i)
        case stateOccupied:
            fmt.Printf("[%d] %s=%d\n", i, m.keys[i], m.values[i])
        }
    }
}

func main() {
    m := NewTombstoneMap(7)
    m.Put("a", 1)
    m.Put("b", 2)
    m.Put("c", 3)
    m.Put("d", 4)

    fmt.Println("Before deletion:")
    m.Display()

    m.Delete("b")
    fmt.Println("\nAfter deleting 'b':")
    m.Display()

    // Can still find 'c' and 'd' even though 'b' is tombstone
    v, ok := m.Get("c")
    fmt.Printf("\nGet('c') = %d, found=%v\n", v, ok)
}
```

---

## Example 9: Collision Statistics

```go
package main

import "fmt"

func collisionStats(keys []string, tableSize int) {
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

    buckets := make(map[int][]string)
    for _, k := range keys {
        idx := hash(k)
        buckets[idx] = append(buckets[idx], k)
    }

    collisions := 0
    maxChain := 0
    for _, chain := range buckets {
        if len(chain) > 1 {
            collisions += len(chain) - 1
        }
        if len(chain) > maxChain {
            maxChain = len(chain)
        }
    }

    fmt.Printf("Table size: %d\n", tableSize)
    fmt.Printf("Keys: %d\n", len(keys))
    fmt.Printf("Collisions: %d\n", collisions)
    fmt.Printf("Max chain length: %d\n", maxChain)
    fmt.Printf("Load factor: %.2f\n", float64(len(keys))/float64(tableSize))

    // Show chains with collisions
    for idx, chain := range buckets {
        if len(chain) > 1 {
            fmt.Printf("Bucket %d: %v\n", idx, chain)
        }
    }
}

func main() {
    keys := []string{
        "apple", "banana", "cherry", "date", "elderberry",
        "fig", "grape", "honeydew", "kiwi", "lemon",
        "mango", "nectarine", "orange", "papaya", "quince",
    }
    collisionStats(keys, 7)
    fmt.Println()
    collisionStats(keys, 16)
}
```

---

## Example 10: Comparison of Collision Resolution Strategies

```go
package main

import "fmt"

func simulateChaining(keys []int, tableSize int) int {
    buckets := make([][]int, tableSize)
    maxChain := 0
    for _, k := range keys {
        idx := k % tableSize
        buckets[idx] = append(buckets[idx], k)
        if len(buckets[idx]) > maxChain {
            maxChain = len(buckets[idx])
        }
    }
    return maxChain
}

func simulateLinearProbing(keys []int, tableSize int) int {
    table := make([]bool, tableSize)
    maxProbes := 0
    for _, k := range keys {
        idx := k % tableSize
        probes := 0
        for table[(idx+probes)%tableSize] {
            probes++
        }
        table[(idx+probes)%tableSize] = true
        if probes > maxProbes {
            maxProbes = probes
        }
    }
    return maxProbes
}

func simulateQuadraticProbing(keys []int, tableSize int) int {
    table := make([]bool, tableSize)
    maxProbes := 0
    for _, k := range keys {
        idx := k % tableSize
        probes := 0
        for table[(idx+probes*probes)%tableSize] {
            probes++
        }
        table[(idx+probes*probes)%tableSize] = true
        if probes > maxProbes {
            maxProbes = probes
        }
    }
    return maxProbes
}

func main() {
    // Sequential keys — worst case for linear probing
    keys := make([]int, 0, 70)
    for i := 0; i < 70; i++ {
        keys = append(keys, i*7) // multiples of 7
    }

    tableSize := 101

    fmt.Println("Sequential keys (multiples of 7):")
    fmt.Printf("Chaining — max chain:    %d\n", simulateChaining(keys, tableSize))
    fmt.Printf("Linear probing — max:    %d\n", simulateLinearProbing(keys, tableSize))
    fmt.Printf("Quadratic probing — max: %d\n", simulateQuadraticProbing(keys, tableSize))
}
```

---

## Collision Resolution Summary

| Strategy         | Lookup (avg) | Lookup (worst) | Clustering | Deletion   |
|-----------------|-------------|---------------|------------|------------|
| Chaining        | O(1 + α)    | O(n)          | None       | Easy       |
| Linear probing  | O(1/(1-α))  | O(n)          | Primary    | Tombstone  |
| Quadratic       | O(1/(1-α))  | O(n)          | Secondary  | Tombstone  |
| Double hashing  | O(1/(1-α))  | O(n)          | Minimal    | Tombstone  |
| Cuckoo hashing  | O(1)        | O(1)          | None       | Easy       |
| Robin Hood      | O(1)        | O(log n)      | Minimal    | Moderate   |

*α = load factor = n/table_size*

## Key Takeaways

1. **Chaining** — simplest, works well with high load factors
2. **Linear probing** — cache-friendly but suffers from primary clustering
3. **Quadratic probing** — reduces clustering but may not find empty slots
4. **Double hashing** — best distribution among probing methods
5. **Cuckoo hashing** — O(1) guaranteed lookup, complex insertion
6. **Tombstones** — needed for deletion in open addressing schemes

> **Next up:** Open Addressing →
