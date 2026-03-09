# Phase 23: Design Data Structures — Randomized Data Structures

## Overview

Randomized data structures use **random choices** to achieve good expected performance, often with simpler implementations than their deterministic counterparts.

| Structure | Expected Time | Description |
|-----------|--------------|-------------|
| **Skip List** | O(log n) search/insert/delete | Layered linked lists with probabilistic promotion |
| **Treap** | O(log n) search/insert/delete | BST + heap with random priorities |
| **Bloom Filter** | O(k) membership test | Probabilistic set — no false negatives |
| **Hash Table** | O(1) amortized | Universal hashing for worst-case avoidance |
| **Randomized BST** | O(log n) expected | Random insertion at root |

---

## Example 1: Skip List

```go
package main

import (
	"fmt"
	"math/rand"
	"strings"
)

const maxLevel = 16

type SkipNode struct {
	key     int
	val     int
	forward []*SkipNode
}

type SkipList struct {
	head  *SkipNode
	level int
}

func NewSkipList() *SkipList {
	head := &SkipNode{forward: make([]*SkipNode, maxLevel)}
	return &SkipList{head: head, level: 0}
}

func randomLevel() int {
	lvl := 0
	for lvl < maxLevel-1 && rand.Float64() < 0.5 { lvl++ }
	return lvl
}

func (sl *SkipList) Search(key int) (int, bool) {
	curr := sl.head
	for i := sl.level; i >= 0; i-- {
		for curr.forward[i] != nil && curr.forward[i].key < key {
			curr = curr.forward[i]
		}
	}
	curr = curr.forward[0]
	if curr != nil && curr.key == key { return curr.val, true }
	return 0, false
}

func (sl *SkipList) Insert(key, val int) {
	update := make([]*SkipNode, maxLevel)
	curr := sl.head
	for i := sl.level; i >= 0; i-- {
		for curr.forward[i] != nil && curr.forward[i].key < key {
			curr = curr.forward[i]
		}
		update[i] = curr
	}
	curr = curr.forward[0]

	if curr != nil && curr.key == key {
		curr.val = val
		return
	}

	lvl := randomLevel()
	if lvl > sl.level {
		for i := sl.level + 1; i <= lvl; i++ { update[i] = sl.head }
		sl.level = lvl
	}

	node := &SkipNode{key: key, val: val, forward: make([]*SkipNode, lvl+1)}
	for i := 0; i <= lvl; i++ {
		node.forward[i] = update[i].forward[i]
		update[i].forward[i] = node
	}
}

func (sl *SkipList) Delete(key int) bool {
	update := make([]*SkipNode, maxLevel)
	curr := sl.head
	for i := sl.level; i >= 0; i-- {
		for curr.forward[i] != nil && curr.forward[i].key < key {
			curr = curr.forward[i]
		}
		update[i] = curr
	}
	curr = curr.forward[0]
	if curr == nil || curr.key != key { return false }

	for i := 0; i <= sl.level; i++ {
		if update[i].forward[i] != curr { break }
		update[i].forward[i] = curr.forward[i]
	}
	for sl.level > 0 && sl.head.forward[sl.level] == nil { sl.level-- }
	return true
}

func (sl *SkipList) String() string {
	var sb strings.Builder
	for i := sl.level; i >= 0; i-- {
		sb.WriteString(fmt.Sprintf("Level %d: ", i))
		curr := sl.head.forward[i]
		for curr != nil {
			sb.WriteString(fmt.Sprintf("%d→", curr.key))
			curr = curr.forward[i]
		}
		sb.WriteString("nil\n")
	}
	return sb.String()
}

func main() {
	sl := NewSkipList()
	for _, k := range []int{3, 6, 7, 9, 12, 19, 17, 26, 21, 25} {
		sl.Insert(k, k*10)
	}

	fmt.Println(sl.String())

	val, found := sl.Search(17)
	fmt.Printf("Search 17: val=%d, found=%v\n", val, found)

	sl.Delete(17)
	_, found = sl.Search(17)
	fmt.Printf("After delete 17: found=%v\n\n", found)

	fmt.Println("Skip list: O(log n) expected, O(n) worst case")
}
```

