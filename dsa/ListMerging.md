# Phase 5: Linked Lists — List Merging

## Overview

**List merging** combines two or more sorted (or unsorted) linked lists into a single list. It appears in merge sort, priority queue implementations, and many interview problems.

| Pattern | Time | Space |
|---------|------|-------|
| Merge 2 sorted | O(n + m) | O(1) |
| Merge K sorted (heap) | O(N log K) | O(K) |
| Merge K sorted (divide & conquer) | O(N log K) | O(log K) stack |

---

## Example 1: Merge Two Sorted Lists (Iterative)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func mergeTwoLists(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy

    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            tail.Next = l1
            l1 = l1.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
        }
        tail = tail.Next
    }

    if l1 != nil {
        tail.Next = l1
    } else {
        tail.Next = l2
    }

    return dummy.Next
}

func toList(vals []int) *Node {
    dummy := &Node{}
    tail := dummy
    for _, v := range vals {
        tail.Next = &Node{Val: v}
        tail = tail.Next
    }
    return dummy.Next
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 3, 5, 7})
    l2 := toList([]int{2, 4, 6, 8})
    printList(mergeTwoLists(l1, l2))

    l3 := toList([]int{1, 1, 1})
    l4 := toList([]int{1, 1})
    printList(mergeTwoLists(l3, l4))

    printList(mergeTwoLists(nil, toList([]int{1, 2})))
    printList(mergeTwoLists(nil, nil))
}
```

**Textual Figure:**
```
Merge two sorted lists (iterative):

l1: ┌───┐   ┌───┐   ┌───┐   ┌───┐
    │ 1 │──→│ 3 │──→│ 5 │──→│ 7 │──→ nil
    └───┘   └───┘   └───┘   └───┘

l2: ┌───┐   ┌───┐   ┌───┐   ┌───┐
    │ 2 │──→│ 4 │──→│ 6 │──→│ 8 │──→ nil
    └───┘   └───┘   └───┘   └───┘

Step-by-step (tail starts at dummy):
  1≤2 → pick 1   D──→[1]
  3>2  → pick 2   D──→[1]──→[2]
  3≤4 → pick 3   D──→[1]──→[2]──→[3]
  5>4  → pick 4   ...──→[3]──→[4]
  5≤6 → pick 5   ...──→[4]──→[5]
  7>6  → pick 6   ...──→[5]──→[6]
  7≤8 → pick 7   ...──→[6]──→[7]
  l1=nil → attach [8]

Result:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 5 │─→│ 6 │─→│ 7 │─→│ 8 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
```

---

## Example 2: Merge Two Sorted Lists (Recursive)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func mergeTwoListsRec(l1, l2 *Node) *Node {
    if l1 == nil {
        return l2
    }
    if l2 == nil {
        return l1
    }

    if l1.Val <= l2.Val {
        l1.Next = mergeTwoListsRec(l1.Next, l2)
        return l1
    }
    l2.Next = mergeTwoListsRec(l1, l2.Next)
    return l2
}

func toList(vals []int) *Node {
    dummy := &Node{}
    t := dummy
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return dummy.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 2, 4})
    l2 := toList([]int{1, 3, 4})
    printList(mergeTwoListsRec(l1, l2))
    // 1 → 1 → 2 → 3 → 4 → 4 → nil
}
```

**Textual Figure:**
```
Recursive merge of l1=[1,2,4] and l2=[1,3,4]:

Call tree (pick smaller head, recurse on rest):
  merge(1→2→4, 1→3→4)
    1≤1 → pick l1(1), l1.Next = merge(2→4, 1→3→4)
    │ merge(2→4, 1→3→4)
    │   2>1 → pick l2(1), l2.Next = merge(2→4, 3→4)
    │   │ merge(2→4, 3→4)
    │   │   2≤3 → pick l1(2), l1.Next = merge(4, 3→4)
    │   │   │ merge(4, 3→4)
    │   │   │   4>3 → pick l2(3), l2.Next = merge(4, 4)
    │   │   │   │ merge(4, 4)
    │   │   │   │   4≤4 → pick l1(4), l1.Next = merge(nil, 4)
    │   │   │   │   │ merge(nil, 4) → return 4
    │   │   │   │   return 4→4
    │   │   │   return 3→4→4
    │   │   return 2→3→4→4
    │   return 1→2→3→4→4
    return 1→1→2→3→4→4

Result:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │─→│ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 4 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
```

