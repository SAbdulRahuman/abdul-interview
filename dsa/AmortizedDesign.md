# Phase 23: Design Data Structures — Amortized Design

## Overview

**Amortized Analysis** shows that even if individual operations are expensive, the average cost per operation over a sequence is low.

| Method | Idea |
|--------|------|
| **Aggregate** | Total cost / n operations |
| **Accounting** | Overpay cheap ops, use credit for expensive ones |
| **Potential** | Define Φ(state), amortized = actual + ΔΦ |

---

## Example 1: Dynamic Array (Slice) — Amortized O(1) Append

```go
package main

import "fmt"

type DynArray struct {
	data []int
	size int
	cap  int
}

func NewDynArray() *DynArray {
	return &DynArray{data: make([]int, 1), size: 0, cap: 1}
}

func (d *DynArray) Append(val int) int {
	cost := 1
	if d.size == d.cap {
		// Double capacity — expensive O(n) copy, but rare
		newCap := d.cap * 2
		newData := make([]int, newCap)
		copy(newData, d.data)
		d.data = newData
		d.cap = newCap
		cost = d.size + 1 // copy + insert
	}
	d.data[d.size] = val
	d.size++
	return cost
}

func main() {
	da := NewDynArray()
	totalCost := 0

	for i := 1; i <= 32; i++ {
		cost := da.Append(i)
		totalCost += cost
		fmt.Printf("  Append(%2d): cost=%2d, size=%2d, cap=%2d\n", i, cost, da.size, da.cap)
	}

	fmt.Printf("\nTotal cost: %d for %d operations\n", totalCost, da.size)
	fmt.Printf("Amortized cost: %.2f per operation\n", float64(totalCost)/float64(da.size))
	fmt.Println("\nDoubling strategy → amortized O(1) append")
}
```

**Textual Figure:**
```
Dynamic Array: Amortized O(1) Append

Append  Size  Cap  Cost  Array State
──────  ────  ───  ────  ──────────────────────────────
  1      1    1    1    [█]
  2      2    2    2    [██]              ← resize! copy 1
  3      3    4    3    [███░]            ← resize! copy 2
  4      4    4    1    [████]
  5      5    8    5    [█████░░░]        ← resize! copy 4
  6      6    8    1    [██████░░]
  7      7    8    1    [███████░]
  8      8    8    1    [████████]
  9      9   16    9    [█████████░░░░░░░]  ← resize! copy 8

Cost pattern: 1, 2, 3, 1, 5, 1, 1, 1, 9, 1, 1, 1, 1, 1, 1, 1, 17...
Resizes at powers of 2: cost n+1, but happen every n ops
Total cost for n appends ≤ 3n → amortized O(1)
```

---

## Example 2: Stack with MultiPop — Amortized O(1)

```go
package main

import "fmt"

type AmortizedStack struct {
	data []int
	ops  int
}

func (s *AmortizedStack) Push(val int) {
	s.data = append(s.data, val)
	s.ops++
}

func (s *AmortizedStack) MultiPop(k int) int {
	popped := 0
	for len(s.data) > 0 && popped < k {
		s.data = s.data[:len(s.data)-1]
		popped++
		s.ops++
	}
	return popped
}

func main() {
	s := &AmortizedStack{}

	// Each element can only be popped once after being pushed once
	// So n pushes + any number of MultiPops ≤ 2n total ops
	operations := []string{}

	for i := 0; i < 10; i++ {
		s.Push(i)
		operations = append(operations, fmt.Sprintf("Push(%d)", i))
	}
	p := s.MultiPop(5)
	operations = append(operations, fmt.Sprintf("MultiPop(5)→%d", p))

	for i := 10; i < 15; i++ {
		s.Push(i)
		operations = append(operations, fmt.Sprintf("Push(%d)", i))
	}
	p = s.MultiPop(100)
	operations = append(operations, fmt.Sprintf("MultiPop(100)→%d", p))

	fmt.Println("Operations:")
	for _, op := range operations { fmt.Printf("  %s\n", op) }
	fmt.Printf("\nTotal actual ops (push+pop): %d\n", s.ops)
	fmt.Printf("User operations: %d\n", len(operations))
	fmt.Println("Amortized O(1) per user operation (accounting method)")
}
```

