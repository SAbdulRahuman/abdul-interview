# Phase 1: Algorithm Complexity — Tradeoffs Between Time and Space

## What is the Time-Space Tradeoff?

The time-space tradeoff is a fundamental concept: **you can often reduce time by using more space, or reduce space by using more time**.

- **Use more memory → faster algorithm** (caching, lookup tables, memoization)
- **Use less memory → slower algorithm** (recomputation, streaming)

The right tradeoff depends on constraints: memory limits, latency requirements, scale.

---

## Example 1: Two Sum — O(n²)/O(1) vs O(n)/O(n)

```go
package main

import "fmt"

// Approach 1: No extra space, but slow — O(n²) time, O(1) space
func twoSumBrute(nums []int, target int) (int, int) {
    for i := 0; i < len(nums); i++ {
        for j := i + 1; j < len(nums); j++ {
            if nums[i]+nums[j] == target {
                return i, j
            }
        }
    }
    return -1, -1
}

// Approach 2: Use hash map for O(1) lookup — O(n) time, O(n) space
func twoSumHash(nums []int, target int) (int, int) {
    seen := make(map[int]int) // extra O(n) space
    for i, num := range nums {
        if j, ok := seen[target-num]; ok {
            return j, i
        }
        seen[num] = i
    }
    return -1, -1
}

func main() {
    nums := []int{2, 7, 11, 15}
    fmt.Println(twoSumBrute(nums, 9)) // 0, 1
    fmt.Println(twoSumHash(nums, 9))  // 0, 1
    // Hash approach: trades O(n) space for O(n²) → O(n) time improvement
}
```

**Textual Figure — Brute Force vs Hash Map:**

```
  Brute Force: Check every pair O(n²)     Hash Map: Single pass O(n)
  ─────────────────────────────────        ─────────────────────────────
  nums = [2, 7, 11, 15]  target=9         nums = [2, 7, 11, 15]  target=9

  i=0: (2,7)✓ found!                      i=0: need 9-2=7 → not in map
  i=0: (2,11) ✗                                 map: {2:0}
  i=0: (2,15) ✗                            i=1: need 9-7=2 → found at 0! ✓
  i=1: (7,11) ✗                                 return (0, 1)
  i=1: (7,15) ✗
  i=2: (11,15) ✗                          Total lookups: 2
  Total comparisons: up to n(n-1)/2

  ┌─────────────────────────────────────────────────────┐
  │  Tradeoff:  O(n) space  ──→  O(n²) time drops to   │
  │             (hash map)       O(n) time              │
  │                                                     │
  │  Time ▲                                             │
  │       │  ·                                          │
  │  n²   │    · Brute                                  │
  │       │      ·                                      │
  │       │        ·                                    │
  │  n    │──────────── Hash ─────                      │
  │       │                                             │
  │       └──────────────────────▶ n                    │
  └─────────────────────────────────────────────────────┘
```

---

## Example 2: Fibonacci — Three Tradeoff Levels

```go
package main

import "fmt"

// Level 1: O(2ⁿ) time, O(n) space (call stack) — no tradeoff
func fibNaive(n int) int {
    if n <= 1 { return n }
    return fibNaive(n-1) + fibNaive(n-2)
}

// Level 2: O(n) time, O(n) space — trade space for time (memoization)
func fibMemo(n int) int {
    memo := make([]int, n+1)
    for i := range memo { memo[i] = -1 }
    return fibHelper(n, memo)
}

func fibHelper(n int, memo []int) int {
    if n <= 1 { return n }
    if memo[n] != -1 { return memo[n] }
    memo[n] = fibHelper(n-1, memo) + fibHelper(n-2, memo)
    return memo[n]
}

// Level 3: O(n) time, O(1) space — optimal tradeoff
func fibOptimal(n int) int {
    if n <= 1 { return n }
    prev, curr := 0, 1
    for i := 2; i <= n; i++ {
        prev, curr = curr, prev+curr
    }
    return curr
}

func main() {
    n := 30
    fmt.Println(fibNaive(n))   // slow!  O(2ⁿ) time, O(n) space
    fmt.Println(fibMemo(n))    // fast   O(n) time, O(n) space
    fmt.Println(fibOptimal(n)) // fast   O(n) time, O(1) space
}
```

