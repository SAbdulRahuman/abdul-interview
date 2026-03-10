# Phase 4: Hash Tables — Hash Map Resizing

## What is Hash Map Resizing?

When a hash table's load factor exceeds a threshold, it must **resize** (usually doubling) and **rehash** all existing entries into the new, larger bucket array. This keeps operations close to O(1).

**Key concepts:**
- **Growth**: double capacity when load factor exceeds threshold
- **Rehashing**: recompute hash positions for all entries (since `h % newCap ≠ h % oldCap`)
- **Amortized O(1)**: costly resize is infrequent → average insert stays O(1)
- **Shrink**: halve capacity when load factor drops too low

---

## Example 1: Basic Resize with Rehash

```go
package main

import "fmt"

type Entry struct {
    Key   string
    Value int
}

type HashMap struct {
    buckets [][]Entry
    size    int
    cap     int
}

func NewHashMap(cap int) *HashMap {
    return &HashMap{buckets: make([][]Entry, cap), cap: cap}
}

func (m *HashMap) hash(key string) int {
    h := 0
    for _, c := range key {
        h = h*31 + int(c)
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *HashMap) resize() {
    fmt.Printf("  RESIZE %d → %d\n", m.cap, m.cap*2)
    old := m.buckets
    m.cap *= 2
    m.buckets = make([][]Entry, m.cap)
    m.size = 0
    for _, bucket := range old {
        for _, e := range bucket {
            m.put(e.Key, e.Value, false)
        }
    }
}

func (m *HashMap) put(key string, value int, checkResize bool) {
    if checkResize && float64(m.size+1)/float64(m.cap) > 0.75 {
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

func (m *HashMap) Put(key string, value int) {
    m.put(key, value, true)
}

func main() {
    m := NewHashMap(4)
    for i := 0; i < 20; i++ {
        k := fmt.Sprintf("key_%d", i)
        m.Put(k, i)
        fmt.Printf("Put %s → size=%d cap=%d load=%.2f\n",
            k, m.size, m.cap, float64(m.size)/float64(m.cap))
    }
}
```

**Textual Figure:**
```
  Basic Resize with Rehash (start cap=4, threshold=0.75)
  ══════════════════════════════════════════════════════════════

  Step 1: Insert key_0, key_1, key_2 into cap=4
  ┌─────┬───────────┐
  │ [0] │ key_0     │     α = 3/4 = 0.75
  │ [1] │ key_1     │
  │ [2] │           │
  │ [3] │ key_2     │
  └─────┴───────────┘
        │
        ▼  Put key_3 triggers: (3+1)/4 = 1.0 > 0.75
  Step 2: RESIZE 4 → 8, rehash all entries
  ┌─────┬───────────┐
  │ [0] │ key_0     │     Rehash: new_idx = hash(key) % 8
  │ [1] │           │
  │ [2] │ key_2     │     key_0: hash%4=0 → hash%8=0
  │ [3] │ key_1     │     key_1: hash%4=1 → hash%8=3
  │ [4] │ key_3     │     key_2: hash%4=3 → hash%8=2
  │ [5] │           │
  │ [6] │           │     Continue inserting key_3..key_5
  │ [7] │ key_5     │     α = 6/8 = 0.75 → RESIZE again!
  └─────┴───────────┘
        │
        ▼  RESIZE 8 → 16
  Step 3: cap=16, rehash, continue to key_19
  ┌──────┬───────────┐
  │ [0]  │ key_0     │     Resize chain:
  │ [1]  │ key_8     │     4 → 8 → 16 → 32
  │ ...  │ ...       │
  │ [15] │ key_11    │     Final: size=20, cap=32, α=0.63
  └──────┴───────────┘
```

---

## Example 2: Doubling vs 1.5x Growth Factor

```go
package main

import (
    "fmt"
    "math"
)

func simulate(growFactor float64, maxLoad float64, numInserts int) (int, int, int) {
    cap := 4
    size := 0
    resizes := 0
    totalRehash := 0

    for i := 0; i < numInserts; i++ {
        if float64(size+1)/float64(cap) > maxLoad {
            totalRehash += size
            cap = int(math.Ceil(float64(cap) * growFactor))
            resizes++
        }
        size++
    }
    return resizes, totalRehash, cap
}

func main() {
    n := 100000
    fmt.Printf("%-15s %-10s %-12s %-12s %-12s\n",
        "Strategy", "MaxLoad", "Resizes", "Rehash Cost", "Final Cap")
    fmt.Println(strings.Repeat("-", 65))

    configs := [][2]float64{
        {2.0, 0.75},   // classic doubling
        {1.5, 0.75},   // 1.5x growth
        {2.0, 0.5},    // aggressive resize
        {4.0, 0.75},   // quadrupling
        {1.5, 0.9},    // lazy resize
    }

    for _, cfg := range configs {
        grow, load := cfg[0], cfg[1]
        resizes, rehash, finalCap := simulate(grow, load, n)
        fmt.Printf("grow=%.1fx      lf=%.2f     %-12d %-12d %d\n",
            grow, load, resizes, rehash, finalCap)
    }
}
```