---

## Example 3: Merge K Sorted Lists (Min-Heap)

```go
package main

import (
    "container/heap"
    "fmt"
)

type Node struct {
    Val  int
    Next *Node
}

type MinHeap []*Node

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].Val < h[j].Val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*Node)) }
func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    *h = old[:n-1]
    return item
}

func mergeKLists(lists []*Node) *Node {
    h := &MinHeap{}
    heap.Init(h)

    for _, l := range lists {
        if l != nil {
            heap.Push(h, l)
        }
    }

    dummy := &Node{}
    tail := dummy

    for h.Len() > 0 {
        smallest := heap.Pop(h).(*Node)
        tail.Next = smallest
        tail = tail.Next
        if smallest.Next != nil {
            heap.Push(h, smallest.Next)
        }
    }

    return dummy.Next
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    lists := []*Node{
        toList([]int{1, 4, 7}),
        toList([]int{2, 5, 8}),
        toList([]int{3, 6, 9}),
    }
    printList(mergeKLists(lists))
    // 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → nil
}
```

**Textual Figure:**
```
Merge K=3 sorted lists using Min-Heap:

  List 0: [1]─→[4]─→[7]
  List 1: [2]─→[5]─→[8]
  List 2: [3]─→[6]─→[9]

Initialize heap with heads:
  Heap: [1, 2, 3]

  Pop 1, push 4  → Heap: [2, 3, 4]  Result: D──→[1]
  Pop 2, push 5  → Heap: [3, 4, 5]  Result: ...──→[2]
  Pop 3, push 6  → Heap: [4, 5, 6]  Result: ...──→[3]
  Pop 4, push 7  → Heap: [5, 6, 7]  Result: ...──→[4]
  Pop 5, push 8  → Heap: [6, 7, 8]  Result: ...──→[5]
  Pop 6, push 9  → Heap: [7, 8, 9]  Result: ...──→[6]
  Pop 7, nil     → Heap: [8, 9]     Result: ...──→[7]
  Pop 8, nil     → Heap: [9]        Result: ...──→[8]
  Pop 9, nil     → Heap: []         Result: ...──→[9]

Result:
  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
  │ 1 │→│ 2 │→│ 3 │→│ 4 │→│ 5 │→│ 6 │→│ 7 │→│ 8 │→│ 9 │→nil
  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘
```

---

## Example 4: Merge K Sorted Lists (Divide and Conquer)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func merge2(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            tail.Next = l1
            l1 = l1.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
        }
        tail = tail.Next
    }
    if l1 != nil {
        tail.Next = l1
    } else {
        tail.Next = l2
    }
    return dummy.Next
}

func mergeKLists(lists []*Node) *Node {
    n := len(lists)
    if n == 0 {
        return nil
    }
    if n == 1 {
        return lists[0]
    }

    mid := n / 2
    left := mergeKLists(lists[:mid])
    right := mergeKLists(lists[mid:])
    return merge2(left, right)
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    lists := []*Node{
        toList([]int{1, 5, 9}),
        toList([]int{2, 6}),
        toList([]int{3, 7, 10}),
        toList([]int{4, 8}),
    }
    printList(mergeKLists(lists))
    // 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → nil
}
```

**Textual Figure:**
```
Merge K=4 sorted lists (Divide & Conquer):

  List 0: [1,5,9]   List 1: [2,6]   List 2: [3,7,10]   List 3: [4,8]

  Recursive split:
                    mergeKLists([0..3])
                   /                    \
       mergeKLists([0,1])          mergeKLists([2,3])
        /            \               /            \
    list[0]       list[1]       list[2]        list[3]
   [1,5,9]        [2,6]       [3,7,10]        [4,8]

  Merge upward:
    merge([1,5,9], [2,6])    → [1,2,5,6,9]
    merge([3,7,10], [4,8])   → [3,4,7,8,10]
    merge([1,2,5,6,9], [3,4,7,8,10]) → [1,2,3,4,5,6,7,8,9,10]