---

## Example 2: Treap (BST + Heap)

```go
package main

import (
	"fmt"
	"math/rand"
)

type TreapNode struct {
	key, priority int
	size          int
	left, right   *TreapNode
}

func newTreapNode(key int) *TreapNode {
	return &TreapNode{key: key, priority: rand.Int(), size: 1}
}

func sz(t *TreapNode) int { if t == nil { return 0 }; return t.size }
func pull(t *TreapNode)   { if t != nil { t.size = 1 + sz(t.left) + sz(t.right) } }

func rotateRight(t *TreapNode) *TreapNode {
	s := t.left; t.left = s.right; s.right = t
	pull(t); pull(s); return s
}

func rotateLeft(t *TreapNode) *TreapNode {
	s := t.right; t.right = s.left; s.left = t
	pull(t); pull(s); return s
}

func insert(t *TreapNode, key int) *TreapNode {
	if t == nil { return newTreapNode(key) }
	if key < t.key {
		t.left = insert(t.left, key)
		if t.left.priority > t.priority { t = rotateRight(t) }
	} else if key > t.key {
		t.right = insert(t.right, key)
		if t.right.priority > t.priority { t = rotateLeft(t) }
	}
	pull(t); return t
}

func erase(t *TreapNode, key int) *TreapNode {
	if t == nil { return nil }
	if key < t.key { t.left = erase(t.left, key) } else if key > t.key {
		t.right = erase(t.right, key)
	} else {
		if t.left == nil { return t.right }
		if t.right == nil { return t.left }
		if t.left.priority > t.right.priority {
			t = rotateRight(t); t.right = erase(t.right, key)
		} else {
			t = rotateLeft(t); t.left = erase(t.left, key)
		}
	}
	pull(t); return t
}

func kth(t *TreapNode, k int) int { // 0-indexed
	leftSz := sz(t.left)
	if k < leftSz { return kth(t.left, k) }
	if k == leftSz { return t.key }
	return kth(t.right, k-leftSz-1)
}

func countLess(t *TreapNode, key int) int {
	if t == nil { return 0 }
	if key <= t.key { return countLess(t.left, key) }
	return sz(t.left) + 1 + countLess(t.right, key)
}

func inorder(t *TreapNode) []int {
	if t == nil { return nil }
	result := inorder(t.left)
	result = append(result, t.key)
	result = append(result, inorder(t.right)...)
	return result
}

func main() {
	var root *TreapNode
	keys := []int{5, 2, 8, 1, 4, 7, 9, 3, 6}
	for _, k := range keys { root = insert(root, k) }

	fmt.Println("In-order:", inorder(root))
	fmt.Println("Size:", sz(root))
	fmt.Println("3rd smallest (0-indexed):", kth(root, 2))
	fmt.Println("Count < 6:", countLess(root, 6))

	root = erase(root, 5)
	fmt.Println("After delete 5:", inorder(root))

	fmt.Println("\nTreap: random priorities ensure O(log n) expected height")
}
```

---

## Example 3: Bloom Filter

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
)

type BloomFilter struct {
	bits []bool
	k    int // number of hash functions
	m    int // number of bits
}

func NewBloomFilter(expectedN int, fpRate float64) *BloomFilter {
	m := int(-float64(expectedN) * math.Log(fpRate) / (math.Log(2) * math.Log(2)))
	k := int(float64(m) / float64(expectedN) * math.Log(2))
	if k < 1 { k = 1 }
	return &BloomFilter{bits: make([]bool, m), k: k, m: m}
}

func (bf *BloomFilter) hash(data string, seed int) int {
	h := fnv.New64a()
	h.Write([]byte(fmt.Sprintf("%d:%s", seed, data)))
	return int(h.Sum64() % uint64(bf.m))
}

