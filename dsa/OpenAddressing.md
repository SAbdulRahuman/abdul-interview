# Phase 4: Hash Tables — Open Addressing

## What is Open Addressing?

In **open addressing**, all elements are stored directly in the hash table array. When a collision occurs, the algorithm **probes** for the next available slot using a probing sequence. No external data structures (like linked lists) are needed.

**Types**: Linear probing, Quadratic probing, Double hashing

---

## Example 1: Linear Probing — Full Implementation

```go
package main

import "fmt"

const (
    empty   = 0
    active  = 1
    deleted = 2
)

type Entry struct {
    key    string
    value  int
    status int
}

type OpenAddressMap struct {
    table []Entry
    cap   int
    size  int
}

func NewOpenAddressMap(capacity int) *OpenAddressMap {
    return &OpenAddressMap{
        table: make([]Entry, capacity),
        cap:   capacity,
    }
}

func (m *OpenAddressMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *OpenAddressMap) Put(key string, value int) {
    if float64(m.size)/float64(m.cap) > 0.7 {
        m.resize()
    }

    idx := m.hash(key)
    firstDeleted := -1

    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        switch m.table[probe].status {
        case empty:
            if firstDeleted != -1 {
                probe = firstDeleted
            }
            m.table[probe] = Entry{key, value, active}
            m.size++
            return
        case deleted:
            if firstDeleted == -1 {
                firstDeleted = probe
            }
        case active:
            if m.table[probe].key == key {
                m.table[probe].value = value
                return
            }
        }
    }
}

func (m *OpenAddressMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if m.table[probe].status == empty {
            return 0, false
        }
        if m.table[probe].status == active && m.table[probe].key == key {
            return m.table[probe].value, true
        }
    }
    return 0, false
}

func (m *OpenAddressMap) Delete(key string) bool {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if m.table[probe].status == empty {
            return false
        }
        if m.table[probe].status == active && m.table[probe].key == key {
            m.table[probe].status = deleted
            m.size--
            return true
        }
    }
    return false
}

func (m *OpenAddressMap) resize() {
    oldTable := m.table
    m.cap *= 2
    m.table = make([]Entry, m.cap)
    m.size = 0
    for _, e := range oldTable {
        if e.status == active {
            m.Put(e.key, e.value)
        }
    }
}

func main() {
    m := NewOpenAddressMap(8)
    for i, w := range []string{"go", "rust", "java", "python", "c", "ruby"} {
        m.Put(w, i+1)
    }

    for _, w := range []string{"go", "python", "swift"} {
        v, ok := m.Get(w)
        fmt.Printf("Get(%q) = %d, found=%v\n", w, v, ok)
    }

    m.Delete("rust")
    v, ok := m.Get("rust")
    fmt.Printf("After delete: Get('rust') = %d, found=%v\n", v, ok)
}
```

---

## Example 2: Quadratic Probing

```go
package main

import "fmt"

type QuadMap struct {
    keys   []string
    values []int
    status []int // 0=empty, 1=active, 2=deleted
    cap    int
    size   int
}

func NewQuadMap(cap int) *QuadMap {
    return &QuadMap{
        keys:   make([]string, cap),
        values: make([]int, cap),
        status: make([]int, cap),
        cap:    cap,
    }
}

func (m *QuadMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

// Probe sequence: h(k), h(k)+1, h(k)+4, h(k)+9, ...
func (m *QuadMap) Put(key string, value int) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i*i) % m.cap
        if m.status[probe] != 1 { // empty or deleted
            m.keys[probe] = key
            m.values[probe] = value
            m.status[probe] = 1
            m.size++
            return
        }
        if m.keys[probe] == key {
            m.values[probe] = value
            return
        }
    }
    fmt.Println("Table full — need resize")
}

func (m *QuadMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for i := 0; i < m.cap; i++ {
        probe := (idx + i*i) % m.cap
        if m.status[probe] == 0 {
            return 0, false
        }
        if m.status[probe] == 1 && m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    return 0, false
}

func main() {
    m := NewQuadMap(11) // prime size works best for quadratic probing
    data := map[string]int{"alpha": 1, "beta": 2, "gamma": 3, "delta": 4, "epsilon": 5}
    for k, v := range data {
        m.Put(k, v)
    }

    for _, k := range []string{"alpha", "gamma", "zeta"} {
        v, ok := m.Get(k)
        fmt.Printf("Get(%q) = %d, found=%v\n", k, v, ok)
    }
}
```

---

## Example 3: Double Hashing

