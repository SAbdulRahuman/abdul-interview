# Phase 23: Design Data Structures — Space-Time Tradeoffs

## Overview

Every algorithm and data structure involves choosing between **using more memory** to save time, or **using more time** to save memory. Understanding these tradeoffs is key to system design and optimization.

| Strategy | Space | Time | Example |
|----------|-------|------|---------|
| **Precomputation** | More | Less | Prefix sum, lookup tables |
| **Caching/Memoization** | More | Less | DP tables, LRU cache |
| **Compression** | Less | More | Run-length, Huffman |
| **Streaming** | Less | More | Single pass, reservoir sampling |
| **Indexing** | More | Less | Hash index, B-tree, inverted index |

---

## Example 1: Precomputed Prefix Sum vs Brute Force

```go
package main

import (
	"fmt"
	"time"
)

// Brute force: O(1) space, O(n) per query
func bruteForceSum(arr []int, l, r int) int {
	sum := 0
	for i := l; i <= r; i++ { sum += arr[i] }
	return sum
}

// Prefix sum: O(n) space, O(1) per query
type PrefixSum struct {
	prefix []int
}

func NewPrefixSum(arr []int) *PrefixSum {
	prefix := make([]int, len(arr)+1)
	for i, v := range arr { prefix[i+1] = prefix[i] + v }
	return &PrefixSum{prefix}
}

func (ps *PrefixSum) Query(l, r int) int {
	return ps.prefix[r+1] - ps.prefix[l]
}

func main() {
	n := 100000
	arr := make([]int, n)
	for i := range arr { arr[i] = i + 1 }
	ps := NewPrefixSum(arr)

	queries := 100000

	start := time.Now()
	for i := 0; i < queries; i++ { bruteForceSum(arr, 0, n-1) }
	brute := time.Since(start)

	start = time.Now()
	for i := 0; i < queries; i++ { ps.Query(0, n-1) }
	precomp := time.Since(start)

	fmt.Println("=== Prefix Sum: Space-Time Tradeoff ===")
	fmt.Printf("Brute force: %v (O(1) space, O(n) per query)\n", brute)
	fmt.Printf("Prefix sum:  %v (O(n) space, O(1) per query)\n", precomp)
	fmt.Printf("Speedup: %.0fx\n", float64(brute)/float64(precomp))

	fmt.Printf("\nAnswer check: %d = %d\n", bruteForceSum(arr, 10, 20), ps.Query(10, 20))
}
```

**Textual Figure:**
```
Prefix Sum: O(n) Space → O(1) Query

  Array:    [3, 1, 4, 1, 5, 9, 2, 6]
  Prefix:   [0, 3, 4, 8, 9,14,23,25,31]
             │  │  │  │  │  │  │  │  │
             0  1  2  3  4  5  6  7  8  (index)

  Query sum(2,5): prefix[6] - prefix[2] = 23 - 4 = 19

  Brute force:                Prefix sum:
  ┌────────────────────┐  ┌────────────────────┐
  │ Space: O(1)          │  │ Space: O(n)          │
  │ Query: O(n) per call │  │ Query: O(1) per call │
  │ Build: none          │  │ Build: O(n) one-time │
  └────────────────────┘  └────────────────────┘

  Tradeoff: spend O(n) memory to get O(1) queries
  Worth it when queries >> 1
```

---

## Example 2: Memoization vs Recomputation

