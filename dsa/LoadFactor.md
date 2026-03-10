# Phase 4: Hash Tables — Load Factor

## What is Load Factor?

The **load factor** (α) measures how full a hash table is:

$$\alpha = \frac{n}{m}$$

where `n` = number of elements, `m` = number of buckets.

- **Low α** → many empty buckets (waste memory, fast lookups)
- **High α** → many collisions (slow lookups, efficient memory)
- **Optimal**: 0.5–0.75 for open addressing, up to 1.0+ for chaining

---

## Example 1: Computing and Monitoring Load Factor

```go
package main

import "fmt"

type HashMap struct {
    buckets [][]string
    size    int
    cap     int
}

func NewHashMap(cap int) *HashMap {
    return &HashMap{
        buckets: make([][]string, cap),
        cap:     cap,
    }
}

func (m *HashMap) loadFactor() float64 {
    return float64(m.size) / float64(m.cap)
}

func (m *HashMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *HashMap) Put(key string) {
    idx := m.hash(key)
    for _, k := range m.buckets[idx] {
        if k == key {
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], key)
    m.size++
}

func main() {
    m := NewHashMap(8)
    words := []string{"go", "rust", "java", "python", "c", "ruby", "swift", "kotlin"}

    for _, w := range words {
        m.Put(w)
        fmt.Printf("Added %q → size=%d, cap=%d, load=%.3f\n",
            w, m.size, m.cap, m.loadFactor())
    }
}
```

**Textual Figure:**
```
  Hash Table (cap=8): Inserting words one by one
  ══════════════════════════════════════════════

  Bucket   State as words are added ("go","rust","java","python","c","ruby","swift","kotlin")
  ┌───────┬──────────────────────────────────────────────────┐
  │  [0]  │                              → "python"         │
  │  [1]  │              → "java"                            │
  │  [2]  │  → "go"                                         │
  │  [3]  │                                       → "ruby"  │
  │  [4]  │                                                  │
  │  [5]  │       → "rust"               → "c"              │
  │  [6]  │                                    → "swift"    │
  │  [7]  │                                         → "kotlin"│
  └───────┴──────────────────────────────────────────────────┘

  Load Factor Progression:
  ┌────────┬──────┬─────┬───────────────────────────────────┐
  │  Word  │ size │ cap │ α = size/cap                      │
  ├────────┼──────┼─────┼───────────────────────────────────┤
  │ "go"   │   1  │  8  │ 0.125  ▓░░░░░░░░░░░░░░░░░░░░░░░ │
  │ "rust" │   2  │  8  │ 0.250  ▓▓▓░░░░░░░░░░░░░░░░░░░░░ │
  │ "java" │   3  │  8  │ 0.375  ▓▓▓▓▓░░░░░░░░░░░░░░░░░░░ │
  │"python"│   4  │  8  │ 0.500  ▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░ │
  │ "c"    │   5  │  8  │ 0.625  ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░ │
  │ "ruby" │   6  │  8  │ 0.750  ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░ │  ← threshold!
  │"swift" │   7  │  8  │ 0.875  ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░ │
  │"kotlin"│   8  │  8  │ 1.000  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │  ← FULL!
  └────────┴──────┴─────┴───────────────────────────────────┘

  Formula:  α = n / m  =  8 / 8  =  1.000
```

---

## Example 2: Auto-Resize at Threshold