| Approach | Time | Space | Strategy |
|----------|------|-------|----------|
| Naive recursion | O(2ⁿ) | O(n) | No tradeoff |
| Memoization | O(n) | O(n) | Trade space for time |
| Iterative | O(n) | O(1) | Optimal |

**Textual Figure — Three Levels of Tradeoff for fib(5):**

```
 Level 1: Naive O(2ⁿ) time, O(n) space — redundant recomputation
 ─────────────────────────────────────────────────────────────
              fib(5)
             /      \
         fib(4)     fib(3)        ← fib(3) computed TWICE
        /    \      /    \
    fib(3) fib(2) fib(2) fib(1)  ← fib(2) computed THREE times
    /   \
 fib(2) fib(1)  ...explosive growth...

 Level 2: Memoization O(n) time, O(n) space — cache results
 ─────────────────────────────────────────────────────────────
  memo[] = [-1, -1, -1, -1, -1, -1]

  fib(5) → fib(4) → fib(3) → fib(2) → fib(1)=1, fib(0)=0
                                        memo[2]=1 ✓
                              memo[3]=2 ✓
                    memo[4]=3 ✓
           memo[5]=5 ✓

  memo[] = [ 0,  1,  1,  2,  3,  5 ]   ← each computed ONCE

 Level 3: Iterative O(n) time, O(1) space — two variables
 ─────────────────────────────────────────────────────────────
  prev=0, curr=1

  i=2: prev=1, curr=0+1=1    [0, 1, 1]
  i=3: prev=1, curr=1+1=2    [0, 1, 1, 2]
  i=4: prev=2, curr=1+2=3    [0, 1, 1, 2, 3]
  i=5: prev=3, curr=2+3=5    [0, 1, 1, 2, 3, 5]  ✓

  Only 2 variables at any time — no array needed!
```

---

## Example 3: Lookup Table vs Computation

```go
package main

import "fmt"

// Approach 1: Compute every time — O(1) space, O(√n) time per call
func isPrimeCompute(n int) bool {
    if n < 2 { return false }
    for i := 2; i*i <= n; i++ {
        if n%i == 0 { return false }
    }
    return true
}

// Approach 2: Pre-build sieve — O(n) space, O(1) time per lookup
func buildSieve(max int) []bool {
    sieve := make([]bool, max+1) // O(n) space
    for i := 2; i <= max; i++ {
        sieve[i] = true
    }
    for i := 2; i*i <= max; i++ {
        if sieve[i] {
            for j := i * i; j <= max; j += i {
                sieve[j] = false
            }
        }
    }
    return sieve
}

func main() {
    // If you check primality once: compute is better
    fmt.Println(isPrimeCompute(997)) // true

    // If you check many numbers: sieve is better
    sieve := buildSieve(1000) // O(n) space, O(n log log n) build
    for _, n := range []int{2, 10, 17, 100, 997} {
        fmt.Printf("%d: %v\n", n, sieve[n]) // O(1) per lookup!
    }
}
```

**Textual Figure — Compute vs Lookup Table:**

```
  Compute on demand: O(√n) per call, O(1) space
  ──────────────────────────────────────────────
  isPrime(17)?
    check 2: 17%2 ≠ 0
    check 3: 17%3 ≠ 0
    check 4: 4² = 16 < 17, 17%4 ≠ 0
    check 5: 5² = 25 > 17, STOP → true
    Cost: √17 ≈ 4 checks

  Sieve precomputation: O(n) space, O(1) per lookup
  ──────────────────────────────────────────────────
  Build once:  idx: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 ...
             sieve: F  F  T  T  F  T  F  T  F  F  F  T  F  T ...
                         ↑  ↑     ↑     ↑              ↑     ↑
                        primes: 2,3,5,7,11,13,...

  Lookup: sieve[17] → true   (instant!)
  Lookup: sieve[100] → false (instant!)

  ┌──────────────────────────────────────────────────────┐
  │ Break-even analysis:                                 │
  │                                                      │
  │  Cost ▲                                              │
  │       │       · Compute: q × O(√n)                   │
  │       │      ·                                       │
  │       │     ·                                        │
  │       │    · ────── Break-even                       │
  │       │  ·─────────── Sieve: O(n) + q × O(1)        │
  │       │·                                             │
  │       └──────────────────────────▶ q (queries)       │
  │  Few queries → compute    Many queries → sieve       │
  └──────────────────────────────────────────────────────┘
```