Result:
  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌────┐
  │ 1 │→│ 2 │→│ 3 │→│ 4 │→│ 5 │→│ 6 │→│ 7 │→│ 8 │→│ 9 │→│ 10 │→nil
  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └────┘
```

---

## Example 5: Merge in Place (No Extra Nodes)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Merges l2 into l1 without creating new nodes
func mergeInPlace(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy

    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            tail.Next = l1
            l1 = l1.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
        }
        tail = tail.Next
    }
    if l1 != nil {
        tail.Next = l1
    } else {
        tail.Next = l2
    }
    return dummy.Next
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 3, 5})
    l2 := toList([]int{2, 4, 6})

    // Count nodes before
    count := 0
    for c := l1; c != nil; c = c.Next {
        count++
    }
    for c := l2; c != nil; c = c.Next {
        count++
    }

    merged := mergeInPlace(l1, l2)
    mergedCount := 0
    for c := merged; c != nil; c = c.Next {
        mergedCount++
    }

    printList(merged)
    fmt.Printf("Total nodes: before=%d, after=%d (same!)\n", count, mergedCount)
}
```

**Textual Figure:**
```
Merge in place (reuse existing nodes, no allocation):

l1: ┌───┐   ┌───┐   ┌───┐
    │ 1 │──→│ 3 │──→│ 5 │──→ nil
    └───┘   └───┘   └───┘

l2: ┌───┐   ┌───┐   ┌───┐
    │ 2 │──→│ 4 │──→│ 6 │──→ nil
    └───┘   └───┘   └───┘

Pointer rewiring (same 6 nodes, no new allocations):
  tail = dummy
  1≤2 → tail.Next = l1[1]  l1 advances
  3>2  → tail.Next = l2[2]  l2 advances
  3≤4 → tail.Next = l1[3]  l1 advances
  5>4  → tail.Next = l2[4]  l2 advances
  5≤6 → tail.Next = l1[5]  l1 advances (nil)
  attach remaining l2[6]

Result (same nodes, relinked):
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→│ 6 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
  Total nodes: before=6, after=6 (same!)
```

---

## Example 6: Merge Sort a Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func mergeSort(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }

    // Split into halves
    slow, fast := head, head.Next
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    mid := slow.Next
    slow.Next = nil

    left := mergeSort(head)
    right := mergeSort(mid)
    return merge(left, right)
}

func merge(a, b *Node) *Node {
    dummy := &Node{}
    tail := dummy
    for a != nil && b != nil {
        if a.Val <= b.Val {
            tail.Next = a
            a = a.Next
        } else {
            tail.Next = b
            b = b.Next
        }
        tail = tail.Next
    }
    if a != nil {
        tail.Next = a
    } else {
        tail.Next = b
    }
    return dummy.Next
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d ", c.Val)
    }
    fmt.Println()
}

func main() {
    tests := [][]int{
        {4, 2, 1, 3},
        {-1, 5, 3, 4, 0},
        {1},
        {},
        {5, 4, 3, 2, 1},
    }

    for _, t := range tests {
        fmt.Printf("Before: %v → After: ", t)
        printList(mergeSort(toList(t)))
    }
}
```

**Textual Figure:**
```
Merge sort linked list [4, 2, 1, 3]:

Split phase (find middle with slow/fast, cut):
                [4,2,1,3]
               /          \
          [4,2]            [1,3]
         /     \          /     \
       [4]     [2]      [1]     [3]

Merge phase (merge sorted halves):
       [4]     [2]      [1]     [3]
         \     /          \     /
          [2,4]            [1,3]
               \          /
                [1,2,3,4]

Result:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→ nil
  └───┘   └───┘   └───┘   └───┘