**Textual Figure:**
```
  Doubling vs 1.5x Growth Factor (100K inserts, start cap=4)
  ════════════════════════════════════════════════════════════

  ┌──────────────┬──────┬─────────┬───────────┬───────────┐
  │  Strategy    │ LF   │ Resizes │ Rehash    │ Final Cap │
  ├──────────────┼──────┼─────────┼───────────┼───────────┤
  │ 2.0x, lf=.75 │ 0.75 │    16   │  ~199K    │  131,072  │
  │ 1.5x, lf=.75 │ 0.75 │    25   │  ~399K    │  153,894  │
  │ 2.0x, lf=.50 │ 0.50 │    17   │  ~199K    │  262,144  │
  │ 4.0x, lf=.75 │ 0.75 │     8   │  ~199K    │  262,144  │
  │ 1.5x, lf=.90 │ 0.90 │    25   │  ~399K    │  127,834  │
  └──────────────┴──────┴─────────┴───────────┴───────────┘

  Capacity growth comparison (2x vs 1.5x):

  cap
  256K │                             ╱ 2.0x
  128K │                         ╱───
   64K │                     ╱───
   32K │                 ╱───      ..···· 1.5x
   16K │             ╱───    ..···
    8K │         ╱───  ..···
    4K │     ╱───···
       │ ╱─··
     4 │╱
       ├──────┬──────┬──────┬──────┬───→ inserts
       0    25K   50K   75K  100K

  Key: 2x = fewer resizes but bigger jumps
       1.5x = more resizes but smoother growth
```

> Need `"strings"` import — add it to see output.

---

## Example 3: Incremental Rehash (Redis-Style)

```go
package main

import "fmt"

type Entry struct {
    Key   string
    Value int
}

type IncrementalMap struct {
    oldBuckets [][]Entry
    newBuckets [][]Entry
    oldCap     int
    newCap     int
    size       int
    rehashIdx  int // next bucket to migrate
    migrating  bool
}

func NewIncrementalMap(cap int) *IncrementalMap {
    return &IncrementalMap{
        oldBuckets: make([][]Entry, cap),
        oldCap:     cap,
        size:       0,
    }
}

func hash(key string, cap int) int {
    h := 0
    for _, c := range key {
        h = h*31 + int(c)
    }
    if h < 0 {
        h = -h
    }
    return h % cap
}

func (m *IncrementalMap) startMigration() {
    m.newCap = m.oldCap * 2
    m.newBuckets = make([][]Entry, m.newCap)
    m.rehashIdx = 0
    m.migrating = true
    fmt.Printf("  Start migration: %d → %d\n", m.oldCap, m.newCap)
}

func (m *IncrementalMap) migrateStep() {
    if !m.migrating || m.rehashIdx >= m.oldCap {
        return
    }
    // Move one bucket per operation
    bucket := m.oldBuckets[m.rehashIdx]
    for _, e := range bucket {
        idx := hash(e.Key, m.newCap)
        m.newBuckets[idx] = append(m.newBuckets[idx], e)
    }
    m.oldBuckets[m.rehashIdx] = nil
    m.rehashIdx++

    fmt.Printf("  Migrated bucket %d/%d\n", m.rehashIdx, m.oldCap)

    if m.rehashIdx >= m.oldCap {
        m.oldBuckets = m.newBuckets
        m.oldCap = m.newCap
        m.newBuckets = nil
        m.newCap = 0
        m.migrating = false
        fmt.Println("  Migration complete!")
    }
}

func (m *IncrementalMap) Put(key string, value int) {
    cap := m.oldCap
    if m.migrating {
        m.migrateStep()
    }

    lf := float64(m.size+1) / float64(cap)
    if !m.migrating && lf > 0.75 {
        m.startMigration()
    }

    // Insert into new if migrating, else old
    if m.migrating {
        idx := hash(key, m.newCap)
        m.newBuckets[idx] = append(m.newBuckets[idx], Entry{key, value})
    } else {
        idx := hash(key, m.oldCap)
        m.oldBuckets[idx] = append(m.oldBuckets[idx], Entry{key, value})
    }
    m.size++
}

func main() {
    m := NewIncrementalMap(4)
    for i := 0; i < 15; i++ {
        k := fmt.Sprintf("k%d", i)
        m.Put(k, i)
    }
    fmt.Printf("Final: size=%d cap=%d\n", m.size, m.oldCap)
}
```

