# Phase 5: Linked Lists — Doubly Linked List

## What is a Doubly Linked List?

A **doubly linked list** has nodes with two pointers:
1. **Next** — points to the next node
2. **Prev** — points to the previous node

```
nil ← [10] ⇄ [20] ⇄ [30] ⇄ [40] → nil
```

This enables **O(1) deletion** when you have a reference to the node, and **bidirectional traversal**.

| Operation | Time |
|-----------|------|
| Insert at head/tail | O(1) |
| Delete given node pointer | O(1) |
| Access by index | O(n) |
| Search | O(n) |

---

## Example 1: Basic Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

type DoublyLinkedList struct {
    Head *Node
    Tail *Node
    Size int
}

func NewDLL() *DoublyLinkedList {
    return &DoublyLinkedList{}
}

func (dll *DoublyLinkedList) PushFront(val int) {
    n := &Node{Val: val}
    if dll.Head == nil {
        dll.Head = n
        dll.Tail = n
    } else {
        n.Next = dll.Head
        dll.Head.Prev = n
        dll.Head = n
    }
    dll.Size++
}

func (dll *DoublyLinkedList) PushBack(val int) {
    n := &Node{Val: val}
    if dll.Tail == nil {
        dll.Head = n
        dll.Tail = n
    } else {
        n.Prev = dll.Tail
        dll.Tail.Next = n
        dll.Tail = n
    }
    dll.Size++
}