```

---

## Example 7: Interleave / Zip Merge

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Interleave: a1→b1→a2→b2→a3→b3...
func interleave(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy
    turn := true

    for l1 != nil || l2 != nil {
        if turn && l1 != nil {
            tail.Next = l1
            l1 = l1.Next
            tail = tail.Next
        } else if !turn && l2 != nil {
            tail.Next = l2
            l2 = l2.Next
            tail = tail.Next
        } else if l1 != nil {
            tail.Next = l1
            l1 = l1.Next
            tail = tail.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
            tail = tail.Next
        }
        turn = !turn
    }
    tail.Next = nil

    return dummy.Next
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 3, 5})
    l2 := toList([]int{2, 4, 6})
    fmt.Print("Equal length: ")
    printList(interleave(l1, l2))
    // 1 → 2 → 3 → 4 → 5 → 6 → nil

    l3 := toList([]int{1, 3, 5, 7, 9})
    l4 := toList([]int{2, 4})
    fmt.Print("Unequal: ")
    printList(interleave(l3, l4))
    // 1 → 2 → 3 → 4 → 5 → 7 → 9 → nil
}
```

**Textual Figure:**
```
Interleave (zip merge) two lists:

l1: ┌───┐   ┌───┐   ┌───┐
    │ 1 │──→│ 3 │──→│ 5 │──→ nil
    └───┘   └───┘   └───┘

l2: ┌───┐   ┌───┐   ┌───┐
    │ 2 │──→│ 4 │──→│ 6 │──→ nil
    └───┘   └───┘   └───┘

Alternating picks (turn toggles l1/l2):
  turn=l1 → pick 1   D──→[1]
  turn=l2 → pick 2   D──→[1]──→[2]
  turn=l1 → pick 3   ...──→[2]──→[3]
  turn=l2 → pick 4   ...──→[3]──→[4]
  turn=l1 → pick 5   ...──→[4]──→[5]
  turn=l2 → pick 6   ...──→[5]──→[6]

Result:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→│ 6 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
  l1       l2       l1       l2       l1       l2
```

---

## Example 8: Merge Two Lists at Alternating Positions

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Insert nodes of l2 into l1 at alternating positions
func mergeAlternate(l1, l2 *Node) (*Node, *Node) {
    if l1 == nil {
        return l2, nil
    }

    curr1 := l1
    curr2 := l2

    for curr1 != nil && curr2 != nil {
        next1 := curr1.Next
        next2 := curr2.Next

        curr1.Next = curr2
        curr2.Next = next1

        curr1 = next1
        curr2 = next2
    }

    return l1, curr2 // remaining of l2
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 2, 3})
    l2 := toList([]int{10, 20, 30, 40, 50})

    merged, remaining := mergeAlternate(l1, l2)
    fmt.Print("Merged:    ")
    printList(merged)
    fmt.Print("Remaining: ")
    printList(remaining)
}
```

**Textual Figure:**
```
Merge at alternating positions:
  l1: [1]──→[2]──→[3]──→nil
  l2: [10]─→[20]─→[30]─→[40]─→[50]─→nil

Step 1: curr1=[1], curr2=[10]
  [1].Next = [10],  [10].Next = [2]  (was [1].Next)
  ┌───┐   ┌────┐   ┌───┐
  │ 1 │──→│ 10 │──→│ 2 │──→...
  └───┘   └────┘   └───┘

Step 2: curr1=[2], curr2=[20]
  [2].Next = [20],  [20].Next = [3]

Step 3: curr1=[3], curr2=[30]
  [3].Next = [30],  [30].Next = nil (was [3].Next)
  curr1 = nil → stop!
  curr2 = [40] → remaining

Merged:
  ┌───┐  ┌────┐  ┌───┐  ┌────┐  ┌───┐  ┌────┐
  │ 1 │─→│ 10 │─→│ 2 │─→│ 20 │─→│ 3 │─→│ 30 │─→nil
  └───┘  └────┘  └───┘  └────┘  └───┘  └────┘

Remaining:
  ┌────┐  ┌────┐
  │ 40 │─→│ 50 │─→nil
  └────┘  └────┘