**Textual Figure:**
```
  Incremental Rehash — Redis-Style (cap=4, migrate 1 bucket per Put)
  ════════════════════════════════════════════════════════════════

  Phase 1: Normal inserts (k0..k2 into oldBuckets, cap=4)
  oldBuckets (cap=4)            newBuckets (nil)
  ┌─────┬─────────┐
  │ [0] │ k0      │
  │ [1] │ k1      │              (not created yet)
  │ [2] │         │
  │ [3] │ k2      │
  └─────┴─────────┘
        │
        ▼  Put k3 → lf > 0.75, start migration!

  Phase 2: Migration starts (cap 4 → 8)
  oldBuckets (cap=4)            newBuckets (cap=8)
  ┌─────┬─────────┐           ┌─────┬─────────┐
  │ [0] │ ───────│──migrate─→│ [0] │ k0      │
  │ [1] │ k1      │           │ [1] │         │
  │ [2] │         │           │ [2] │         │
  │ [3] │ k2      │           │ [3] │ k3 (new)│
  └─────┴─────────┘           │ [4] │         │
    rehashIdx=1                 │ [5] │         │
    (bucket 0 migrated)         │ [6] │         │
                                │ [7] │         │
                                └─────┴─────────┘
  Phase 3: Continue — each Put migrates 1 more bucket
  ┌────────────┬────────────┬──────────────────────────────┐
  │  Put k4   │ migrate [1] │ rehashIdx=2                    │
  │  Put k5   │ migrate [2] │ rehashIdx=3                    │
  │  Put k6   │ migrate [3] │ rehashIdx=4 → DONE!            │
  └────────────┴────────────┴──────────────────────────────┘

  After migration complete: oldBuckets = newBuckets, cap=8
  No single operation pays full rehash cost!

  Traditional vs Incremental:
  ┌─────────────┬─────────────────────────┐
  │ Traditional │ ░░░░░░░░░▓▓▓▓▓▓▓▓░░░░░░ │  big spike
  │ Incremental │ ░▒░▒░▒░▒░▒░▒░▒░▒░▒░▒░░ │  smooth
  └─────────────┴─────────────────────────┘
```

---

## Example 4: Resize Preserves All Data

```go
package main

import "fmt"

type KV struct {
    K string
    V int
}

type SafeMap struct {
    buckets [][]KV
    cap     int
    size    int
}

func NewSafeMap(cap int) *SafeMap {
    return &SafeMap{buckets: make([][]KV, cap), cap: cap}
}

func (m *SafeMap) hash(k string) int {
    h := 0
    for _, c := range k {
        h = h*31 + int(c)
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *SafeMap) Put(k string, v int) {
    if float64(m.size+1)/float64(m.cap) > 0.75 {
        m.grow()
    }
    idx := m.hash(k)
    for i, kv := range m.buckets[idx] {
        if kv.K == k {
            m.buckets[idx][i].V = v
            return
        }
    }
    m.buckets[idx] = append(m.buckets[idx], KV{k, v})
    m.size++
}

func (m *SafeMap) Get(k string) (int, bool) {
    idx := m.hash(k)
    for _, kv := range m.buckets[idx] {
        if kv.K == k {
            return kv.V, true
        }
    }
    return 0, false
}

func (m *SafeMap) grow() {
    old := m.buckets
    m.cap *= 2
    m.buckets = make([][]KV, m.cap)
    m.size = 0
    for _, b := range old {
        for _, kv := range b {
            m.Put(kv.K, kv.V)
        }
    }
}

func main() {
    m := NewSafeMap(2)

    // Insert 100 entries
    for i := 0; i < 100; i++ {
        m.Put(fmt.Sprintf("item_%d", i), i*10)
    }

    // Verify ALL data preserved after multiple resizes
    allFound := true
    for i := 0; i < 100; i++ {
        v, ok := m.Get(fmt.Sprintf("item_%d", i))
        if !ok || v != i*10 {
            allFound = false
            fmt.Printf("MISSING: item_%d\n", i)
        }
    }
    fmt.Printf("All 100 entries preserved after resize: %v\n", allFound)
    fmt.Printf("Final: size=%d cap=%d load=%.2f\n", m.size, m.cap, float64(m.size)/float64(m.cap))
}
```