**Textual Figure:**
```
Stack with MultiPop: Amortized O(1)

Operations sequence:
  Push(0..9)  → stack: [0,1,2,3,4,5,6,7,8,9]
  MultiPop(5) → stack: [0,1,2,3,4]         pops 5
  Push(10..14)→ stack: [0,1,2,3,4,10,11,12,13,14]
  MultiPop(100)→ stack: []                  pops 10

Accounting method:
  ┌─────────────────────────────────────────┐
  │ Each Push pays $2:                      │
  │   $1 for the push itself                │
  │   $1 saved as credit on the element     │
  │                                         │
  │ Each Pop costs $1, paid by credit       │
  │                                         │
  │ Push: [○$][○$][○$][○$][○$]             │
  │ Pop:  [○✗][○✗][○✗][○✗][○✗]  uses $     │
  │                                         │
  │ Total: n pushes ≤ 2n cost               │
  │ Any pops ≤ n (can't pop more than push) │
  │ Amortized: O(1) per operation            │
  └─────────────────────────────────────────┘
```

---

## Example 3: Incrementing Binary Counter

```go
package main

import "fmt"

type BinaryCounter struct {
	bits    []int
	flips   int
}

func NewBinaryCounter(k int) *BinaryCounter {
	return &BinaryCounter{bits: make([]int, k)}
}

func (c *BinaryCounter) Increment() int {
	flips := 0
	i := 0
	for i < len(c.bits) && c.bits[i] == 1 {
		c.bits[i] = 0
		flips++
		i++
	}
	if i < len(c.bits) {
		c.bits[i] = 1
		flips++
	}
	c.flips += flips
	return flips
}

func (c *BinaryCounter) String() string {
	s := ""
	for i := len(c.bits) - 1; i >= 0; i-- {
		s += fmt.Sprintf("%d", c.bits[i])
	}
	return s
}

func main() {
	c := NewBinaryCounter(8)

	for i := 1; i <= 16; i++ {
		flips := c.Increment()
		fmt.Printf("  %2d: %s (flips=%d)\n", i, c, flips)
	}

	fmt.Printf("\nTotal flips: %d for 16 increments\n", c.flips)
	fmt.Printf("Amortized: %.2f flips per increment\n", float64(c.flips)/16.0)
	fmt.Println("\nBit k flips every 2^k increments → amortized O(1)")
}
```

**Textual Figure:**
```
Binary Counter: 8-bit, 16 increments

 Inc  Binary    Flips
 ───  ────────  ─────
  1   00000001    1
  2   00000010    2     ← bit 0 flips every 1
  3   00000011    1
  4   00000100    3     ← bit 1 flips every 2
  5   00000101    1
  6   00000110    2
  7   00000111    1
  8   00001000    4     ← bit 2 flips every 4
  9   00001001    1
  10  00001010    2
  11  00001011    1
  12  00001100    3
  13  00001101    1
  14  00001110    2
  15  00001111    1
  16  00010000    5     ← bit 3 flips every 8

  Total flips: 31 for 16 increments
  Amortized: 31/16 ≈ 1.94 flips/increment

  Bit k flips n/2^k times in n increments:
    Σ(n/2^k) = n(1 + 1/2 + 1/4 + ...) < 2n
    → amortized O(1)
```

---

## Example 4: MinStack — O(1) Push/Pop/GetMin

```go
package main

import "fmt"

// LeetCode 155: Min Stack
type MinStack struct {
	stack    []int
	minStack []int
}

func NewMinStack() *MinStack {
	return &MinStack{}
}

func (s *MinStack) Push(val int) {
	s.stack = append(s.stack, val)
	if len(s.minStack) == 0 || val <= s.minStack[len(s.minStack)-1] {
		s.minStack = append(s.minStack, val)
	}
}

func (s *MinStack) Pop() {
	top := s.stack[len(s.stack)-1]
	s.stack = s.stack[:len(s.stack)-1]
	if top == s.minStack[len(s.minStack)-1] {
		s.minStack = s.minStack[:len(s.minStack)-1]
	}
}

func (s *MinStack) Top() int { return s.stack[len(s.stack)-1] }
func (s *MinStack) GetMin() int { return s.minStack[len(s.minStack)-1] }

func main() {
	s := NewMinStack()
	ops := []struct{ action string; val int }{
		{"push", -2}, {"push", 0}, {"push", -3},
	}

	for _, op := range ops {
		s.Push(op.val)
		fmt.Printf("  Push(%d) → min=%d\n", op.val, s.GetMin())
	}

	s.Pop()
	fmt.Printf("  Pop()   → min=%d, top=%d\n", s.GetMin(), s.Top())
}
```