func (bf *BloomFilter) Add(item string) {
	for i := 0; i < bf.k; i++ { bf.bits[bf.hash(item, i)] = true }
}

func (bf *BloomFilter) MayContain(item string) bool {
	for i := 0; i < bf.k; i++ {
		if !bf.bits[bf.hash(item, i)] { return false }
	}
	return true // might be false positive
}

func main() {
	bf := NewBloomFilter(1000, 0.01) // 1% false positive rate
	fmt.Printf("Bloom filter: %d bits, %d hashes\n\n", bf.m, bf.k)

	words := []string{"apple", "banana", "cherry", "date", "elderberry"}
	for _, w := range words { bf.Add(w) }

	// Check membership
	for _, w := range words {
		fmt.Printf("  Contains '%s'? %v ✓\n", w, bf.MayContain(w))
	}

	// Check non-members
	nonMembers := []string{"fig", "grape", "kiwi", "mango"}
	falsePos := 0
	for _, w := range nonMembers {
		result := bf.MayContain(w)
		if result { falsePos++ }
		fmt.Printf("  Contains '%s'? %v\n", w, result)
	}

	fmt.Println("\nNo false negatives guaranteed!")
	fmt.Println("False positive rate depends on m/n ratio and k")
}
```

---

## Example 4: Count-Min Sketch

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
)

// Probabilistic frequency counter: slight overcount, never undercount

type CountMinSketch struct {
	table [][]int
	d     int // rows (hash functions)
	w     int // columns (width)
}

func NewCMS(epsilon, delta float64) *CountMinSketch {
	w := int(math.Ceil(math.E / epsilon))
	d := int(math.Ceil(math.Log(1.0 / delta)))
	table := make([][]int, d)
	for i := range table { table[i] = make([]int, w) }
	return &CountMinSketch{table: table, d: d, w: w}
}

func (cms *CountMinSketch) hash(key string, row int) int {
	h := fnv.New64a()
	h.Write([]byte(fmt.Sprintf("%d:%s", row, key)))
	return int(h.Sum64() % uint64(cms.w))
}

func (cms *CountMinSketch) Add(key string, count int) {
	for i := 0; i < cms.d; i++ {
		cms.table[i][cms.hash(key, i)] += count
	}
}

func (cms *CountMinSketch) Estimate(key string) int {
	minCount := math.MaxInt64
	for i := 0; i < cms.d; i++ {
		c := cms.table[i][cms.hash(key, i)]
		if c < minCount { minCount = c }
	}
	return minCount
}

func main() {
	cms := NewCMS(0.001, 0.01) // ε=0.1%, δ=1%
	fmt.Printf("CMS: %d rows × %d cols\n\n", cms.d, cms.w)

	data := map[string]int{"apple": 100, "banana": 50, "cherry": 200, "date": 10}
	for word, count := range data {
		for i := 0; i < count; i++ { cms.Add(word, 1) }
	}

	for word, actual := range data {
		est := cms.Estimate(word)
		fmt.Printf("  %-8s actual=%3d, estimate=%3d (error=%d)\n", word, actual, est, est-actual)
	}

	fmt.Println("\nNever undercounts! May overcount by ε·N")
}
```

---

## Example 5: Randomized Quick Select (K-th Smallest)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Expected O(n) vs deterministic O(n) median-of-medians

func quickSelect(arr []int, k int) int {
	if len(arr) == 1 { return arr[0] }

	// Random pivot
	pivotIdx := rand.Intn(len(arr))
	pivot := arr[pivotIdx]

	var lo, eq, hi []int
	for _, v := range arr {
		if v < pivot { lo = append(lo, v) } else if v == pivot { eq = append(eq, v) } else { hi = append(hi, v) }
	}

	if k < len(lo) { return quickSelect(lo, k) }
	if k < len(lo)+len(eq) { return pivot }
	return quickSelect(hi, k-len(lo)-len(eq))
}

