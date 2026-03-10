# Phase 23: Design Data Structures — Iterator Design Patterns

## Overview

Iterators provide a **uniform way to traverse** collections without exposing internal structure. In Go, iterators are typically implemented with channels, closures, or explicit `Next()/HasNext()` methods.

| Pattern | Description |
|---------|------------|
| **hasNext/Next** | Java-style explicit iterator |
| **Channel-based** | Go idiomatic with goroutines |
| **Closure-based** | Returns a function that yields next |
| **In-order BST** | O(h) space controlled traversal |
| **Flatten nested** | Stack-based for nested structures |

---

## Example 1: BST Iterator (LeetCode 173)

```go
package main

import "fmt"

type TreeNode struct {
	Val         int
	Left, Right *TreeNode
}

// O(h) space BST iterator using stack
type BSTIterator struct {
	stack []*TreeNode
}

func NewBSTIterator(root *TreeNode) *BSTIterator {
	it := &BSTIterator{}
	it.pushLeft(root)
	return it
}

func (it *BSTIterator) pushLeft(node *TreeNode) {
	for node != nil {
		it.stack = append(it.stack, node)
		node = node.Left
	}
}

func (it *BSTIterator) Next() int {
	top := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	it.pushLeft(top.Right)
	return top.Val
}

func (it *BSTIterator) HasNext() bool {
	return len(it.stack) > 0
}

func main() {
	//      7
	//     / \
	//    3   15
	//       / \
	//      9   20
	root := &TreeNode{7,
		&TreeNode{3, nil, nil},
		&TreeNode{15,
			&TreeNode{9, nil, nil},
			&TreeNode{20, nil, nil}}}

	it := NewBSTIterator(root)
	fmt.Print("In-order: ")
	for it.HasNext() {
		fmt.Printf("%d ", it.Next())
	}
	fmt.Println("\n\nO(h) space, O(1) amortized Next()")
}
```

**Textual Figure:**
```
BST Iterator: Stack-Based In-Order Traversal

  Tree:
          7
         ╱ ╲
        3   15
           ╱  ╲
          9    20

  Stack progression (pushLeft on init → push 7, 3):

  Step    Stack          Action             Output
  ────    ─────────────  ─────────────────  ──────
  init    [7, 3]         pushLeft(root)       —
  Next()  [7]            pop 3, pushL(nil)    3
  Next()  []             pop 7, pushL(15→9)   7
          [15, 9]        ← pushed 15, then 9
  Next()  [15]           pop 9, pushL(nil)    9
  Next()  []             pop 15, pushL(20)    15
          [20]
  Next()  []             pop 20, pushL(nil)   20

  Result: 3 → 7 → 9 → 15 → 20  (in-order ✓)

  ┌─────────────────────────────────────────┐
  │ Space: O(h) — at most h nodes on stack  │
  │ Time: O(1) amortized per Next()         │
  │ Each node pushed/popped exactly once    │
  └─────────────────────────────────────────┘
```

---

## Example 2: Flatten Nested List Iterator (LeetCode 341)

```go
package main

import "fmt"

type NestedInteger struct {
	val  int
	list []*NestedInteger
}

func (ni *NestedInteger) IsInteger() bool      { return ni.list == nil }
func (ni *NestedInteger) GetInteger() int       { return ni.val }
func (ni *NestedInteger) GetList() []*NestedInteger { return ni.list }

func NewInt(v int) *NestedInteger              { return &NestedInteger{val: v} }
func NewList(items ...*NestedInteger) *NestedInteger { return &NestedInteger{list: items} }

type NestedIterator struct {
	stack []*NestedInteger
}

func NewNestedIterator(nestedList []*NestedInteger) *NestedIterator {
	it := &NestedIterator{}
	// Push in reverse order so first element is on top
	for i := len(nestedList) - 1; i >= 0; i-- {
		it.stack = append(it.stack, nestedList[i])
	}
	return it
}

func (it *NestedIterator) HasNext() bool {
	for len(it.stack) > 0 {
		top := it.stack[len(it.stack)-1]
		if top.IsInteger() { return true }
		it.stack = it.stack[:len(it.stack)-1]
		list := top.GetList()
		for i := len(list) - 1; i >= 0; i-- {
			it.stack = append(it.stack, list[i])
		}
	}
	return false
}

func (it *NestedIterator) Next() int {
	top := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	return top.GetInteger()
}

func main() {
	// [[1,1],2,[1,1]]
	nested := []*NestedInteger{
		NewList(NewInt(1), NewInt(1)),
		NewInt(2),
		NewList(NewInt(1), NewInt(1)),
	}

	it := NewNestedIterator(nested)
	fmt.Print("Flattened: ")
	for it.HasNext() {
		fmt.Printf("%d ", it.Next())
	}
	fmt.Println() // 1 1 2 1 1

	// Deeply nested: [1,[2,[3,[4]]]]
	deep := []*NestedInteger{
		NewInt(1),
		NewList(NewInt(2), NewList(NewInt(3), NewList(NewInt(4)))),
	}
	it2 := NewNestedIterator(deep)
	fmt.Print("Deep:      ")
	for it2.HasNext() {
		fmt.Printf("%d ", it2.Next())
	}
	fmt.Println() // 1 2 3 4
}
```