```

---

## Example 9: Union and Intersection of Two Sorted Lists

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func unionSorted(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy

    for l1 != nil && l2 != nil {
        if l1.Val < l2.Val {
            tail.Next = &Node{Val: l1.Val}
            l1 = l1.Next
        } else if l1.Val > l2.Val {
            tail.Next = &Node{Val: l2.Val}
            l2 = l2.Next
        } else {
            tail.Next = &Node{Val: l1.Val}
            l1 = l1.Next
            l2 = l2.Next
        }
        tail = tail.Next
    }
    for l1 != nil {
        tail.Next = &Node{Val: l1.Val}
        tail = tail.Next
        l1 = l1.Next
    }
    for l2 != nil {
        tail.Next = &Node{Val: l2.Val}
        tail = tail.Next
        l2 = l2.Next
    }
    return dummy.Next
}

func intersectionSorted(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy

    for l1 != nil && l2 != nil {
        if l1.Val < l2.Val {
            l1 = l1.Next
        } else if l1.Val > l2.Val {
            l2 = l2.Next
        } else {
            tail.Next = &Node{Val: l1.Val}
            tail = tail.Next
            l1 = l1.Next
            l2 = l2.Next
        }
    }
    return dummy.Next
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d → ", c.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := toList([]int{1, 2, 3, 5, 7})
    l2 := toList([]int{2, 4, 5, 6, 7, 8})

    fmt.Print("Union:        ")
    printList(unionSorted(toList([]int{1, 2, 3, 5, 7}), toList([]int{2, 4, 5, 6, 7, 8})))

    fmt.Print("Intersection: ")
    printList(intersectionSorted(l1, l2))
}
```

**Textual Figure:**
```
Union and Intersection of sorted lists:

  l1: ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
      │ 1 │─→│ 2 │─→│ 3 │─→│ 5 │─→│ 7 │─→nil
      └───┘  └───┘  └───┘  └───┘  └───┘

  l2: ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
      │ 2 │─→│ 4 │─→│ 5 │─→│ 6 │─→│ 7 │─→│ 8 │─→nil
      └───┘  └───┘  └───┘  └───┘  └───┘  └───┘

Union (all unique values):
  1<2 → add 1 | 2==2 → add 2 | 3<4 → add 3 | 4<5 → add 4
  5==5 → add 5 | 6<7 → add 6 | 7==7 → add 7 | add remaining [8]
  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
  │ 1 │→│ 2 │→│ 3 │→│ 4 │→│ 5 │→│ 6 │→│ 7 │→│ 8 │→nil
  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘

Intersection (only common values):
  1<2 skip l1 | 2==2 add 2 | 3<4 skip l1 | 5==5 add 5 | 7==7 add 7
  ┌───┐   ┌───┐   ┌───┐
  │ 2 │──→│ 5 │──→│ 7 │──→ nil
  └───┘   └───┘   └───┘
```

---

## Example 10: Merge K Sorted Lists (Iterative Pairwise)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func merge2(l1, l2 *Node) *Node {
    dummy := &Node{}
    tail := dummy
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            tail.Next = l1
            l1 = l1.Next
        } else {
            tail.Next = l2
            l2 = l2.Next
        }
        tail = tail.Next
    }
    if l1 != nil {
        tail.Next = l1
    } else {
        tail.Next = l2
    }
    return dummy.Next
}

// Iterative pairwise merge — reduces K lists to 1
func mergeKIterative(lists []*Node) *Node {
    if len(lists) == 0 {
        return nil
    }

    for len(lists) > 1 {
        merged := make([]*Node, 0, (len(lists)+1)/2)
        for i := 0; i < len(lists); i += 2 {
            if i+1 < len(lists) {
                merged = append(merged, merge2(lists[i], lists[i+1]))
            } else {
                merged = append(merged, lists[i])
            }
        }
        lists = merged
    }
    return lists[0]
}

func toList(vals []int) *Node {
    d := &Node{}
    t := d
    for _, v := range vals {
        t.Next = &Node{Val: v}
        t = t.Next
    }
    return d.Next
}

func printList(head *Node) {
    for c := head; c != nil; c = c.Next {
        fmt.Printf("%d ", c.Val)
    }
    fmt.Println()
}

func main() {
    lists := []*Node{
        toList([]int{1, 10, 20}),
        toList([]int{4, 11, 13}),
        toList([]int{3, 8, 9}),
        toList([]int{2, 6}),
        toList([]int{5, 7, 14, 15}),
    }

    fmt.Print("Merged: ")
    printList(mergeKIterative(lists))
    // 1 2 3 4 5 6 7 8 9 10 11 13 14 15 20
}
```

**Textual Figure:**
```
Merge K=5 sorted lists (Iterative Pairwise):

  L0: [1,10,20]  L1: [4,11,13]  L2: [3,8,9]  L3: [2,6]  L4: [5,7,14,15]