```go
package main

import "fmt"

type DoubleHashMap struct {
    keys   []string
    values []int
    status []int
    cap    int
    size   int
}

func NewDoubleHashMap(cap int) *DoubleHashMap {
    return &DoubleHashMap{
        keys:   make([]string, cap),
        values: make([]int, cap),
        status: make([]int, cap),
        cap:    cap,
    }
}

func (m *DoubleHashMap) h1(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *DoubleHashMap) h2(key string) int {
    h := 7
    for _, c := range key {
        h = (h*37 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return 1 + h%(m.cap-1)
}

// Probe: h1(k), h1(k)+h2(k), h1(k)+2*h2(k), ...
func (m *DoubleHashMap) Put(key string, value int) {
    a := m.h1(key)
    b := m.h2(key)

    for i := 0; i < m.cap; i++ {
        probe := (a + i*b) % m.cap
        if m.status[probe] != 1 {
            m.keys[probe] = key
            m.values[probe] = value
            m.status[probe] = 1
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
    a := m.h1(key)
    b := m.h2(key)
    for i := 0; i < m.cap; i++ {
        probe := (a + i*b) % m.cap
        if m.status[probe] == 0 {
            return 0, false
        }
        if m.status[probe] == 1 && m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    return 0, false
}

func main() {
    m := NewDoubleHashMap(13)
    words := []string{"sun", "moon", "star", "planet", "comet", "asteroid"}
    for i, w := range words {
        m.Put(w, i)
    }

    for _, w := range words {
        v, _ := m.Get(w)
        fmt.Printf("%s → %d\n", w, v)
    }
}
```

---

## Example 4: Probe Sequence Visualization

```go
package main

import "fmt"

func linearProbeSequence(hash, cap int) []int {
    seq := make([]int, cap)
    for i := 0; i < cap; i++ {
        seq[i] = (hash + i) % cap
    }
    return seq
}

func quadraticProbeSequence(hash, cap int) []int {
    seq := make([]int, cap)
    for i := 0; i < cap; i++ {
        seq[i] = (hash + i*i) % cap
    }
    return seq
}

func doubleHashSequence(h1, h2, cap int) []int {
    seq := make([]int, cap)
    for i := 0; i < cap; i++ {
        seq[i] = (h1 + i*h2) % cap
    }
    return seq
}

func main() {
    hash := 3
    cap := 11

    fmt.Printf("hash=%d, capacity=%d\n\n", hash, cap)
    fmt.Println("Linear:    ", linearProbeSequence(hash, cap))
    fmt.Println("Quadratic: ", quadraticProbeSequence(hash, cap))
    fmt.Println("Double(h2=5):", doubleHashSequence(hash, 5, cap))
    // Linear: predictable clustering
    // Quadratic: wider spread but may miss slots
    // Double: best distribution
}
```

---

## Example 5: Load Factor Impact on Performance

```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func benchmarkLinearProbing(numKeys, tableSize int) float64 {
    table := make([]bool, tableSize)
    totalProbes := 0

    r := rand.New(rand.NewSource(42))
    for i := 0; i < numKeys; i++ {
        key := r.Intn(tableSize * 10)
        idx := key % tableSize
        probes := 0
        for table[(idx+probes)%tableSize] {
            probes++
        }
        table[(idx+probes)%tableSize] = true
        totalProbes += probes + 1
    }

    return float64(totalProbes) / float64(numKeys)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    tableSize := 10000

    fmt.Println("Load Factor | Avg Probes (Linear Probing)")
    fmt.Println("------------|---------------------------")
    for _, lf := range []float64{0.1, 0.2, 0.3, 0.5, 0.7, 0.8, 0.9, 0.95} {
        numKeys := int(lf * float64(tableSize))
        avgProbes := benchmarkLinearProbing(numKeys, tableSize)
        fmt.Printf("    %.2f     |     %.2f\n", lf, avgProbes)
    }
    // Performance degrades rapidly above 70% load factor
}
```

---

## Example 6: Back-Shift Deletion (No Tombstones)