func main() {
	arr := []int{7, 10, 4, 3, 20, 15, 8, 1, 6, 12}
	fmt.Println("Array:", arr)

	for k := 0; k < len(arr); k++ {
		result := quickSelect(append([]int{}, arr...), k)
		fmt.Printf("  %d-th smallest = %d\n", k+1, result)
	}

	fmt.Println("\nRandomized pivot → O(n) expected, O(n²) worst case")
	fmt.Println("Deterministic median-of-medians → O(n) worst case but larger constant")
}
```

---

## Example 6: Randomized Treap Split/Merge (Implicit)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Implicit treap: array with O(log n) split/merge
// Supports: insert at position, delete at position, reverse range

type ImplicitNode struct {
	priority    int
	val         int
	size        int
	rev         bool
	left, right *ImplicitNode
}

func newImplicit(val int) *ImplicitNode {
	return &ImplicitNode{val: val, priority: rand.Int(), size: 1}
}

func sz2(t *ImplicitNode) int { if t == nil { return 0 }; return t.size }

func pull2(t *ImplicitNode) {
	if t != nil { t.size = 1 + sz2(t.left) + sz2(t.right) }
}

func push2(t *ImplicitNode) {
	if t != nil && t.rev {
		t.left, t.right = t.right, t.left
		if t.left != nil { t.left.rev = !t.left.rev }
		if t.right != nil { t.right.rev = !t.right.rev }
		t.rev = false
	}
}

func split(t *ImplicitNode, pos int) (*ImplicitNode, *ImplicitNode) {
	if t == nil { return nil, nil }
	push2(t)
	leftSz := sz2(t.left)
	if pos <= leftSz {
		l, r := split(t.left, pos)
		t.left = r; pull2(t)
		return l, t
	}
	l, r := split(t.right, pos-leftSz-1)
	t.right = l; pull2(t)
	return t, r
}

func merge(l, r *ImplicitNode) *ImplicitNode {
	push2(l); push2(r)
	if l == nil { return r }
	if r == nil { return l }
	if l.priority > r.priority {
		l.right = merge(l.right, r); pull2(l); return l
	}
	r.left = merge(l, r.left); pull2(r); return r
}

func insertAt(root *ImplicitNode, pos, val int) *ImplicitNode {
	l, r := split(root, pos)
	return merge(merge(l, newImplicit(val)), r)
}

func deleteAt(root *ImplicitNode, pos int) *ImplicitNode {
	l, mr := split(root, pos)
	_, r := split(mr, 1)
	return merge(l, r)
}

func reverseRange(root *ImplicitNode, l, r int) *ImplicitNode {
	a, bc := split(root, l)
	b, c := split(bc, r-l+1)
	b.rev = !b.rev
	return merge(merge(a, b), c)
}

func toArray(t *ImplicitNode) []int {
	if t == nil { return nil }
	push2(t)
	result := toArray(t.left)
	result = append(result, t.val)
	result = append(result, toArray(t.right)...)
	return result
}

func main() {
	var root *ImplicitNode
	for _, v := range []int{1, 2, 3, 4, 5, 6, 7, 8} {
		root = merge(root, newImplicit(v))
	}
	fmt.Println("Array:", toArray(root))

	root = insertAt(root, 3, 99)
	fmt.Println("Insert 99 at pos 3:", toArray(root))

	root = deleteAt(root, 3)
	fmt.Println("Delete at pos 3:", toArray(root))

	root = reverseRange(root, 2, 5)
	fmt.Println("Reverse [2,5]:", toArray(root))

	fmt.Println("\nImplicit treap: rope-like operations in O(log n)")
}
```

---

## Example 7: Hash Table with Universal Hashing