```go
package main

import "fmt"

const maxLoadFactor = 0.75

type Entry struct {
    Key   string
    Value int
}

type AutoMap struct {
    buckets [][]Entry
    size    int
    cap     int
}

func NewAutoMap(cap int) *AutoMap {
    return &AutoMap{
        buckets: make([][]Entry, cap),
        cap:     cap,
    }
}

func (m *AutoMap) loadFactor() float64 {
    return float64(m.size) / float64(m.cap)
}

func (m *AutoMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *AutoMap) resize() {
    oldBuckets := m.buckets
    m.cap *= 2
    m.buckets = make([][]Entry, m.cap)
    m.size = 0

    for _, bucket := range oldBuckets {
        for _, e := range bucket {
            m.Put(e.Key, e.Value)
        }
    }
}

func (m *AutoMap) Put(key string, value int) {
    if m.loadFactor() >= maxLoadFactor {
        fmt.Printf("  ⚡ Resizing: %d → %d (load=%.2f)\n", m.cap, m.cap*2, m.loadFactor())
        m.resize()
    }

    idx := m.hash(key)
    for i, e := range m.buckets[idx] {
        if e.Key == key {
            m.buckets[idx][i].Value = value
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], Entry{key, value})
    m.size++
}

func (m *AutoMap) Get(key string) (int, bool) {
    idx := m.hash(key)
    for _, e := range m.buckets[idx] {
        if e.Key == key {
            return e.Value, true
        }
    }
    return 0, false
}

func main() {
    m := NewAutoMap(4) // start very small
    for i := 0; i < 20; i++ {
        key := fmt.Sprintf("key_%d", i)
        m.Put(key, i)
    }
    fmt.Printf("\nFinal: size=%d, cap=%d, load=%.2f\n", m.size, m.cap, m.loadFactor())
}
```

**Textual Figure:**
```
  Auto-Resize at maxLoad = 0.75 (start cap=4)
  ════════════════════════════════════════════

  Phase 1: cap=4, insert key_0..key_2
  ┌─────┬─────────┐        α = 3/4 = 0.75 → RESIZE!
  │ [0] │ key_0   │
  │ [1] │ key_1   │
  │ [2] │         │
  │ [3] │ key_2   │
  └─────┴─────────┘
        │
        ▼  ⚡ Resize 4 → 8
  Phase 2: cap=8, rehash + continue inserting
  ┌─────┬─────────┐
  │ [0] │ key_0   │
  │ [1] │         │        After rehash, keys
  │ [2] │ key_2   │        move to new positions
  │ [3] │ key_1   │        (h % 8 ≠ h % 4)
  │ [4] │ key_3   │
  │ [5] │ key_4   │
  │ [6] │         │
  │ [7] │ key_5   │        α = 6/8 = 0.75 → RESIZE!
  └─────┴─────────┘
        │
        ▼  ⚡ Resize 8 → 16
  Phase 3: cap=16, continue...
  ┌──────┬─────────┐
  │ [0]  │ key_0   │       Resize pattern:
  │ [1]  │ key_8   │       cap:  4 → 8 → 16 → 32
  │ ...  │ ...     │       at n:  3    6    12
  │ [15] │ key_11  │       α after resize: ~0.38
  └──────┴─────────┘

  Final: size=20, cap=32, load=0.63

  Resize Timeline:
  ┌──────────┬────────┬────────┬────────────────┐
  │ Insert # │ Action │ cap    │ α after        │
  ├──────────┼────────┼────────┼────────────────┤
  │    3     │ RESIZE │ 4 → 8  │ 3/8  = 0.375  │
  │    6     │ RESIZE │ 8 → 16 │ 6/16 = 0.375  │
  │   12     │ RESIZE │ 16→ 32 │ 12/32= 0.375  │
  │   20     │ done   │ 32     │ 20/32= 0.625  │
  └──────────┴────────┴────────┴────────────────┘
```

---

## Example 3: Load Factor Impact on Collision Rate

```go
package main

import "fmt"

func measureCollisions(numKeys, tableSize int) (int, int, float64) {
    buckets := make([]int, tableSize)
    for i := 0; i < numKeys; i++ {
        h := (i * 2654435761) % tableSize // multiplicative hash
        if h < 0 {
            h = -h
        }
        buckets[h]++
    }

    collisions := 0
    maxChain := 0
    for _, count := range buckets {
        if count > 1 {
            collisions += count - 1
        }
        if count > maxChain {
            maxChain = count
        }
    }

    loadFactor := float64(numKeys) / float64(tableSize)
    return collisions, maxChain, loadFactor
}

func main() {
    numKeys := 10000

    fmt.Printf("%-12s %-12s %-12s %s\n", "Load Factor", "Table Size", "Collisions", "Max Chain")
    fmt.Println("-----------------------------------------------------")

    for _, tableSize := range []int{50000, 20000, 10000, 5000, 2000, 1000} {
        collisions, maxChain, lf := measureCollisions(numKeys, tableSize)
        fmt.Printf("%-12.2f %-12d %-12d %d\n", lf, tableSize, collisions, maxChain)
    }
}
```