```go
package main

import "fmt"

// Alternative to tombstones: shift elements back after deletion
type BackShiftMap struct {
    keys   []string
    values []int
    used   []bool
    cap    int
    size   int
}

func NewBackShiftMap(cap int) *BackShiftMap {
    return &BackShiftMap{
        keys:   make([]string, cap),
        values: make([]int, cap),
        used:   make([]bool, cap),
        cap:    cap,
    }
}

func (m *BackShiftMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *BackShiftMap) Put(key string, value int) {
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
            m.values[probe] = value
            return
        }
    }
}

func (m *BackShiftMap) Delete(key string) bool {
    idx := m.hash(key)
    pos := -1
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if !m.used[probe] {
            return false
        }
        if m.keys[probe] == key {
            pos = probe
            break
        }
    }
    if pos == -1 {
        return false
    }

    // Back-shift: move subsequent elements back
    m.used[pos] = false
    m.size--

    i := (pos + 1) % m.cap
    for m.used[i] {
        ideal := m.hash(m.keys[i])
        // Check if element at i should be moved back
        if (i >= pos && ideal <= pos) ||
            (i < pos && (ideal <= pos || ideal > i)) ||
            (ideal <= pos && pos < i) {
            m.keys[pos] = m.keys[i]
            m.values[pos] = m.values[i]
            m.used[pos] = true
            m.used[i] = false
            pos = i
        }
        i = (i + 1) % m.cap
    }
    return true
}

func (m *BackShiftMap) Get(key string) (int, bool) {
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

func main() {
    m := NewBackShiftMap(11)
    for i, w := range []string{"a", "b", "c", "d", "e"} {
        m.Put(w, i)
    }

    m.Delete("c")
    for _, w := range []string{"a", "b", "c", "d", "e"} {
        v, ok := m.Get(w)
        fmt.Printf("Get(%q) = %d, found=%v\n", w, v, ok)
    }
    // No tombstones needed — elements are shifted back
}
```

---

## Example 7: Open Addressing with Generic Types

```go
package main

import "fmt"

type OAMap[K comparable, V any] struct {
    keys   []K
    values []V
    used   []bool
    cap    int
    size   int
    hashFn func(K) int
}

func NewOAMap[K comparable, V any](cap int, hashFn func(K) int) *OAMap[K, V] {
    return &OAMap[K, V]{
        keys:   make([]K, cap),
        values: make([]V, cap),
        used:   make([]bool, cap),
        cap:    cap,
        hashFn: hashFn,
    }
}

func (m *OAMap[K, V]) Put(key K, value V) {
    idx := m.hashFn(key) % m.cap
    if idx < 0 {
        idx += m.cap
    }
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
            m.values[probe] = value
            return
        }
    }
}

func (m *OAMap[K, V]) Get(key K) (V, bool) {
    idx := m.hashFn(key) % m.cap
    if idx < 0 {
        idx += m.cap
    }
    for i := 0; i < m.cap; i++ {
        probe := (idx + i) % m.cap
        if !m.used[probe] {
            var zero V
            return zero, false
        }
        if m.keys[probe] == key {
            return m.values[probe], true
        }
    }
    var zero V
    return zero, false
}

func main() {
    intHash := func(k int) int { return k }
    m := NewOAMap[int, string](16, intHash)

    m.Put(1, "one")
    m.Put(2, "two")
    m.Put(17, "seventeen") // would collide with 1 in table of 16

    for _, k := range []int{1, 2, 17, 3} {
        v, ok := m.Get(k)
        fmt.Printf("Get(%d) = %q, found=%v\n", k, v, ok)
    }
}
```

---

## Example 8: Clustering Analysis

```go
package main

import "fmt"

func analyzeCluster(tableSize, numKeys int) {
    table := make([]bool, tableSize)

    for i := 0; i < numKeys; i++ {
        key := i * 7 // keys that would cause clustering
        idx := key % tableSize
        for table[idx] {
            idx = (idx + 1) % tableSize
        }
        table[idx] = true
    }

    // Find clusters
    maxCluster := 0
    currentCluster := 0
    clusters := 0
    for i := 0; i < tableSize; i++ {
        if table[i] {
            currentCluster++
        } else {
            if currentCluster > 0 {
                clusters++
                if currentCluster > maxCluster {
                    maxCluster = currentCluster
                }
                currentCluster = 0
            }
        }
    }
    if currentCluster > 0 {
        clusters++
        if currentCluster > maxCluster {
            maxCluster = currentCluster
        }
    }

    fmt.Printf("Table: %d, Keys: %d, Load: %.2f\n",
        tableSize, numKeys, float64(numKeys)/float64(tableSize))
    fmt.Printf("Clusters: %d, Max cluster size: %d\n\n", clusters, maxCluster)
}

func main() {
    fmt.Println("=== Primary Clustering in Linear Probing ===")
    analyzeCluster(101, 30)
    analyzeCluster(101, 50)
    analyzeCluster(101, 70)
    analyzeCluster(101, 90)
}
```

---

## Example 9: Robin Hood Hashing (Variance Reduction)