**Textual Figure:**
```
Flatten Nested List Iterator: Stack-Based Lazy Flattening

  Input: [[1,1], 2, [1,1]]

  Initial stack (pushed in reverse):
    ┌──────────┐
    │ [1,1]    │  ← top (first element)
    │ 2        │
    │ [1,1]    │  ← bottom
    └──────────┘

  HasNext()/Next() trace:

  Call       Stack (top→bottom)         Action              Output
  ────────   ────────────────────────   ──────────────────  ──────
  HasNext()  [1,1] | 2 | [1,1]         top=[1,1] → expand    —
             1 | 1 | 2 | [1,1]         top=1 → integer ✓
  Next()     1 | 2 | [1,1]             pop 1                  1
  Next()     2 | [1,1]                  pop 1                  1
  Next()     [1,1]                      pop 2                  2
  HasNext()  [1,1]                      top=[1,1] → expand
             1 | 1                      top=1 → integer ✓
  Next()     1                          pop 1                  1
  Next()     (empty)                    pop 1                  1

  Result: 1 1 2 1 1 ✓

  Deep nested: [1,[2,[3,[4]]]]
    Expansion: stack unfolds lazily:
    [1,[2,[3,[4]]]] → 1 | [2,[3,[4]]] → ... → 1 2 3 4
```

---

## Example 3: Peeking Iterator (LeetCode 284)

```go
package main

import "fmt"

type Iterator struct {
	data []int
	pos  int
}

func NewIterator(data []int) *Iterator { return &Iterator{data: data} }
func (it *Iterator) HasNext() bool     { return it.pos < len(it.data) }
func (it *Iterator) Next() int         { v := it.data[it.pos]; it.pos++; return v }

// PeekingIterator wraps an Iterator and adds peek capability
type PeekingIterator struct {
	iter    *Iterator
	peeked  bool
	peekVal int
}

func NewPeekingIterator(it *Iterator) *PeekingIterator {
	return &PeekingIterator{iter: it}
}

func (pi *PeekingIterator) Peek() int {
	if !pi.peeked {
		pi.peekVal = pi.iter.Next()
		pi.peeked = true
	}
	return pi.peekVal
}

func (pi *PeekingIterator) Next() int {
	if pi.peeked {
		pi.peeked = false
		return pi.peekVal
	}
	return pi.iter.Next()
}

func (pi *PeekingIterator) HasNext() bool {
	return pi.peeked || pi.iter.HasNext()
}

func main() {
	data := []int{1, 2, 3, 4, 5}
	pi := NewPeekingIterator(NewIterator(data))

	fmt.Println("Peek:", pi.Peek())     // 1
	fmt.Println("Peek:", pi.Peek())     // 1 (doesn't advance)
	fmt.Println("Next:", pi.Next())     // 1
	fmt.Println("Next:", pi.Next())     // 2
	fmt.Println("Peek:", pi.Peek())     // 3
	fmt.Println("Next:", pi.Next())     // 3
	fmt.Println("HasNext:", pi.HasNext()) // true
}
```

