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