func (dll *DoublyLinkedList) PrintForward() {
    for curr := dll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func (dll *DoublyLinkedList) PrintBackward() {
    for curr := dll.Tail; curr != nil; curr = curr.Prev {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    dll := NewDLL()
    dll.PushBack(10)
    dll.PushBack(20)
    dll.PushBack(30)
    dll.PushFront(5)

    fmt.Print("Forward:  ")
    dll.PrintForward()

    fmt.Print("Backward: ")
    dll.PrintBackward()

    fmt.Println("Size:", dll.Size)
}
```

**Textual Figure:**
```
Building the doubly linked list:

PushBack(10), PushBack(20), PushBack(30):
  head                                                tail
   ↓                                                   ↓
  ┌──────┬────┬──────┐    ┌──────┬────┬──────┐    ┌──────┬────┬──────┐
  │ Prev │ 10 │ Next │────│ Prev │ 20 │ Next │────│ Prev │ 30 │ Next │
  │ nil  │    │  ──→ │    │  ←── │    │  ──→ │    │  ←── │    │ nil  │
  └──────┴────┴──────┘    └──────┴────┴──────┘    └──────┴────┴──────┘

PushFront(5) — insert before head:
  head                                                              tail
   ↓                                                                 ↓
  ┌─────┬───┬─────┐    ┌─────┬────┬─────┐    ┌─────┬────┬─────┐    ┌─────┬────┬─────┐
  │ nil │ 5 │  ──→ │ ←── │ 10 │  ──→ │ ←── │ 20 │  ──→ │ ←── │ 30 │ nil │
  └─────┴───┴─────┘    └─────┴────┴─────┘    └─────┴────┴─────┘    └─────┴────┴─────┘

Forward:  5 ⇔ 10 ⇔ 20 ⇔ 30 ⇔ nil
Backward: 30 ⇔ 20 ⇔ 10 ⇔ 5 ⇔ nil
Size: 4
```

---

## Example 2: Delete Any Node in O(1)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

type DLL struct {
    Head *Node
    Tail *Node
    Size int
}

func NewDLL() *DLL { return &DLL{} }

func (dll *DLL) PushBack(val int) *Node {
    n := &Node{Val: val}
    if dll.Tail == nil {
        dll.Head = n
        dll.Tail = n
    } else {
        n.Prev = dll.Tail
        dll.Tail.Next = n
        dll.Tail = n
    }
    dll.Size++
    return n
}

func (dll *DLL) Remove(n *Node) {
    if n.Prev != nil {
        n.Prev.Next = n.Next
    } else {
        dll.Head = n.Next
    }
    if n.Next != nil {
        n.Next.Prev = n.Prev
    } else {
        dll.Tail = n.Prev
    }
    dll.Size--
}

func (dll *DLL) Print() {
    for curr := dll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    dll := NewDLL()
    n1 := dll.PushBack(10)
    _ = dll.PushBack(20)
    n3 := dll.PushBack(30)
    _ = dll.PushBack(40)

    dll.Print()

    // Remove middle node — O(1)
    dll.Remove(n3)
    fmt.Print("After remove 30: ")
    dll.Print()

    // Remove head — O(1)
    dll.Remove(n1)
    fmt.Print("After remove 10: ")
    dll.Print()
}
```

**Textual Figure:**
```
Initial list:
  ┌────┬────┐    ┌────┬────┐    ┌────┬────┐    ┌────┬────┐
  │ 10 │ ──→┼────│ 20 │ ──→┼────│ 30 │ ──→┼────│ 40 │ ──→ nil
  │    │    │    │    │    │    │    │    │    │    │
  nil ←─┼    │←── │    │    │←── │    │    │←── │    │
  └────┴────┘    └────┴────┘    └────┴────┘    └────┴────┘
   n1                          n3

Remove(n3) — O(1), relink prev/next:
  n3.Prev.Next = n3.Next    (20.Next = 40)
  n3.Next.Prev = n3.Prev    (40.Prev = 20)
  ┌────┬────┐    ┌────┬────┐    ┌────┬────┐
  │ 10 │ ──→┼────│ 20 │ ──→┼────│ 40 │ ──→ nil
  nil ←─┼    │←── │    │    │←── │    │
  └────┴────┘    └────┴────┘    └────┴────┘

Remove(n1) — head removal, head = n1.Next:
  ┌────┬────┐    ┌────┬────┐
  │ 20 │ ──→┼────│ 40 │ ──→ nil
  nil ←─┼    │←── │    │
  └────┴────┘    └────┴────┘

Result: 20 ⇔ 40 ⇔ nil
```

---

## Example 3: Sentinel (Dummy) Head and Tail

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

// Sentinel-based DLL: head and tail are dummy nodes
// All operations become simpler — no nil checks
type SentinelDLL struct {
    head *Node // dummy head
    tail *Node // dummy tail
    size int
}

func NewSentinelDLL() *SentinelDLL {
    h := &Node{}
    t := &Node{}
    h.Next = t
    t.Prev = h
    return &SentinelDLL{head: h, tail: t}
}

func (dll *SentinelDLL) insertAfter(node *Node, val int) *Node {
    n := &Node{Val: val, Prev: node, Next: node.Next}
    node.Next.Prev = n
    node.Next = n
    dll.size++
    return n
}

func (dll *SentinelDLL) PushFront(val int) *Node {
    return dll.insertAfter(dll.head, val)
}

func (dll *SentinelDLL) PushBack(val int) *Node {
    return dll.insertAfter(dll.tail.Prev, val)
}

func (dll *SentinelDLL) Remove(n *Node) int {
    n.Prev.Next = n.Next
    n.Next.Prev = n.Prev
    dll.size--
    return n.Val
}

func (dll *SentinelDLL) PopFront() (int, bool) {
    if dll.size == 0 {
        return 0, false
    }
    return dll.Remove(dll.head.Next), true
}

func (dll *SentinelDLL) PopBack() (int, bool) {
    if dll.size == 0 {
        return 0, false
    }
    return dll.Remove(dll.tail.Prev), true
}

func (dll *SentinelDLL) MoveToFront(n *Node) {
    dll.Remove(n)
    dll.size++ // Remove decremented, undo
    n.Prev = dll.head
    n.Next = dll.head.Next
    dll.head.Next.Prev = n
    dll.head.Next = n
}

func (dll *SentinelDLL) Print() {
    for curr := dll.head.Next; curr != dll.tail; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    dll := NewSentinelDLL()
    dll.PushBack(1)
    n2 := dll.PushBack(2)
    dll.PushBack(3)
    dll.PushFront(0)

    dll.Print() // 0 ⇄ 1 ⇄ 2 ⇄ 3 ⇄ nil

    dll.MoveToFront(n2)
    fmt.Print("After move 2 to front: ")
    dll.Print() // 2 ⇄ 0 ⇄ 1 ⇄ 3 ⇄ nil

    dll.PopBack()
    fmt.Print("After pop back: ")
    dll.Print()
}
```

**Textual Figure:**
```
Sentinel (dummy) head and tail eliminate nil checks:

Initial structure (PushBack 1, PushBack 2, PushBack 3, PushFront 0):
  dummy                                                    dummy
  head                                                     tail
   │    ┌───┬────┐  ┌───┬────┐  ┌───┬────┐  ┌───┬────┐    │
   │    │ 0 │ ──→┼──│ 1 │ ──→┼──│ 2 │ ──→┼──│ 3 │ ──→┼    │
   └──→ │   │    │  │   │    │  │   │    │  │   │    │←──┘
        │   │←── │  │   │←── │  │   │←── │  │   │←── │
        └───┴────┘  └───┴────┘  └───┴────┘  └───┴────┘
  Output: 0 ⇔ 1 ⇔ 2 ⇔ 3 ⇔ nil

MoveToFront(n2) — remove node 2, reinsert after dummy head:
   dummy   ┌───┐  ┌───┐  ┌───┐  ┌───┐  dummy
   head → │ 2 │⇔ │ 0 │⇔ │ 1 │⇔ │ 3 │← tail
          └───┘  └───┘  └───┘  └───┘
  Output: 2 ⇔ 0 ⇔ 1 ⇔ 3 ⇔ nil

PopBack() — remove tail.Prev (node 3):
   dummy   ┌───┐  ┌───┐  ┌───┐  dummy
   head → │ 2 │⇔ │ 0 │⇔ │ 1 │← tail
          └───┘  └───┘  └───┘
  Output: 2 ⇔ 0 ⇔ 1 ⇔ nil
```

---

## Example 4: LRU Cache (Classic DLL + HashMap)

```go
package main

import "fmt"

type LRUNode struct {
    Key  int
    Val  int
    Prev *LRUNode
    Next *LRUNode
}

type LRUCache struct {
    cap   int
    cache map[int]*LRUNode
    head  *LRUNode // dummy
    tail  *LRUNode // dummy
}

func NewLRUCache(capacity int) *LRUCache {
    h := &LRUNode{}
    t := &LRUNode{}
    h.Next = t
    t.Prev = h
    return &LRUCache{
        cap:   capacity,
        cache: make(map[int]*LRUNode),
        head:  h,
        tail:  t,
    }
}

func (c *LRUCache) remove(n *LRUNode) {
    n.Prev.Next = n.Next
    n.Next.Prev = n.Prev
}

func (c *LRUCache) addToFront(n *LRUNode) {
    n.Next = c.head.Next
    n.Prev = c.head
    c.head.Next.Prev = n
    c.head.Next = n
}

func (c *LRUCache) Get(key int) int {
    if n, ok := c.cache[key]; ok {
        c.remove(n)
        c.addToFront(n)
        return n.Val
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    if n, ok := c.cache[key]; ok {
        n.Val = value
        c.remove(n)
        c.addToFront(n)
        return
    }

    n := &LRUNode{Key: key, Val: value}
    c.cache[key] = n
    c.addToFront(n)

    if len(c.cache) > c.cap {
        // Evict LRU (tail.Prev)
        lru := c.tail.Prev
        c.remove(lru)
        delete(c.cache, lru.Key)
    }
}

func main() {
    cache := NewLRUCache(3)
    cache.Put(1, 100)
    cache.Put(2, 200)
    cache.Put(3, 300)
    fmt.Println("Get 1:", cache.Get(1))  // 100 (moves to front)
    cache.Put(4, 400)                     // evicts key 2
    fmt.Println("Get 2:", cache.Get(2))  // -1 (evicted)
    fmt.Println("Get 3:", cache.Get(3))  // 300
    fmt.Println("Get 4:", cache.Get(4))  // 400
}
```

**Textual Figure:**
```
LRU Cache (capacity=3) — DLL + HashMap:

Put(1,100), Put(2,200), Put(3,300):
  MRU                          LRU
  dummy    ┌───────┐  ┌───────┐  ┌───────┐  dummy
  head ─→ │ 3:300 │⇔ │ 2:200 │⇔ │ 1:100 │ ←─ tail
          └───────┘  └───────┘  └───────┘
  map: {1:•, 2:•, 3:•}

Get(1) → 100 — move key 1 to front:
  dummy    ┌───────┐  ┌───────┐  ┌───────┐  dummy
  head ─→ │ 1:100 │⇔ │ 3:300 │⇔ │ 2:200 │ ←─ tail
          └───────┘  └───────┘  └───────┘

Put(4,400) — capacity full, evict LRU (key 2):
  dummy    ┌───────┐  ┌───────┐  ┌───────┐  dummy
  head ─→ │ 4:400 │⇔ │ 1:100 │⇔ │ 3:300 │ ←─ tail
          └───────┘  └───────┘  └───────┘
  map: {1:•, 3:•, 4:•}    (key 2 evicted)

Get(2) → -1 (evicted)
Get(3) → 300
Get(4) → 400
```

---

## Example 5: Insert Before and After Given Node

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

type DLL struct {
    Head *Node
    Tail *Node
}

func (dll *DLL) PushBack(val int) *Node {
    n := &Node{Val: val}
    if dll.Tail == nil {
        dll.Head = n
        dll.Tail = n
    } else {
        n.Prev = dll.Tail
        dll.Tail.Next = n
        dll.Tail = n
    }
    return n
}

func (dll *DLL) InsertBefore(node *Node, val int) *Node {
    n := &Node{Val: val, Next: node, Prev: node.Prev}
    if node.Prev != nil {
        node.Prev.Next = n
    } else {
        dll.Head = n
    }
    node.Prev = n
    return n
}

func (dll *DLL) InsertAfter(node *Node, val int) *Node {
    n := &Node{Val: val, Prev: node, Next: node.Next}
    if node.Next != nil {
        node.Next.Prev = n
    } else {
        dll.Tail = n
    }
    node.Next = n
    return n
}

func (dll *DLL) Print() {
    for curr := dll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    dll := &DLL{}
    n1 := dll.PushBack(10)
    n2 := dll.PushBack(30)
    _ = dll.PushBack(50)

    dll.Print() // 10 ⇄ 30 ⇄ 50 ⇄ nil

    dll.InsertAfter(n1, 20)
    dll.InsertBefore(n2, 25)
    dll.Print() // 10 ⇄ 20 ⇄ 25 ⇄ 30 ⇄ 50 ⇄ nil
}
```

**Textual Figure:**
```
Initial list (PushBack 10, 30, 50):
  head                                tail
   ↓                                   ↓
  ┌────┬────┐    ┌────┬────┐    ┌────┬────┐
  │ 10 │ ──→├────│ 30 │ ──→├────│ 50 │→ nil
  nil ←─┤    │←── │    │    │←── │    │
  └────┴────┘    └────┴────┘    └────┴────┘
   n1               n2

InsertAfter(n1, 20) — new node between 10 and 30:
  ┌────┐    ┌────┐    ┌────┐    ┌────┐
  │ 10 │⇄   │ 20 │⇄   │ 30 │⇄   │ 50 │→ nil
  └────┘    └────┘    └────┘    └────┘
              new!

InsertBefore(n2=30, 25) — new node before 30:
  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
  │ 10 │⇄ │ 20 │⇄ │ 25 │⇄ │ 30 │⇄ │ 50 │→ nil
  └────┘  └────┘  └────┘  └────┘  └────┘
                    new!

Result: 10 ⇄ 20 ⇄ 25 ⇄ 30 ⇄ 50 ⇄ nil
```

---

## Example 6: Reverse a Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

func reverse(head *Node) (*Node, *Node) {
    var newTail *Node
    curr := head
    if curr != nil {
        newTail = curr // original head becomes tail
    }
    for curr != nil {
        curr.Prev, curr.Next = curr.Next, curr.Prev
        if curr.Prev == nil {
            // curr is now the new head
            return curr, newTail
        }
        curr = curr.Prev // was curr.Next before swap
    }
    return nil, nil
}

func printForward(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    // Build: 1 ⇄ 2 ⇄ 3 ⇄ 4 ⇄ 5
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < len(nodes)-1; i++ {
        nodes[i].Next = nodes[i+1]
        nodes[i+1].Prev = nodes[i]
    }

    fmt.Print("Original: ")
    printForward(nodes[0])

    newHead, _ := reverse(nodes[0])
    fmt.Print("Reversed: ")
    printForward(newHead)
}
```

**Textual Figure:**
```
Reverse a doubly linked list — swap Prev and Next for every node:

Original:
  nil ← [1] ⇄ [2] ⇄ [3] ⇄ [4] ⇄ [5] → nil
        head                        tail

Step-by-step (for each node, swap Prev ↔ Next):
  Node 1: Prev=nil,Next=2 → Prev=2,Next=nil
  Node 2: Prev=1,Next=3   → Prev=3,Next=1
  Node 3: Prev=2,Next=4   → Prev=4,Next=2
  Node 4: Prev=3,Next=5   → Prev=5,Next=3
  Node 5: Prev=4,Next=nil → Prev=nil,Next=4  ← new head!

Reversed:
  nil ← [5] ⇄ [4] ⇄ [3] ⇄ [2] ⇄ [1] → nil
        head                        tail
         (was tail)                 (was head)

Output: 5 ⇄ 4 ⇄ 3 ⇄ 2 ⇄ 1 ⇄ nil
```

---

## Example 7: Go Standard Library container/list

```go
package main

import (
    "container/list"
    "fmt"
)

func main() {
    l := list.New()

    // Insert
    l.PushBack(10)
    l.PushBack(20)
    l.PushFront(5)
    e30 := l.PushBack(30)

    // Print forward
    fmt.Print("Forward:  ")
    for e := l.Front(); e != nil; e = e.Next() {
        fmt.Printf("%v → ", e.Value)
    }
    fmt.Println("nil")

    // Print backward
    fmt.Print("Backward: ")
    for e := l.Back(); e != nil; e = e.Prev() {
        fmt.Printf("%v → ", e.Value)
    }
    fmt.Println("nil")

    // Remove
    l.Remove(e30)
    fmt.Print("After remove 30: ")
    for e := l.Front(); e != nil; e = e.Next() {
        fmt.Printf("%v → ", e.Value)
    }
    fmt.Println("nil")

    // InsertBefore / InsertAfter
    e20 := l.Front().Next() // the element with value 10's next = 20
    l.InsertBefore(15, e20)
    fmt.Print("After insert 15: ")
    for e := l.Front(); e != nil; e = e.Next() {
        fmt.Printf("%v → ", e.Value)
    }
    fmt.Println("nil")

    fmt.Println("Size:", l.Len())
}
```

**Textual Figure:**
```
Go container/list — built-in doubly linked list:

After PushBack(10), PushBack(20), PushFront(5), PushBack(30):
  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │  5  │⇄ │ 10  │⇄ │ 20  │⇄ │ 30  │
  └─────┘  └─────┘  └─────┘  └─────┘
  Front()                      Back()

Forward:  5 → 10 → 20 → 30 → nil
Backward: 30 → 20 → 10 → 5 → nil

Remove(e30):
  ┌─────┐  ┌─────┐  ┌─────┐
  │  5  │⇄ │ 10  │⇄ │ 20  │
  └─────┘  └─────┘  └─────┘

InsertBefore(15, e20):
  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │  5  │⇄ │ 10  │⇄ │ 15  │⇄ │ 20  │
  └─────┘  └─────┘  └─────┘  └─────┘
                      new!

Result: 5 → 10 → 15 → 20 → nil   Size: 4
```

---

## Example 8: Browser History (Back/Forward)

```go
package main

import "fmt"

type Page struct {
    URL  string
    Prev *Page
    Next *Page
}

type BrowserHistory struct {
    current *Page
}

func NewBrowserHistory(homepage string) *BrowserHistory {
    return &BrowserHistory{
        current: &Page{URL: homepage},
    }
}

func (bh *BrowserHistory) Visit(url string) {
    page := &Page{URL: url, Prev: bh.current}
    bh.current.Next = page
    bh.current = page
    // Clear forward history
    bh.current.Next = nil
}

func (bh *BrowserHistory) Back(steps int) string {
    for steps > 0 && bh.current.Prev != nil {
        bh.current = bh.current.Prev
        steps--
    }
    return bh.current.URL
}

func (bh *BrowserHistory) Forward(steps int) string {
    for steps > 0 && bh.current.Next != nil {
        bh.current = bh.current.Next
        steps--
    }
    return bh.current.URL
}

func main() {
    bh := NewBrowserHistory("google.com")
    bh.Visit("youtube.com")
    bh.Visit("facebook.com")

    fmt.Println("Back 1:", bh.Back(1))     // youtube.com
    fmt.Println("Back 1:", bh.Back(1))     // google.com
    fmt.Println("Forward 1:", bh.Forward(1)) // youtube.com
    bh.Visit("linkedin.com")                  // clears forward
    fmt.Println("Forward 2:", bh.Forward(2)) // linkedin.com (no more forward)
    fmt.Println("Back 2:", bh.Back(2))       // google.com
}
```

**Textual Figure:**
```
Browser history — doubly linked list navigation:

Visit: google.com → youtube.com → facebook.com
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ google.com  │ ⇄  │ youtube.com │ ⇄  │ facebook.com │
  └─────────────┘    └─────────────┘    └──────────────┘
                                          current ↑

Back(1) → youtube.com:
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ google.com  │ ⇄  │ youtube.com │ ⇄  │ facebook.com │
  └─────────────┘    └─────────────┘    └──────────────┘
                       current ↑

Back(1) → google.com:
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ google.com  │ ⇄  │ youtube.com │ ⇄  │ facebook.com │
  └─────────────┘    └─────────────┘    └──────────────┘
    current ↑

Forward(1) → youtube.com:
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ google.com  │ ⇄  │ youtube.com │ ⇄  │ facebook.com │
  └─────────────┘    └─────────────┘    └──────────────┘
                       current ↑

Visit(linkedin.com) — clears forward history:
  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐
  │ google.com  │ ⇄  │ youtube.com │ ⇄  │ linkedin.com │
  └─────────────┘    └─────────────┘    └──────────────┘
                                          current ↑
  (facebook.com discarded)

Back(2) → google.com
```

---

## Example 9: Deque with Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

type Deque struct {
    head *Node
    tail *Node
    size int
}

func (d *Deque) PushFront(val int) {
    n := &Node{Val: val}
    if d.head == nil {
        d.head = n
        d.tail = n
    } else {
        n.Next = d.head
        d.head.Prev = n
        d.head = n
    }
    d.size++
}

func (d *Deque) PushBack(val int) {
    n := &Node{Val: val}
    if d.tail == nil {
        d.head = n
        d.tail = n
    } else {
        n.Prev = d.tail
        d.tail.Next = n
        d.tail = n
    }
    d.size++
}

func (d *Deque) PopFront() (int, bool) {
    if d.head == nil {
        return 0, false
    }
    val := d.head.Val
    d.head = d.head.Next
    if d.head != nil {
        d.head.Prev = nil
    } else {
        d.tail = nil
    }
    d.size--
    return val, true
}

func (d *Deque) PopBack() (int, bool) {
    if d.tail == nil {
        return 0, false
    }
    val := d.tail.Val
    d.tail = d.tail.Prev
    if d.tail != nil {
        d.tail.Next = nil
    } else {
        d.head = nil
    }
    d.size--
    return val, true
}

func (d *Deque) PeekFront() (int, bool) {
    if d.head == nil {
        return 0, false
    }
    return d.head.Val, true
}

func (d *Deque) PeekBack() (int, bool) {
    if d.tail == nil {
        return 0, false
    }
    return d.tail.Val, true
}

func main() {
    d := &Deque{}
    d.PushBack(1)
    d.PushBack(2)
    d.PushBack(3)
    d.PushFront(0)
    d.PushFront(-1)

    fmt.Println("Size:", d.size)

    f, _ := d.PeekFront()
    b, _ := d.PeekBack()
    fmt.Printf("Front=%d, Back=%d\n", f, b)

    for d.size > 0 {
        v, _ := d.PopFront()
        fmt.Printf("Pop front: %d\n", v)
    }
}
```

**Textual Figure:**
```
Deque with doubly linked list:

PushBack(1), PushBack(2), PushBack(3), PushFront(0), PushFront(-1):
  head                                                tail
   ↓                                                   ↓
  ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
  │ -1 │ ⇄  │  0 │ ⇄  │  1 │ ⇄  │  2 │ ⇄  │  3 │
  └────┘    └────┘    └────┘    └────┘    └────┘
  PeekFront=-1                           PeekBack=3
  Size: 5

PopFront sequence:
  Pop -1: [0] ⇄ [1] ⇄ [2] ⇄ [3]
  Pop  0: [1] ⇄ [2] ⇄ [3]
  Pop  1: [2] ⇄ [3]
  Pop  2: [3]
  Pop  3: (empty)
```

---

## Example 10: Flatten a Multilevel Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val   int
    Prev  *Node
    Next  *Node
    Child *Node
}

func flatten(head *Node) *Node {
    curr := head
    for curr != nil {
        if curr.Child != nil {
            child := curr.Child
            // Find tail of child list
            childTail := child
            for childTail.Next != nil {
                childTail = childTail.Next
            }

            // Splice child list between curr and curr.Next
            childTail.Next = curr.Next
            if curr.Next != nil {
                curr.Next.Prev = childTail
            }
            curr.Next = child
            child.Prev = curr
            curr.Child = nil
        }
        curr = curr.Next
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    // Build: 1 ⇄ 2 ⇄ 3  with 2.Child = 10 ⇄ 11
    n1 := &Node{Val: 1}
    n2 := &Node{Val: 2}
    n3 := &Node{Val: 3}
    n1.Next = n2; n2.Prev = n1
    n2.Next = n3; n3.Prev = n2

    c1 := &Node{Val: 10}
    c2 := &Node{Val: 11}
    c1.Next = c2; c2.Prev = c1
    n2.Child = c1

    fmt.Println("Before flatten:")
    printList(n1)
    fmt.Print("  Child of 2: ")
    printList(c1)

    flatten(n1)
    fmt.Print("After flatten:  ")
    printList(n1)
    // 1 ⇄ 2 ⇄ 10 ⇄ 11 ⇄ 3 ⇄ nil
}
```

**Textual Figure:**
```
Flatten a multilevel doubly linked list:

Before:
  Level 0:  [1] ⇄ [2] ⇄ [3]
                   │
                   └─ Child
                   ↓
  Level 1:        [10] ⇄ [11]

Flatten process (splice child list between curr and curr.Next):
  curr = node 2, has child [10 ⇄ 11]
  1. Find child tail: 11
  2. childTail.Next = curr.Next (11.Next = 3)
  3. curr.Next.Prev = childTail (3.Prev = 11)
  4. curr.Next = child (2.Next = 10)
  5. child.Prev = curr (10.Prev = 2)
  6. curr.Child = nil

After:
  ┌───┐    ┌───┐    ┌────┐    ┌────┐    ┌───┐
  │ 1 │ ⇄  │ 2 │ ⇄  │ 10 │ ⇄  │ 11 │ ⇄  │ 3 │ → nil
  └───┘    └───┘    └────┘    └────┘    └───┘
                     spliced in!

Result: 1 ⇄ 2 ⇄ 10 ⇄ 11 ⇄ 3 ⇄ nil
```

---

## Doubly Linked List Summary

| Feature | Singly | Doubly |
|---------|--------|--------|
| Memory per node | 1 pointer | 2 pointers |
| Traverse backward | No | Yes |
| Delete given node | O(n) need predecessor | O(1) |
| Insert before node | O(n) | O(1) |
| Complexity | Simpler | Slightly more complex |

## Key Takeaways

1. **Two pointers** (Prev, Next) enable O(1) removal given a node reference
2. **Sentinel nodes** (dummy head/tail) eliminate edge cases
3. **LRU Cache** = DLL + HashMap — the classic use case
4. **container/list** — Go standard library provides a doubly linked list
5. **Deque** is naturally implemented with a DLL
6. **Browser history** — perfect example of bidirectional traversal
7. **Always update both Prev and Next** when inserting/removing

> **Next up:** Circular Linked List →