**Textual Figure:**
```
Peeking Iterator: Cached One-Ahead Value

  Data: [1, 2, 3, 4, 5]
  Wraps an underlying Iterator and caches one value.

  Op        peeked  peekVal  iter.pos  Output
  ────────  ──────  ───────  ────────  ──────
  (init)    false    —        0          —
  Peek()    true     1        1          1
             └─ called iter.Next(), cached result
  Peek()    true     1        1          1
             └─ already peeked, return cache
  Next()    false    —        1          1
             └─ return cached, clear flag
  Next()    false    —        2          2
             └─ no cache, call iter.Next()
  Peek()    true     3        3          3
  Next()    false    —        3          3
  HasNext() true              3          true
             └─ peeked || iter.HasNext()

  ┌────────────────────────────────────────┐
  │ State: { peeked: bool, peekVal: int } │
  │ Peek: cache if not cached, return     │
  │ Next: return cache if any, else iter  │
  └────────────────────────────────────────┘
```

---

## Example 4: Closure-Based Iterator (Go Idiomatic)

```go
package main

import "fmt"

// Go-style: return a closure that yields values
func fibIterator() func() (int, bool) {
	a, b := 0, 1
	return func() (int, bool) {
		val := a
		a, b = b, a+b
		return val, true // infinite sequence
	}
}

// Bounded iterator
func rangeIterator(start, end int) func() (int, bool) {
	current := start
	return func() (int, bool) {
		if current >= end { return 0, false }
		v := current
		current++
		return v, true
	}
}

// Filter iterator: composes with another iterator
func filterIterator(next func() (int, bool), pred func(int) bool) func() (int, bool) {
	return func() (int, bool) {
		for {
			v, ok := next()
			if !ok { return 0, false }
			if pred(v) { return v, true }
		}
	}
}

// Map iterator
func mapIterator(next func() (int, bool), f func(int) int) func() (int, bool) {
	return func() (int, bool) {
		v, ok := next()
		if !ok { return 0, false }
		return f(v), true
	}
}

func main() {
	// Fibonacci: take first 10
	fib := fibIterator()
	fmt.Print("Fibonacci: ")
	for i := 0; i < 10; i++ {
		v, _ := fib()
		fmt.Printf("%d ", v)
	}
	fmt.Println()

	// Range [0, 10) → filter even → square
	even := filterIterator(rangeIterator(0, 10), func(x int) bool { return x%2 == 0 })
	squared := mapIterator(even, func(x int) int { return x * x })

	fmt.Print("Even² [0,10): ")
	for {
		v, ok := squared()
		if !ok { break }
		fmt.Printf("%d ", v)
	}
	fmt.Println()
}
```

**Textual Figure:**
```
Closure-Based Iterator: Composable Functional Pipeline

  fibIterator():
    Closure state: a=0, b=1
    Call sequence:  0, 1, 1, 2, 3, 5, 8, 13, 21, 34

  Pipeline: range(0,10) → filter(even) → map(x²)

    rangeIterator(0,10):
      yields: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9
                │
                ▼
    filterIterator(even?):
      passes: 0, 2, 4, 6, 8
                │
                ▼
    mapIterator(x²):
      yields: 0, 4, 16, 36, 64

  Data flow:
    ┌─────────┐     ┌──────────┐     ┌─────────┐
    │ range   │────▶│ filter   │────▶│  map    │────▶ output
    │ 0..9    │     │ x%2==0   │     │  x*x    │
    └─────────┘     └──────────┘     └─────────┘

  Each iterator is a func() (int, bool)
  Lazy evaluation — values computed on demand
```

---

## Example 5: Channel-Based Iterator (Goroutine)

```go
package main

import "fmt"

// Tree traversal via channel
type TreeNode struct {
	Val         int
	Left, Right *TreeNode
}

func inorderChannel(root *TreeNode) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		var walk func(*TreeNode)
		walk = func(node *TreeNode) {
			if node == nil { return }
			walk(node.Left)
			ch <- node.Val
			walk(node.Right)
		}
		walk(root)
	}()
	return ch
}

func preorderChannel(root *TreeNode) <-chan int {
	ch := make(chan int)
	go func() {
		defer close(ch)
		var walk func(*TreeNode)
		walk = func(node *TreeNode) {
			if node == nil { return }
			ch <- node.Val
			walk(node.Left)
			walk(node.Right)
		}
		walk(root)
	}()
	return ch
}

func main() {
	root := &TreeNode{4,
		&TreeNode{2,
			&TreeNode{1, nil, nil},
			&TreeNode{3, nil, nil}},
		&TreeNode{6,
			&TreeNode{5, nil, nil},
			&TreeNode{7, nil, nil}}}

	fmt.Print("In-order:  ")
	for v := range inorderChannel(root) { fmt.Printf("%d ", v) }
	fmt.Println()

	fmt.Print("Pre-order: ")
	for v := range preorderChannel(root) { fmt.Printf("%d ", v) }
	fmt.Println()

	fmt.Println("\nChannel iterators: elegant but have goroutine overhead")
}
```