Round 1 — pair & merge:
  merge(L0, L1) → [1,4,10,11,13,20]
  merge(L2, L3) → [2,3,6,8,9]
  L4 unpaired   → [5,7,14,15]
  → 3 lists remain

Round 2 — pair & merge:
  merge([1,4,10,11,13,20], [2,3,6,8,9]) → [1,2,3,4,6,8,9,10,11,13,20]
  [5,7,14,15] unpaired
  → 2 lists remain

Round 3 — final merge:
  merge([1,2,3,4,6,8,9,10,11,13,20], [5,7,14,15])

Result:
  1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 13 → 14 → 15 → 20

  Tournament bracket:
    L0  L1    L2  L3    L4
     \ /       \ /       |
    merge     merge      |
       \       /         |
        merge            |
            \           /
             final merge
```

---

## Example 11: Find Intersection Point of Two Lists

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Two lists share a common tail (Y-shaped)
func getIntersectionNode(headA, headB *Node) *Node {
    if headA == nil || headB == nil {
        return nil
    }

    a, b := headA, headB
    for a != b {
        if a == nil {
            a = headB
        } else {
            a = a.Next
        }
        if b == nil {
            b = headA
        } else {
            b = b.Next
        }
    }
    return a
}

func main() {
    // Shared tail: 7 → 8 → nil
    shared := &Node{Val: 7, Next: &Node{Val: 8}}

    // List A: 1 → 3 → 7 → 8
    headA := &Node{Val: 1, Next: &Node{Val: 3, Next: shared}}

    // List B: 2 → 4 → 5 → 7 → 8
    headB := &Node{Val: 2, Next: &Node{Val: 4, Next: &Node{Val: 5, Next: shared}}}

    inter := getIntersectionNode(headA, headB)
    if inter != nil {
        fmt.Printf("Intersection at node %d\n", inter.Val) // 7
    }

    // Why this works:
    // A traverses: lenA + lenB - common
    // B traverses: lenB + lenA - common
    // They travel the same distance → meet at intersection or nil
}
```

**Textual Figure:**
```
Find intersection point of two Y-shaped lists:

  List A: [1]──→[3]──┐
                     │
                     └──→[7]──→[8]──→nil     (shared tail)
                     ┌──┘
  List B: [2]─→[4]─→[5]

Two-pointer technique:
  a starts at headA, b starts at headB
  When a reaches nil, redirect to headB
  When b reaches nil, redirect to headA

  Pointer a: 1 → 3 → 7 → 8 → nil → 2 → 4 → 5 → [7]
  Pointer b: 2 → 4 → 5 → 7 → 8 → nil → 1 → 3 → [7]
                                                  ↑
                                       a == b at node 7!

  Why it works:
    a travels: lenA + lenB - common = 2 + 3 - 2 = 3 extra
    b travels: lenB + lenA - common = 3 + 2 - 2 = 3 extra
    Same total distance → they meet at intersection!

  Intersection at node 7 ✓
```

---

## Merge K Sorted Lists: Approach Comparison

| Approach | Time | Space | Notes |
|----------|------|-------|-------|
| Brute force (merge one-by-one) | O(NK) | O(1) | Slow for large K |
| Min-Heap | O(N log K) | O(K) | Best for streaming |
| Divide & Conquer | O(N log K) | O(log K) | Best overall |
| Iterative Pairwise | O(N log K) | O(1)* | Tournament style |

*O(1) if we reuse the lists slice.

## Key Takeaways

1. **Dummy node** is essential — avoids special-casing the head
2. **Merge 2 sorted lists** is the building block for all K-list approaches
3. **Heap approach** maintains K list heads — always pop smallest, push its next
4. **Divide & conquer** recursively pairs lists — O(N log K) like merge sort
5. **Interleave merge** preserves original order — not sorted, just alternating
6. **Intersection point**: two-pointer technique with length equalization
7. **Merge sort on linked list**: O(n log n) time, O(log n) stack — linked lists are ideal for merge sort since splitting/merging needs no extra array

> **Phase 5 Complete!** Next up: Phase 6 — Stack →