---

## Example 4: Sorting Trade Space for Stability

```go
package main

import "fmt"

// In-place sort: O(1) space but NOT stable (quick sort)
func quickSortInPlace(nums []int, low, high int) {
    if low < high {
        pivot := nums[high]
        i := low - 1
        for j := low; j < high; j++ {
            if nums[j] <= pivot {
                i++
                nums[i], nums[j] = nums[j], nums[i]
            }
        }
        nums[i+1], nums[high] = nums[high], nums[i+1]
        quickSortInPlace(nums, low, i)
        quickSortInPlace(nums, i+2, high)
    }
}

// Stable sort: O(n) space, preserves relative order
func mergeSort(nums []int) []int {
    if len(nums) <= 1 { return nums }
    mid := len(nums) / 2
    left := mergeSort(append([]int{}, nums[:mid]...))
    right := mergeSort(append([]int{}, nums[mid:]...))
    return merge(left, right)
}

func merge(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i]); i++
        } else {
            result = append(result, b[j]); j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}

func main() {
    nums := []int{38, 27, 43, 3, 9, 82, 10}
    fmt.Println("Merge sort (stable, O(n) space):", mergeSort(nums))

    nums2 := []int{38, 27, 43, 3, 9, 82, 10}
    quickSortInPlace(nums2, 0, len(nums2)-1)
    fmt.Println("Quick sort (in-place, O(1) space):", nums2)
}
```

**Textual Figure — Stability vs Space Tradeoff:**

```
  Stability test: sort by value, preserving original order for ties
  Input: [(3,"a"), (1,"b"), (3,"c"), (2,"d")]

  Quick Sort (in-place, O(1) space, UNSTABLE):
  ┌───────────────────────────────────────────────┐
  │ (3,"a") (1,"b") (3,"c") (2,"d")             │
  │    ↓ partition around pivot                   │
  │ (1,"b") (2,"d") (3,"c") (3,"a")             │
  │                       ↑       ↑               │
  │              3,"c" before 3,"a" → UNSTABLE    │
  │              (original: "a" came before "c")  │
  └───────────────────────────────────────────────┘

  Merge Sort (O(n) space, STABLE):
  ┌───────────────────────────────────────────────┐
  │ (3,"a") (1,"b") (3,"c") (2,"d")             │
  │    ↓ split + merge with ≤ (preserves order)   │
  │ (1,"b") (2,"d") (3,"a") (3,"c")             │
  │                       ↑       ↑               │
  │              3,"a" before 3,"c" → STABLE ✓    │
  └───────────────────────────────────────────────┘

  ┌──────────────┬──────────┬─────────┬──────────┐
  │  Algorithm   │  Time    │  Space  │  Stable? │
  ├──────────────┼──────────┼─────────┼──────────┤
  │  Quick Sort  │ O(n lg n)│  O(1)   │    No    │
  │  Merge Sort  │ O(n lg n)│  O(n)   │   Yes    │
  └──────────────┴──────────┴─────────┴──────────┘
```

---

## Example 5: String Duplicate Check — Space Tradeoffs

