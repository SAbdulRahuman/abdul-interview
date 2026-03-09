# Phase 9: BST — Self-Balancing BST Trade-offs

## Overview

Each self-balancing BST makes different trade-offs between strictness of balance, cost of maintenance, and practical performance.

| Tree | Balance Guarantee | Insert Cost | Delete Cost | Search Cost | Extra Space |
|------|-------------------|-------------|-------------|-------------|-------------|
| **AVL** | |h_L - h_R| ≤ 1 | O(log n), ≤2 rotations | O(log n), O(log n) rotations | O(log n) | Height per node |
| **Red-Black** | h ≤ 2 log₂(n+1) | O(log n), ≤2 rotations | O(log n), ≤3 rotations | O(log n) | 1 bit color |
| **Splay** | Amortized O(log n) | Amortized O(log n) | Amortized O(log n) | Amortized O(log n) | None |
| **Treap** | Expected O(log n) | Expected O(log n) | Expected O(log n) | Expected O(log n) | Priority |
| **B-Tree** | All leaves same depth | O(log n) | O(log n) | O(log n) | Children array |
| **Skip List** | Expected O(log n) | Expected O(log n) | Expected O(log n) | Expected O(log n) | Forward ptrs |

---

## Example 1: AVL — Strict Balance, Great for Reads

```go
package main

import (
	"fmt"
	"time"
)

type AVL struct{ Val, H int; L, R *AVL }
func h(n *AVL) int { if n == nil { return 0 }; return n.H }
func mx(a, b int) int { if a > b { return a }; return b }
func up(n *AVL) { n.H = 1 + mx(h(n.L), h(n.R)) }
func bf(n *AVL) int { return h(n.L) - h(n.R) }
func rr(y *AVL) *AVL { x := y.L; y.L = x.R; x.R = y; up(y); up(x); return x }
func lr(x *AVL) *AVL { y := x.R; x.R = y.L; y.L = x; up(x); up(y); return y }
func bal(n *AVL) *AVL { up(n); b := bf(n); if b > 1 { if bf(n.L) < 0 { n.L = lr(n.L) }; return rr(n) }; if b < -1 { if bf(n.R) > 0 { n.R = rr(n.R) }; return lr(n) }; return n }
func ins(n *AVL, v int) *AVL { if n == nil { return &AVL{Val: v, H: 1} }; if v < n.Val { n.L = ins(n.L, v) } else if v > n.Val { n.R = ins(n.R, v) }; return bal(n) }
func search(n *AVL, v int) bool { for n != nil { if v == n.Val { return true }; if v < n.Val { n = n.L } else { n = n.R } }; return false }

func main() {
	var root *AVL
	n := 100000
	start := time.Now()
	for i := 0; i < n; i++ { root = ins(root, i) }
	insertTime := time.Since(start)

	start = time.Now()
	for i := 0; i < n; i++ { search(root, i) }
	searchTime := time.Since(start)

	fmt.Printf("AVL: n=%d height=%d insert=%v search=%v\n", n, h(root), insertTime, searchTime)
	fmt.Println("Strength: Fastest searches (lowest height)")
	fmt.Println("Weakness: More rotations on delete")
}
```

---

## Example 2: Red-Black — Fewer Rotations, Good All-Round