**Textual Figure:**
```
MinStack: O(1) Push/Pop/GetMin

Operation     Stack        MinStack     GetMin
──────────    ─────────    ─────────    ──────

Push(-2)      [-2]         [-2]         -2
Push(0)       [-2, 0]      [-2]         -2
Push(-3)      [-2, 0, -3]  [-2, -3]     -3

  Stack:     MinStack:
  ┌───┐     ┌───┐
  │-3 │     │-3 │  ← current min
  ├───┤     ├───┤
  │ 0 │     │-2 │  ← previous min
  ├───┤     └───┘
  │-2 │
  └───┘

Pop()         [-2, 0]      [-2]         -2
  (popped -3, which was minStack top, so pop minStack too)

All operations O(1)!
```

---

## Example 5: Queue with Two Stacks — Amortized O(1)

```go
package main

import "fmt"

type QueueTwoStacks struct {
	inbox  []int
	outbox []int
}

func NewQueue() *QueueTwoStacks {
	return &QueueTwoStacks{}
}

func (q *QueueTwoStacks) Enqueue(val int) {
	q.inbox = append(q.inbox, val)
}

func (q *QueueTwoStacks) transfer() {
	for len(q.inbox) > 0 {
		top := q.inbox[len(q.inbox)-1]
		q.inbox = q.inbox[:len(q.inbox)-1]
		q.outbox = append(q.outbox, top)
	}
}

func (q *QueueTwoStacks) Dequeue() int {
	if len(q.outbox) == 0 { q.transfer() }
	val := q.outbox[len(q.outbox)-1]
	q.outbox = q.outbox[:len(q.outbox)-1]
	return val
}

func (q *QueueTwoStacks) Peek() int {
	if len(q.outbox) == 0 { q.transfer() }
	return q.outbox[len(q.outbox)-1]
}

func main() {
	q := NewQueue()

	for i := 1; i <= 5; i++ {
		q.Enqueue(i)
		fmt.Printf("  Enqueue(%d)\n", i)
	}

	for i := 0; i < 3; i++ {
		fmt.Printf("  Dequeue() = %d\n", q.Dequeue())
	}

	q.Enqueue(6)
	q.Enqueue(7)
	fmt.Println("  Enqueue(6), Enqueue(7)")

	for len(q.inbox) > 0 || len(q.outbox) > 0 {
		fmt.Printf("  Dequeue() = %d\n", q.Dequeue())
	}

	fmt.Println("\nEach element transferred at most once → amortized O(1)")
}
```

**Textual Figure:**
```
Queue with Two Stacks: Amortized O(1)

Enqueue 1,2,3,4,5:
  Inbox:  [1,2,3,4,5]    Outbox: []
          ↑ push here

Dequeue (outbox empty → transfer):
  Inbox:  []    Outbox: [5,4,3,2,1]
                         ↑ pop here = 1 (FIFO!)

  Transfer visualization:
  Inbox:           Outbox:
  ┌───┐            ┌───┐
  │ 5 │───┐        │ 5 │ bottom
  │ 4 │   │        │ 4 │
  │ 3 │   │───▶    │ 3 │
  │ 2 │   │  flip  │ 2 │
  │ 1 │───┘        │ 1 │ top ← pop this
  └───┘            └───┘

Dequeue: 1, 2, 3 (from outbox)
Enqueue 6, 7 (to inbox)
Dequeue: 4, 5, 6, 7

Each element: push to inbox once, transfer once, pop once
→ Amortized O(1) per operation
```

---

## Example 6: Union-Find with Path Compression + Rank

```go
package main

import "fmt"

type UnionFind struct {
	parent, rank []int
	ops          int
}

func NewUF(n int) *UnionFind {
	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	return &UnionFind{parent: parent, rank: make([]int, n)}
}

func (uf *UnionFind) Find(x int) int {
	uf.ops++
	if uf.parent[x] != x {
		uf.parent[x] = uf.Find(uf.parent[x]) // path compression
	}
	return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) bool {
	px, py := uf.Find(x), uf.Find(y)
	if px == py { return false }
	if uf.rank[px] < uf.rank[py] { px, py = py, px }
	uf.parent[py] = px
	if uf.rank[px] == uf.rank[py] { uf.rank[px]++ }
	return true
}

func main() {
	n := 16
	uf := NewUF(n)

	// Chain of unions
	for i := 0; i < n-1; i++ {
		uf.Union(i, i+1)
	}

	fmt.Printf("After %d unions:\n", n-1)
	uf.ops = 0

	// Many finds — path compression makes subsequent ones fast
	for i := 0; i < n; i++ { uf.Find(i) }
	fmt.Printf("  First pass finds: %d ops\n", uf.ops)

	uf.ops = 0
	for i := 0; i < n; i++ { uf.Find(i) }
	fmt.Printf("  Second pass finds: %d ops (all compressed)\n", uf.ops)

	fmt.Println("\nWith path compression + union by rank:")
	fmt.Println("  Amortized O(α(n)) ≈ O(1) per operation")
	fmt.Println("  α = inverse Ackermann function, practically constant")
}
```

