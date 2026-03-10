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

**Textual Figure:**

```
Linear Probing — Insert, Get, Delete

  Input keys: "go"=1, "rust"=2, "java"=3, "python"=4, "c"=5, "ruby"=6
  Table capacity: 8

  Step 1: Insert "go"       → hash("go")=0       → slot 0 empty   → place at 0
  Step 2: Insert "rust"     → hash("rust")=4      → slot 4 empty   → place at 4
  Step 3: Insert "java"     → hash("java")=2      → slot 2 empty   → place at 2
  Step 4: Insert "python"   → hash("python")=4    → COLLISION at 4 ("rust")
           linear probe → slot 5 empty             → place at 5
  Step 5: Insert "c"        → hash("c")=3         → slot 3 empty   → place at 3
  Step 6: Insert "ruby"     → hash("ruby")=6      → slot 6 empty   → place at 6

  Hash Table after all inserts:
  ┌───────┬──────────┬───────┬────────┐
  │ Index │   Key    │ Value │ Status │
  ├───────┼──────────┼───────┼────────┤
  │   0   │  "go"    │   1   │ active │
  │   1   │          │       │ empty  │
  │   2   │  "java"  │   3   │ active │
  │   3   │  "c"     │   5   │ active │
  │   4   │  "rust"  │   2   │ active │
  │   5   │ "python" │   4   │ active │  ← linear probe from 4
  │   6   │  "ruby"  │   6   │ active │
  │   7   │          │       │ empty  │
  └───────┴──────────┴───────┴────────┘
  Load factor: 6/8 = 0.75

  Get("go"):     hash=0 → slot 0 → key matches       → return 1, true
  Get("python"): hash=4 → slot 4 "rust"≠"python"
                 → probe 5 → match                   → return 4, true
  Get("swift"):  hash → probe until empty slot        → return 0, false

  Delete("rust"): hash=4 → slot 4 → match → mark DELETED (tombstone)
  ┌───────┬──────────┬───────┬─────────┐
  │   4   │  "rust"  │   2   │ DELETED │  ← tombstone
  └───────┴──────────┴───────┴─────────┘
  Get("rust") after delete:
    hash=4 → slot 4 status=DELETED → skip
    → probe 5 → "python"≠"rust" → probe 6 → ... → not found
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

**Textual Figure:**

```
Quadratic Probing — Probe Sequence

  Input: "alpha"=1, "beta"=2, "gamma"=3, "delta"=4, "epsilon"=5
  Table capacity: 11 (prime)
  Probe formula: (hash + i²) % 11   for i = 0, 1, 2, 3, ...

  Suppose hash values (mod 11):
    hash("alpha")   = 3
    hash("beta")    = 7
    hash("gamma")   = 3   ← collision with "alpha"
    hash("delta")   = 9
    hash("epsilon") = 3   ← collision with "alpha" and "gamma"

  Insert "alpha":   probe(3+0²)= 3 → empty   → place at 3
  Insert "beta":    probe(7+0²)= 7 → empty   → place at 7
  Insert "gamma":   probe(3+0²)= 3 → occupied!
                    probe(3+1²)= 4 → empty   → place at 4
  Insert "delta":   probe(9+0²)= 9 → empty   → place at 9
  Insert "epsilon": probe(3+0²)= 3 → occupied!
                    probe(3+1²)= 4 → occupied!
                    probe(3+2²)= 7 → occupied!
                    probe(3+3²)= 1 → empty   → place at 1

  Quadratic probe sequence from hash=3:
  i:     0    1    2    3    4    5    ...
  i²:    0    1    4    9   16   25    ...
  slot:  3    4    7    1    8    6    ...
                                 ↑ wider spread than linear

  Final Hash Table:
  ┌───────┬───────────┬───────┐
  │ Index │    Key    │ Value │
  ├───────┼───────────┼───────┤
  │   0   │           │       │
  │   1   │ "epsilon" │   5   │  ← quadratic probe from 3
  │   2   │           │       │
  │   3   │ "alpha"   │   1   │  ← ideal slot
  │   4   │ "gamma"   │   3   │  ← quadratic probe from 3
  │   5   │           │       │
  │   6   │           │       │
  │   7   │ "beta"    │   2   │  ← ideal slot
  │   8   │           │       │
  │   9   │ "delta"   │   4   │  ← ideal slot
  │  10   │           │       │
  └───────┴───────────┴───────┘
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