**Textual Figure:**
```
  Load Factor vs Collision Rate (n=10,000 keys, multiplicative hash)
  ══════════════════════════════════════════════════════════════════

  Load Factor   Table Size   Collisions   Max Chain   Collision Visual
  ┌───────────┬────────────┬────────────┬───────────┬──────────────────────────┐
  │   0.20    │   50,000   │    ~950    │     2     │ ▓░░░░░░░░░░░░░░░░░░░░░░ │
  │   0.50    │   20,000   │   ~3800    │     4     │ ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░ │
  │   1.00    │   10,000   │   ~5000    │     6     │ ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░ │
  │   2.00    │    5,000   │   ~7500    │    10     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░ │
  │   5.00    │    2,000   │   ~9000    │    17     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░ │
  │  10.00    │    1,000   │   ~9500    │    28     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░ │
  └───────────┴────────────┴────────────┴───────────┴──────────────────────────┘

  Visualization at α=0.20 (sparse):     Visualization at α=5.00 (overloaded):
  ┌───┬───┬───┬───┬───┬───┬───┬───┐    ┌───┬───┬───┬───┐
  │   │ A │   │   │ B │   │   │   │    │ABC│DEF│GHI│JKL│
  └───┴───┴───┴───┴───┴───┴───┴───┘    │MN │OPQ│RST│UV │
  Few collisions, short chains          │WX │YZ │   │   │
                                        └───┴───┴───┴───┘
                                        Many collisions, long chains

  Key insight: Collisions grow rapidly once α > 1.0
```

---

## Example 4: Expected Probe Count by Load Factor

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    fmt.Println("Load Factor vs Expected Probes")
    fmt.Println("================================")
    fmt.Printf("%-8s %-15s %-15s %-15s\n",
        "α", "Linear Probe", "Double Hash", "Chaining")
    fmt.Println("------------------------------------------------------")

    for _, alpha := range []float64{0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 0.95} {
        // Expected probes for successful search:
        // Linear probing: 0.5 * (1 + 1/(1-α))
        linear := 0.5 * (1 + 1/(1-alpha))

        // Double hashing: -ln(1-α)/α
        doubleH := -math.Log(1-alpha) / alpha

        // Chaining: 1 + α/2
        chaining := 1 + alpha/2

        fmt.Printf("%-8.2f %-15.2f %-15.2f %-15.2f\n",
            alpha, linear, doubleH, chaining)
    }

    // Key insight: at α=0.9, linear probing averages 5.5 probes!
}
```

**Textual Figure:**
```
  Expected Probe Count by Load Factor (Successful Search)
  ═══════════════════════════════════════════════════════

  Formulas:
    Linear Probing:  E[probes] = ½(1 + 1/(1−α))
    Double Hashing:  E[probes] = −ln(1−α) / α
    Chaining:        E[probes] = 1 + α/2

  ┌──────┬──────────────┬──────────────┬──────────────┐
  │  α   │ Linear Probe │ Double Hash  │   Chaining   │
  ├──────┼──────────────┼──────────────┼──────────────┤
  │ 0.10 │     1.06     │     1.05     │     1.05     │
  │ 0.30 │     1.21     │     1.19     │     1.15     │
  │ 0.50 │     1.50     │     1.39     │     1.25     │
  │ 0.70 │     2.17     │     1.72     │     1.35     │
  │ 0.80 │     3.00     │     2.01     │     1.40     │
  │ 0.90 │     5.50     │     2.56     │     1.45     │ ← Linear explodes!
  │ 0.95 │    10.50     │     3.15     │     1.48     │
  └──────┴──────────────┴──────────────┴──────────────┘

  Probes vs Load Factor (visual):
  probes
    11 │                                              ╱ Linear
    10 │                                            ╱
     8 │                                          ╱
     6 │                                        ╱
     4 │                                    ╱─╱
     3 │                              ╱──╱    ╱ Double Hash
     2 │                  ╱─────────╱    ╱──╱
     1 │ ─────────────────────────────────── Chaining
     0 ├──────┬──────┬──────┬──────┬──────┬──→ α
       0.0   0.2   0.4   0.6   0.8   1.0

  Takeaway: Keep α < 0.7 for open addressing!