```go
package main

import (
    "fmt"
    "sort"
)

// Approach 1: O(1) space, O(n²) time — compare every pair
func hasDupBrute(s string) bool {
    for i := 0; i < len(s); i++ {
        for j := i + 1; j < len(s); j++ {
            if s[i] == s[j] { return true }
        }
    }
    return false
}

// Approach 2: O(n) space, O(n) time — use hash set
func hasDupHash(s string) bool {
    seen := make(map[byte]bool)
    for i := 0; i < len(s); i++ {
        if seen[s[i]] { return true }
        seen[s[i]] = true
    }
    return false
}

// Approach 3: O(1) space, O(n log n) time — sort first (if mutable)
func hasDupSort(s string) bool {
    chars := []byte(s)
    sort.Slice(chars, func(i, j int) bool { return chars[i] < chars[j] })
    for i := 1; i < len(chars); i++ {
        if chars[i] == chars[i-1] { return true }
    }
    return false
}

// Approach 4: O(1) space, O(n) time — bit vector (only for a-z)
func hasDupBit(s string) bool {
    var checker int32
    for i := 0; i < len(s); i++ {
        val := int32(s[i] - 'a')
        if checker&(1<<val) > 0 { return true }
        checker |= 1 << val
    }
    return false
}

func main() {
    s1 := "abcdefg"
    s2 := "abcdefa"

    fmt.Printf("Brute: %s=%v, %s=%v\n", s1, hasDupBrute(s1), s2, hasDupBrute(s2))
    fmt.Printf("Hash:  %s=%v, %s=%v\n", s1, hasDupHash(s1), s2, hasDupHash(s2))
    fmt.Printf("Sort:  %s=%v, %s=%v\n", s1, hasDupSort(s1), s2, hasDupSort(s2))
    fmt.Printf("Bit:   %s=%v, %s=%v\n", s1, hasDupBit(s1), s2, hasDupBit(s2))
}
```

**Textual Figure — Four Approaches on the Tradeoff Spectrum:**

```
  Input: s = "abcdefa"  (duplicate 'a')

  1. Brute O(n²) time, O(1) space — compare every pair:
     a-b a-c a-d a-e a-f a-a ✓ found!
     ────  6 comparisons to find it

  2. Hash  O(n) time, O(n) space — set lookup:
     seen={}  → 'a' add → 'b' add → ... → 'a' FOUND ✓
     seen={a,b,c,d,e,f}  ← up to 26 entries

  3. Sort  O(n lg n) time, O(1) space — sort then scan:
     "abcdefa" → sort → "aabcdef"
      ↑↑ adjacent match → found ✓

  4. Bit   O(n) time, O(1) space — bit vector (a-z only):
     checker: 00000000000000000000000000  (26 bits)
     'a' → set bit 0:  00000000000000000000000001
     'b' → set bit 1:  00000000000000000000000011
     ...
     'a' → bit 0 already set → FOUND ✓

  Tradeoff Map:
  ┌─────────────────────────────────────────────┐
  │  Time ▲                                     │
  │       │                                     │
  │  n²   │  ● Brute                            │
  │       │                                     │
  │ n lg n│     ● Sort                           │
  │       │                                     │
  │  n    │        ● Hash    ● Bit (best!)      │
  │       │                                     │
  │       └────────────────────────────▶ Space   │
  │           O(1)     O(n)    O(1)*            │
  │                          *only for a-z      │
  └─────────────────────────────────────────────┘
```

---

## Example 6: Caching — Classic Time-Space Tradeoff

```go
package main

import (
    "fmt"
    "sync"
)

// LRU-style cache: trade O(n) space for O(1) retrieval
type Cache struct {
    data map[string]int
    mu   sync.RWMutex
}

func NewCache() *Cache {
    return &Cache{data: make(map[string]int)}
}

func (c *Cache) Get(key string) (int, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok // O(1) lookup
}

func (c *Cache) Set(key string, val int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = val // O(1) insert
}

// Expensive computation
func expensiveCompute(n int) int {
    result := 0
    for i := 0; i < n*n; i++ { // O(n²)
        result += i
    }
    return result
}

func main() {
    cache := NewCache()

    // First call: compute and cache — O(n²)
    key := "1000"
    result := expensiveCompute(1000)
    cache.Set(key, result)
    fmt.Printf("Computed: %d\n", result)

    // Subsequent calls: O(1) from cache
    if val, ok := cache.Get(key); ok {
        fmt.Printf("Cached:   %d\n", val)
    }
}
```

**Textual Figure — Cache Hit/Miss Flow:**