```go
package main

import "fmt"

type RHEntry struct {
    key    string
    value  int
    psl    int  // probe sequence length
    active bool
}

type RHMap struct {
    table []RHEntry
    cap   int
    size  int
}

func NewRHMap(cap int) *RHMap {
    return &RHMap{table: make([]RHEntry, cap), cap: cap}
}

func (m *RHMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *RHMap) Put(key string, value int) {
    entry := RHEntry{key: key, value: value, psl: 0, active: true}
    idx := m.hash(key)

    for {
        probe := (idx + entry.psl) % m.cap
        if !m.table[probe].active {
            m.table[probe] = entry
            m.size++
            return
        }
        if m.table[probe].key == entry.key {
            m.table[probe].value = entry.value
            return
        }
        // Robin Hood: swap if current entry has traveled farther
        if entry.psl > m.table[probe].psl {
            entry, m.table[probe] = m.table[probe], entry
            idx = m.hash(entry.key)
            entry.psl = (probe - idx + m.cap) % m.cap
        }
        entry.psl++
    }
}

func (m *RHMap) Stats() {
    totalPSL := 0
    maxPSL := 0
    for _, e := range m.table {
        if e.active {
            totalPSL += e.psl
            if e.psl > maxPSL {
                maxPSL = e.psl
            }
        }
    }
    fmt.Printf("Size: %d, Max PSL: %d, Avg PSL: %.2f\n",
        m.size, maxPSL, float64(totalPSL)/float64(m.size))
}

func main() {
    m := NewRHMap(17)
    words := []string{"cat", "dog", "rat", "bat", "hat", "mat", "sat", "fat", "pat", "vat"}
    for i, w := range words {
        m.Put(w, i)
    }
    m.Stats()
    // Robin Hood hashing minimizes the max PSL
}
```

---

## Example 10: Open Addressing vs Chaining Benchmark

```go
package main

import (
    "fmt"
    "time"
)

func benchmarkOpenAddressing(n int) time.Duration {
    cap := n * 2
    keys := make([]string, cap)
    values := make([]int, cap)
    used := make([]bool, cap)

    hash := func(s string) int {
        h := 0
        for _, c := range s {
            h = (h*31 + int(c))
        }
        if h < 0 {
            h = -h
        }
        return h % cap
    }

    start := time.Now()

    // Insert
    for i := 0; i < n; i++ {
        key := fmt.Sprintf("key_%d", i)
        idx := hash(key)
        for used[idx] {
            idx = (idx + 1) % cap
        }
        keys[idx] = key
        values[idx] = i
        used[idx] = true
    }

    // Lookup all
    for i := 0; i < n; i++ {
        key := fmt.Sprintf("key_%d", i)
        idx := hash(key)
        for keys[idx] != key {
            idx = (idx + 1) % cap
        }
        _ = values[idx]
    }

    return time.Since(start)
}

func benchmarkChaining(n int) time.Duration {
    cap := n
    type node struct {
        key   string
        value int
        next  *node
    }
    buckets := make([]*node, cap)

    hash := func(s string) int {
        h := 0
        for _, c := range s {
            h = (h*31 + int(c))
        }
        if h < 0 {
            h = -h
        }
        return h % cap
    }

    start := time.Now()

    // Insert
    for i := 0; i < n; i++ {
        key := fmt.Sprintf("key_%d", i)
        idx := hash(key)
        buckets[idx] = &node{key: key, value: i, next: buckets[idx]}
    }

    // Lookup all
    for i := 0; i < n; i++ {
        key := fmt.Sprintf("key_%d", i)
        idx := hash(key)
        for nd := buckets[idx]; nd != nil; nd = nd.next {
            if nd.key == key {
                _ = nd.value
                break
            }
        }
    }

    return time.Since(start)
}

func main() {
    n := 100000
    oa := benchmarkOpenAddressing(n)
    ch := benchmarkChaining(n)
    fmt.Printf("Open Addressing: %v\n", oa)
    fmt.Printf("Chaining:        %v\n", ch)
    // Open addressing is typically faster due to cache locality
}
```

---

## Open Addressing Summary

| Probing Method   | Probe Sequence          | Clustering  | Cache  |
|-----------------|------------------------|-------------|--------|
| Linear           | h+1, h+2, h+3, ...     | Primary    | Best   |
| Quadratic        | h+1², h+2², h+3², ...  | Secondary  | Good   |
| Double hash      | h+k, h+2k, h+3k, ...  | Minimal    | Fair   |
| Robin Hood       | Linear + swap if PSL >  | Balanced   | Best   |

## Key Takeaways

1. **No extra pointers** — all data stored in array (cache-friendly)
2. **Load factor must stay < 0.7-0.8** for good performance
3. **Deletion requires tombstones** or back-shifting
4. **Linear probing** — fastest due to cache, but clusters
5. **Double hashing** — best distribution among simple methods
6. **Robin Hood** — reduces variance, makes worst case predictable

> **Next up:** Chaining →