**Textual Figure:**

```
Double Hashing — Two Independent Hash Functions

  Input: "sun"=0, "moon"=1, "star"=2, "planet"=3, "comet"=4, "asteroid"=5
  Table capacity: 13

  Hash functions:
    h1(key) = (Σ key[i]*31^i) mod 13   (primary position)
    h2(key) = 1 + (Σ key[i]*37^i) mod 12  (step size, always ≥ 1)

  Probe formula: (h1 + i × h2) mod 13

  Suppose:
    h1("sun")=5,      h2("sun")=3
    h1("moon")=8,     h2("moon")=5
    h1("star")=5,     h2("star")=7     ← collision with "sun" at 5
    h1("planet")=2,   h2("planet")=4
    h1("comet")=8,    h2("comet")=2    ← collision with "moon" at 8
    h1("asteroid")=5, h2("asteroid")=2 ← collision with "sun" at 5

  Insert "sun":      h1=5 → slot 5 empty → place at 5
  Insert "moon":     h1=8 → slot 8 empty → place at 8
  Insert "star":     h1=5 → slot 5 occupied!
                     (5 + 1×7) % 13 = 12 → empty → place at 12
  Insert "planet":   h1=2 → slot 2 empty → place at 2
  Insert "comet":    h1=8 → slot 8 occupied!
                     (8 + 1×2) % 13 = 10 → empty → place at 10
  Insert "asteroid": h1=5 → slot 5 occupied!
                     (5 + 1×2) % 13 = 7  → empty → place at 7

  Final Hash Table:
  ┌───────┬────────────┬───────┬─────────────────────────┐
  │ Index │    Key     │ Value │ Notes                   │
  ├───────┼────────────┼───────┼─────────────────────────┤
  │   0   │            │       │                         │
  │   1   │            │       │                         │
  │   2   │ "planet"   │   3   │ h1=2                    │
  │   3   │            │       │                         │
  │   4   │            │       │                         │
  │   5   │ "sun"      │   0   │ h1=5                    │
  │   6   │            │       │                         │
  │   7   │ "asteroid" │   5   │ h1=5, step h2=2         │
  │   8   │ "moon"     │   1   │ h1=8                    │
  │   9   │            │       │                         │
  │  10   │ "comet"    │   4   │ h1=8, step h2=2         │
  │  11   │            │       │                         │
  │  12   │ "star"     │   2   │ h1=5, step h2=7         │
  └───────┴────────────┴───────┴─────────────────────────┘

  Key insight: Different h2 values spread colliding keys apart
  "sun", "star", "asteroid" all hash to h1=5 but scatter via h2:
    sun      → 5
    star     → 5 + 7 = 12
    asteroid → 5 + 2 = 7
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

**Textual Figure:**

```
Comparing Probe Sequences (hash=3, capacity=11)

  Starting position: hash = 3

  ┌─────┬────────────────┬───────────────────┬────────────────────┐
  │  i  │ Linear (h+i)   │ Quadratic (h+i²)  │ Double (h+i×5)     │
  ├─────┼────────────────┼───────────────────┼────────────────────┤
  │  0  │       3        │        3          │         3          │
  │  1  │       4        │        4          │         8          │
  │  2  │       5        │        7          │         2          │
  │  3  │       6        │        1          │         7          │
  │  4  │       7        │        8          │         1          │
  │  5  │       8        │        6          │         6          │
  │  6  │       9        │        6*         │         0          │
  │  7  │      10        │        8*         │         5          │
  │  8  │       0        │        2          │        10          │
  │  9  │       1        │        9          │         4          │
  │ 10  │       2        │        7*         │         9          │
  └─────┴────────────────┴───────────────────┴────────────────────┘
  (* = repeated slot — quadratic may not visit all slots)

  Visualization on table (first 5 probes, numbered by visit order):

  Index:  0    1    2    3    4    5    6    7    8    9   10
        ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  Lin:  │    │    │    │ 1  │ 2  │ 3  │ 4  │ 5  │    │    │    │ clustered!
        ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
  Quad: │    │ 4  │    │ 1  │ 2  │    │    │ 3  │ 5  │    │    │ spread out
        ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
  Dbl:  │    │ 5  │ 3  │ 1  │    │    │    │ 4  │ 2  │    │    │ best spread
        └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

  Linear   → primary clustering (consecutive slots fill up)
  Quadratic→ wider jumps, but may miss some slots
  Double   → best distribution, step size independent of position
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