```go
package main

import (
	"fmt"
	"time"
)

// No memoization: O(1) space, O(2^n) time
func fibNaive(n int) int64 {
	if n <= 1 { return int64(n) }
	return fibNaive(n-1) + fibNaive(n-2)
}

// Memoization: O(n) space, O(n) time
func fibMemo(n int, memo map[int]int64) int64 {
	if n <= 1 { return int64(n) }
	if v, ok := memo[n]; ok { return v }
	memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo)
	return memo[n]
}

// Bottom-up DP: O(n) space, O(n) time
func fibDP(n int) int64 {
	if n <= 1 { return int64(n) }
	dp := make([]int64, n+1)
	dp[1] = 1
	for i := 2; i <= n; i++ { dp[i] = dp[i-1] + dp[i-2] }
	return dp[n]
}

// Space-optimized: O(1) space, O(n) time
func fibOpt(n int) int64 {
	if n <= 1 { return int64(n) }
	a, b := int64(0), int64(1)
	for i := 2; i <= n; i++ { a, b = b, a+b }
	return b
}

func main() {
	n := 40

	start := time.Now()
	r1 := fibNaive(n)
	t1 := time.Since(start)

	start = time.Now()
	r2 := fibMemo(n, make(map[int]int64))
	t2 := time.Since(start)

	start = time.Now()
	r3 := fibDP(n)
	t3 := time.Since(start)

	start = time.Now()
	r4 := fibOpt(n)
	t4 := time.Since(start)

	fmt.Printf("fib(%d) = %d\n\n", n, r1)
	fmt.Printf("Naive (no memo):  %12v  O(2^n) time, O(n) stack   [%d]\n", t1, r1)
	fmt.Printf("Memoized:         %12v  O(n)   time, O(n) memo    [%d]\n", t2, r2)
	fmt.Printf("Bottom-up DP:     %12v  O(n)   time, O(n) array   [%d]\n", t3, r3)
	fmt.Printf("Space-optimized:  %12v  O(n)   time, O(1) vars    [%d]\n", t4, r4)
}
```

**Textual Figure:**
```
Memoization vs Recomputation: Fibonacci

  Naive fib(5) call tree — O(2^n) redundant calls:

               fib(5)
              ╱      ╲
          fib(4)     fib(3)
          ╱    ╲      ╱    ╲
       fib(3) fib(2) fib(2) fib(1)
       ╱   ╲    ╱  ╲    ╱  ╲
    fib(2) f(1) f(1) f(0) f(1) f(0)
    ╱   ╲
  f(1)  f(0)     ← fib(2) computed 3 times!

  With memoization — O(n) calls:

    fib(5) → fib(4) → fib(3) → fib(2) → fib(1)
              └─ fib(3) [cached ✓]
    └─ fib(3) [cached ✓]

  ┌──────────────┬───────────┬───────────┐
  │ Method       │ Time      │ Space     │
  ├──────────────┼───────────┼───────────┤
  │ Naive        │ O(2^n)    │ O(n) stack│
  │ Memoized     │ O(n)      │ O(n) memo │
  │ Bottom-up DP │ O(n)      │ O(n) array│
  │ Optimized    │ O(n)      │ O(1) vars │
  └──────────────┴───────────┴───────────┘
```

---

## Example 3: Lookup Table for Popcount

```go
package main

import "fmt"

// Precomputed 16-bit popcount table: 64KB space, O(1) per 16-bit lookup
var popTable [1 << 16]byte

func init() {
	for i := 1; i < len(popTable); i++ {
		popTable[i] = popTable[i>>1] + byte(i&1)
	}
}

func popcountTable(x uint64) int {
	return int(popTable[x&0xFFFF] +
		popTable[(x>>16)&0xFFFF] +
		popTable[(x>>32)&0xFFFF] +
		popTable[(x>>48)&0xFFFF])
}

func popcountLoop(x uint64) int {
	count := 0
	for x != 0 { count++; x &= x - 1 }
	return count
}

func main() {
	tests := []uint64{0, 1, 7, 255, 0xDEADBEEF, 0xFFFFFFFFFFFFFFFF}

	fmt.Println("=== Popcount: Table Lookup vs Kernighan ===\n")
	for _, x := range tests {
		t := popcountTable(x)
		k := popcountLoop(x)
		fmt.Printf("  0x%016X → table=%d, loop=%d\n", x, t, k)
	}

	fmt.Println("\nTable: 64KB space, O(1) per 16-bit chunk")
	fmt.Println("Loop: O(1) space, O(popcount) per call")
}
```