**Textual Figure:**
```
Channel-Based Iterator: Goroutine Tree Traversal

  Tree:
          4
         ╱ ╲
        2    6
       ╱ ╲  ╱ ╲
      1  3  5  7

  In-order channel:
    goroutine walks tree recursively:

    walk(4)
    ├─ walk(2)
    │  ├─ walk(1) → ch ← 1
    │  └─ ch ← 2
    │  └─ walk(3) → ch ← 3
    └─ ch ← 4
    └─ walk(6)
       ├─ walk(5) → ch ← 5
       └─ ch ← 6
       └─ walk(7) → ch ← 7
    close(ch)

    ┌──────────┐    ch     ┌──────────────┐
    │ goroutine│───────────▶│ for v := range│
    │ walk()   │  1,2,3,4  │   ch { ... }  │
    │          │  5,6,7    │               │
    └──────────┘           └──────────────┘

  In-order:  1 2 3 4 5 6 7
  Pre-order: 4 2 1 3 6 5 7

  Elegant for range syntax, but goroutine lifecycle
  must be managed (defer close(ch))
```

---

## Example 6: Zigzag Iterator (Interleaving)

```go
package main

import "fmt"

// Zigzag between multiple lists: [1,2] [3,4,5] → 1,3,2,4,5

type ZigzagIterator struct {
	lists [][]int
	idxs  []int
	curr  int
}

func NewZigzag(lists ...[]int) *ZigzagIterator {
	var filtered [][]int
	for _, l := range lists {
		if len(l) > 0 { filtered = append(filtered, l) }
	}
	return &ZigzagIterator{lists: filtered, idxs: make([]int, len(filtered))}
}

func (zi *ZigzagIterator) HasNext() bool {
	for range zi.lists {
		if zi.idxs[zi.curr] < len(zi.lists[zi.curr]) { return true }
		zi.curr = (zi.curr + 1) % len(zi.lists)
	}
	return false
}

func (zi *ZigzagIterator) Next() int {
	for zi.idxs[zi.curr] >= len(zi.lists[zi.curr]) {
		zi.curr = (zi.curr + 1) % len(zi.lists)
	}
	val := zi.lists[zi.curr][zi.idxs[zi.curr]]
	zi.idxs[zi.curr]++
	zi.curr = (zi.curr + 1) % len(zi.lists)
	return val
}

func main() {
	zi := NewZigzag([]int{1, 2}, []int{3, 4, 5, 6}, []int{7})

	fmt.Print("Zigzag [1,2] [3,4,5,6] [7]: ")
	for zi.HasNext() { fmt.Printf("%d ", zi.Next()) }
	fmt.Println() // 1 3 7 2 4 5 6
}
```

**Textual Figure:**
```
Zigzag Iterator: Round-Robin Interleaving

  Input lists:
    List 0: [1, 2]
    List 1: [3, 4, 5, 6]
    List 2: [7]

  Round-robin traversal (curr cycles 0 → 1 → 2 → 0 → ...):

  Step  curr  List  idx   Value  Lists state
  ────  ────  ────  ───   ─────  ───────────────────
   1     0     0    0      1     [_,2]  [3,4,5,6]  [7]
   2     1     1    0      3     [_,2]  [_,4,5,6]  [7]
   3     2     2    0      7     [_,2]  [_,4,5,6]  [_]
   4     0     0    1      2     [_,_]  [_,4,5,6]  [_]
   5     1     1    1      4     [_,_]  [_,_,5,6]  [_]
   6     1     1    2      5     (skip exhausted lists)
   7     1     1    3      6     

  Output: 1 3 7 2 4 5 6

  Exhausted lists are skipped in round-robin:
    ┌───┐  ┌───┐  ┌───┐
    │ L0│→│ L1│→│ L2│→ (cycle)
    └───┘  └───┘  └───┘
```