**Textual Figure:**

```
Load Factor vs Average Probes (Linear Probing)

  Expected probes formula: E[probes] ≈ ½(1 + 1/(1−α))  where α = load factor

  α (Load Factor) │ Avg Probes │ Performance
  ─────────────────┼────────────┼────────────────────────────────────
       0.10        │    ~1.06   │ ██ excellent
       0.20        │    ~1.13   │ ██ excellent
       0.30        │    ~1.21   │ ███ very good
       0.50        │    ~1.50   │ ████ good
       0.70        │    ~2.17   │ ██████ acceptable
       0.80        │    ~3.00   │ █████████ degrading
       0.90        │    ~5.50   │ █████████████████ poor
       0.95        │   ~10.50   │ █████████████████████████████████ terrible

  Probe count growth:
  Probes
   11 │                                                ×
   10 │                                             ×
    9 │
    8 │
    7 │
    6 │                                        ×
    5 │
    4 │
    3 │                                  ×
    2 │                            ×
    1 │  ×     ×     ×      ×
    0 ┼─────────────────────────────────────────────
      0.1  0.2  0.3  0.5  0.7  0.8  0.9  0.95
                    Load Factor (α)

  ⚠ Keep load factor below 0.7 for linear probing!
  Performance degrades dramatically above 70% occupancy.
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

**Textual Figure:**

```
Back-Shift Deletion (No Tombstones)

  Insert: "a"=0, "b"=1, "c"=2, "d"=3, "e"=4   capacity=11

  Suppose hash values (mod 11):
    hash("a")=8, hash("b")=9, hash("c")=8, hash("d")=8, hash("e")=4

  After inserts (linear probing for collisions):
    "a" → slot 8 (ideal)     "b" → slot 9 (ideal)
    "c" → hash 8 → 8,9 occupied → slot 10
    "d" → hash 8 → 8,9,10 occupied → slot 0
    "e" → slot 4 (ideal)

  Table before delete:
  ┌───────┬─────┬───────┬────────────┐
  │ Index │ Key │ Value │ Ideal Slot │
  ├───────┼─────┼───────┼────────────┤
  │   0   │ "d" │   3   │     8      │  ← displaced from 8
  │   4   │ "e" │   4   │     4      │
  │   8   │ "a" │   0   │     8      │
  │   9   │ "b" │   1   │     9      │
  │  10   │ "c" │   2   │     8      │  ← displaced from 8
  └───────┴─────┴───────┴────────────┘

  Delete("c") — found at slot 10:
  Step 1: Remove "c" from slot 10 → mark empty
  Step 2: Check slot 0 → "d" has ideal=8
          Since ideal(8) ≤ 10 < 0(+11) → "d" should move back
          Move "d" from slot 0 → slot 10
  ┌───────┬───────────────────────────┐
  │  10   │ empty → "d" moves here  │
  │   0   │ "d"   → now empty       │
  └───────┴───────────────────────────┘

  Table after back-shift:
  ┌───────┬─────┬───────┬───────────────────────────┐
  │ Index │ Key │ Value │ Notes                     │
  ├───────┼─────┼───────┼───────────────────────────┤
  │   0   │     │       │ empty (no tombstone!)     │
  │   4   │ "e" │   4   │                           │
  │   8   │ "a" │   0   │                           │
  │   9   │ "b" │   1   │                           │
  │  10   │ "d" │   3   │ shifted back from slot 0  │
  └───────┴─────┴───────┴───────────────────────────┘

  Result: Get("c") → not found ✓
          Get("d") → found at slot 10 ✓
  Advantage: No tombstones polluting the table!
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

**Textual Figure:**