**Textual Figure:**
```
Popcount Lookup Table: 64KB Space → O(1) Time

  Precomputed table (16-bit chunks):
    popTable[0x0000] = 0
    popTable[0x0001] = 1    (0000 0000 0000 0001)
    popTable[0x0003] = 2    (0000 0000 0000 0011)
    popTable[0x00FF] = 8    (0000 0000 1111 1111)
    popTable[0xFFFF] = 16   (1111 1111 1111 1111)

  64-bit popcount via 4 table lookups:
    x = 0xDEADBEEF_00000000

    ┌──────────┬──────────┬──────────┬──────────┐
    │ chunk 3  │ chunk 2  │ chunk 1  │ chunk 0  │
    │ bits48-63│ bits32-47│ bits16-31│ bits 0-15│
    └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
         │           │           │           │
         ▼           ▼           ▼           ▼
     popTable[]   popTable[]  popTable[]  popTable[]
         │           │           │           │
         └────────── + ────────── + ──────────┘
                       = total popcount

  ┌──────────────┬─────────────┬───────────────┐
  │ Method       │ Space       │ Time            │
  ├──────────────┼─────────────┼───────────────┤
  │ Table lookup │ 64KB        │ O(1) = 4 lookups│
  │ Kernighan    │ O(1)        │ O(popcount)     │
  └──────────────┴─────────────┴───────────────┘
```

---

## Example 4: Hash Set vs Sorted Slice for Membership

```go
package main

import (
	"fmt"
	"sort"
	"time"
)

func main() {
	n := 1000000
	data := make([]int, n)
	for i := range data { data[i] = i * 2 }

	// Build hash set: O(n) space, O(1) lookup
	hashSet := make(map[int]bool, n)
	for _, v := range data { hashSet[v] = true }

	// Build sorted slice: O(n) space, O(log n) lookup
	sorted := make([]int, n)
	copy(sorted, data)
	sort.Ints(sorted)

	lookups := 100000
	targets := make([]int, lookups)
	for i := range targets { targets[i] = i * 3 }

	// Hash set lookup
	start := time.Now()
	hashHits := 0
	for _, t := range targets {
		if hashSet[t] { hashHits++ }
	}
	hashTime := time.Since(start)

	// Binary search lookup
	start = time.Now()
	bsHits := 0
	for _, t := range targets {
		i := sort.SearchInts(sorted, t)
		if i < len(sorted) && sorted[i] == t { bsHits++ }
	}
	bsTime := time.Since(start)

	fmt.Println("=== Hash Set vs Binary Search ===")
	fmt.Printf("Hash set:  %v (%d hits) — O(n) extra space, O(1) lookup\n", hashTime, hashHits)
	fmt.Printf("Bin search: %v (%d hits) — O(1) extra space*, O(log n) lookup\n", bsTime, bsHits)
	fmt.Println("\n*sorted in-place uses no extra space")
}
```

**Textual Figure:**
```
Hash Set vs Binary Search: Membership Testing

  Data: [0, 2, 4, 6, 8, ..., 1999998]  (n=1,000,000)
  Query: is target in data?

  Hash Set approach:
    Build: iterate all n elements → O(n) time, O(n) space
    ┌────────────────────────────────┐
    │ hashSet[target] → true/false │  O(1)
    └────────────────────────────────┘

  Binary Search approach:
    Build: sort O(n log n), then search
    ┌──────────────────────────────────────┐
    │ [0, 2, 4, ..., target?, ..., 1999998] │
    │         ↑ binary search O(log n)        │
    └──────────────────────────────────────┘

  ┌───────────────┬──────────────┬──────────────┐
  │ Method        │ Extra Space  │ Lookup Time  │
  ├───────────────┼──────────────┼──────────────┤
  │ Hash Set      │ O(n)         │ O(1) avg     │
  │ Binary Search │ O(1)*        │ O(log n)     │
  └───────────────┴──────────────┴──────────────┘
  * sorted in-place uses no extra space
```

---

## Example 5: Rolling Hash vs Naive String Matching