**Textual Figure:**
```
  Resize Preserves All Data (insert 100 items, start cap=2)
  ═════════════════════════════════════════════════════════

  Resize chain (each doubles cap):
  ┌───────┬────────┬───────────┬─────────────────────────────┐
  │ cap   │ resize │ at size   │ Items rehashed               │
  ├───────┼────────┼───────────┼─────────────────────────────┤
  │   2   │  #1    │     2     │ 2 items → cap 4             │
  │   4   │  #2    │     4     │ 4 items → cap 8             │
  │   8   │  #3    │     7     │ 7 items → cap 16            │
  │  16   │  #4    │    13     │ 13 items → cap 32           │
  │  32   │  #5    │    25     │ 25 items → cap 64           │
  │  64   │  #6    │    49     │ 49 items → cap 128          │
  │ 128   │  ─     │   100     │ (no more resizes needed)    │
  └───────┴────────┴───────────┴─────────────────────────────┘

  Data integrity verification:
  ┌──────────────────────────────────────────────┐
  │  For each item_i (i=0..99):                    │
  │    Get("item_i") == i*10  ?                     │
  │                                                │
  │  Before resize:  item_5 at bucket hash%16 = 3   │
  │  After resize:   item_5 at bucket hash%32 = 19  │
  │                  (different bucket, same value!) │
  │                                                │
  │  Result: ✓ All 100 entries preserved = true      │
  │  Final:  size=100  cap=128  load=0.78           │
  └──────────────────────────────────────────────┘
```

---

## Example 5: Power-of-2 vs Prime-Size Resizing

```go
package main

import "fmt"

func isPrime(n int) bool {
    if n < 2 {
        return false
    }
    for i := 2; i*i <= n; i++ {
        if n%i == 0 {
            return false
        }
    }
    return true
}

func nextPrime(n int) int {
    for !isPrime(n) {
        n++
    }
    return n
}

func countCollisions(keys []int, tableSize int) int {
    buckets := make([]int, tableSize)
    for _, k := range keys {
        h := k % tableSize
        buckets[h]++
    }
    collisions := 0
    for _, c := range buckets {
        if c > 1 {
            collisions += c - 1
        }
    }
    return collisions
}

func main() {
    // Generate keys that are multiples of common numbers
    keys := make([]int, 1000)
    for i := range keys {
        keys[i] = i * 16 // pathological for power-of-2
    }

    fmt.Printf("%-10s %-10s %-12s\n", "Size", "Type", "Collisions")
    fmt.Println("----------------------------------")

    sizes := []int{1024, 2048, 4096}
    for _, s := range sizes {
        c := countCollisions(keys, s)
        fmt.Printf("%-10d %-10s %-12d\n", s, "power-2", c)

        p := nextPrime(s)
        c2 := countCollisions(keys, p)
        fmt.Printf("%-10d %-10s %-12d\n", p, "prime", c2)
    }
}
```

**Textual Figure:**
```
  Power-of-2 vs Prime-Size Resizing (1000 keys = i*16, pathological)
  ══════════════════════════════════════════════════════════

  Keys = [0, 16, 32, 48, 64, ...] (multiples of 16)

  Power-of-2 table (size=1024):     Prime table (size=1031):
  key % 1024:                       key % 1031:
  ┌─────┬─────────────────┐       ┌─────┬─────────────────┐
  │ [0] │ 0,1024,2048...  │       │ [0] │ 0              │
  │ [16]│ 16,1040,2064..│       │ [16]│ 16             │
  │ [32]│ 32,1056,2080..│       │ [32]│ 32             │
  │ [48]│ 48,1072...     │       │ ... │ (spread evenly)│
  │ ... │ ...            │       └─────┴─────────────────┘
  └─────┴─────────────────┘
  Only 64 buckets used!             All 1031 buckets used!
  (1024/16 = 64 distinct slots)     (gcd(16,1031)=1 → spread)

  ┌────────┬─────────┬─────────────┐
  │ Size   │  Type   │ Collisions  │
  ├────────┼─────────┼─────────────┤
  │  1024  │ power-2 │   ~936      │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  │  1031  │ prime   │   ~0        │  ░
  │  2048  │ power-2 │   ~872      │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  │  2053  │ prime   │   ~0        │  ░
  │  4096  │ power-2 │   ~744      │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
  │  4099  │ prime   │   ~0        │  ░
  └────────┴─────────┴─────────────┘

  Why? key % (power-of-2) preserves low bits only!
  If keys share a common factor with table size → clustering.
  Prime sizes break this pattern.
```

---

## Example 6: Shrink on Delete