```go
package main

import (
	"fmt"
	"math/rand"
)

// Universal hash family: h(x) = ((a*x + b) mod p) mod m
// For any x ≠ y: Pr[h(x)=h(y)] ≤ 1/m

type UniversalHashMap struct {
	m       int
	a, b    int64
	p       int64
	buckets [][]struct{ key, val int }
}

func NewUniversalHashMap(size int) *UniversalHashMap {
	p := int64(1_000_000_007) // large prime
	a := int64(rand.Intn(int(p)-1)) + 1
	b := int64(rand.Intn(int(p)))
	return &UniversalHashMap{
		m: size, a: a, b: b, p: p,
		buckets: make([][]struct{ key, val int }, size),
	}
}

func (hm *UniversalHashMap) hash(key int) int {
	h := ((hm.a*int64(key) + hm.b) % hm.p)
	if h < 0 { h += hm.p }
	return int(h % int64(hm.m))
}

func (hm *UniversalHashMap) Put(key, val int) {
	h := hm.hash(key)
	for i := range hm.buckets[h] {
		if hm.buckets[h][i].key == key { hm.buckets[h][i].val = val; return }
	}
	hm.buckets[h] = append(hm.buckets[h], struct{ key, val int }{key, val})
}

func (hm *UniversalHashMap) Get(key int) (int, bool) {
	h := hm.hash(key)
	for _, kv := range hm.buckets[h] {
		if kv.key == key { return kv.val, true }
	}
	return 0, false
}

func main() {
	hm := NewUniversalHashMap(16)
	for i := 0; i < 20; i++ { hm.Put(i*16, i) } // worst case for mod-16 hash

	fmt.Println("Universal hashing distributes even adversarial inputs:")
	maxBucket := 0
	for _, b := range hm.buckets {
		if len(b) > maxBucket { maxBucket = len(b) }
	}
	fmt.Printf("Max bucket size: %d (with 20 items in 16 buckets)\n", maxBucket)

	for i := 0; i < 5; i++ {
		v, _ := hm.Get(i * 16)
		fmt.Printf("  Get(%d) = %d\n", i*16, v)
	}

	fmt.Println("\nUniversal hashing: protects against adversarial inputs")
}
```

---

## Example 8: Randomized Set with Weighted Sampling

```go
package main

import (
	"fmt"
	"math/rand"
)

// Weighted random selection: each item has a weight
// Uses alias method for O(1) sampling after O(n) preprocessing

type WeightedSampler struct {
	n     int
	prob  []float64
	alias []int
}

func NewWeightedSampler(weights []float64) *WeightedSampler {
	n := len(weights)
	total := 0.0
	for _, w := range weights { total += w }

	prob := make([]float64, n)
	alias := make([]int, n)
	scaled := make([]float64, n)
	for i, w := range weights { scaled[i] = w * float64(n) / total }

	var small, large []int
	for i, s := range scaled {
		if s < 1.0 { small = append(small, i) } else { large = append(large, i) }
	}

	for len(small) > 0 && len(large) > 0 {
		s := small[len(small)-1]; small = small[:len(small)-1]
		l := large[len(large)-1]; large = large[:len(large)-1]
		prob[s] = scaled[s]
		alias[s] = l
		scaled[l] = scaled[l] + scaled[s] - 1.0
		if scaled[l] < 1.0 { small = append(small, l) } else { large = append(large, l) }
	}
	for _, l := range large { prob[l] = 1.0 }
	for _, s := range small { prob[s] = 1.0 }

	return &WeightedSampler{n, prob, alias}
}

func (ws *WeightedSampler) Sample() int {
	i := rand.Intn(ws.n)
	if rand.Float64() < ws.prob[i] { return i }
	return ws.alias[i]
}

func main() {
	weights := []float64{1, 2, 3, 4} // probabilities: 10%, 20%, 30%, 40%
	ws := NewWeightedSampler(weights)

	counts := make([]int, len(weights))
	trials := 100000
	for i := 0; i < trials; i++ { counts[ws.Sample()]++ }

	fmt.Println("Weighted sampling (Alias method): O(n) build, O(1) sample\n")
	fmt.Println("Weights:", weights)
	for i, c := range counts {
		fmt.Printf("  Item %d: weight=%.0f, sampled=%d (%.1f%%)\n", i, weights[i], c, float64(c)/float64(trials)*100)
	}
}
```