```go
package main

import (
	"fmt"
	"time"
)

const (RED = true; BLACK = false)
type RB struct { Val int; L, R *RB; C bool }
func red(n *RB) bool { return n != nil && n.C }
func rlr(h *RB) *RB { x := h.R; h.R = x.L; x.L = h; x.C = h.C; h.C = RED; return x }
func rrr(h *RB) *RB { x := h.L; h.L = x.R; x.R = h; x.C = h.C; h.C = RED; return x }
func flip(h *RB) { h.C = !h.C; h.L.C = !h.L.C; h.R.C = !h.R.C }
func rbIns(h *RB, v int) *RB {
	if h == nil { return &RB{Val: v, C: RED} }
	if v < h.Val { h.L = rbIns(h.L, v) } else if v > h.Val { h.R = rbIns(h.R, v) }
	if red(h.R) && !red(h.L) { h = rlr(h) }
	if red(h.L) && red(h.L.L) { h = rrr(h) }
	if red(h.L) && red(h.R) { flip(h) }
	return h
}
func rbSearch(n *RB, v int) bool { for n != nil { if v == n.Val { return true }; if v < n.Val { n = n.L } else { n = n.R } }; return false }
func rbH(n *RB) int { if n == nil { return 0 }; l, r := rbH(n.L), rbH(n.R); if l > r { return l + 1 }; return r + 1 }

func main() {
	var root *RB
	n := 100000
	start := time.Now()
	for i := 0; i < n; i++ { root = rbIns(root, i); root.C = BLACK }
	insertTime := time.Since(start)

	start = time.Now()
	for i := 0; i < n; i++ { rbSearch(root, i) }
	searchTime := time.Since(start)

	fmt.Printf("RB:  n=%d height=%d insert=%v search=%v\n", n, rbH(root), insertTime, searchTime)
	fmt.Println("Strength: ≤3 rotations per delete, widely used")
	fmt.Println("Weakness: Taller than AVL (up to 2x)")
}
```

---

## Example 3: Splay Tree — Amortized, Great for Temporal Locality

```go
package main

import "fmt"

type Splay struct{ Val int; L, R *Splay }

func splay(root *Splay, key int) *Splay {
	if root == nil || root.Val == key { return root }
	if key < root.Val {
		if root.L == nil { return root }
		if key < root.L.Val {
			root.L.L = splay(root.L.L, key)
			root = rotateR(root)
		} else if key > root.L.Val {
			root.L.R = splay(root.L.R, key)
			if root.L.R != nil { root.L = rotateL(root.L) }
		}
		if root.L == nil { return root }
		return rotateR(root)
	} else {
		if root.R == nil { return root }
		if key > root.R.Val {
			root.R.R = splay(root.R.R, key)
			root = rotateL(root)
		} else if key < root.R.Val {
			root.R.L = splay(root.R.L, key)
			if root.R.L != nil { root.R = rotateR(root.R) }
		}
		if root.R == nil { return root }
		return rotateL(root)
	}
}

func rotateR(y *Splay) *Splay { x := y.L; y.L = x.R; x.R = y; return x }
func rotateL(x *Splay) *Splay { y := x.R; x.R = y.L; y.L = x; return y }

func insert(root *Splay, key int) *Splay {
	if root == nil { return &Splay{Val: key} }
	root = splay(root, key)
	if root.Val == key { return root }
	n := &Splay{Val: key}
	if key < root.Val { n.R = root; n.L = root.L; root.L = nil } else { n.L = root; n.R = root.R; root.R = nil }
	return n
}

func search(root *Splay, key int) (*Splay, bool) {
	root = splay(root, key)
	return root, root != nil && root.Val == key
}

func main() {
	var root *Splay
	for i := 1; i <= 10; i++ { root = insert(root, i) }

	// Access 3 repeatedly — it stays near root
	for i := 0; i < 5; i++ {
		var found bool
		root, found = search(root, 3)
		fmt.Printf("Search 3: found=%v root=%d\n", found, root.Val)
	}
	fmt.Println("Strength: Recently accessed items near root")
	fmt.Println("Weakness: Individual ops can be O(n)")
}
```

---

## Example 4: Treap — Simple Randomized Balance

```go
package main

import (
	"fmt"
	"math/rand"
)

type Treap struct {
	Val, Pri int
	L, R     *Treap
}

func rotateR(y *Treap) *Treap { x := y.L; y.L = x.R; x.R = y; return x }
func rotateL(x *Treap) *Treap { y := x.R; x.R = y.L; y.L = x; return y }

func insert(root *Treap, val int) *Treap {
	if root == nil { return &Treap{Val: val, Pri: rand.Int()} }
	if val < root.Val {
		root.L = insert(root.L, val)
		if root.L.Pri > root.Pri { root = rotateR(root) }
	} else if val > root.Val {
		root.R = insert(root.R, val)
		if root.R.Pri > root.Pri { root = rotateL(root) }
	}
	return root
}

func height(n *Treap) int {
	if n == nil { return 0 }
	l, r := height(n.L), height(n.R)
	if l > r { return l + 1 }
	return r + 1
}

func main() {
	var root *Treap
	for i := 1; i <= 100000; i++ {
		root = insert(root, i)
	}
	fmt.Printf("Treap: 100k sequential inserts, height=%d\n", height(root))
	fmt.Println("Strength: Simple implementation, good expected bounds")
	fmt.Println("Weakness: Not deterministic, worst case O(n)")
}
```