```go
package main

import "fmt"

const (
	base = 31
	mod  = 1_000_000_007
)

// Naive: O(1) space, O(n*m) time
func naiveSearch(text, pattern string) []int {
	var results []int
	for i := 0; i <= len(text)-len(pattern); i++ {
		if text[i:i+len(pattern)] == pattern {
			results = append(results, i)
		}
	}
	return results
}

// Rabin-Karp: O(m) precomputation space, O(n+m) time (average)
func rabinKarp(text, pattern string) []int {
	n, m := len(text), len(pattern)
	if m > n { return nil }

	// Compute pattern hash and power
	pHash, tHash := int64(0), int64(0)
	pow := int64(1)
	for i := 0; i < m; i++ {
		pHash = (pHash*base + int64(pattern[i])) % mod
		tHash = (tHash*base + int64(text[i])) % mod
		if i > 0 { pow = pow * base % mod }
	}

	var results []int
	for i := 0; i <= n-m; i++ {
		if tHash == pHash && text[i:i+m] == pattern {
			results = append(results, i)
		}
		if i+m < n {
			tHash = (tHash - int64(text[i])*pow%mod + mod) % mod
			tHash = (tHash*base + int64(text[i+m])) % mod
		}
	}
	return results
}

func main() {
	text := "abcabcabcabc"
	pattern := "abcabc"

	fmt.Printf("Text:    %s\n", text)
	fmt.Printf("Pattern: %s\n\n", pattern)

	r1 := naiveSearch(text, pattern)
	r2 := rabinKarp(text, pattern)

	fmt.Printf("Naive matches at:      %v  (O(nm) time, O(1) space)\n", r1)
	fmt.Printf("Rabin-Karp matches at: %v  (O(n+m) avg, O(m) space)\n", r2)
}
```

**Textual Figure:**
```
Rolling Hash (Rabin-Karp) vs Naive String Matching

  Text:    "abcabcabcabc"  (n=12)
  Pattern: "abcabc"        (m=6)

  Naive: slide pattern across text, compare all m chars
    abcabcabcabc       abcabcabcabc
    abcabc             ✗  (shift 1)
    ✓ match at 0         abcabc
                         ✗
    ...O(n·m) worst case

  Rolling hash: compute hash of window, slide in O(1)

    Window:  [a b c a b c] a b c a b c    hash = H1
    Pattern:  a b c a b c                 hash = P
              H1 == P? → verify ✓ match at 0

    Slide:   a [b c a b c a] b c a b c    hash = H2
              Remove 'a', add 'a':
              H2 = (H1 - val('a')·base^(m-1)) · base + val('a')
              H2 ≠ P → skip

    ...continue sliding...

    Window:  a b c [a b c a b c] a b c    hash = H3
              H3 == P? → verify ✓ match at 3

  ┌───────────────┬──────────────┬──────────────┐
  │ Method        │ Time         │ Space        │
  ├───────────────┼──────────────┼──────────────┤
  │ Naive         │ O(n·m) worst │ O(1)         │
  │ Rabin-Karp    │ O(n+m) avg  │ O(m)         │
  └───────────────┴──────────────┴──────────────┘
  Matches at: [0, 3, 6]
```

---

## Example 6: In-Place vs Extra Array for Merge Sort

```go
package main

import "fmt"

// Standard merge sort: O(n) extra space
func mergeSort(arr []int) []int {
	if len(arr) <= 1 { return arr }
	mid := len(arr) / 2
	left := mergeSort(append([]int{}, arr[:mid]...))
	right := mergeSort(append([]int{}, arr[mid:]...))
	return merge(left, right)
}

func merge(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] { result = append(result, a[i]); i++ } else { result = append(result, b[j]); j++ }
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

// In-place sort (insertion sort): O(1) extra space, O(n²) time
func insertionSort(arr []int) {
	for i := 1; i < len(arr); i++ {
		key := arr[i]
		j := i - 1
		for j >= 0 && arr[j] > key { arr[j+1] = arr[j]; j-- }
		arr[j+1] = key
	}
}

func main() {
	arr1 := []int{38, 27, 43, 3, 9, 82, 10}
	arr2 := make([]int, len(arr1))
	copy(arr2, arr1)

	fmt.Printf("Original: %v\n\n", arr1)

	sorted := mergeSort(arr1)
	fmt.Printf("Merge sort:     %v  (O(n log n) time, O(n) space)\n", sorted)

	insertionSort(arr2)
	fmt.Printf("Insertion sort: %v  (O(n²) time, O(1) space)\n", arr2)

	fmt.Println("\nTradeoff: merge sort uses O(n) extra memory but is O(n log n)")
	fmt.Println("Insertion sort is in-place but O(n²)")
	fmt.Println("Go's sort.Slice uses introsort: O(n log n) time, O(log n) space")
}
```