```go
package main

import "fmt"

type ResizableMap struct {
    buckets  [][]string
    cap      int
    size     int
    minCap   int
}

func NewResizableMap() *ResizableMap {
    return &ResizableMap{
        buckets: make([][]string, 8),
        cap:     8,
        minCap:  8,
    }
}

func (m *ResizableMap) hash(k string) int {
    h := 0
    for _, c := range k {
        h = h*31 + int(c)
    }
    if h < 0 {
        h = -h
    }
    return h % m.cap
}

func (m *ResizableMap) rehash(newCap int) {
    old := m.buckets
    m.cap = newCap
    m.buckets = make([][]string, newCap)
    m.size = 0
    for _, b := range old {
        for _, k := range b {
            idx := m.hash(k)
            m.buckets[idx] = append(m.buckets[idx], k)
            m.size++
        }
    }
}

func (m *ResizableMap) Add(k string) {
    if float64(m.size+1)/float64(m.cap) > 0.75 {
        fmt.Printf("  GROW %d → %d\n", m.cap, m.cap*2)
        m.rehash(m.cap * 2)
    }
    idx := m.hash(k)
    m.buckets[idx] = append(m.buckets[idx], k)
    m.size++
}

func (m *ResizableMap) Remove(k string) bool {
    idx := m.hash(k)
    for i, key := range m.buckets[idx] {
        if key == k {
            m.buckets[idx] = append(m.buckets[idx][:i], m.buckets[idx][i+1:]...)
            m.size--

            // Shrink if load < 0.125 (too sparse)
            if m.cap > m.minCap && float64(m.size)/float64(m.cap) < 0.125 {
                fmt.Printf("  SHRINK %d → %d\n", m.cap, m.cap/2)
                m.rehash(m.cap / 2)
            }
            return true
        }
    }
    return false
}

func main() {
    m := NewResizableMap()

    // Add 50 items
    for i := 0; i < 50; i++ {
        m.Add(fmt.Sprintf("item_%d", i))
    }
    fmt.Printf("After 50 adds: size=%d cap=%d load=%.3f\n\n", m.size, m.cap, float64(m.size)/float64(m.cap))

    // Delete 47 items
    for i := 0; i < 47; i++ {
        m.Remove(fmt.Sprintf("item_%d", i))
    }
    fmt.Printf("After 47 deletes: size=%d cap=%d load=%.3f\n", m.size, m.cap, float64(m.size)/float64(m.cap))
}
```

**Textual Figure:**
```
  Shrink on Delete (grow at α>0.75, shrink at α<0.125, minCap=8)
  ══════════════════════════════════════════════════════════

  Phase 1: GROW (add 50 items, start cap=8)
  cap:  8 ──→ 16 ──→ 32 ──→ 64
  ┌─────┬────┬────┬────┬────┬─────── ─ ─┬────┐
  │ [0] │ .. │ .. │ .. │ .. │  ...     │[63]│  cap=64, 50 items
  └─────┴────┴────┴────┴────┴─────── ─ ─┴────┘
  After 50 adds: size=50, cap=64, load=0.781

  Phase 2: SHRINK (delete 47 items)
  ┌─────────────┬───────┬────────┬────────┬──────────────────┐
  │ Deletes     │ size  │ cap    │  α     │ Action           │
  ├─────────────┼───────┼────────┼────────┼──────────────────┤
  │ del 0..42   │   7   │   64   │ 0.109  │ SHRINK 64→32     │
  │ del 43..44  │   5   │   32   │ 0.156  │                  │
  │ del 45      │   4   │   32   │ 0.125  │                  │
  │ del 46      │   3   │   32   │ 0.094  │ SHRINK 32→16     │
  └─────────────┴───────┴────────┴────────┴──────────────────┘

  Cap over time:
   64 │─────────────────────────┐
   32 │                         └────────┐
   16 │                                  └────
    8 │
      ├──────────────┬───────────┬─────────┬───→
      add 50       del 43    del 46   del 47

  After 47 deletes: size=3, cap=16, load=0.188
```

---

## Example 7: Resize with Open Addressing (Linear Probing)

