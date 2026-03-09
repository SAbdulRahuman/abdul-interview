# Phase 4: Hash Tables — Hash Functions

## What is a Hash Function?

A **hash function** maps data of arbitrary size to a fixed-size value (the hash). In hash tables, hash functions determine the **bucket index** where key-value pairs are stored. A good hash function:
- **Deterministic** — same input always produces same output
- **Uniform distribution** — minimizes clustering/collisions
- **Fast to compute** — O(1) for fixed-size keys

---

## Example 1: Simple Modular Hash

```go
package main

import "fmt"

func modHash(key int, tableSize int) int {
    return key % tableSize
}

func main() {
    tableSize := 10
    keys := []int{15, 25, 35, 42, 99, 100, 3, 73}
    for _, k := range keys {
        fmt.Printf("hash(%d) = %d\n", k, modHash(k, tableSize))
    }
    // Notice: 15, 25, 35 all hash to 5 → collision!
}
```

---

## Example 2: String Hash Function

```go
package main

import "fmt"

func stringHash(s string, tableSize int) int {
    hash := 0
    for _, c := range s {
        hash = (hash*31 + int(c)) % tableSize
    }
    return hash
}

func main() {
    tableSize := 16
    words := []string{"hello", "world", "go", "hash", "table", "function"}
    for _, w := range words {
        fmt.Printf("hash(%q) = %d\n", w, stringHash(w, tableSize))
    }
}
```

---

## Example 3: DJB2 Hash (Dan Bernstein)

```go
package main

import "fmt"

func djb2(s string) uint32 {
    hash := uint32(5381)
    for _, c := range s {
        hash = ((hash << 5) + hash) + uint32(c) // hash * 33 + c
    }
    return hash
}

func main() {
    words := []string{"hello", "world", "abc", "cba", "Go", "go"}
    for _, w := range words {
        fmt.Printf("djb2(%q) = %d (bucket: %d)\n", w, djb2(w), djb2(w)%16)
    }
}
```

---

## Example 4: FNV-1a Hash

```go
package main

import "fmt"

func fnv1a(data []byte) uint32 {
    const (
        offset = uint32(2166136261)
        prime  = uint32(16777619)
    )
    hash := offset
    for _, b := range data {
        hash ^= uint32(b)
        hash *= prime
    }
    return hash
}

func main() {
    strings := []string{"hello", "world", "hash", "function"}
    for _, s := range strings {
        h := fnv1a([]byte(s))
        fmt.Printf("fnv1a(%q) = %d (bucket: %d)\n", s, h, h%16)
    }
}
```

---

## Example 5: Using Go's Built-in Map (hash under the hood)

```go
package main

import "fmt"

func main() {
    // Go's map uses a built-in hash function for each key type
    m := make(map[string]int)
    words := []string{"apple", "banana", "cherry", "apple", "banana", "apple"}

    for _, w := range words {
        m[w]++
    }

    for k, v := range m {
        fmt.Printf("%s: %d\n", k, v)
    }
    // Go automatically handles hashing, collision resolution, and resizing
}
```

---

## Example 6: Polynomial Rolling Hash for Substrings

```go
package main

import "fmt"

const (
    base = 31
    mod  = 1_000_000_007
)

type PolyHash struct {
    prefix []int64
    power  []int64
}

func NewPolyHash(s string) *PolyHash {
    n := len(s)
    ph := &PolyHash{
        prefix: make([]int64, n+1),
        power:  make([]int64, n+1),
    }
    ph.power[0] = 1
    for i := 0; i < n; i++ {
        ph.prefix[i+1] = (ph.prefix[i]*base + int64(s[i]-'a'+1)) % mod
        ph.power[i+1] = (ph.power[i] * base) % mod
    }
    return ph
}

func (ph *PolyHash) Hash(l, r int) int64 {
    return (ph.prefix[r+1] - ph.prefix[l]*ph.power[r-l+1]%mod + mod*2) % mod
}

func main() {
    s := "abcabcabc"
    ph := NewPolyHash(s)

    fmt.Printf("hash(%q) = %d\n", s[0:3], ph.Hash(0, 2))
    fmt.Printf("hash(%q) = %d\n", s[3:6], ph.Hash(3, 5))
    fmt.Printf("hash(%q) = %d\n", s[6:9], ph.Hash(6, 8))
    // All three should be equal since they're all "abc"
}
```

---

## Example 7: Hash Function Quality — Distribution Test