---

## Example 5: Skip List — Alternative to Balanced BST

```go
package main

import (
	"fmt"
	"math/rand"
)

const maxLevel = 16

type SkipNode struct {
	Val     int
	Forward []*SkipNode
}

type SkipList struct {
	Head  *SkipNode
	Level int
}

func NewSkipList() *SkipList {
	return &SkipList{Head: &SkipNode{Forward: make([]*SkipNode, maxLevel)}, Level: 0}
}

func randomLevel() int {
	lvl := 0
	for lvl < maxLevel-1 && rand.Float64() < 0.5 { lvl++ }
	return lvl
}

func (sl *SkipList) Insert(val int) {
	update := make([]*SkipNode, maxLevel)
	cur := sl.Head
	for i := sl.Level; i >= 0; i-- {
		for cur.Forward[i] != nil && cur.Forward[i].Val < val { cur = cur.Forward[i] }
		update[i] = cur
	}
	lvl := randomLevel()
	if lvl > sl.Level {
		for i := sl.Level + 1; i <= lvl; i++ { update[i] = sl.Head }
		sl.Level = lvl
	}
	newNode := &SkipNode{Val: val, Forward: make([]*SkipNode, lvl+1)}
	for i := 0; i <= lvl; i++ {
		newNode.Forward[i] = update[i].Forward[i]
		update[i].Forward[i] = newNode
	}
}

func (sl *SkipList) Search(val int) bool {
	cur := sl.Head
	for i := sl.Level; i >= 0; i-- {
		for cur.Forward[i] != nil && cur.Forward[i].Val < val { cur = cur.Forward[i] }
	}
	cur = cur.Forward[0]
	return cur != nil && cur.Val == val
}

func main() {
	sl := NewSkipList()
	for i := 1; i <= 20; i++ { sl.Insert(i) }
	fmt.Println("Search 10:", sl.Search(10))  // true
	fmt.Println("Search 25:", sl.Search(25))  // false
	fmt.Println("Skip list levels:", sl.Level)
	fmt.Println("Strength: Lock-free variants, simple concurrent access")
	fmt.Println("Weakness: More space (forward pointers)")
}
```

---

## Example 6: B-Tree Properties (Used in Databases)

```go
package main

import "fmt"

// A simplified 2-3 tree (B-tree of order 3) concept demo
type BTreeNode struct {
	Keys     []int
	Children []*BTreeNode
	Leaf     bool
}

func search(node *BTreeNode, key int) bool {
	if node == nil { return false }
	i := 0
	for i < len(node.Keys) && key > node.Keys[i] { i++ }
	if i < len(node.Keys) && key == node.Keys[i] { return true }
	if node.Leaf { return false }
	return search(node.Children[i], key)
}

func main() {
	// Manual 2-3 tree:     [10, 20]
	//                     /   |   \
	//                  [5]  [15]  [25,30]
	root := &BTreeNode{
		Keys: []int{10, 20},
		Children: []*BTreeNode{
			{Keys: []int{5}, Leaf: true},
			{Keys: []int{15}, Leaf: true},
			{Keys: []int{25, 30}, Leaf: true},
		},
	}
	for _, k := range []int{5, 10, 15, 20, 25, 30, 12} {
		fmt.Printf("Search(%d): %v\n", k, search(root, k))
	}
	fmt.Println("\nStrength: Minimizes disk I/O, all leaves same depth")
	fmt.Println("Weakness: Complex implementation, overkill for in-memory")
}
```

---

## Example 7: When to Use Which — Decision Framework