---

## Example 7: Level-Order Iterator (BFS)

```go
package main

import "fmt"

type TreeNode2 struct {
	Val         int
	Left, Right *TreeNode2
}

type LevelIterator struct {
	queue     []*TreeNode2
	levelDone bool
	level     int
}

func NewLevelIterator(root *TreeNode2) *LevelIterator {
	li := &LevelIterator{}
	if root != nil { li.queue = append(li.queue, root) }
	return li
}

func (li *LevelIterator) HasNext() bool { return len(li.queue) > 0 }

func (li *LevelIterator) NextLevel() ([]int, int) {
	if len(li.queue) == 0 { return nil, -1 }
	size := len(li.queue)
	level := li.level
	var vals []int

	for i := 0; i < size; i++ {
		node := li.queue[0]
		li.queue = li.queue[1:]
		vals = append(vals, node.Val)
		if node.Left != nil { li.queue = append(li.queue, node.Left) }
		if node.Right != nil { li.queue = append(li.queue, node.Right) }
	}
	li.level++
	return vals, level
}

func main() {
	root := &TreeNode2{3,
		&TreeNode2{9, nil, nil},
		&TreeNode2{20,
			&TreeNode2{15, nil, nil},
			&TreeNode2{7, nil, nil}}}

	it := NewLevelIterator(root)
	fmt.Println("Level-order iteration:")
	for it.HasNext() {
		vals, level := it.NextLevel()
		fmt.Printf("  Level %d: %v\n", level, vals)
	}
}
```

**Textual Figure:**
```
Level-Order Iterator: BFS Queue

  Tree:
        3
       ╱ ╲
      9   20
         ╱  ╲
        15    7

  BFS iteration by level:

  Level  Queue before       Process        Queue after
  ─────  ───────────────  ─────────────  ───────────────
   0     [3]              dequeue 3        [9, 20]
   1     [9, 20]           dequeue 9, 20   [15, 7]
   2     [15, 7]           dequeue 15, 7   []

  Output:
    Level 0: [3]
    Level 1: [9, 20]
    Level 2: [15, 7]

  Queue state visualization:
    ┌───┐
    │ 3 │  → process, add children
    └───┘
    ┌───┬────┐
    │ 9 │ 20 │  → process size=2, add children
    └───┴────┘
    ┌────┬───┐
    │ 15 │ 7 │  → process size=2, no children
    └────┴───┘
```

---

## Example 8: Skip Iterator

```go
package main

import "fmt"

// Iterator that supports skip(val): next occurrence of val is skipped

type SkipIterator struct {
	data   []int
	pos    int
	skipMap map[int]int  // val → count to skip
}

func NewSkipIterator(data []int) *SkipIterator {
	return &SkipIterator{data: data, skipMap: make(map[int]int)}
}

func (si *SkipIterator) advance() {
	for si.pos < len(si.data) {
		if si.skipMap[si.data[si.pos]] > 0 {
			si.skipMap[si.data[si.pos]]--
			si.pos++
		} else {
			break
		}
	}
}

func (si *SkipIterator) HasNext() bool {
	si.advance()
	return si.pos < len(si.data)
}

func (si *SkipIterator) Next() int {
	si.advance()
	v := si.data[si.pos]
	si.pos++
	return v
}

func (si *SkipIterator) Skip(val int) {
	si.skipMap[val]++
}

func main() {
	si := NewSkipIterator([]int{2, 3, 5, 6, 5, 7, 5, -1, 5, 10})
	fmt.Println("Data: [2, 3, 5, 6, 5, 7, 5, -1, 5, 10]")
	fmt.Println()

	fmt.Println("Next:", si.Next()) // 2
	si.Skip(5)
	fmt.Println("Skip(5)")
	fmt.Println("Next:", si.Next()) // 3 
	fmt.Println("Next:", si.Next()) // 6 (first 5 was skipped)
	fmt.Println("Next:", si.Next()) // 5
	si.Skip(5)
	si.Skip(5)
	fmt.Println("Skip(5), Skip(5)")
	fmt.Println("Next:", si.Next()) // 7 (both 5s skipped)
	fmt.Println("Next:", si.Next()) // -1
	fmt.Println("Next:", si.Next()) // 10
}
```