```
  Without Cache:                     With Cache:
  ────────────────────────          ────────────────────────────
  Query(1000)                        Query(1000)
    │                                  │
    ▼                                  ▼
  ┌───────────────┐                 ┌──────────────┐
  │  Compute O(n²) │                 │ Cache lookup │
  │  1,000,000    │                 │    O(1)      │
  │  iterations   │                 └──────┬───────┘
  └───────┬───────┘                     │
          │                           Hit? ────────────┐
          ▼                        Yes ▼              No ▼
       result                     return val    ┌───────────┐
                                    O(1)        │ Compute  │
  Query(1000) again:                            │   O(n²)  │
    │                                          └────┬──────┘
    ▼                                               │
  ┌───────────────┐                            cache.Set()
  │  Compute O(n²) │                                │
  │  AGAIN!        │                              result
  └───────────────┘
                                   Next call: O(1) from cache!

  Cost over Q queries:
  No cache:   Q × O(n²) = O(Q·n²)
  With cache: O(n²) + (Q-1) × O(1) ≈ O(n²)
```

---

## Example 7: Matrix Path — DP Space Optimization

```go
package main

import "fmt"

// Full DP: O(m×n) space
func uniquePathsFull(m, n int) int {
    dp := make([][]int, m)
    for i := range dp {
        dp[i] = make([]int, n)
        dp[i][0] = 1
    }
    for j := 0; j < n; j++ {
        dp[0][j] = 1
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
        }
    }
    return dp[m-1][n-1]
}

// Space-optimized: O(n) space — only keep current and previous row
func uniquePathsOptimized(m, n int) int {
    dp := make([]int, n)
    for j := range dp {
        dp[j] = 1
    }
    for i := 1; i < m; i++ {
        for j := 1; j < n; j++ {
            dp[j] += dp[j-1] // dp[j] already has value from previous row
        }
    }
    return dp[n-1]
}

func main() {
    m, n := 3, 7
    fmt.Println(uniquePathsFull(m, n))      // 28 — O(m×n) space
    fmt.Println(uniquePathsOptimized(m, n)) // 28 — O(n) space
    // Same time O(m×n), but 7x less memory!
}
```

**Textual Figure — Full DP Grid vs Space-Optimized Row:**

```
  Full DP: O(m×n) space — 3×7 grid
  ┌──────────────────────────────────────┐
  │      j=0  j=1  j=2  j=3  j=4  j=5  j=6  │
  │ i=0 [ 1    1    1    1    1    1    1  ] │
  │ i=1 [ 1    2    3    4    5    6    7  ] │
  │ i=2 [ 1    3    6   10   15   21   28 ] │
  │                                    ↑↑    │
  │                               answer=28  │
  │      Total cells stored: 3×7 = 21         │
  └──────────────────────────────────────┘

  Space-Optimized: O(n) — reuse single row
  ┌──────────────────────────────────────┐
  │ Init:   dp = [1, 1, 1, 1, 1, 1, 1]       │
  │                                           │
  │ i=1:    dp = [1, 2, 3, 4, 5, 6, 7]       │
  │         dp[j] += dp[j-1]                   │
  │         dp[1] = 1+1=2                      │
  │         dp[2] = 1+2=3  ...                 │
  │                                           │
  │ i=2:    dp = [1, 3, 6, 10, 15, 21, 28]   │
  │         dp[j] already has prev row value!   │
  │                                     ↑↑     │
  │                                answer=28   │
  │      Total cells stored: 7 (not 21!)       │
  └──────────────────────────────────────┘
```

---

## Example 8: Prefix Sum — Precompute for Fast Queries

```go
package main

import "fmt"

// Without prefix sum: each range query is O(n)
func rangeSumBrute(nums []int, left, right int) int {
    sum := 0
    for i := left; i <= right; i++ {
        sum += nums[i]
    }
    return sum
}

// With prefix sum: O(n) space, but each query is O(1)
type PrefixSum struct {
    prefix []int
}

func NewPrefixSum(nums []int) *PrefixSum {
    prefix := make([]int, len(nums)+1)
    for i, num := range nums {
        prefix[i+1] = prefix[i] + num
    }
    return &PrefixSum{prefix: prefix}
}

func (ps *PrefixSum) RangeSum(left, right int) int {
    return ps.prefix[right+1] - ps.prefix[left] // O(1)!
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Brute force: O(n) per query
    fmt.Println(rangeSumBrute(nums, 2, 7)) // 33

    // Prefix sum: O(n) build, O(1) per query
    ps := NewPrefixSum(nums)
    fmt.Println(ps.RangeSum(2, 7)) // 33
    fmt.Println(ps.RangeSum(0, 9)) // 55
    fmt.Println(ps.RangeSum(4, 4)) // 5

    // For 1 million queries: brute=O(n×q), prefix=O(n+q)
}
```