```go
package main

import "fmt"

const empty = -1
const deleted = -2

type OAMap struct {
    keys   []int
    values []int
    cap    int
    size   int
}

func NewOAMap(cap int) *OAMap {
    m := &OAMap{
        keys:   make([]int, cap),
        values: make([]int, cap),
        cap:    cap,
    }
    for i := range m.keys {
        m.keys[i] = empty
    }
    return m
}

func (m *OAMap) resize() {
    oldKeys := m.keys
    oldValues := m.values
    oldCap := m.cap

    m.cap *= 2
    m.keys = make([]int, m.cap)
    m.values = make([]int, m.cap)
    m.size = 0
    for i := range m.keys {
        m.keys[i] = empty
    }

    fmt.Printf("  Resize %d → %d\n", oldCap, m.cap)

    for i := 0; i < oldCap; i++ {
        if oldKeys[i] != empty && oldKeys[i] != deleted {
            m.Put(oldKeys[i], oldValues[i])
        }
    }
}

func (m *OAMap) Put(key, value int) {
    if float64(m.size+1)/float64(m.cap) > 0.5 { // open addressing needs lower threshold
        m.resize()
    }

    idx := key % m.cap
    if idx < 0 {
        idx += m.cap
    }

    for m.keys[idx] != empty && m.keys[idx] != deleted && m.keys[idx] != key {
        idx = (idx + 1) % m.cap
    }
    if m.keys[idx] != key {
        m.size++
    }
    m.keys[idx] = key
    m.values[idx] = value
}

func (m *OAMap) Get(key int) (int, bool) {
    idx := key % m.cap
    if idx < 0 {
        idx += m.cap
    }
    for m.keys[idx] != empty {
        if m.keys[idx] == key {
            return m.values[idx], true
        }
        idx = (idx + 1) % m.cap
    }
    return 0, false
}

func main() {
    m := NewOAMap(4)
    for i := 0; i < 20; i++ {
        m.Put(i, i*100)
    }

    // Verify
    for i := 0; i < 20; i++ {
        v, ok := m.Get(i)
        fmt.Printf("Get(%d) = %d, found=%v\n", i, v, ok)
    }
}
```

**Textual Figure:**
```
  Resize with Open Addressing / Linear Probing (thresh=0.5)
  ═════════════════════════════════════════════════════════

  cap=4: insert key 0, 1 → α=0.50, next triggers resize
  ┌─────┬─────┬───────┐
  │ idx │ key │ value │    No chaining — one entry per slot!
  ├─────┼─────┼───────┤
  │  0  │  0  │   0   │    key 0 → 0%4 = 0  ✓
  │  1  │  1  │  100  │    key 1 → 1%4 = 1  ✓
  │  2  │  -1 │       │    (empty)
  │  3  │  -1 │       │    (empty)
  └─────┴─────┴───────┘
        │
        ▼  Insert key 2: (2+1)/4 = 0.75 > 0.5 → RESIZE!
  cap=8: rehash existing, then insert key 2
  ┌─────┬─────┬───────┐
  │ idx │ key │ value │    Rehash: new_idx = key % 8
  ├─────┼─────┼───────┤
  │  0  │  0  │   0   │    key 0: 0%8=0
  │  1  │  1  │  100  │    key 1: 1%8=1
  │  2  │  2  │  200  │    key 2: 2%8=2 (new)
  │  3  │  -1 │       │
  │  4  │  -1 │       │    Collision handling with
  │  5  │  -1 │       │    linear probing:
  │  6  │  -1 │       │    if slot taken, try idx+1
  │  7  │  -1 │       │
  └─────┴─────┴───────┘

  Resize chain: 4 → 8 → 16 → 32 → 64
  Final: 20 keys in cap=64, α = 20/64 = 0.31

  Why lower threshold (0.5) for open addressing?
  ┌────────────────────────────────────────────────┐
  │  Chaining: overflow into lists (tolerates α>1) │
  │  Open Addr: must find an empty slot!           │
  │  At α=0.9: avg ~5.5 probes (linear)            │
  │  At α=0.5: avg ~1.5 probes (fast!)             │
  └────────────────────────────────────────────────┘
```

---

## Example 8: Measuring Resize Impact on Latency

```go
package main

import (
    "fmt"
    "time"
)

type LatencyMap struct {
    buckets [][]int
    cap     int
    size    int
}

func NewLatencyMap(cap int) *LatencyMap {
    return &LatencyMap{buckets: make([][]int, cap), cap: cap}
}

func (m *LatencyMap) put(k int) time.Duration {
    start := time.Now()

    if float64(m.size+1)/float64(m.cap) > 0.75 {
        old := m.buckets
        m.cap *= 2
        m.buckets = make([][]int, m.cap)
        m.size = 0
        for _, b := range old {
            for _, v := range b {
                idx := v % m.cap
                if idx < 0 {
                    idx += m.cap
                }
                m.buckets[idx] = append(m.buckets[idx], v)
                m.size++
            }
        }
    }

    idx := k % m.cap
    if idx < 0 {
        idx += m.cap
    }
    m.buckets[idx] = append(m.buckets[idx], k)
    m.size++

    return time.Since(start)
}

func main() {
    m := NewLatencyMap(4)

    fmt.Printf("%-8s %-8s %-12s %s\n", "Insert#", "Size", "Latency", "Note")
    fmt.Println("------------------------------------------")

    for i := 0; i < 100000; i++ {
        d := m.put(i)
        if d > time.Microsecond*50 || (i < 50) || i%10000 == 0 {
            note := ""
            if d > time.Microsecond*50 {
                note = "RESIZE SPIKE"
            }
            fmt.Printf("%-8d %-8d %-12v %s\n", i, m.size, d, note)
        }
    }
}
```