```go
package main

import "fmt"

func main() {
	scenarios := []struct {
		Scenario string
		Best     string
		Why      string
	}{
		{
			"Read-heavy, rarely updated data",
			"AVL Tree",
			"Stricter balance → shorter height → faster lookups",
		},
		{
			"Frequent inserts and deletes",
			"Red-Black Tree",
			"Fewer rotations per update (≤3 per delete vs O(log n))",
		},
		{
			"Cache-like access (80/20 rule)",
			"Splay Tree",
			"Recently accessed items bubble to root, no extra space",
		},
		{
			"Need simplicity + good average case",
			"Treap",
			"Randomized priorities, simple code, expected O(log n)",
		},
		{
			"Disk-based storage / database index",
			"B-Tree / B+ Tree",
			"Multi-key nodes minimize disk reads",
		},
		{
			"Concurrent sorted data structure",
			"Skip List",
			"Lock-free variants possible, simpler concurrent access",
		},
		{
			"Need sorted iteration in Go",
			"AVL or RB Tree (custom)",
			"Go's built-in map is hash-based; implement tree for sorted order",
		},
		{
			"Competitive programming",
			"Treap or Splay",
			"Short code, implicit key treap is very versatile",
		},
	}

	fmt.Println("=== Self-Balancing BST Decision Guide ===")
	for i, s := range scenarios {
		fmt.Printf("\n%d. Scenario: %s\n   Best: %s\n   Why: %s\n", i+1, s.Scenario, s.Best, s.Why)
	}
}
```

---

## Example 8: Memory Overhead Comparison

```go
package main

import (
	"fmt"
	"unsafe"
)

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

type RBNode struct {
	Val   int
	Left  *RBNode
	Right *RBNode
	Color bool
}

type TreapNode struct {
	Val      int
	Priority int
	Left     *TreapNode
	Right    *TreapNode
}

type SplayNode struct {
	Val   int
	Left  *SplayNode
	Right *SplayNode
}

func main() {
	fmt.Println("=== Memory per Node ===")
	fmt.Printf("AVL:   %d bytes (Val + Left + Right + Height)\n", unsafe.Sizeof(AVLNode{}))
	fmt.Printf("RB:    %d bytes (Val + Left + Right + Color)\n", unsafe.Sizeof(RBNode{}))
	fmt.Printf("Treap: %d bytes (Val + Priority + Left + Right)\n", unsafe.Sizeof(TreapNode{}))
	fmt.Printf("Splay: %d bytes (Val + Left + Right)\n", unsafe.Sizeof(SplayNode{}))

	fmt.Println("\n=== For 1M nodes ===")
	n := 1_000_000
	fmt.Printf("AVL:   %.1f MB\n", float64(n)*float64(unsafe.Sizeof(AVLNode{}))/1e6)
	fmt.Printf("RB:    %.1f MB\n", float64(n)*float64(unsafe.Sizeof(RBNode{}))/1e6)
	fmt.Printf("Treap: %.1f MB\n", float64(n)*float64(unsafe.Sizeof(TreapNode{}))/1e6)
	fmt.Printf("Splay: %.1f MB\n", float64(n)*float64(unsafe.Sizeof(SplayNode{}))/1e6)
}
```

---