**Textual Figure:**
```
In-Place vs Extra Array Sorting

  Input: [38, 27, 43, 3, 9, 82, 10]

  Merge Sort (O(n) extra space):
    [38,27,43,3,9,82,10]
    ┌─────────────────────────┐
    │ split:  [38,27,43,3] [9,82,10] │
    │ split:  [38,27] [43,3]         │
    │ merge:  [27,38] [3,43]         │ ← temp arrays
    │ merge:  [3,27,38,43]           │ ← temp arrays
    │ merge:  [3,9,10,27,38,43,82]   │ ← temp arrays
    └─────────────────────────┘
    Time: O(n log n)    Space: O(n)

  Insertion Sort (O(1) extra space):
    [38, 27, 43, 3, 9, 82, 10]
     ───  ↑
    [27, 38, 43, 3, 9, 82, 10]  insert 27
    [27, 38, 43, 3, 9, 82, 10]  43 in place
    [ 3, 27, 38, 43, 9, 82, 10]  insert 3 (shift 3 elements)
    [ 3, 9, 27, 38, 43, 82, 10]  insert 9
    [ 3, 9, 27, 38, 43, 82, 10]  82 in place
    [ 3, 9, 10, 27, 38, 43, 82]  insert 10
    Time: O(n²)         Space: O(1)

  ┌────────────────┬─────────────┬──────────┐
  │ Algorithm      │ Time        │ Space    │
  ├────────────────┼─────────────┼──────────┤
  │ Merge sort     │ O(n log n)  │ O(n)     │
  │ Insertion sort │ O(n²)       │ O(1)     │
  │ Introsort      │ O(n log n)  │ O(log n) │
  └────────────────┴─────────────┴──────────┘
```

---

## Example 7: Adjacency Matrix vs Adjacency List

```go
package main

import (
	"fmt"
	"unsafe"
)

// Matrix: O(V²) space, O(1) edge check
type AdjMatrix struct {
	n    int
	data [][]bool
}

func NewMatrix(n int) *AdjMatrix {
	data := make([][]bool, n)
	for i := range data { data[i] = make([]bool, n) }
	return &AdjMatrix{n, data}
}

func (m *AdjMatrix) AddEdge(u, v int) { m.data[u][v] = true; m.data[v][u] = true }
func (m *AdjMatrix) HasEdge(u, v int) bool { return m.data[u][v] }
func (m *AdjMatrix) Neighbors(u int) []int {
	var res []int
	for v := 0; v < m.n; v++ { if m.data[u][v] { res = append(res, v) } }
	return res
}

// List: O(V+E) space, O(degree) edge check
type AdjList struct {
	n    int
	adj  [][]int
}

func NewList(n int) *AdjList { return &AdjList{n, make([][]int, n)} }

func (l *AdjList) AddEdge(u, v int) { l.adj[u] = append(l.adj[u], v); l.adj[v] = append(l.adj[v], u) }
func (l *AdjList) HasEdge(u, v int) bool {
	for _, w := range l.adj[u] { if w == v { return true } }
	return false
}
func (l *AdjList) Neighbors(u int) []int { return l.adj[u] }

func main() {
	n := 1000
	mat := NewMatrix(n)
	lst := NewList(n)

	// Sparse graph: each node has ~5 edges
	for i := 0; i < n; i++ {
		for j := 1; j <= 5 && i+j < n; j++ {
			mat.AddEdge(i, i+j)
			lst.AddEdge(i, i+j)
		}
	}

	matSize := int(unsafe.Sizeof(*mat)) + n*n // approximate
	lstSize := int(unsafe.Sizeof(*lst)) + n*5*8 // approximate

	fmt.Println("=== Adjacency Matrix vs List ===")
	fmt.Printf("V=%d, E≈%d (sparse)\n\n", n, n*5)
	fmt.Printf("Matrix: ~%d bytes (%d×%d)\n", matSize, n, n)
	fmt.Printf("List:   ~%d bytes\n\n", lstSize)

	fmt.Printf("HasEdge(0,3): matrix=%v, list=%v\n", mat.HasEdge(0, 3), lst.HasEdge(0, 3))
	fmt.Printf("HasEdge(0,999): matrix=%v, list=%v\n", mat.HasEdge(0, 999), lst.HasEdge(0, 999))

	fmt.Println("\nMatrix: dense graphs, O(1) edge check, O(V²) space")
	fmt.Println("List: sparse graphs, O(deg) edge check, O(V+E) space")
}
```