**Textual Figure:**
```
  Measuring Resize Impact on Latency (100K inserts, start cap=4)
  ════════════════════════════════════════════════════════

  Latency timeline (not to scale):

  latency
    │
  5ms│                                         │ RESIZE SPIKE
    │                                         │ (rehash 49K items)
  2ms│                              │           
    │                              │ SPIKE
  1ms│                   │          (rehash 24K)
    │                   │ SPIKE
 50µs│       │   │        (rehash 12K)
    │       │   │ SPIKE
  1µs│─────────────────────────────────────  normal
    ├────┬────┬────┬────┬────┬────┬────┬─────→ insert#
    0   6   12    25   50        100K
         ↑    ↑     ↑    ↑          ↑
       cap8 cap16 cap32 cap64    cap131K

  ┌──────────┬────────────┬────────────────────────┐
  │ Insert#  │ Rehash Cnt │ Latency Impact           │
  ├──────────┼────────────┼────────────────────────┤
  │ Normal   │     0      │ ~1 µs  ░               │
  │ Resize@6 │     6      │ ~5 µs  ▓               │
  │ Resize@K │   ~50K     │ ~5 ms  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │
  └──────────┴────────────┴────────────────────────┘
  Spikes grow geometrically, but occur exponentially less often!
```

---

## Example 9: Two-Level Resize (Smooth Growth)

```go
package main

import "fmt"

// Instead of doubling all at once, split into two halves
// to spread rehash cost

type SmoothMap struct {
    primary   [][]string
    overflow  [][]string
    pCap      int
    oCap      int
    size      int
    migrated  int
}

func NewSmoothMap(cap int) *SmoothMap {
    return &SmoothMap{
        primary: make([][]string, cap),
        pCap:    cap,
    }
}

func hashStr(s string, cap int) int {
    h := 0
    for _, c := range s {
        h = h*31 + int(c)
    }
    if h < 0 {
        h = -h
    }
    return h % cap
}

func (m *SmoothMap) Put(key string) {
    lf := float64(m.size+1) / float64(m.pCap)

    if m.overflow != nil {
        // Migrate one bucket from overflow to primary
        if m.migrated < m.oCap {
            for _, k := range m.overflow[m.migrated] {
                idx := hashStr(k, m.pCap)
                m.primary[idx] = append(m.primary[idx], k)
            }
            m.overflow[m.migrated] = nil
            m.migrated++
            if m.migrated >= m.oCap {
                m.overflow = nil
                fmt.Println("  Migration complete!")
            }
        }
    } else if lf > 0.75 {
        // Start new resize
        m.overflow = m.primary
        m.oCap = m.pCap
        m.pCap *= 2
        m.primary = make([][]string, m.pCap)
        m.migrated = 0
        m.size = 0
        fmt.Printf("  Start resize: %d → %d\n", m.oCap, m.pCap)
    }

    idx := hashStr(key, m.pCap)
    m.primary[idx] = append(m.primary[idx], key)
    m.size++
}

func main() {
    m := NewSmoothMap(4)
    for i := 0; i < 30; i++ {
        k := fmt.Sprintf("k%d", i)
        m.Put(k)
        fmt.Printf("Put %s: size=%d primaryCap=%d\n", k, m.size, m.pCap)
    }
}
```