```
Generic Open Addressing — OAMap[int, string]

  Table capacity: 16,  hash(k) = k

  Insert key=1:  hash(1) % 16 = 1  → slot 1 empty   → place
  Insert key=2:  hash(2) % 16 = 2  → slot 2 empty   → place
  Insert key=17: hash(17) % 16 = 1 → COLLISION with key=1!
                 probe slot 2 → occupied (key=2)
                 probe slot 3 → empty → place at 3

  ┌───────┬─────┬─────────────┬───────┬─────────────────────────┐
  │ Index │ Key │    Value    │ Used  │ Notes                   │
  ├───────┼─────┼─────────────┼───────┼─────────────────────────┤
  │   0   │     │             │ false │                         │
  │   1   │  1  │ "one"       │ true  │ hash(1)=1               │
  │   2   │  2  │ "two"       │ true  │ hash(2)=2               │
  │   3   │ 17  │ "seventeen" │ true  │ hash(17)=1, probed to 3 │
  │   4   │     │             │ false │                         │
  │  ... │     │             │ false │                         │
  └───────┴─────┴─────────────┴───────┴─────────────────────────┘

  Lookups:
  Get(1)  → hash=1 → slot 1 → match     → "one", true
  Get(2)  → hash=2 → slot 2 → match     → "two", true
  Get(17) → hash=1 → slot 1 (1≠17)
           → slot 2 (2≠17) → slot 3 (17=17) → "seventeen", true
  Get(3)  → hash=3 → slot 3 (17≠3)
           → slot 4 (empty) → "", false

  Generic type system: OAMap[K comparable, V any]
  ┌──────────────┬──────────────────────────────┐
  │ Type Param   │ Constraint                   │
  ├──────────────┼──────────────────────────────┤
  │ K comparable │ supports == operator         │
  │ V any        │ no constraint on value type  │
  └──────────────┴──────────────────────────────┘
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

**Textual Figure:**

```
Primary Clustering in Linear Probing

  Keys: i×7 for i=0,1,2,...   Table size: 101
  hash(key) = key % 101

  With 30 keys (load ≈ 0.30):
  ┌─────────────────────────────────────────────────────┐
  │ · · · · · · · ■ · · · · · · ■ · · · · · · ■ · · · │  scattered
  │ · · · · · · · ■ · · · · · · ■ · · · · ·           │
  └─────────────────────────────────────────────────────┘
  Clusters: many small (size 1)    Max cluster: 1

  With 50 keys (load ≈ 0.50):
  ┌─────────────────────────────────────────────────────┐
  │ · · · · · ■ ■ ■ · · · · ■ ■ ■ · · · · · ■ ■ ■ · · │  small clusters
  │ · · · · · ■ ■ ■ · · · · ■ ■ ■ · · · · ·           │
  └─────────────────────────────────────────────────────┘
  Clusters forming, max cluster: ~3-5

  With 70 keys (load ≈ 0.69):
  ┌─────────────────────────────────────────────────────┐
  │ · ■ ■ ■ ■ ■ ■ ■ ■ · · ■ ■ ■ ■ ■ ■ ■ · ■ ■ ■ ■ ■ ■ │  merging
  │ ■ ■ ■ · · ■ ■ ■ ■ ■ ■ ■ ■ · · ■ ■ ■ ■ ■             │
  └─────────────────────────────────────────────────────┘
  Clusters merging, max cluster: ~10-15

  With 90 keys (load ≈ 0.89):
  ┌─────────────────────────────────────────────────────┐
  │ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ · ■ ■ ■ ■ ■ ■ ■ ■ │  massive
  │ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■ · ■ ■             │
  └─────────────────────────────────────────────────────┘
  Clusters dominate, max cluster: ~30+

  Summary:
  ┌─────────┬─────────┬──────────────┬──────────────────┐
  │  Keys   │  Load   │  # Clusters  │  Max Cluster     │
  ├─────────┼─────────┼──────────────┼──────────────────┤
  │   30    │  0.30   │    ~30       │     ~1           │
  │   50    │  0.50   │    ~20       │     ~5           │
  │   70    │  0.69   │    ~10       │     ~15          │
  │   90    │  0.89   │     ~3       │     ~35          │
  └─────────┴─────────┴──────────────┴──────────────────┘
  Clusters grow and merge as load factor increases!
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

**Textual Figure:**