**Textual Figure:**
```
Adjacency Matrix vs Adjacency List

  Graph (V=5, E=5, sparse):
    0 ── 1 ── 2
    │       │
    3 ── 4 ─┘

  Adjacency Matrix (V² = 25 cells):
      0  1  2  3  4
    ┌───┬──┬──┬──┬──┐
  0 │ 0 │ 1│ 0│ 1│ 0│
    ├───┼──┼──┼──┼──┤
  1 │ 1 │ 0│ 1│ 0│ 0│
    ├───┼──┼──┼──┼──┤
  2 │ 0 │ 1│ 0│ 0│ 1│
    ├───┼──┼──┼──┼──┤
  3 │ 1 │ 0│ 0│ 0│ 1│
    ├───┼──┼──┼──┼──┤
  4 │ 0 │ 0│ 1│ 1│ 0│
    └───┴──┴──┴──┴──┘
    Space: O(V²) = 25    HasEdge: O(1)

  Adjacency List:
    0 → [1, 3]
    1 → [0, 2]
    2 → [1, 4]
    3 → [0, 4]
    4 → [2, 3]
    Space: O(V+E) = 15   HasEdge: O(deg)

  ┌─────────┬────────────┬────────────┬─────────────┐
  │         │ Space      │ Edge check │ Best for    │
  ├─────────┼────────────┼────────────┼─────────────┤
  │ Matrix  │ O(V²)      │ O(1)       │ Dense graph │
  │ List    │ O(V+E)     │ O(deg)     │ Sparse graph│
  └─────────┴────────────┴────────────┴─────────────┘
```

---

## Example 8: Compressed Trie vs Full Array Trie

```go
package main

import "fmt"

// Full trie: each node has 26 children pointers
type FullNode struct {
	children [26]*FullNode
	isEnd    bool
}

type FullTrie struct {
	root *FullNode
}

func NewFullTrie() *FullTrie { return &FullTrie{root: &FullNode{}} }

func (t *FullTrie) Insert(word string) {
	node := t.root
	for _, c := range word {
		idx := c - 'a'
		if node.children[idx] == nil { node.children[idx] = &FullNode{} }
		node = node.children[idx]
	}
	node.isEnd = true
}

func (t *FullTrie) countNodes() int {
	count := 0
	var dfs func(*FullNode)
	dfs = func(n *FullNode) {
		if n == nil { return }
		count++
		for _, c := range n.children { dfs(c) }
	}
	dfs(t.root)
	return count
}

// Compressed trie: uses map, saves space for sparse alphabets
type CompNode struct {
	children map[byte]*CompNode
	isEnd    bool
}

type CompTrie struct {
	root *CompNode
}

func NewCompTrie() *CompTrie { return &CompTrie{root: &CompNode{children: make(map[byte]*CompNode)}} }

func (t *CompTrie) Insert(word string) {
	node := t.root
	for i := 0; i < len(word); i++ {
		c := word[i]
		if node.children[c] == nil { node.children[c] = &CompNode{children: make(map[byte]*CompNode)} }
		node = node.children[c]
	}
	node.isEnd = true
}

func (t *CompTrie) countNodes() int {
	count := 0
	var dfs func(*CompNode)
	dfs = func(n *CompNode) {
		if n == nil { return }
		count++
		for _, c := range n.children { dfs(c) }
	}
	dfs(t.root)
	return count
}

func main() {
	words := []string{"apple", "app", "application", "apply", "banana", "band", "bat"}

	full := NewFullTrie()
	comp := NewCompTrie()
	for _, w := range words { full.Insert(w); comp.Insert(w) }

	fmt.Println("=== Full Array Trie vs Map Trie ===")
	fmt.Printf("Words: %v\n\n", words)
	fmt.Printf("Full trie nodes: %d (each = 26 pointers = %d bytes)\n", full.countNodes(), 26*8)
	fmt.Printf("Comp trie nodes: %d (each = map overhead, but fewer pointers)\n\n", comp.countNodes())
	fmt.Println("Full trie: faster access, more memory (good for dense alphabets)")
	fmt.Println("Map trie: slower access, less memory (good for sparse alphabets)")
}
```