**Textual Figure:**
```
Skip Iterator: Skip-Count Map

  Data: [2, 3, 5, 6, 5, 7, 5, -1, 5, 10]
                pos →

  skipMap tracks how many future occurrences of a value to skip.

  Op          pos  skipMap       advance()         Output
  ──────────  ───  ───────────  ────────────────  ──────
  Next()       0   {}            —                  2
  Skip(5)      1   {5:1}         —                  —
  Next()       1   {5:1}         —                  3
  Next()       2   {5:1}         pos 2 is 5 → skip  6
                                 skip [{5:0}] pos→4
  Next()       4   {}            —                  5
  Skip(5)      5   {5:1}         —                  —
  Skip(5)      5   {5:2}         —                  —
  Next()       5   {5:2}         pos5=5→skip        7
                                 pos6=5→skip
                                 {5:0} pos→7
  Next()       7   {}            —                  -1
  Next()       8   {}            —                  10
                                 (pos8=5, but skip count=0)
  Wait: pos8=5, skipMap empty → not skipped? Re-read:
  Actually after 2 skips: pos5=5 skipped, pos6=5 skipped, pos7=-1
  Next()=7, Next()=-1, Next()=5(pos8), Next()=10(pos9)
```

---

## Example 9: Iterator for 2D Matrix (Spiral Order)

```go
package main

import "fmt"

type SpiralIterator struct {
	matrix         [][]int
	top, bottom    int
	left, right    int
	dir            int // 0=right, 1=down, 2=left, 3=up
	row, col       int
	remaining      int
}

func NewSpiral(matrix [][]int) *SpiralIterator {
	if len(matrix) == 0 { return &SpiralIterator{} }
	m, n := len(matrix), len(matrix[0])
	return &SpiralIterator{
		matrix: matrix, top: 0, bottom: m - 1, left: 0, right: n - 1,
		remaining: m * n,
	}
}

func (si *SpiralIterator) HasNext() bool { return si.remaining > 0 }

func (si *SpiralIterator) Next() int {
	val := si.matrix[si.row][si.col]
	si.remaining--

	switch si.dir {
	case 0: // right
		if si.col == si.right { si.top++; si.dir = 1; si.row++ } else { si.col++ }
	case 1: // down
		if si.row == si.bottom { si.right--; si.dir = 2; si.col-- } else { si.row++ }
	case 2: // left
		if si.col == si.left { si.bottom--; si.dir = 3; si.row-- } else { si.col-- }
	case 3: // up
		if si.row == si.top { si.left++; si.dir = 0; si.col++ } else { si.row-- }
	}
	return val
}

func main() {
	matrix := [][]int{
		{1, 2, 3, 4},
		{5, 6, 7, 8},
		{9, 10, 11, 12},
	}

	it := NewSpiral(matrix)
	fmt.Print("Spiral: ")
	for it.HasNext() { fmt.Printf("%d ", it.Next()) }
	fmt.Println()
	// 1 2 3 4 8 12 11 10 9 5 6 7
}
```

**Textual Figure:**
```
Spiral Order Iterator: Direction State Machine

  Matrix:
    ┌────┬────┬────┬────┐
    │  1 │  2 │  3 │  4 │
    ├────┼────┼────┼────┤
    │  5 │  6 │  7 │  8 │
    ├────┼────┼────┼────┤
    │  9 │ 10 │ 11 │ 12 │
    └────┴────┴────┴────┘

  Direction state: 0=right, 1=down, 2=left, 3=up
  Boundaries: top, bottom, left, right (shrink inward)

  Step  dir    Boundary hit?       Elements     Boundary update
  ────  ─────  ────────────────  ───────────  ───────────────
   1    right  col hits right=3   1,2,3,4       top: 0→1
   2    down   row hits bottom=2  8,12          right: 3→2
   3    left   col hits left=0    11,10,9       bottom: 2→1
   4    up     row hits top=1     5             left: 0→1
   5    right  col hits right=2   6,7           done!

  Spiral path:
    1 ─▶ 2 ─▶ 3 ─▶ 4
                    │
    5 ─▶ 6 ─▶ 7    8
    │              │
    9 ◀─ 10 ◀ 11 ◀ 12

  Output: 1 2 3 4 8 12 11 10 9 5 6 7
```