---

## Example 9: Random Shuffle Verification (Fisher-Yates)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Verify Fisher-Yates produces uniform permutations

func fisherYates(arr []int) {
	for i := len(arr) - 1; i > 0; i-- {
		j := rand.Intn(i + 1)
		arr[i], arr[j] = arr[j], arr[i]
	}
}

// WRONG shuffle for comparison
func naiveShuffle(arr []int) {
	n := len(arr)
	for i := 0; i < n; i++ {
		j := rand.Intn(n) // bug: should be rand.Intn(i+1) or similar
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	n := 3
	trials := 300000

	// Count permutation frequencies for Fisher-Yates
	fyCount := map[string]int{}
	for t := 0; t < trials; t++ {
		arr := []int{0, 1, 2}
		fisherYates(arr)
		fyCount[fmt.Sprint(arr)]++
	}

	// Count for naive shuffle
	naiveCount := map[string]int{}
	for t := 0; t < trials; t++ {
		arr := []int{0, 1, 2}
		naiveShuffle(arr)
		naiveCount[fmt.Sprint(arr)]++
	}

	expected := trials / 6 // 3! = 6 permutations
	fmt.Printf("Expected per permutation: %d\n\n", expected)

	fmt.Println("Fisher-Yates (uniform):")
	for perm, count := range fyCount {
		fmt.Printf("  %s: %d (%.1f%%)\n", perm, count, float64(count)/float64(trials)*100)
	}

	fmt.Println("\nNaive shuffle (biased!):")
	for perm, count := range naiveCount {
		fmt.Printf("  %s: %d (%.1f%%)\n", perm, count, float64(count)/float64(trials)*100)
	}
}
```

---

## Example 10: Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Randomized Data Structures Patterns ===\n")

	patterns := []struct {
		name, complexity, keyIdea string
	}{
		{"Skip List", "O(log n) expected", "Probabilistic level promotion"},
		{"Treap", "O(log n) expected", "Random priority maintains balance"},
		{"Implicit Treap", "O(log n) expected", "Split/merge for array operations"},
		{"Bloom Filter", "O(k) query", "Multiple hashes, no false negatives"},
		{"Count-Min Sketch", "O(d) query", "Minimum of d counters, never undercounts"},
		{"Quick Select", "O(n) expected", "Random pivot for k-th element"},
		{"Universal Hashing", "O(1) expected", "Random hash avoids adversarial collisions"},
		{"Alias Method", "O(1) sample", "Precomputed probability table"},
		{"Fisher-Yates", "O(n) shuffle", "Each permutation equally likely"},
		{"Reservoir Sampling", "O(n) pass", "Uniform sample from stream"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-18s %16s — %s\n", p.name, p.complexity, p.keyIdea)
	}

	fmt.Println("\n--- When to Use Randomization ---")
	fmt.Println("✅ Simpler implementation than deterministic alternative")
	fmt.Println("✅ Good expected performance with high probability")
	fmt.Println("✅ Adversary-resistant (can't construct worst case)")
	fmt.Println("⚠️  Non-deterministic: different runs may give different results")
	fmt.Println("⚠️  Worst case still exists (just very unlikely)")
}
```

---

## Key Takeaways

1. **Skip List**: simpler than balanced BSTs, O(log n) expected for all operations
2. **Treap**: random priorities automatically balance the BST
3. **Implicit Treap**: split/merge enables rope-like array operations
4. **Bloom Filter**: space-efficient set membership — no false negatives
5. **Count-Min Sketch**: approximate frequency counting in streaming
6. **Quick Select**: O(n) expected k-th element with random pivot
7. **Universal Hashing**: protects against adversarial inputs
8. **Alias Method**: O(1) weighted sampling after O(n) preprocessing

> **Phase 23 Complete!** Next up: Phase 24 — Divide & Conquer →