**Textual Figure:**
```
Full Array Trie vs Map (Compressed) Trie

  Words: ["app", "apple", "bat"]

  Full Array Trie (26 pointers per node):
    (root)
    ├─ a [0:nil, 1:nil, ..., 15:●, ..., 25:nil]  ← 26 ptrs each!
    │  └─ p [all nil except 15:●]
    │     └─ p✓ [all nil except 11:●]
    │        └─ l [all nil except 4:●]
    │           └─ e✓
    └─ b [all nil except 0:●]
       └─ a [all nil except 19:●]
          └─ t✓

    7 nodes × 26 pointers = 182 pointers (mostly nil!)

  Map Trie (only used pointers):
    (root)
    ├─ 'a' → ┌─'p'→┌─'p'✓→┌─'l'→┌─'e'✓
    └─ 'b' → ┌─'a'→┌─'t'✓

    7 nodes, only non-nil entries stored
    Fewer pointers, but map overhead per node

  ┌───────────┬───────────────┬───────────────┐
  │ Trie Type │ Access Speed  │ Memory        │
  ├───────────┼───────────────┼───────────────┤
  │ Full [26] │ O(1) index    │ 26×8B per node│
  │ Map-based │ O(1) avg hash │ ~8B per child │
  └───────────┴───────────────┴───────────────┘
```

---

## Example 9: Bit Manipulation for Space Savings

```go
package main

import "fmt"

// Represent a set of integers 0..63 using a single uint64

type BitSet64 uint64

func (bs *BitSet64) Add(x int)      { *bs |= 1 << x }
func (bs *BitSet64) Remove(x int)   { *bs &^= 1 << x }
func (bs BitSet64) Contains(x int) bool { return bs>>x&1 == 1 }
func (bs BitSet64) Count() int {
	count := 0
	v := uint64(bs)
	for v != 0 { count++; v &= v - 1 }
	return count
}
func (bs BitSet64) Union(other BitSet64) BitSet64       { return bs | other }
func (bs BitSet64) Intersect(other BitSet64) BitSet64    { return bs & other }

func main() {
	var s1, s2 BitSet64
	s1.Add(1); s1.Add(3); s1.Add(5); s1.Add(7)
	s2.Add(3); s2.Add(5); s2.Add(9); s2.Add(11)

	fmt.Println("=== BitSet vs map[int]bool ===\n")
	fmt.Printf("Set 1: contains 3=%v, 4=%v, count=%d\n", s1.Contains(3), s1.Contains(4), s1.Count())
	fmt.Printf("Set 2: contains 9=%v, count=%d\n", s2.Contains(9), s2.Count())

	union := s1.Union(s2)
	inter := s1.Intersect(s2)
	fmt.Printf("Union count: %d\n", union.Count())
	fmt.Printf("Intersection count: %d\n", inter.Count())

	// Space comparison
	mapSet := map[int]bool{1: true, 3: true, 5: true, 7: true}
	_ = mapSet

	fmt.Println("\nBitSet: 8 bytes for up to 64 elements")
	fmt.Println("map[int]bool: ~50+ bytes per element")
	fmt.Println("For n=64: BitSet=8B vs map≈3200B = 400x savings!")
}
```

**Textual Figure:**
```
BitSet vs map[int]bool: Space Comparison

  Set 1: {1, 3, 5, 7}    Set 2: {3, 5, 9, 11}

  BitSet (uint64 = 8 bytes):
    Bit position: 63 ... 11 10 9  8  7  6  5  4  3  2  1  0
    Set 1:         0      0  0 0  0  1  0  1  0  1  0  1  0
    Set 2:         0      1  0 1  0  0  0  1  0  1  0  0  0
    Union (|):     0      1  0 1  0  1  0  1  0  1  0  1  0
    Intersect(&):  0      0  0 0  0  0  0  1  0  1  0  0  0 = {3,5}

  map[int]bool (per element):
    ┌─────┬──────┬──────┬──────┐
    │ key │ hash │ bool │ next │  × each element
    └─────┴──────┴──────┴──────┘
    ~50+ bytes per entry

  Space comparison for 64 elements:
    BitSet:  8 bytes (1 uint64)
    Map:     ~3200 bytes (50B × 64)
    Ratio:   400x savings!

  Operations:
    Union:     O(1) bitwise OR   vs  O(n) map merge
    Intersect: O(1) bitwise AND  vs  O(n) map iterate
    Contains:  O(1) bit check    vs  O(1) map lookup
```