**Textual Figure — Prefix Sum Construction and Query:**

```
  nums:     [1,  2,  3,  4,  5,  6,  7,  8,  9, 10]
  index:     0   1   2   3   4   5   6   7   8   9

  Build prefix sum array (one-time O(n)):
  prefix: [0,  1,  3,  6, 10, 15, 21, 28, 36, 45, 55]
  index:   0   1   2   3   4   5   6   7   8   9  10
           ↑                                       ↑
         dummy                                    sum(all)

  Query: RangeSum(2, 7) = sum of indices 2..7
  ┌────────────────────────────────────────────────┐
  │  Brute: loop from 2 to 7                          │
  │    3 + 4 + 5 + 6 + 7 + 8 = 33     O(n) per query  │
  │                                                   │
  │  Prefix Sum: prefix[8] - prefix[2]                │
  │    = 36 - 3 = 33                   O(1) per query  │
  │                                                   │
  │  nums:   [ 1  2 |3  4  5  6  7  8| 9  10]         │
  │                  └───────────────┘                   │
  │                   sum = 33                        │
  │  prefix[right+1] - prefix[left] = instant!         │
  └────────────────────────────────────────────────┘

  Cost comparison for Q queries:
  Brute:      O(n × Q)
  Prefix Sum: O(n) build + O(Q) queries = O(n + Q)
```

---

## Example 9: Adjacency Matrix vs Adjacency List

```go
package main

import "fmt"

// Adjacency Matrix: O(V²) space, O(1) edge lookup
type AdjMatrix struct {
    matrix [][]bool
    V      int
}

func NewAdjMatrix(v int) *AdjMatrix {
    m := make([][]bool, v)
    for i := range m { m[i] = make([]bool, v) }
    return &AdjMatrix{matrix: m, V: v}
}

func (g *AdjMatrix) AddEdge(u, v int) { g.matrix[u][v] = true; g.matrix[v][u] = true }
func (g *AdjMatrix) HasEdge(u, v int) bool { return g.matrix[u][v] } // O(1)

// Adjacency List: O(V+E) space, O(degree) edge lookup
type AdjList struct {
    list map[int][]int
}

func NewAdjList() *AdjList { return &AdjList{list: make(map[int][]int)} }

func (g *AdjList) AddEdge(u, v int) {
    g.list[u] = append(g.list[u], v)
    g.list[v] = append(g.list[v], u)
}

func (g *AdjList) HasEdge(u, v int) bool {
    for _, neighbor := range g.list[u] { // O(degree)
        if neighbor == v { return true }
    }
    return false
}

func main() {
    // Dense graph (many edges): matrix is better
    // Sparse graph (few edges): list is better

    matrix := NewAdjMatrix(5)
    matrix.AddEdge(0, 1); matrix.AddEdge(1, 2)
    fmt.Println("Matrix HasEdge(0,1):", matrix.HasEdge(0, 1)) // O(1)

    list := NewAdjList()
    list.AddEdge(0, 1); list.AddEdge(1, 2)
    fmt.Println("List HasEdge(0,1):", list.HasEdge(0, 1)) // O(degree)

    fmt.Println("\nTradeoff:")
    fmt.Println("  Matrix: O(V²) space, O(1) edge check")
    fmt.Println("  List:   O(V+E) space, O(degree) edge check")
}
```

**Textual Figure — Matrix vs List Representation:**