**Textual Figure:**
```
Union-Find: Path Compression + Union by Rank

Before path compression (chain):
  0 → 1 → 2 → 3 → ... → 15

After Find(0) with path compression:
          ┌─── 15 (root) ───┐
          │    │  │  │       │
          0    1  2  3  ...  14
          (all point directly to root)

First pass finds: O(n) total (long chains)
Second pass finds: O(n) total (all compressed → O(1) each)

  ┌────────────────────────────────────┐
  │ Union by rank:                     │
  │   Shorter tree attached under      │
  │   taller tree's root               │
  │   → tree height ≤ O(log n)        │
  │                                    │
  │ Path compression:                  │
  │   Every node points to root        │
  │   after Find                       │
  │   → subsequent Finds O(1)         │
  │                                    │
  │ Combined: amortized O(α(n)) ≈ O(1)│
  └────────────────────────────────────┘
```

---

## Example 7: Splay Tree Operations (Amortized O(log n))

```go
package main

import "fmt"

type SplayNode struct {
	key         int
	left, right *SplayNode
}

func rotateRight(node *SplayNode) *SplayNode {
	left := node.left
	node.left = left.right
	left.right = node
	return left
}

func rotateLeft(node *SplayNode) *SplayNode {
	right := node.right
	node.right = right.left
	right.left = node
	return right
}

func splay(node *SplayNode, key int) *SplayNode {
	if node == nil || node.key == key { return node }

	if key < node.key {
		if node.left == nil { return node }
		if key < node.left.key {
			node.left.left = splay(node.left.left, key)
			node = rotateRight(node)
		} else if key > node.left.key {
			node.left.right = splay(node.left.right, key)
			if node.left.right != nil { node.left = rotateLeft(node.left) }
		}
		if node.left == nil { return node }
		return rotateRight(node)
	} else {
		if node.right == nil { return node }
		if key < node.right.key {
			node.right.left = splay(node.right.left, key)
			if node.right.left != nil { node.right = rotateRight(node.right) }
		} else if key > node.right.key {
			node.right.right = splay(node.right.right, key)
			node = rotateLeft(node)
		}
		if node.right == nil { return node }
		return rotateLeft(node)
	}
}

func insert(root *SplayNode, key int) *SplayNode {
	if root == nil { return &SplayNode{key: key} }
	root = splay(root, key)
	if root.key == key { return root }
	node := &SplayNode{key: key}
	if key < root.key {
		node.left = root.left
		node.right = root
		root.left = nil
	} else {
		node.right = root.right
		node.left = root
		root.right = nil
	}
	return node
}

func inorder(node *SplayNode, result *[]int) {
	if node == nil { return }
	inorder(node.left, result)
	*result = append(*result, node.key)
	inorder(node.right, result)
}

func main() {
	var root *SplayNode

	keys := []int{10, 20, 30, 15, 25, 5}
	for _, k := range keys {
		root = insert(root, k)
		fmt.Printf("Insert %d → root=%d\n", k, root.key)
	}

	// Search splays accessed node to root
	root = splay(root, 10)
	fmt.Printf("\nAfter splay(10): root=%d\n", root.key)

	result := []int{}
	inorder(root, &result)
	fmt.Printf("Inorder: %v\n", result)

	fmt.Println("\nSplay tree: amortized O(log n) per operation")
	fmt.Println("Recently accessed elements are near root → cache friendly")
}
```

**Textual Figure:**
```
Splay Tree: Amortized O(log n)

Insert sequence: 10, 20, 30, 15, 25, 5

After insertions (each insert splays to root):
  Insert 10:  root=10
  Insert 20:  root=20 (splay 20 to top)
  Insert 30:  root=30
  Insert 15:  root=15
  Insert 25:  root=25
  Insert  5:  root=5

After splay(10):
       10  ← accessed node becomes root
      ┏  ┓
     5    25
         ┏  ┓
        15   30
           ┓
           20

  ┌─────────────────────────────────────┐
  │ Worst case single op: O(n)           │
  │ But: splaying balances the tree       │
  │ Amortized over m ops: O(m log n)     │
  │ Recently accessed → near root        │
  │ → great cache locality              │
  └─────────────────────────────────────┘
```

