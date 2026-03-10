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

**Textual Figure: Simple Modular Hash**

```
Input: keys = [15, 25, 35, 42, 99, 100, 3, 73]   tableSize = 10
Formula: hash(key) = key % tableSize

┌───────┬───────────────┬────────┐
│  Key  │  key % 10     │ Bucket │
├───────┼───────────────┼────────┤
│   15  │  15 % 10 = 5  │   5    │
│   25  │  25 % 10 = 5  │   5    │
│   35  │  35 % 10 = 5  │   5    │
│   42  │  42 % 10 = 2  │   2    │
│   99  │  99 % 10 = 9  │   9    │
│  100  │ 100 % 10 = 0  │   0    │
│    3  │   3 % 10 = 3  │   3    │
│   73  │  73 % 10 = 3  │   3    │
└───────┴───────────────┴────────┘

Bucket Layout (tableSize = 10):
  [0] ──→ [100]
  [1] ──→ (empty)
  [2] ──→ [42]
  [3] ──→ [3] ──→ [73]                 ← collision
  [4] ──→ (empty)
  [5] ──→ [15] ──→ [25] ──→ [35]       ← collision cluster!
  [6] ──→ (empty)
  [7] ──→ (empty)
  [8] ──→ (empty)
  [9] ──→ [99]

Result: 15, 25, 35 all map to bucket 5 (poor distribution for multiples of 5)
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

**Textual Figure: String Hash Function**

```
Input: words = ["hello", "world", "go", ...]   tableSize = 16
Formula: hash = (hash × 31 + charCode) % tableSize

Step-by-step trace for "go":
┌──────┬──────┬───────┬────────────────────────────┬──────┐
│ Step │ Char │ ASCII │ Computation                │ hash │
├──────┼──────┼───────┼────────────────────────────┼──────┤
│ init │  —   │   —   │ hash = 0                   │   0  │
│  1   │ 'g'  │  103  │ (0 × 31 + 103) % 16 = 7   │   7  │
│  2   │ 'o'  │  111  │ (7 × 31 + 111) % 16 = 8   │   8  │
└──────┴──────┴───────┴────────────────────────────┴──────┘
                                            result ──→ 8

Step-by-step trace for "hello":
┌──────┬──────┬───────┬────────────────────────────┬──────┐
│ Step │ Char │ ASCII │ Computation                │ hash │
├──────┼──────┼───────┼────────────────────────────┼──────┤
│ init │  —   │   —   │ hash = 0                   │   0  │
│  1   │ 'h'  │  104  │ (0 × 31 + 104) % 16 = 8   │   8  │
│  2   │ 'e'  │  101  │ (8 × 31 + 101) % 16 = 13  │  13  │
│  3   │ 'l'  │  108  │ (13 × 31 + 108) % 16 = 15 │  15  │
│  4   │ 'l'  │  108  │ (15 × 31 + 108) % 16 = 13 │  13  │
│  5   │ 'o'  │  111  │ (13 × 31 + 111) % 16 = 2  │   2  │
└──────┴──────┴───────┴────────────────────────────┴──────┘
                                            result ──→ 2

Bucket Mapping (tableSize = 16):
  "go"    ──→ Bucket  8
  "hello" ──→ Bucket  2
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

**Textual Figure: DJB2 Hash (hash × 33 + c)**

```
Formula: hash = 5381 (seed); for each char c: hash = hash × 33 + c
         (implemented as: (hash << 5) + hash + c)

Trace for "abc" (uint32 arithmetic):
┌──────┬──────┬─────┬───────────────────────────────┬────────────┐
│ Step │ Char │ Val │ Computation                   │    hash    │
├──────┼──────┼─────┼───────────────────────────────┼────────────┤
│ init │  —   │  —  │ seed                          │       5381 │
│  1   │ 'a'  │  97 │ 5381 × 33 + 97                │     177670 │
│  2   │ 'b'  │  98 │ 177670 × 33 + 98              │    5863208 │
│  3   │ 'c'  │  99 │ 5863208 × 33 + 99             │  193485963 │
└──────┴──────┴─────┴───────────────────────────────┴────────────┘

Final: djb2("abc") = 193485963  →  bucket = 193485963 % 16 = 11

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────────┐
  │hash=5381 │──→ │ ×33 + 'a'│──→ │ ×33 + 'b'│──→ │ ×33 + 'c' │
  │  (seed)  │    │ = 177670 │    │= 5863208 │    │=193485963 │
  └──────────┘    └──────────┘    └──────────┘    └─────┬─────┘
                                                        │
                                                   % 16 = 11
                                                        ↓
                                                  ┌──────────┐
                                                  │ Bucket 11│
                                                  └──────────┘

Key insight: (hash << 5) + hash = hash × 32 + hash = hash × 33
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

**Textual Figure: FNV-1a Hash Algorithm**

```
Constants: offset = 2166136261 (0x811C9DC5)
           prime  = 16777619   (0x01000193)