```

---

## Example 5: Shrink on Low Load Factor

```go
package main

import "fmt"

type ShrinkMap struct {
    buckets [][]string
    size    int
    cap     int
}

func NewShrinkMap(cap int) *ShrinkMap {
    return &ShrinkMap{
        buckets: make([][]string, cap),
        cap:     cap,
    }
}

func (m *ShrinkMap) lf() float64 {
    return float64(m.size) / float64(m.cap)
}

func (m *ShrinkMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = (h*31 + int(c))
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *ShrinkMap) rehash(newCap int) {
    old := m.buckets
    m.cap = newCap
    m.buckets = make([][]string, newCap)
    m.size = 0
    for _, bucket := range old {
        for _, k := range bucket {
            m.Add(k)
        }
    }
}

func (m *ShrinkMap) Add(key string) {
    if m.lf() > 0.75 {
        m.rehash(m.cap * 2)
    }
    idx := m.hash(key)
    m.buckets[idx] = append(m.buckets[idx], key)
    m.size++
}

func (m *ShrinkMap) Remove(key string) {
    idx := m.hash(key)
    for i, k := range m.buckets[idx] {
        if k == key {
            m.buckets[idx] = append(m.buckets[idx][:i], m.buckets[idx][i+1:]...)
            m.size--
            break
        }
    }

    // Shrink if load factor drops below 0.25
    if m.cap > 4 && m.lf() < 0.25 {
        fmt.Printf("  Shrinking: %d → %d (load=%.2f)\n", m.cap, m.cap/2, m.lf())
        m.rehash(m.cap / 2)
    }
}

func main() {
    m := NewShrinkMap(4)

    // Add many keys
    for i := 0; i < 20; i++ {
        m.Add(fmt.Sprintf("k%d", i))
    }
    fmt.Printf("After adds: size=%d, cap=%d, load=%.2f\n", m.size, m.cap, m.lf())

    // Remove most keys
    for i := 0; i < 18; i++ {
        m.Remove(fmt.Sprintf("k%d", i))
    }
    fmt.Printf("After removes: size=%d, cap=%d, load=%.2f\n", m.size, m.cap, m.lf())
}
```

**Textual Figure:**
```
  Shrink on Low Load Factor (grow threshold=0.75, shrink threshold=0.25)
  ═════════════════════════════════════════════════════════════════════

  Phase 1: GROW — Adding 20 keys (start cap=4)
  ┌───────────────────────────────────────────────────────┐
  │ cap=4  → add k0-k2  → α=0.75 → RESIZE!              │
  │ cap=8  → add k3-k5  → α=0.75 → RESIZE!              │
  │ cap=16 → add k6-k11 → α=0.75 → RESIZE!              │
  │ cap=32 → add k12-k19→ α=0.63                         │
  └───────────────────────────────────────────────────────┘
  After adds: size=20, cap=32, load=0.63

  ┌─────┬───┬───┬───┬───┬───┬───┬───┬─── ─ ─ ───┬─────┐
  │ idx │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │    ...    │ 31  │  cap = 32
  │ cnt │ 1 │ 0 │ 1 │ 1 │ 0 │ 1 │ 1 │    ...    │  1  │  20 keys
  └─────┴───┴───┴───┴───┴───┴───┴───┴─── ─ ─ ───┴─────┘

  Phase 2: SHRINK — Removing 18 keys
  ┌───────────────────────────────────────────────────────┐
  │ remove k0..k14 → size=5, cap=32, α=0.16 → SHRINK!   │
  │ cap=16 → remove k15 → size=4, α=0.25                │
  │ remove k16 → size=3, α=0.19 → SHRINK!               │
  │ cap=8  → remove k17 → size=2, α=0.25                │
  └───────────────────────────────────────────────────────┘

  Cap transitions:
  GROW:    4 ──→ 8 ──→ 16 ──→ 32
                                │
  SHRINK: 8 ←── 16 ←────────────┘

  After removes: size=2, cap=8, load=0.25
  ┌─────┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ idx │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  cap = 8
  │     │   │   │k18│   │   │   │k19│   │  2 keys
  └─────┴───┴───┴───┴───┴───┴───┴───┴───┘