---

## Example 8: MaxQueue — O(1) Amortized Max

```go
package main

import "fmt"

type MaxQueue struct {
	queue []int
	deque []int // monotonic decreasing deque
}

func NewMaxQueue() *MaxQueue { return &MaxQueue{} }

func (mq *MaxQueue) Enqueue(val int) {
	mq.queue = append(mq.queue, val)
	for len(mq.deque) > 0 && mq.deque[len(mq.deque)-1] < val {
		mq.deque = mq.deque[:len(mq.deque)-1]
	}
	mq.deque = append(mq.deque, val)
}

func (mq *MaxQueue) Dequeue() int {
	val := mq.queue[0]
	mq.queue = mq.queue[1:]
	if val == mq.deque[0] {
		mq.deque = mq.deque[1:]
	}
	return val
}

func (mq *MaxQueue) Max() int { return mq.deque[0] }

func main() {
	mq := NewMaxQueue()

	ops := []int{3, 1, 5, 2, 8, 4}
	for _, v := range ops {
		mq.Enqueue(v)
		fmt.Printf("  Enqueue(%d) → max=%d\n", v, mq.Max())
	}

	for len(mq.queue) > 0 {
		m := mq.Max()
		v := mq.Dequeue()
		fmt.Printf("  Dequeue()=%d (was max=%d)\n", v, m)
	}

	fmt.Println("\nMonotonic deque maintains max in amortized O(1)")
}
```

**Textual Figure:**
```
MaxQueue: Monotonic Deque for O(1) Max

Enqueue sequence: 3, 1, 5, 2, 8, 4

  Op          Queue      Deque (monotonic ↓)  Max
  ────────    ───────    ─────────────────    ───
  Enq(3)      [3]        [3]                  3
  Enq(1)      [3,1]      [3,1]                3
  Enq(5)      [3,1,5]    [5]   ← flush 3,1    5
  Enq(2)      [3,1,5,2]  [5,2]                5
  Enq(8)      [..5,2,8]  [8]   ← flush 5,2    8
  Enq(4)      [...,8,4]  [8,4]                8

  Deque invariant: always decreasing
  ┌────────┐
  │ 8 │ 4  │  ← front is always the max
  └────────┘
  When dequeuing, if value == deque front, pop deque too
```

---

## Example 9: LRU Cache (O(1) Amortized All Operations)

```go
package main

import "fmt"

// LeetCode 146: LRU Cache

type LRUNode struct {
	key, val   int
	prev, next *LRUNode
}

type LRUCache struct {
	capacity   int
	cache      map[int]*LRUNode
	head, tail *LRUNode
}

func NewLRU(capacity int) *LRUCache {
	head := &LRUNode{}
	tail := &LRUNode{}
	head.next = tail
	tail.prev = head
	return &LRUCache{capacity: capacity, cache: make(map[int]*LRUNode), head: head, tail: tail}
}

func (l *LRUCache) remove(node *LRUNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
}

func (l *LRUCache) addToFront(node *LRUNode) {
	node.next = l.head.next
	node.prev = l.head
	l.head.next.prev = node
	l.head.next = node
}

func (l *LRUCache) Get(key int) int {
	if node, ok := l.cache[key]; ok {
		l.remove(node)
		l.addToFront(node)
		return node.val
	}
	return -1
}

func (l *LRUCache) Put(key, value int) {
	if node, ok := l.cache[key]; ok {
		node.val = value
		l.remove(node)
		l.addToFront(node)
		return
	}
	node := &LRUNode{key: key, val: value}
	l.cache[key] = node
	l.addToFront(node)
	if len(l.cache) > l.capacity {
		last := l.tail.prev
		l.remove(last)
		delete(l.cache, last.key)
	}
}

func main() {
	lru := NewLRU(2)
	lru.Put(1, 1)
	lru.Put(2, 2)
	fmt.Printf("Get(1) = %d\n", lru.Get(1))
	lru.Put(3, 3)
	fmt.Printf("Get(2) = %d (evicted)\n", lru.Get(2))
	lru.Put(4, 4)
	fmt.Printf("Get(1) = %d (evicted)\n", lru.Get(1))
	fmt.Printf("Get(3) = %d\n", lru.Get(3))
	fmt.Printf("Get(4) = %d\n", lru.Get(4))
}
```

