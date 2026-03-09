# Phase 5: Linked Lists — Dummy Nodes

## What are Dummy Nodes?

A **dummy node** (also called a **sentinel node**) is a placeholder node placed at the beginning of a linked list. It simplifies edge cases by eliminating special handling for:
- Empty list
- Operations on the head node
- Returning the new head after modifications

```
dummy → [real nodes...] → nil
         ^
         dummy.Next = actual head
```

**Pattern:**
```go
dummy := &Node{}
// ... build or modify list ...
return dummy.Next  // actual head
```

---

## Example 1: Merge Two Sorted Lists

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func mergeTwoLists(l1, l2 *Node) *Node {
    dummy := &Node{} // no special head handling needed
    curr := dummy

    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            curr.Next = l1
            l1 = l1.Next
        } else {
            curr.Next = l2
            l2 = l2.Next
        }
        curr = curr.Next
    }

    if l1 != nil {
        curr.Next = l1
    } else {
        curr.Next = l2
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    l1 := fromSlice([]int{1, 3, 5, 7})
    l2 := fromSlice([]int{2, 4, 6, 8})

    merged := mergeTwoLists(l1, l2)
    printList(merged)
    // 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → nil
}
```

---

## Example 2: Remove Nth from End (Without Dummy vs With Dummy)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// WITHOUT dummy — need special case for head removal
func removeNthFromEndNoDummy(head *Node, n int) *Node {
    length := 0
    for curr := head; curr != nil; curr = curr.Next {
        length++
    }
    if n == length {
        return head.Next // special case: remove head
    }
    curr := head
    for i := 0; i < length-n-1; i++ {
        curr = curr.Next
    }
    curr.Next = curr.Next.Next
    return head
}

// WITH dummy — clean and uniform
func removeNthFromEnd(head *Node, n int) *Node {
    dummy := &Node{Next: head}
    fast, slow := dummy, dummy

    // Move fast n+1 steps ahead
    for i := 0; i <= n; i++ {
        fast = fast.Next
    }

    // Move both until fast reaches end
    for fast != nil {
        slow = slow.Next
        fast = fast.Next
    }
    slow.Next = slow.Next.Next

    return dummy.Next // handles head removal automatically
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    // Remove last
    h1 := fromSlice([]int{1, 2, 3, 4, 5})
    printList(removeNthFromEnd(h1, 2))

    // Remove head (edge case)
    h2 := fromSlice([]int{1, 2, 3})
    printList(removeNthFromEnd(h2, 3))

    // Single element
    h3 := fromSlice([]int{1})
    printList(removeNthFromEnd(h3, 1)) // nil
}
```

---