Algorithm flow (per byte):
  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
  │ hash = offset  │──→ │ hash XOR byte  │──→ │ hash × prime   │──→ next byte
  │ 0x811C9DC5     │    │                │    │                │
  └────────────────┘    └────────────────┘    └────────────────┘
                              ↑                       │
                              └───────────────────────┘
                                  repeat for each byte

Trace for "hash" → bytes: ['h','a','s','h'] = [104, 97, 115, 104]
┌──────┬──────┬──────┬───────────────────────────────────────────┐
│ Step │ Byte │ Hex  │ Operations (uint32 with overflow)         │
├──────┼──────┼──────┼───────────────────────────────────────────┤
│  0   │  —   │  —   │ hash = 0x811C9DC5 (offset basis)         │
│  1   │ 'h'  │ 0x68 │ hash ^= 0x68  →  hash *= 16777619       │
│  2   │ 'a'  │ 0x61 │ hash ^= 0x61  →  hash *= 16777619       │
│  3   │ 's'  │ 0x73 │ hash ^= 0x73  →  hash *= 16777619       │
│  4   │ 'h'  │ 0x68 │ hash ^= 0x68  →  hash *= 16777619       │
└──────┴──────┴──────┴───────────────────────────────────────────┘
                          Final: bucket = hash % 16

FNV-1a vs FNV-1:
  FNV-1a: hash = (hash XOR byte) × prime   ← XOR first (better avalanche)
  FNV-1:  hash = (hash × prime) XOR byte   ← multiply first
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

**Textual Figure: Go Built-in Map (Frequency Counting)**

```
Input: words = ["apple", "banana", "cherry", "apple", "banana", "apple"]

Processing each word (m[w]++):
┌──────┬──────────┬──────────────────────────────────────────────┐
│ Step │   Word   │ Map State                                    │
├──────┼──────────┼──────────────────────────────────────────────┤
│  1   │ "apple"  │ {"apple": 1}                                 │
│  2   │ "banana" │ {"apple": 1, "banana": 1}                    │
│  3   │ "cherry" │ {"apple": 1, "banana": 1, "cherry": 1}       │
│  4   │ "apple"  │ {"apple": 2, "banana": 1, "cherry": 1}       │
│  5   │ "banana" │ {"apple": 2, "banana": 2, "cherry": 1}       │
│  6   │ "apple"  │ {"apple": 3, "banana": 2, "cherry": 1}       │
└──────┴──────────┴──────────────────────────────────────────────┘

Under the hood (Go runtime):
  ┌───────────────────────────────────────────────────┐
  │              Go Map Internals                     │
  ├───────────────────────────────────────────────────┤
  │  "apple"  ──hash()──→ bucket[i] ──→ count: 3     │
  │  "banana" ──hash()──→ bucket[j] ──→ count: 2     │
  │  "cherry" ──hash()──→ bucket[k] ──→ count: 1     │
  ├───────────────────────────────────────────────────┤
  │  • Hash function chosen per key type              │
  │  • Automatic collision resolution (chaining)      │
  │  • Dynamic resizing when load factor > 6.5        │
  └───────────────────────────────────────────────────┘

Final output: apple: 3, banana: 2, cherry: 1
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

**Textual Figure: Polynomial Rolling Hash**

```
Input: s = "abcabcabc"   base = 31   mod = 1_000_000_007
Char values: a=1, b=2, c=3  (mapped as char - 'a' + 1)

