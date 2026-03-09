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