```
Robin Hood Hashing — PSL-Based Swapping

  Concept: "Steal from the rich, give to the poor"
  If inserting element has traveled farther (higher PSL)
  than the element at current slot, SWAP them.

  Insert into table of size 17:
    "cat"=0, "dog"=1, "rat"=2, "bat"=3, "hat"=4

  Suppose hash values (mod 17):
    hash("cat")=5, hash("dog")=8, hash("rat")=5,
    hash("bat")=5, hash("hat")=8

  Insert "cat": slot 5, PSL=0 → empty → place
  Insert "dog": slot 8, PSL=0 → empty → place
  Insert "rat": slot 5 → "cat"(PSL=0), my PSL=0 → no swap
                slot 6 → empty, PSL=1 → place
  Insert "bat": slot 5 → "cat"(PSL=0), my PSL=0 → no swap
                slot 6 → "rat"(PSL=1), my PSL=1 → equal, no swap
                slot 7 → empty, PSL=2 → place
  Insert "hat": slot 8 → "dog"(PSL=0), my PSL=0 → no swap
                slot 9 → empty, PSL=1 → place

  Table state:
  ┌───────┬───────┬───────┬─────┬────────────────────────┐
  │ Index │  Key  │ Value │ PSL │ Notes                  │
  ├───────┼───────┼───────┼─────┼────────────────────────┤
  │   5   │ "cat" │   0   │  0  │ ideal slot             │
  │   6   │ "rat" │   2   │  1  │ 1 away from ideal (5)  │
  │   7   │ "bat" │   3   │  2  │ 2 away from ideal (5)  │
  │   8   │ "dog" │   1   │  0  │ ideal slot             │
  │   9   │ "hat" │   4   │  1  │ 1 away from ideal (8)  │
  └───────┴───────┴───────┴─────┴────────────────────────┘

  Robin Hood swap example — Insert "mat" with hash=5:
  slot 5: "cat" PSL=0, my PSL=0 → equal, continue
  slot 6: "rat" PSL=1, my PSL=1 → equal, continue
  slot 7: "bat" PSL=2, my PSL=2 → equal, continue
  slot 8: "dog" PSL=0, my PSL=3 → MY PSL > THEIRS → SWAP!
          ┌──────────────────────────────────────┐
          │ "mat"(PSL=3) swaps with "dog"(PSL=0) │
          │ "mat" placed at slot 8               │
          │ "dog" continues insertion at PSL=1   │
          └──────────────────────────────────────┘
  slot 9: "hat" PSL=1, "dog" PSL=1 → equal, continue
  slot 10: empty → place "dog" at PSL=2

  Result: Max PSL is bounded → more predictable performance
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

**Textual Figure:**

```
Open Addressing vs Chaining — Comparison

  n = 100,000 keys

  Open Addressing (Linear Probing):
  ┌─────────────────────────────────────────────┐
  │ Array (contiguous memory)                   │
  │ ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐     │
  │ │k1│k2│  │k3│k4│k5│  │k6│k7│  │k8│k9│     │
  │ └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘     │
  │ ← cache line →  ← cache line →            │
  │ ✓ Sequential memory access                 │
  │ ✓ CPU cache friendly (spatial locality)    │
  │ ✓ No pointer overhead                      │
  │ ✗ Load factor must stay < 0.7              │
  │ Table size: 2n = 200,000 slots             │
  └─────────────────────────────────────────────┘

  Chaining (Linked Lists):
  ┌─────────────────────────────────────────────┐
  │ Buckets     Linked Nodes (heap-allocated)  │
  │ ┌──┐       ┌──────┐   ┌──────┐            │
  │ │ ─┼──────→│ k1 ─┼──→│ k5 ─┼──→ nil      │
  │ ├──┤       └──────┘   └──────┘            │
  │ │ ─┼──────→ nil                            │
  │ ├──┤       ┌──────┐                        │
  │ │ ─┼──────→│ k2 ─┼──→ nil                 │
  │ ├──┤       └──────┘                        │
  │ │ ─┼──────→ nil                            │
  │ └──┘                                       │
  │ ✗ Pointer chasing (cache misses)           │
  │ ✗ Extra memory for pointers                │
  │ ✓ Load factor can exceed 1.0               │
  │ ✓ Simple deletion                          │
  │ Table size: n = 100,000 buckets            │
  └─────────────────────────────────────────────┘

  Performance Comparison (n=100,000):
  ┌──────────────────┬─────────────┬──────────────┐
  │  Metric          │ Open Addr.  │ Chaining     │
  ├──────────────────┼─────────────┼──────────────┤
  │ Memory layout    │ contiguous  │ scattered    │
  │ Cache behavior   │ excellent   │ poor         │
  │ Avg lookup time  │ faster      │ slower       │
  │ Memory overhead  │ empty slots │ pointers     │
  │ Deletion         │ complex     │ simple       │
  │ Load factor      │ < 0.7       │ any          │
  └──────────────────┴─────────────┴──────────────┘
  → Open addressing typically wins on speed due to cache locality
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