```

---

## Example 6: Amortized Cost of Resizing

```go
package main

import "fmt"

func main() {
    // Simulate insertions and track resize costs
    cap := 4
    size := 0
    totalCost := 0
    resizes := 0

    for i := 1; i <= 100; i++ {
        if float64(size)/float64(cap) >= 0.75 {
            // Resize: costs O(n) to rehash all elements
            totalCost += size
            cap *= 2
            resizes++
            fmt.Printf("  Resize at i=%d: rehash %d elements, new cap=%d\n", i, size, cap)
        }
        totalCost++ // regular insert O(1)
        size++
    }

    fmt.Printf("\nTotal insertions: %d\n", size)
    fmt.Printf("Total cost: %d\n", totalCost)
    fmt.Printf("Amortized cost per insert: %.2f\n", float64(totalCost)/float64(size))
    fmt.Printf("Resizes: %d\n", resizes)
    // Amortized cost per insert ≈ O(1) due to geometric growth
}
```

**Textual Figure:**
```
  Amortized Cost of Resizing (100 inserts, threshold=0.75, cap starts at 4)
  ═════════════════════════════════════════════════════════════

  Resize events and costs:
  ┌──────────┬─────────┬───────────┬───────────────┐
  │ Insert # │ Resize  │ Rehash    │ New Cap       │
  ├──────────┼─────────┼───────────┼───────────────┤
  │    4     │  #1     │  3 elems  │ 4 → 8         │
  │    7     │  #2     │  6 elems  │ 8 → 16        │
  │   13     │  #3     │ 12 elems  │ 16 → 32       │
  │   25     │  #4     │ 24 elems  │ 32 → 64       │
  │   49     │  #5     │ 48 elems  │ 64 → 128      │
  │   97     │  #6     │ 96 elems  │ 128 → 256     │
  └──────────┴─────────┴───────────┴───────────────┘

  Cost per insert (visual):
  cost
    96 │                                              │ (resize #6)
       │
    48 │                             │                  (resize #5)
       │
    24 │                │                               (resize #4)
    12 │          │                                     (resize #3)
     6 │     │                                          (resize #2)
     3 │  │                                             (resize #1)
     1 │ ───────────────────────────────  (normal inserts)
     0 ├───────┬──────┬──────┬──────┬──────┬──→
       0     20    40    60    80   100  insert #

  Total:  100 inserts + 189 rehash ops = 289 total cost
  Amortized: 289/100 ≈ 2.89 per insert ≈ O(1)
```

---

## Example 7: Optimal Load Factor Selection

```go
package main

import (
    "fmt"
    "math"
)

type Config struct {
    Name        string
    MaxLoad     float64
    GrowFactor  float64
}

func analyzeConfig(cfg Config, numOps int) {
    cap := 16
    size := 0
    totalRehashCost := 0
    resizes := 0
    peakMemory := cap

    for i := 0; i < numOps; i++ {
        if float64(size+1)/float64(cap) > cfg.MaxLoad {
            totalRehashCost += size
            cap = int(math.Ceil(float64(cap) * cfg.GrowFactor))
            resizes++
            if cap > peakMemory {
                peakMemory = cap
            }
        }
        size++
    }

    fmt.Printf("%-20s maxLoad=%.2f grow=%.1fx\n", cfg.Name, cfg.MaxLoad, cfg.GrowFactor)
    fmt.Printf("  Final: size=%d cap=%d load=%.2f\n", size, cap, float64(size)/float64(cap))
    fmt.Printf("  Resizes: %d, Rehash cost: %d\n", resizes, totalRehashCost)
    fmt.Printf("  Peak memory (slots): %d\n\n", peakMemory)
}

func main() {
    configs := []Config{
        {"Go map (lf=6.5/b)", 0.8, 2.0},
        {"Java HashMap", 0.75, 2.0},
        {"Python dict", 0.67, 2.0},
        {"Conservative", 0.5, 2.0},
        {"Aggressive", 0.9, 1.5},
    }

    numOps := 100000
    fmt.Printf("Simulating %d insertions:\n\n", numOps)
    for _, cfg := range configs {
        analyzeConfig(cfg, numOps)
    }
}
```

**Textual Figure:**
```
  Optimal Load Factor Selection — 100,000 Insertions Comparison
  ════════════════════════════════════════════════════════════════

  ┌──────────────────┬────────┬──────┬─────────┬──────────┬──────────┐
  │    Config        │ maxLF  │ grow │ Resizes │ Rehash   │ Peak Mem │
  ├──────────────────┼────────┼──────┼─────────┼──────────┼──────────┤
  │ Go map (lf=0.8) │  0.80  │ 2.0x │   16    │ ~199k   │ 131,072  │
  │ Java HashMap     │  0.75  │ 2.0x │   16    │ ~199k   │ 131,072  │
  │ Python dict      │  0.67  │ 2.0x │   17    │ ~199k   │ 262,144  │
  │ Conservative     │  0.50  │ 2.0x │   17    │ ~199k   │ 262,144  │
  │ Aggressive       │  0.90  │ 1.5x │   25    │ ~399k   │ 153,894  │
  └──────────────────┴────────┴──────┴─────────┴──────────┴──────────┘

  Trade-off Spectrum:
  ┌────────────────────────────────────────────────────────────┐
  │  Low α (0.5)          High α (0.9)                      │
  │  ────────────────────────────────────────────    │
  │  ↑ More memory              ↑ More collisions          │
  │  ↑ Fewer collisions         ↑ Less memory              │
  │  ↑ Fewer resizes             ↑ More resizes (if 1.5x)   │
  │                                                          │
  │  Sweet spot: α ≈ 0.70–0.75 with 2x growth                  │
  └────────────────────────────────────────────────────────────┘
```

---

## Example 8: Load Factor in Go's Built-in Map

```go
package main

import (
    "fmt"
    "runtime"
    "unsafe"
)

func main() {
    // Go's map uses a load factor of ~6.5 per bucket
    // Each bucket holds 8 key-value pairs
    // Effective load factor ≈ 6.5/8 ≈ 0.8125

    m := make(map[int]int)
    var prev runtime.MemStats

    checkpoints := []int{100, 1000, 10000, 100000}
    ci := 0

    runtime.ReadMemStats(&prev)

    for i := 0; i < 100001; i++ {
        m[i] = i
        if ci < len(checkpoints) && i == checkpoints[ci] {
            var ms runtime.MemStats
            runtime.ReadMemStats(&ms)
            fmt.Printf("Size: %6d | Map memory ≈ %d KB | Overhead per entry ≈ %d bytes\n",
                len(m),
                (ms.HeapAlloc-prev.HeapAlloc)/1024,
                int(unsafe.Sizeof(0))*2, // approximate
            )
            ci++
        }
    }

    // Go manages load factor internally
    // Buckets overflow at 8 entries → linked overflow buckets
}
```

**Textual Figure:**
```
  Go's Built-in Map: Internal Bucket Structure
  ═════════════════════════════════════════════════

  Go map load factor = 6.5 per bucket (each bucket holds 8 slots)
  Effective α ≈ 6.5 / 8 ≈ 0.8125

  Map Header (hmap)
  ┌────────────────┐
  │ count: 5       │
  │ B: 1 (2 bkts)  │ ───┐
  │ buckets: *      │     │
  └────────────────┘     │
                         ▼
  Bucket Array (2^B = 2 buckets)
  ┌───────────────────────────────────────────┐
  │ Bucket 0:                                     │
  │  tophash: [h0][h1][h2][ ][ ][ ][ ][ ]          │
  │  keys:    [k0][k1][k2][ ][ ][ ][ ][ ]          │
  │  values:  [v0][v1][v2][ ][ ][ ][ ][ ]          │
  │  overflow: nil                                  │
  ├───────────────────────────────────────────┤
  │ Bucket 1:                                     │
  │  tophash: [h3][h4][ ][ ][ ][ ][ ][ ]           │
  │  keys:    [k3][k4][ ][ ][ ][ ][ ][ ]           │
  │  values:  [v3][v4][ ][ ][ ][ ][ ][ ]           │
  │  overflow: nil                                  │
  └───────────────────────────────────────────┘

  When bucket overflows (>8 entries):
  ┌────────────────┐     ┌────────────────┐
  │ Bucket (8 slots)│─→─│ Overflow Bucket │─→ nil
  │  [full........] │     │  [extra entries]│
  └────────────────┘     └────────────────┘

  Memory growth (approximate):
  ┌─────────┬───────────┬──────────────────┐
  │  Size   │ Heap (KB) │ Overhead/entry   │
  ├─────────┼───────────┼──────────────────┤
  │    100  │     ~8    │  ~16 bytes       │
  │  1,000  │    ~64    │  ~16 bytes       │
  │ 10,000  │   ~640    │  ~16 bytes       │
  │100,000  │  ~6,400   │  ~16 bytes       │
  └─────────┴───────────┴──────────────────┘
```

---

## Example 9: Load Factor and Hash Table Performance Benchmark

```go
package main

import (
    "fmt"
    "time"
)

func benchmarkAtLoadFactor(targetLF float64, numOps int) time.Duration {
    tableSize := int(float64(numOps) / targetLF)
    if tableSize < 1 {
        tableSize = 1
    }

    buckets := make([][]int, tableSize)
    hash := func(k int) int { return k % tableSize }

    // Insert
    for i := 0; i < numOps; i++ {
        idx := hash(i)
        buckets[idx] = append(buckets[idx], i)
    }

    // Benchmark lookups
    start := time.Now()
    for i := 0; i < numOps; i++ {
        idx := hash(i)
        for _, v := range buckets[idx] {
            if v == i {
                break
            }
        }
    }
    return time.Since(start)
}

func main() {
    numOps := 100000

    fmt.Printf("%-12s %-12s\n", "Load Factor", "Lookup Time")
    fmt.Println("-------------------------")
    for _, lf := range []float64{0.25, 0.5, 0.75, 1.0, 2.0, 5.0, 10.0} {
        t := benchmarkAtLoadFactor(lf, numOps)
        fmt.Printf("%-12.2f %-12v\n", lf, t)
    }
}
```

**Textual Figure:**
```
  Hash Table Performance Benchmark (100K ops, chaining)
  ═══════════════════════════════════════════════════

  Setup: tableSize = int(numKeys / targetLF)

  ┌───────────┬────────────┬───────────┬────────────────────────────┐
  │ Load Fac. │ Table Size │ Avg Chain │ Relative Lookup Speed      │
  ├───────────┼────────────┼───────────┼────────────────────────────┤
  │   0.25    │  400,000   │   ~1.1    │ ▓░░░░░░░░░  fastest      │
  │   0.50    │  200,000   │   ~1.3    │ ▓▓░░░░░░░░               │
  │   0.75    │  133,333   │   ~1.4    │ ▓▓▓░░░░░░░  optimal      │
  │   1.00    │  100,000   │   ~1.5    │ ▓▓▓▓░░░░░░               │
  │   2.00    │   50,000   │   ~2.0    │ ▓▓▓▓▓▓░░░░               │
  │   5.00    │   20,000   │   ~3.5    │ ▓▓▓▓▓▓▓▓░░               │
  │  10.00    │   10,000   │   ~6.0    │ ▓▓▓▓▓▓▓▓▓▓  slowest     │
  └───────────┴────────────┴───────────┴────────────────────────────┘

  Bucket state at α=0.25:          Bucket state at α=10.0:
  ┌───┐┌───┐┌───┐┌───┐┌───┐   ┌───┐┌───┐
  │   ││ A ││   ││   ││ B │   │AB ││CD │
  └───┘└───┘└───┘└───┘└───┘   │EF ││GH │
  Mostly empty, O(1) lookup        │IJ ││KL │
                                   │MN ││OP │
                                   └───┘└───┘
                                   Long chains, O(n) lookup
```

---

## Example 10: Dynamic Load Factor Visualizer

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    cap := 8
    size := 0
    threshold := 0.75

    fmt.Println("Insertion sequence with load factor tracking:")
    fmt.Println("==============================================")

    for i := 0; i < 30; i++ {
        lf := float64(size) / float64(cap)

        // Visual bar
        barLen := int(lf * 40)
        if barLen > 40 {
            barLen = 40
        }
        bar := strings.Repeat("█", barLen) + strings.Repeat("░", 40-barLen)

        marker := ""
        if lf >= threshold {
            marker = " ← RESIZE!"
            cap *= 2
        }

        fmt.Printf("i=%2d size=%2d cap=%2d lf=%.3f |%s|%s\n",
            i, size, cap, lf, bar, marker)

        size++
    }
}
```

**Textual Figure:**
```
  Dynamic Load Factor Visualizer (cap=8, threshold=0.75, 30 inserts)
  ════════════════════════════════════════════════════════════

  i= 0  size= 0  cap= 8  lf=0.000  |░░░░░░░░░░░░░░░░░░░░|
  i= 1  size= 1  cap= 8  lf=0.125  |▓░░░░░░░░░░░░░░░░░░░|
  i= 2  size= 2  cap= 8  lf=0.250  |▓▓▓░░░░░░░░░░░░░░░░░|
  i= 3  size= 3  cap= 8  lf=0.375  |▓▓▓▓▓░░░░░░░░░░░░░░░|
  i= 4  size= 4  cap= 8  lf=0.500  |▓▓▓▓▓▓▓░░░░░░░░░░░░░|
  i= 5  size= 5  cap= 8  lf=0.625  |▓▓▓▓▓▓▓▓▓░░░░░░░░░░░|
  i= 6  size= 6  cap=16  lf=0.750  |▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░| ← RESIZE!  8→16
  i= 7  size= 7  cap=16  lf=0.438  |▓▓▓▓▓▓░░░░░░░░░░░░░░|   (load drops)
  i= 8  size= 8  cap=16  lf=0.500  |▓▓▓▓▓▓▓░░░░░░░░░░░░░|
  ...                                                  
  i=12  size=12  cap=32  lf=0.750  |▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░| ← RESIZE! 16→32
  i=13  size=13  cap=32  lf=0.406  |▓▓▓▓▓▓░░░░░░░░░░░░░░|   (load drops)
  ...                                                  
  i=24  size=24  cap=64  lf=0.750  |▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░| ← RESIZE! 32→64
  i=25  size=25  cap=64  lf=0.391  |▓▓▓▓▓░░░░░░░░░░░░░░░|   (load drops)
  ...                                                  
  i=29  size=29  cap=64  lf=0.453  |▓▓▓▓▓▓░░░░░░░░░░░░░░|

  Load factor over time (sawtooth pattern):
  α
  0.75 │    /│      /│           /│
  0.50 │   / │     / │          / │
  0.38 │  /  │    /  │ drop    /  │ drop
  0.25 │ /   │   /   │  ↓    /   │  ↓
  0.00 │/    │  /    │     /    │
       ├────┼─────┼───────┼─────────→ inserts
       0   6    12       24
         cap8  cap16    cap32   cap64
```

---

## Load Factor Summary

| Hash Table Impl | Default Load Factor | Growth Factor |
|-----------------|-------------------|---------------|
| Go map          | ~6.5/bucket (≈0.81)| 2x           |
| Java HashMap    | 0.75               | 2x           |
| Python dict     | 0.67               | 2x (approx)  |
| C++ unordered_map | 1.0              | 2x           |
| Redis dict      | 1.0                | 2x           |

## Key Takeaways

1. **α = n/m** — ratio of elements to buckets
2. **Open addressing**: keep α < 0.7 for good performance
3. **Chaining**: can tolerate α > 1.0 gracefully
4. **Resize** when α exceeds threshold (typically 0.75)
5. **Amortized O(1)** — geometric growth ensures low average cost
6. **Shrink** when α drops below 0.25 to save memory
7. **Trade-off**: memory (low α) vs speed (high α causes collisions)

> **Next up:** Hash Map Resizing →