---

## Example 10: Space-Time Tradeoffs Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Space-Time Tradeoffs Decision Guide ===\n")

	decisions := []struct {
		scenario     string
		moreSpace    string
		moreTime     string
		recommendation string
	}{
		{
			"Range sum queries",
			"O(n) prefix sum → O(1) query",
			"O(1) space → O(n) per query",
			"Use prefix sum if queries >> updates",
		},
		{
			"Repeated subproblems",
			"O(n) memo table → O(n) time",
			"No memo → exponential recalculation",
			"Always memoize",
		},
		{
			"Graph edge check",
			"O(V²) adj matrix → O(1) check",
			"O(V+E) adj list → O(degree) check",
			"Matrix for dense, list for sparse",
		},
		{
			"String search",
			"O(m) hash → O(n+m) avg",
			"O(1) → O(nm) worst case",
			"Use rolling hash for repeated searches",
		},
		{
			"Set membership",
			"O(n) hash set → O(1) lookup",
			"O(1) sorted array → O(log n)",
			"Hash set unless memory constrained",
		},
		{
			"Priority queue",
			"O(n) heap → O(log n) ops",
			"O(1) unsorted → O(n) extract-min",
			"Use heap for repeated min/max",
		},
		{
			"Boolean set (small universe)",
			"Bitset: O(n/64) words",
			"map[int]bool: O(n) × overhead",
			"Bitset when universe fits",
		},
	}

	for _, d := range decisions {
		fmt.Printf("Scenario: %s\n", d.scenario)
		fmt.Printf("  More space: %s\n", d.moreSpace)
		fmt.Printf("  More time:  %s\n", d.moreTime)
		fmt.Printf("  → %s\n\n", d.recommendation)
	}

	fmt.Println("--- General Rules ---")
	fmt.Println("• Memory is cheap; use it to avoid recomputation")
	fmt.Println("• But watch cache locality — sometimes O(n²) is faster than O(n log n)")
	fmt.Println("• Precompute anything that's queried more than once")
	fmt.Println("• Compress when data has patterns (RLE, bitpacking)")
	fmt.Println("• Stream when data doesn't fit in memory")
}
```
**Textual Figure:**
```
Space-Time Tradeoffs: Decision Guide

  ┌───────────────────────────────────────────────┐
  │                Space-Time Spectrum                │
  │                                                   │
  │  ◀── More Space             More Time ──▶         │
  │  ◀── Less Time              Less Space ──▶        │
  │                                                   │
  │  Prefix Sum    Hash Set    In-Place Sort  Streaming│
  │  Memo Table    Trie        Bit Manip      Compress │
  │  B-Tree Index  Cache       2-ptr          RLE      │
  └───────────────────────────────────────────────┘

  Decision flowchart:
    Queried > 1 time? ─── Yes → Precompute (prefix sum, table)
         │ No
         ▼
    Memory tight? ─────── Yes → In-place / streaming / bits
         │ No
         ▼
    Need O(1) lookup? ─── Yes → Hash set/map
         │ No
         ▼
    Need ordering? ────── Yes → Sorted + binary search
         │ No
         ▼
    Default: Hash for speed, sort for order
```
---

## Key Takeaways

1. **Prefix sums/lookup tables** trade O(n) space for O(1) query time
2. **Memoization** is almost always worth it — avoids exponential recomputation
3. **Hash tables** trade O(n) space for O(1) average lookup
4. **Bit manipulation** provides massive space savings for small-universe sets
5. **Graph representation** choice depends on density: matrix for dense, list for sparse
6. **Tries**: array-based vs map-based depends on alphabet density
7. **General rule**: spend memory to save time unless memory-constrained

> **Next up:** Iterator Design Patterns →