**Textual Figure:**
```
LRU Cache (capacity = 2)

  HashMap + Doubly Linked List:
  HEAD ↔ [node] ↔ [node] ↔ TAIL
  (most recent)    (least recent)

Operations:
  Put(1,1):  HEAD ↔ [1] ↔ TAIL           map:{1}
  Put(2,2):  HEAD ↔ [2] ↔ [1] ↔ TAIL     map:{1,2}
  Get(1):    HEAD ↔ [1] ↔ [2] ↔ TAIL     (move 1 to front)
  Put(3,3):  HEAD ↔ [3] ↔ [1] ↔ TAIL     (evict 2, LRU)
             Get(2) = -1 (evicted)
  Put(4,4):  HEAD ↔ [4] ↔ [3] ↔ TAIL     (evict 1)
             Get(1) = -1 (evicted)
  Get(3)=3:  HEAD ↔ [3] ↔ [4] ↔ TAIL
  Get(4)=4:  HEAD ↔ [4] ↔ [3] ↔ TAIL

  ┌─────────────────────────────────┐
  │ All operations O(1):            │
  │   Get: HashMap lookup + move    │
  │   Put: HashMap insert + evict   │
  └─────────────────────────────────┘
```

---

## Example 10: Amortized Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Amortized Analysis Patterns ===")
	fmt.Println()

	patterns := []struct{ ds, operation, worst, amortized, method string }{
		{"Dynamic Array", "Append", "O(n)", "O(1)", "Aggregate: total cost ≤ 3n"},
		{"Stack+MultiPop", "Mixed ops", "O(n)", "O(1)", "Accounting: push pays for pop"},
		{"Binary Counter", "Increment", "O(k)", "O(1)", "Aggregate: bit k flips every 2^k"},
		{"Union-Find", "Find+Union", "O(log n)", "O(α(n))", "Potential: path compression"},
		{"Splay Tree", "Any op", "O(n)", "O(log n)", "Potential: tree balance"},
		{"Two-Stack Queue", "Dequeue", "O(n)", "O(1)", "Accounting: each element moves once"},
		{"MinStack", "Push/Pop/Min", "O(1)", "O(1)", "Auxiliary stack"},
		{"MaxQueue", "All ops", "O(n)", "O(1)", "Monotonic deque"},
		{"LRU Cache", "Get/Put", "O(1)", "O(1)", "HashMap + doubly linked list"},
		{"Hash Table", "Insert", "O(n)", "O(1)", "Resize doubles capacity"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-16s %-12s worst=%-8s amortized=%-8s %s\n",
			p.ds, p.operation, p.worst, p.amortized, p.method)
	}

	fmt.Println()
	fmt.Println("Three Methods of Amortized Analysis:")
	fmt.Println("  Aggregate: Total cost / n = amortized per op")
	fmt.Println("  Accounting: Assign credits, pay from savings")
	fmt.Println("  Potential: Φ(state) function, amortized = actual + ΔΦ")
}
```

**Textual Figure:**
```
Amortized Analysis: Three Methods

┌────────────────┬────────────┬────────┬───────────────────┐
│ Data Structure │ Worst Case │ Amort. │ Method            │
├────────────────┼────────────┼────────┼───────────────────┤
│ Dynamic Array  │ O(n)       │ O(1)   │ Aggregate         │
│ Stack+MultiPop │ O(n)       │ O(1)   │ Accounting        │
│ Binary Counter │ O(k)       │ O(1)   │ Aggregate         │
│ Union-Find     │ O(log n)   │ O(α(n))│ Potential         │
│ Splay Tree     │ O(n)       │ O(logn)│ Potential         │
│ Two-Stack Queue│ O(n)       │ O(1)   │ Accounting        │
│ MaxQueue       │ O(n)       │ O(1)   │ Monotonic deque   │
│ LRU Cache      │ O(1)       │ O(1)   │ HashMap+DLL       │
└────────────────┴────────────┴────────┴───────────────────┘

Key insight: Amortized ≠ Average!
Amortized is a GUARANTEE over any sequence.
```

---

## Key Takeaways

1. **Amortized ≠ Average**: it's a guarantee over any sequence of operations
2. **Doubling trick**: core pattern — resize costs O(n) but happens every O(n) ops
3. **Each element moves once**: stack+multipop, two-stack queue — key insight
4. **Path compression**: makes repeated operations nearly free (Union-Find)
5. **Monotonic structures**: deques for MaxQueue/MinQueue in amortized O(1)

> **Next up:** Lazy Propagation Patterns →