Building prefix[] array (prefix[i+1] = prefix[i] × 31 + val):
┌───┬──────┬───────┬─────────────────────────────────────┐
│ i │ char │ value │ prefix[i+1]                         │
├───┼──────┼───────┼─────────────────────────────────────┤
│ 0 │ 'a'  │   1   │ (0 × 31 + 1) = 1                    │
│ 1 │ 'b'  │   2   │ (1 × 31 + 2) = 33                   │
│ 2 │ 'c'  │   3   │ (33 × 31 + 3) = 1026                │
│ 3 │ 'a'  │   1   │ (1026 × 31 + 1) = 31807             │
│ 4 │ 'b'  │   2   │ (31807 × 31 + 2) = 986019           │
│ 5 │ 'c'  │   3   │ (986019 × 31 + 3) = 30566592        │
└───┴──────┴───────┴─────────────────────────────────────┘

power[]: [1, 31, 961, 29791, 923521, ...]

Substring hash formula: Hash(l, r) = prefix[r+1] - prefix[l] × power[r-l+1]

Comparing substrings:
  s[0:3] = "abc" ──→ Hash(0,2) = prefix[3] - prefix[0]×power[3]
                                = 1026 - 0 = 1026
  s[3:6] = "abc" ──→ Hash(3,5) = prefix[6] - prefix[3]×power[3]
                                = 30566592 - 31807×29791 = 1026  ✓
  s[6:9] = "abc" ──→ Hash(6,8) = same computation = 1026         ✓

  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │ s[0:3]="abc" │   │ s[3:6]="abc" │   │ s[6:9]="abc" │
  │ hash = 1026 │   │ hash = 1026 │   │ hash = 1026 │
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                 │
         └────── ALL EQUAL ──────────────────┘
              → O(1) substring comparison!
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

**Textual Figure: Hash Function Quality — Distribution Test**

```
Input: 100 keys ("key_0" .. "key_99"),  10 buckets
Metric: Standard deviation of bucket counts (lower = better)

┌───────────────────────────────────────────────────────┐
│  Simple Sum Hash — Poor Distribution                  │
├────────┬──────────────────────────────────────────────┤
│Bucket 0│ ████████████████████ (20)                    │
│Bucket 1│ ███████████ (11)                             │
│Bucket 2│ ███████████ (11)                             │
│Bucket 3│ ███████████ (11)                             │
│Bucket 4│ ██████████ (10)                              │
│Bucket 5│ █████████ (9)                                │
│Bucket 6│ ████████ (8)                                 │
│Bucket 7│ ███████ (7)                                  │
│Bucket 8│ ███████ (7)                                  │
│Bucket 9│ ██████ (6)                                   │
├────────┴──────────────────────────────────────────────┤
│  StdDev: HIGH → uneven spread, more collisions        │
└───────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│  DJB2 Hash — Good Distribution                        │
├────────┬──────────────────────────────────────────────┤
│Bucket 0│ ██████████ (10)                              │
│Bucket 1│ ███████████ (11)                             │
│Bucket 2│ █████████ (9)                                │
│Bucket 3│ ██████████ (10)                              │
│Bucket 4│ ██████████ (10)                              │
│Bucket 5│ ██████████ (10)                              │
│Bucket 6│ ██████████ (10)                              │
│Bucket 7│ ██████████ (10)                              │
│Bucket 8│ ██████████ (10)                              │
│Bucket 9│ ██████████ (10)                              │
├────────┴──────────────────────────────────────────────┤
│  StdDev: LOW → near-uniform spread, fewer collisions  │
└───────────────────────────────────────────────────────┘

Conclusion: Lower standard deviation = better hash function
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

**Textual Figure: Perfect Hash Function**

```
Input: keywords = ["if", "for", "func", "var", "return", "go", "unknown"]

Perfect Hash — Each key maps to a unique slot, zero collisions:
┌──────────┬───────────────────┬───────┐
│ Keyword  │ perfectHash(key)  │ Index │
├──────────┼───────────────────┼───────┤
│ "if"     │         0         │   0   │
│ "for"    │         1         │   1   │
│ "func"   │         2         │   2   │
│ "var"    │         3         │   3   │
│ "return" │         4         │   4   │
│ "go"     │         5         │   5   │
│ "unknown"│        -1         │  N/A  │
└──────────┴───────────────────┴───────┘