```go
package main

import (
    "fmt"
    "math"
)

func testDistribution(hashFunc func(string) uint32, keys []string, buckets int) {
    counts := make([]int, buckets)
    for _, k := range keys {
        counts[hashFunc(k)%uint32(buckets)]++
    }

    // Calculate standard deviation
    mean := float64(len(keys)) / float64(buckets)
    variance := 0.0
    for _, c := range counts {
        diff := float64(c) - mean
        variance += diff * diff
    }
    variance /= float64(buckets)
    stddev := math.Sqrt(variance)

    fmt.Printf("Keys: %d, Buckets: %d\n", len(keys), buckets)
    fmt.Printf("Mean: %.2f, StdDev: %.2f\n", mean, stddev)
    fmt.Printf("Distribution: %v\n\n", counts)
}

func simpleHash(s string) uint32 {
    h := uint32(0)
    for _, c := range s {
        h += uint32(c)
    }
    return h
}

func djb2(s string) uint32 {
    h := uint32(5381)
    for _, c := range s {
        h = ((h << 5) + h) + uint32(c)
    }
    return h
}

func main() {
    keys := []string{}
    for i := 0; i < 100; i++ {
        keys = append(keys, fmt.Sprintf("key_%d", i))
    }

    fmt.Println("=== Simple Sum Hash ===")
    testDistribution(simpleHash, keys, 10)

    fmt.Println("=== DJB2 Hash ===")
    testDistribution(djb2, keys, 10)
    // DJB2 typically has much better distribution
}
```

---

## Example 8: Perfect Hash Function (Compile-Time Known Keys)

```go
package main

import "fmt"

// For a known set of keys, create a minimal perfect hash
func perfectHash(keyword string) int {
    // Pre-computed for {"if", "for", "func", "var", "return", "go"}
    table := map[string]int{
        "if":     0,
        "for":    1,
        "func":   2,
        "var":    3,
        "return": 4,
        "go":     5,
    }
    if idx, ok := table[keyword]; ok {
        return idx
    }
    return -1
}

func main() {
    keywords := []string{"if", "for", "func", "var", "return", "go", "unknown"}
    for _, k := range keywords {
        fmt.Printf("perfectHash(%q) = %d\n", k, perfectHash(k))
    }
}
```

---

## Example 9: Multiplicative Hash Function

```go
package main

import (
    "fmt"
    "math"
)

func multiplicativeHash(key int, tableSize int) int {
    // Knuth's multiplicative method
    // A = (√5 - 1) / 2 ≈ 0.6180339887
    A := (math.Sqrt(5) - 1) / 2
    frac := float64(key) * A
    frac = frac - math.Floor(frac) // fractional part
    return int(math.Floor(float64(tableSize) * frac))
}

func main() {
    tableSize := 16
    for key := 0; key < 20; key++ {
        fmt.Printf("hash(%2d) = %d\n", key, multiplicativeHash(key, tableSize))
    }
    // Good distribution even for sequential keys
}
```

---

## Example 10: Custom Hash Map with Configurable Hash Function

```go
package main

import "fmt"

type HashFunc func(string) int

type Entry struct {
    Key   string
    Value int
}

type HashMap struct {
    buckets  [][]Entry
    size     int
    hashFunc HashFunc
}

func NewHashMap(capacity int, hf HashFunc) *HashMap {
    return &HashMap{
        buckets:  make([][]Entry, capacity),
        hashFunc: hf,
    }
}

func (hm *HashMap) bucket(key string) int {
    h := hm.hashFunc(key)
    if h < 0 {
        h = -h
    }
    return h % len(hm.buckets)
}

func (hm *HashMap) Put(key string, value int) {
    idx := hm.bucket(key)
    for i, e := range hm.buckets[idx] {
        if e.Key == key {
            hm.buckets[idx][i].Value = value
            return
        }
    }
    hm.buckets[idx] = append(hm.buckets[idx], Entry{key, value})
    hm.size++
}

func (hm *HashMap) Get(key string) (int, bool) {
    idx := hm.bucket(key)
    for _, e := range hm.buckets[idx] {
        if e.Key == key {
            return e.Value, true
        }
    }
    return 0, false
}

func main() {
    djb2 := func(s string) int {
        h := 5381
        for _, c := range s {
            h = ((h << 5) + h) + int(c)
        }
        return h
    }

    hm := NewHashMap(16, djb2)

    hm.Put("apple", 1)
    hm.Put("banana", 2)
    hm.Put("cherry", 3)

    for _, key := range []string{"apple", "banana", "cherry", "grape"} {
        v, ok := hm.Get(key)
        fmt.Printf("Get(%q) = %d, found=%v\n", key, v, ok)
    }
}
```

---

## Hash Function Properties Summary

| Property       | Description                          |
|---------------|--------------------------------------|
| Deterministic | Same input → same output always     |
| Uniform       | Even distribution across buckets     |
| Fast          | O(1) for fixed-size keys             |
| Avalanche     | Small input change → big hash change |
| Low collision | Different keys → different hashes    |

## Key Takeaways

1. **Modular hash** (`key % size`) — simple but poor distribution for patterns
2. **Polynomial hash** — good for strings, uses base and prime mod
3. **DJB2 / FNV-1a** — popular general-purpose string hash functions
4. **Multiplicative hash** — Knuth's method with golden ratio
5. **Distribution testing** — measure standard deviation across buckets
6. **Go's map** handles hashing internally — understand what's under the hood

> **Next up:** Collision Resolution →