## Example 3: Remove All Occurrences of Value

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 203: Remove all nodes with Val == target
func removeElements(head *Node, target int) *Node {
    dummy := &Node{Next: head}
    curr := dummy

    for curr.Next != nil {
        if curr.Next.Val == target {
            curr.Next = curr.Next.Next
        } else {
            curr = curr.Next
        }
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    head := fromSlice([]int{6, 1, 2, 6, 3, 4, 5, 6})
    fmt.Print("Before: ")
    printList(head)

    head = removeElements(head, 6)
    fmt.Print("After removing 6: ")
    printList(head)
    // 1 → 2 → 3 → 4 → 5 → nil

    // All same values
    head2 := fromSlice([]int{7, 7, 7})
    head2 = removeElements(head2, 7)
    fmt.Print("All 7s removed: ")
    printList(head2) // nil
}
```

---

## Example 4: Partition List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 86: Partition list around x
// All nodes < x come before nodes >= x, preserve order
func partition(head *Node, x int) *Node {
    beforeDummy := &Node{}
    afterDummy := &Node{}
    before := beforeDummy
    after := afterDummy

    for curr := head; curr != nil; curr = curr.Next {
        if curr.Val < x {
            before.Next = curr
            before = before.Next
        } else {
            after.Next = curr
            after = after.Next
        }
    }

    after.Next = nil             // important: terminate the after list
    before.Next = afterDummy.Next // connect before → after
    return beforeDummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    head := fromSlice([]int{1, 4, 3, 2, 5, 2})
    fmt.Print("Before partition(3): ")
    printList(head)

    head = partition(head, 3)
    fmt.Print("After partition(3):  ")
    printList(head)
    // 1 → 2 → 2 → 4 → 3 → 5 → nil
}
```

---

## Example 5: Remove Duplicates from Sorted List II

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 82: Remove ALL nodes that have duplicates
func deleteDuplicates(head *Node) *Node {
    dummy := &Node{Next: head}
    prev := dummy

    for prev.Next != nil {
        curr := prev.Next
        // Check if curr has duplicates
        if curr.Next != nil && curr.Val == curr.Next.Val {
            // Skip all nodes with this value
            val := curr.Val
            for prev.Next != nil && prev.Next.Val == val {
                prev.Next = prev.Next.Next
            }
        } else {
            prev = prev.Next
        }
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    tests := [][]int{
        {1, 2, 3, 3, 4, 4, 5},
        {1, 1, 1, 2, 3},
        {1, 1},
    }

    for _, t := range tests {
        head := fromSlice(t)
        fmt.Printf("Input:  ")
        printList(head)
        fmt.Printf("Output: ")
        printList(deleteDuplicates(fromSlice(t)))
        fmt.Println()
    }
}
```

---

## Example 6: Add Two Numbers

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 2: Numbers stored in reverse order
func addTwoNumbers(l1, l2 *Node) *Node {
    dummy := &Node{}
    curr := dummy
    carry := 0

    for l1 != nil || l2 != nil || carry > 0 {
        sum := carry
        if l1 != nil {
            sum += l1.Val
            l1 = l1.Next
        }
        if l2 != nil {
            sum += l2.Val
            l2 = l2.Next
        }
        carry = sum / 10
        curr.Next = &Node{Val: sum % 10}
        curr = curr.Next
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    // 342 + 465 = 807
    l1 := fromSlice([]int{2, 4, 3}) // 342
    l2 := fromSlice([]int{5, 6, 4}) // 465

    fmt.Print("  ")
    printList(l1)
    fmt.Print("+ ")
    printList(l2)
    fmt.Print("= ")
    printList(addTwoNumbers(l1, l2))

    // 999 + 1 = 1000
    fmt.Println()
    a := fromSlice([]int{9, 9, 9})
    b := fromSlice([]int{1})
    printList(addTwoNumbers(a, b))
}
```

---

## Example 7: Swap Pairs

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 24: Swap every two adjacent nodes
func swapPairs(head *Node) *Node {
    dummy := &Node{Next: head}
    prev := dummy

    for prev.Next != nil && prev.Next.Next != nil {
        first := prev.Next
        second := prev.Next.Next

        // Swap
        first.Next = second.Next
        second.Next = first
        prev.Next = second

        prev = first
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    head := fromSlice([]int{1, 2, 3, 4})
    fmt.Print("Before: ")
    printList(head)
    fmt.Print("After:  ")
    printList(swapPairs(head))
    // 2 → 1 → 4 → 3 → nil

    head2 := fromSlice([]int{1, 2, 3})
    fmt.Print("\nBefore: ")
    printList(head2)
    fmt.Print("After:  ")
    printList(swapPairs(head2))
    // 2 → 1 → 3 → nil
}
```

---

## Example 8: Insert into Sorted List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func insertSorted(head *Node, val int) *Node {
    dummy := &Node{Next: head}
    curr := dummy

    for curr.Next != nil && curr.Next.Val < val {
        curr = curr.Next
    }

    newNode := &Node{Val: val, Next: curr.Next}
    curr.Next = newNode

    return dummy.Next
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    var head *Node

    // Build sorted list by inserting
    for _, v := range []int{5, 1, 8, 3, 10, 2} {
        head = insertSorted(head, v)
        printList(head)
    }
}
```

---

## Example 9: Reverse Linked List II (Reverse Sublist)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// LeetCode 92: Reverse from position left to right (1-indexed)
func reverseBetween(head *Node, left, right int) *Node {
    dummy := &Node{Next: head}
    prev := dummy

    // Move to node before 'left'
    for i := 1; i < left; i++ {
        prev = prev.Next
    }

    // Reverse between left and right
    curr := prev.Next
    for i := 0; i < right-left; i++ {
        next := curr.Next
        curr.Next = next.Next
        next.Next = prev.Next
        prev.Next = next
    }

    return dummy.Next
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    head := fromSlice([]int{1, 2, 3, 4, 5})
    fmt.Print("Before: ")
    printList(head)

    head = reverseBetween(head, 2, 4)
    fmt.Print("Rev [2,4]: ")
    printList(head)
    // 1 → 4 → 3 → 2 → 5 → nil

    // Reverse entire list
    head2 := fromSlice([]int{1, 2, 3, 4, 5})
    head2 = reverseBetween(head2, 1, 5)
    fmt.Print("Rev [1,5]: ")
    printList(head2)
    // 5 → 4 → 3 → 2 → 1 → nil
}
```

---

## Example 10: Build List Without Tracking Head

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    // Without dummy: must track head separately
    withoutDummy := func(vals []int) *Node {
        var head *Node
        var tail *Node
        for _, v := range vals {
            n := &Node{Val: v}
            if head == nil {
                head = n
                tail = n
            } else {
                tail.Next = n
                tail = n
            }
        }
        return head
    }

    // With dummy: cleaner — no head check
    withDummy := func(vals []int) *Node {
        dummy := &Node{}
        tail := dummy
        for _, v := range vals {
            tail.Next = &Node{Val: v}
            tail = tail.Next
        }
        return dummy.Next
    }

    fmt.Print("Without dummy: ")
    printList(withoutDummy([]int{1, 2, 3, 4, 5}))

    fmt.Print("With dummy:    ")
    printList(withDummy([]int{1, 2, 3, 4, 5}))

    // Empty list — dummy handles it naturally
    fmt.Print("Empty:         ")
    printList(withDummy(nil)) // nil
}
```

---

## When to Use Dummy Nodes

| Situation | Without Dummy | With Dummy |
|-----------|--------------|------------|
| Delete head | Special case needed | Same logic as any node |
| Build list from scratch | Track head + tail | Just tail |
| Merge lists | Check which is head | Append freely |
| Modify and return head | Track new head | `dummy.Next` |
| Empty list input | nil checks everywhere | Works naturally |

## Key Takeaways

1. **Dummy node** = placeholder before real head → `dummy.Next` is the actual head
2. **Eliminates edge cases**: head deletion, empty list, building from scratch
3. **Pattern**: `dummy := &Node{Next: head}` ... `return dummy.Next`
4. **No memory waste**: single extra node, garbage collected after function returns
5. **Essential for**: merge, partition, remove by value, reverse sublist
6. **Used in 90%+ of linked list interview solutions** — always consider it first
7. **Two dummies**: partition problems use two dummy nodes (before/after)

> **Next up:** Reversal Algorithms →