---

## Example 10: Merge k Sorted Iterators

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntIterator struct {
	data []int
	pos  int
}

func (it *IntIterator) HasNext() bool { return it.pos < len(it.data) }
func (it *IntIterator) Peek() int    { return it.data[it.pos] }
func (it *IntIterator) Next() int    { v := it.data[it.pos]; it.pos++; return v }

type HeapItem struct {
	val int
	idx int // which iterator
}

type MinH []HeapItem
func (h MinH) Len() int            { return len(h) }
func (h MinH) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h MinH) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinH) Push(x any)         { *h = append(*h, x.(HeapItem)) }
func (h *MinH) Pop() any           { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

type MergeKIterator struct {
	iters []*IntIterator
	h     *MinH
}

func NewMergeK(lists [][]int) *MergeKIterator {
	iters := make([]*IntIterator, len(lists))
	h := &MinH{}
	for i, l := range lists {
		iters[i] = &IntIterator{data: l}
		if iters[i].HasNext() {
			heap.Push(h, HeapItem{iters[i].Peek(), i})
		}
	}
	return &MergeKIterator{iters, h}
}

func (mk *MergeKIterator) HasNext() bool { return mk.h.Len() > 0 }

func (mk *MergeKIterator) Next() int {
	top := heap.Pop(mk.h).(HeapItem)
	mk.iters[top.idx].Next()
	if mk.iters[top.idx].HasNext() {
		heap.Push(mk.h, HeapItem{mk.iters[top.idx].Peek(), top.idx})
	}
	return top.val
}

func main() {
	lists := [][]int{
		{1, 4, 7, 10},
		{2, 5, 8},
		{3, 6, 9, 11, 12},
	}

	mk := NewMergeK(lists)
	fmt.Print("Merged: ")
	for mk.HasNext() { fmt.Printf("%d ", mk.Next()) }
	fmt.Println()
	// 1 2 3 4 5 6 7 8 9 10 11 12

	fmt.Println("\nO(log k) per Next() — k = number of iterators")
}
```

**Textual Figure:**
```
Merge k Sorted Iterators: Min-Heap of Heads

  Input iterators:
    Iter 0: [1, 4, 7, 10]
    Iter 1: [2, 5, 8]
    Iter 2: [3, 6, 9, 11, 12]

  Min-Heap state (tracks head of each iterator):

  Step  Heap (val,iter)     Pop    Push next       Output
  ────  ─────────────────  ─────  ──────────────  ──────
  init  [(1,0),(2,1),(3,2)]  —      —               —
   1    [(2,1),(3,2),(4,0)]  1,i0   push (4,i0)     1
   2    [(3,2),(4,0),(5,1)]  2,i1   push (5,i1)     2
   3    [(4,0),(5,1),(6,2)]  3,i2   push (6,i2)     3
   4    [(5,1),(6,2),(7,0)]  4,i0   push (7,i0)     4
   5    [(6,2),(7,0),(8,1)]  5,i1   push (8,i1)     5
   6    [(7,0),(8,1),(9,2)]  6,i2   push (9,i2)     6
   ...  ...                  ...    ...             ...

  Heap visualization (step 1):
        ┌───┐
        │ 1 │ ← min (from iter 0)
        └┬──┘
       ╱    ╲
    ┌───┐  ┌───┐
    │ 2 │  │ 3 │
    └───┘  └───┘
    iter1   iter2

  Output: 1 2 3 4 5 6 7 8 9 10 11 12
  Time: O(N log k) total, O(log k) per Next()
```

---

## Key Takeaways

1. **BST Iterator**: use stack to simulate in-order — O(h) space, O(1) amortized
2. **Flatten Nested**: stack-based, flatten lazily in `HasNext()`
3. **Peeking Iterator**: cache one value, flag-based control
4. **Go closures**: elegant functional iterators, composable
5. **Go channels**: clean `for range` syntax but goroutine overhead
6. **Zigzag/Interleave**: round-robin through multiple sources
7. **Skip Iterator**: maintain a skip-count map
8. **Merge K**: min-heap of iterator heads

> **Next up:** Randomized Data Structures →