## Example 9: Worst Case Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Worst-Case Height for n=1,000,000 ===")
	n := 1_000_000

	fmt.Printf("BST (no balancing):   %d\n", n) // degenerate

	// AVL: h ≤ 1.44 * log2(n)
	avlHeight := int(1.44 * 20) // log2(1M) ≈ 20
	fmt.Printf("AVL:                  %d\n", avlHeight)

	// RB: h ≤ 2 * log2(n+1)
	rbHeight := 2 * 20
	fmt.Printf("Red-Black:            %d\n", rbHeight)

	// Splay: worst case single op is O(n), but amortized O(log n)
	fmt.Printf("Splay (single op):    %d (amortized: ~%d)\n", n, 20)

	// Treap: expected O(log n) but worst case O(n)
	fmt.Printf("Treap (expected):     ~%d (worst: %d)\n", 20, n)

	fmt.Println("\n=== Rotations per Operation ===")
	fmt.Printf("%-15s %-20s %-20s\n", "Tree", "Insert Rotations", "Delete Rotations")
	fmt.Printf("%-15s %-20s %-20s\n", "AVL", "≤ 2", "O(log n)")
	fmt.Printf("%-15s %-20s %-20s\n", "Red-Black", "≤ 2", "≤ 3")
	fmt.Printf("%-15s %-20s %-20s\n", "Splay", "O(log n) amortized", "O(log n) amortized")
	fmt.Printf("%-15s %-20s %-20s\n", "Treap", "O(log n) expected", "O(log n) expected")
}
```

---

## Example 10: Practical Benchmark — All Trees

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// ---- AVL ----
type AVL struct{ V, H int; L, R *AVL }
func avlH(n *AVL) int { if n == nil { return 0 }; return n.H }
func avlMx(a, b int) int { if a > b { return a }; return b }
func avlUp(n *AVL) { n.H = 1 + avlMx(avlH(n.L), avlH(n.R)) }
func avlBf(n *AVL) int { return avlH(n.L) - avlH(n.R) }
func avlRR(y *AVL) *AVL { x := y.L; y.L = x.R; x.R = y; avlUp(y); avlUp(x); return x }
func avlLR(x *AVL) *AVL { y := x.R; x.R = y.L; y.L = x; avlUp(x); avlUp(y); return y }
func avlBal(n *AVL) *AVL { avlUp(n); b := avlBf(n); if b > 1 { if avlBf(n.L) < 0 { n.L = avlLR(n.L) }; return avlRR(n) }; if b < -1 { if avlBf(n.R) > 0 { n.R = avlRR(n.R) }; return avlLR(n) }; return n }
func avlIns(n *AVL, v int) *AVL { if n == nil { return &AVL{V: v, H: 1} }; if v < n.V { n.L = avlIns(n.L, v) } else if v > n.V { n.R = avlIns(n.R, v) }; return avlBal(n) }
func avlS(n *AVL, v int) bool { for n != nil { if v == n.V { return true }; if v < n.V { n = n.L } else { n = n.R } }; return false }

// ---- Treap ----
type Treap struct{ V, P int; L, R *Treap }
func tRR(y *Treap) *Treap { x := y.L; y.L = x.R; x.R = y; return x }
func tLR(x *Treap) *Treap { y := x.R; x.R = y.L; y.L = x; return y }
func tIns(n *Treap, v int) *Treap {
	if n == nil { return &Treap{V: v, P: rand.Int()} }
	if v < n.V { n.L = tIns(n.L, v); if n.L.P > n.P { n = tRR(n) } } else if v > n.V { n.R = tIns(n.R, v); if n.R.P > n.P { n = tLR(n) } }
	return n
}
func tS(n *Treap, v int) bool { for n != nil { if v == n.V { return true }; if v < n.V { n = n.L } else { n = n.R } }; return false }

func benchmark(name string, n int, insertFn func(int), searchFn func(int) bool) {
	data := rand.Perm(n)

	start := time.Now()
	for _, v := range data { insertFn(v) }
	insertTime := time.Since(start)

	start = time.Now()
	for _, v := range data { searchFn(v) }
	searchTime := time.Since(start)

	fmt.Printf("%-10s insert=%10v search=%10v\n", name, insertTime, searchTime)
}

func main() {
	n := 200000
	fmt.Printf("Benchmarking with n=%d random values:\n", n)

	var avlRoot *AVL
	benchmark("AVL", n,
		func(v int) { avlRoot = avlIns(avlRoot, v) },
		func(v int) bool { return avlS(avlRoot, v) })

	var treapRoot *Treap
	benchmark("Treap", n,
		func(v int) { treapRoot = tIns(treapRoot, v) },
		func(v int) bool { return tS(treapRoot, v) })

	// Go built-in map for comparison
	m := map[int]bool{}
	benchmark("HashMap", n,
		func(v int) { m[v] = true },
		func(v int) bool { return m[v] })
}
```

---

## Key Takeaways

1. **AVL**: Best for read-heavy — shortest height, fastest lookups
2. **Red-Black**: Best for write-heavy — bounded rotations per delete (≤3)
3. **Splay**: Best for temporal locality — recently accessed items near root, no extra metadata
4. **Treap**: Best for simplicity — randomized, short code, good expected performance
5. **B-Tree**: Best for disk — minimizes I/O with multi-key nodes
6. **Skip List**: Best for concurrency — lock-free variants available
7. Go has no built-in sorted data structure — implement AVL or RB tree when needed

> **Phase 9 Complete! Next up:** Phase 10 — Binary Search →