Slot assignments (1-to-1 mapping):
  ┌──────┬──────┬──────┬──────┬────────┬──────┐
  │  0   │  1   │  2   │  3   │   4    │  5   │
  │"if"  │"for" │"func"│"var" │"return"│"go"  │
  └──────┴──────┴──────┴──────┴────────┴──────┘
    ↑       ↑       ↑      ↑       ↑       ↑
    │       │       │      │       │       │
   "if"   "for"  "func"  "var" "return"  "go"

Properties:
  • Every known key → unique slot → zero collisions
  • Only works when all keys known at compile time
  • O(1) guaranteed lookup, no probing/chaining needed
  • Unknown keys return -1
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

**Textual Figure: Multiplicative Hash (Knuth's Golden Ratio Method)**

```
Formula: hash(key) = floor(tableSize × frac(key × A))
  where A = (√5 - 1) / 2 ≈ 0.6180339887  (golden ratio conjugate)
  tableSize = 16

Trace for sequential keys:
┌─────┬───────────────┬──────────────┬───────────────┬────────┐
│ Key │ key × A       │ frac(key×A)  │ 16 × frac     │ Bucket │
├─────┼───────────────┼──────────────┼───────────────┼────────┤
│   0 │ 0.000         │ 0.000        │  0.000        │    0   │
│   1 │ 0.618         │ 0.618        │  9.889        │    9   │
│   2 │ 1.236         │ 0.236        │  3.777        │    3   │
│   3 │ 1.854         │ 0.854        │ 13.666        │   13   │
│   4 │ 2.472         │ 0.472        │  7.554        │    7   │
│   5 │ 3.090         │ 0.090        │  1.443        │    1   │
│   6 │ 3.708         │ 0.708        │ 11.332        │   11   │
│   7 │ 4.326         │ 0.326        │  5.221        │    5   │
│   8 │ 4.944         │ 0.944        │ 15.109        │   15   │
│   9 │ 5.563         │ 0.563        │  8.998        │    8   │
└─────┴───────────────┴──────────────┴───────────────┴────────┘

Bucket distribution for keys 0-9:
  [ 0] █  [ 1] █  [ 2]    [ 3] █  [ 4]    [ 5] █  [ 6]    [ 7] █
  [ 8] █  [ 9] █  [10]    [11] █  [12]    [13] █  [14]    [15] █

  → Sequential keys are scattered across buckets!
  → Golden ratio ensures maximum spacing between consecutive hashes
  → No clustering, even for worst-case sequential input
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

**Textual Figure: Custom Hash Map with Configurable Hash Function**

```
Hash Map: capacity = 16 buckets, hashFunc = djb2

Put Operations:
  Put("apple", 1)  → djb2("apple")  → |hash| % 16 → bucket[i] → store
  Put("banana", 2) → djb2("banana") → |hash| % 16 → bucket[j] → store
  Put("cherry", 3) → djb2("cherry") → |hash| % 16 → bucket[k] → store

Bucket Layout (separate chaining with slices):
  ┌─────────┬──────────────────────────────────────┐
  │  Bucket  │ Chain ([]Entry)                      │
  ├─────────┼──────────────────────────────────────┤
  │  [0]    │ → nil                                │
  │  [1]    │ → nil                                │
  │   ...   │ → nil                                │
  │  [i]    │ → {"apple", 1}                       │
  │   ...   │ → nil                                │
  │  [j]    │ → {"banana", 2}                      │
  │   ...   │ → nil                                │
  │  [k]    │ → {"cherry", 3}                      │
  │   ...   │ → nil                                │
  │  [15]   │ → nil                                │
  └─────────┴──────────────────────────────────────┘

Get Operations:
  ┌────────────────────┬──────────────┬──────────┬────────────┐
  │ Get(key)           │ Bucket       │ Scan     │ Result     │
  ├────────────────────┼──────────────┼──────────┼────────────┤
  │ Get("apple")       │ bucket[i]    │ match!   │ {1, true}  │
  │ Get("banana")      │ bucket[j]    │ match!   │ {2, true}  │
  │ Get("cherry")      │ bucket[k]    │ match!   │ {3, true}  │
  │ Get("grape")       │ bucket[?]    │ no match │ {0, false} │
  └────────────────────┴──────────────┴──────────┴────────────┘

Collision handling (if keys collide to same bucket):
  bucket[i] → {key₁, val₁} → {key₂, val₂} → ...
  Linear scan through slice to find matching key
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