```
  Graph: 5 nodes, edges: 0-1, 1-2

  Adjacency Matrix: O(V²) space
  ┌─────────────────────────────────┐
  │      0   1   2   3   4        │
  │  0 [ 0   1   0   0   0 ]      │
  │  1 [ 1   0   1   0   0 ]      │
  │  2 [ 0   1   0   0   0 ]      │
  │  3 [ 0   0   0   0   0 ]      │
  │  4 [ 0   0   0   0   0 ]      │
  │                               │
  │  Total cells: 5×5 = 25        │
  │  HasEdge(0,1) → matrix[0][1]  │
  │                = 1 → O(1) ✓   │
  └─────────────────────────────────┘

  Adjacency List: O(V+E) space
  ┌─────────────────────────────────┐
  │  0: [1]                       │
  │  1: [0, 2]                    │
  │  2: [1]                       │
  │  3: []                        │
  │  4: []                        │
  │                               │
  │  Total entries: 5 + 4 = 9     │
  │  HasEdge(0,1) → scan list[0]  │
  │              = O(degree) = O(1)│
  └─────────────────────────────────┘

  When to use which:
  ┌──────────────┬────────────────┬────────────────┐
  │  Graph Type  │    Matrix      │      List      │
  ├──────────────┼────────────────┼────────────────┤
  │ Dense E≈V²  │  ✓ Efficient  │  Wasteful       │
  │ Sparse E≪V² │  Wasteful     │  ✓ Efficient   │
  └──────────────┴────────────────┴────────────────┘
```

---

## Example 10: Frequency Count — Precompute vs On-Demand

```go
package main

import "fmt"

// On-demand: O(1) space, O(n) time per query
func countOnDemand(nums []int, target int) int {
    count := 0
    for _, num := range nums {
        if num == target { count++ }
    }
    return count
}

// Precomputed: O(n) space, O(1) time per query
func buildFreqMap(nums []int) map[int]int {
    freq := make(map[int]int)
    for _, num := range nums {
        freq[num]++
    }
    return freq
}

func main() {
    nums := []int{1, 3, 2, 3, 1, 3, 4, 2, 1, 3, 5, 3}

    // 1 query → on-demand is fine
    fmt.Println("Count of 3:", countOnDemand(nums, 3)) // 5

    // Many queries → precompute is better
    freq := buildFreqMap(nums)
    for _, target := range []int{1, 2, 3, 4, 5, 6} {
        fmt.Printf("Count of %d: %d\n", target, freq[target])
    }
}
```

**Textual Figure — On-Demand vs Precomputed Frequency:**

```
  nums = [1, 3, 2, 3, 1, 3, 4, 2, 1, 3, 5, 3]

  On-Demand: Count of 3? → scan entire array
  ┌───────────────────────────────────────┐
  │  [1, 3, 2, 3, 1, 3, 4, 2, 1, 3, 5, 3] │
  │       ↑     ↑     ↑              ↑     ↑  │
  │      match match match          match match│
  │   count = 5              O(n) per query │
  └───────────────────────────────────────┘

  Precomputed: Build freq map once O(n), then O(1) lookups
  ┌───────────────────────────────────────┐
  │  freq = {1:3, 2:2, 3:5, 4:1, 5:1}      │
  │                                         │
  │  freq[3] = 5      O(1) ✓                │
  │  freq[1] = 3      O(1) ✓                │
  │  freq[6] = 0      O(1) ✓ (not found)    │
  └───────────────────────────────────────┘

  Cost for Q queries on n elements:
    On-demand:   O(n × Q)     ← scan for each query
    Precomputed: O(n + Q)     ← build once, lookup many
```

---

## Tradeoff Decision Guide

| Scenario | Prefer Time | Prefer Space |
|----------|-------------|-------------|
| Many repeated queries | Precompute (cache) | N/A |
| Single query | N/A | Compute on-demand |
| Latency-critical | More memory | N/A |
| Memory-constrained | N/A | Recompute |
| Large dataset | Streaming / O(1) | N/A |
| Real-time systems | Lookup tables | N/A |

---

## Key Takeaways

1. **Time-space tradeoff is everywhere**: hash maps, caching, DP, precomputation
2. **Hash maps**: trade O(n) space for O(1) lookup (vs O(n) linear scan)
3. **Memoization/DP**: trade O(n) space for exponential → linear time
4. **Prefix sums**: trade O(n) space for O(1) range queries
5. **Always mention the tradeoff** in interviews — shows depth of understanding
6. **No universal "best"** — depends on constraints (memory, latency, query count)

> **Next up:** Recurrence Relations →