**Textual Figure:**
```
  Two-Level Resize — Smooth Growth (start cap=4)
  ═══════════════════════════════════════════════════════════

  Phase 1: Insert k0..k2 into primary (cap=4)
  primary (cap=4)              overflow (nil)
  ┌─────┬───────┐
  │ [0] │ k0    │
  │ [1] │ k1    │                (not yet)
  │ [2] │       │
  │ [3] │ k2    │
  └─────┴───────┘
        │
        ▼  Put k3: lf > 0.75, start resize!

  Phase 2: Resize triggered
  ─ old primary becomes overflow
  ─ new empty primary (cap=8) created
  ─ each subsequent Put migrates 1 bucket from overflow

  overflow (cap=4)             primary (cap=8)
  ┌─────┬───────┐            ┌─────┬───────┐
  │ [0] │  k0   │ ─migrate→ │ [0] │ k0    │
  │ [1] │  k1   │            │ [1] │       │
  │ [2] │       │            │ [2] │       │
  │ [3] │  k2   │            │ [3] │ k3    │ ← new insert
  └─────┴───────┘            │ [4] │       │
    migrated=1                  │ [5] │       │
                                │ [6] │       │
                                │ [7] │       │
                                └─────┴───────┘
  Phase 3: Migration progresses (1 bucket/Put)
  ┌────────┬───────────────┬────────────────────────────┐
  │  Put   │ Migrates      │ Status                      │
  ├────────┼───────────────┼────────────────────────────┤
  │  k4    │ overflow[1]   │ migrated=2                  │
  │  k5    │ overflow[2]   │ migrated=3                  │
  │  k6    │ overflow[3]   │ migrated=4 → Complete!       │
  └────────┴───────────────┴────────────────────────────┘

  Advantage: Cost of resize spread across multiple insertions
```

---

## Example 10: Go Map Growth Behavior Observation

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    m := make(map[int]struct{})
    var prevAlloc uint64

    runtime.GC()
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    prevAlloc = ms.HeapAlloc

    fmt.Printf("%-10s %-15s %-15s\n", "Size", "HeapAlloc(KB)", "Delta(KB)")
    fmt.Println("------------------------------------------")

    for i := 0; i < 1000000; i++ {
        m[i] = struct{}{}

        if i > 0 && (i&(i-1)) == 0 { // check at powers of 2
            runtime.GC()
            runtime.ReadMemStats(&ms)
            delta := int64(ms.HeapAlloc) - int64(prevAlloc)
            fmt.Printf("%-10d %-15d %-+15d\n",
                len(m), ms.HeapAlloc/1024, delta/1024)
            prevAlloc = ms.HeapAlloc
        }
    }
}
```

**Textual Figure:**
```
  Go Map Growth Behavior (insert 1M entries, measure at powers of 2)
  ═════════════════════════════════════════════════════════

  Go runtime.MemStats snapshots at size = powers of 2:
  ┌──────────┬───────────────┬───────────────┬──────────────────┐
  │   Size   │ HeapAlloc(KB) │  Delta(KB)    │   Observation     │
  ├──────────┼───────────────┼───────────────┼──────────────────┤
  │      2   │       ~1      │     +1        │                  │
  │      8   │       ~2      │     +1        │                  │
  │     64   │       ~8      │     +6        │ small growth     │
  │    512   │      ~40      │    +32        │                  │
  │   4096   │     ~300      │   +260        │ bigger jumps     │
  │  32768   │    ~2400      │  +2100        │                  │
  │ 262144   │   ~19000      │ +16600        │ large allocs     │
  │ 524288   │   ~38000      │ +19000        │ doubling visible │
  └──────────┴───────────────┴───────────────┴──────────────────┘

  HeapAlloc growth (log scale):
  KB
  40000 │                                         ╱
  20000 │                                     ╱───
  10000 │                                 ╱───
   5000 │                             ╱───
   2500 │                         ╱───
    300 │                   ..····
     40 │             ..···
      8 │        ..··
      1 │ ..···
        ├───┬───┬───┬───┬───┬───┬───┬───┬───┬──→ size
        2   8  64  512 4K  32K 262K 524K 1M

  Go map internally uses B (bucket power) and grows
  by doubling number of buckets (2^B → 2^(B+1))
  with incremental evacuation to avoid latency spikes.
```

---

## Resize Strategy Comparison

| Strategy | Pros | Cons |
|----------|------|------|
| Double (2x) | Simple, amortized O(1) | Large spikes, 2x memory |
| 1.5x growth | Less memory waste | More frequent resizes |
| Incremental | No latency spikes | Complex implementation |
| Prime sizes | Better distribution | Hard to compute next size |

## Key Takeaways

1. **Resize = allocate new array + rehash all entries** → O(n) per resize
2. **Geometric growth** (2x) gives **amortized O(1)** insertions
3. **Incremental rehash** (Redis-style) avoids latency spikes
4. **Open addressing** needs earlier resize (α < 0.5–0.7) than chaining
5. **Shrink** on delete to avoid wasting memory
6. **Power-of-2** sizes allow bitwise modulo but risk poor distribution
7. Go's built-in map handles resizing transparently

> **Next up:** Hash Sets →
