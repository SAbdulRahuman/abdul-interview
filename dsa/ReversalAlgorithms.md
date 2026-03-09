# Phase 5: Linked Lists — Reversal Algorithms

## Overview

Reversing a linked list is one of the most fundamental operations. There are several variations:

| Variation | Description |
|-----------|-------------|
| Full reversal | Reverse entire list |
| Partial reversal | Reverse between positions m and n |
| K-group reversal | Reverse in groups of k |
| Alternating reversal | Reverse every other group |

---

## Example 1: Iterative Full Reversal

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseIterative(head *Node) *Node {
    var prev *Node
    curr := head

    for curr != nil {
        next := curr.Next  // save next
        curr.Next = prev   // reverse pointer
        prev = curr        // advance prev
        curr = next        // advance curr
    }
    return prev
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

    head = reverseIterative(head)
    fmt.Print("After:  ")
    printList(head)
}
```

---

## Example 2: Recursive Full Reversal

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseRecursive(head *Node) *Node {
    // Base case: empty or single node
    if head == nil || head.Next == nil {
        return head
    }

    // Recurse: reverse the rest
    newHead := reverseRecursive(head.Next)

    // head.Next is the LAST node of the reversed sublist
    // Make it point back to head
    head.Next.Next = head
    head.Next = nil

    return newHead
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

    head = reverseRecursive(head)
    fmt.Print("After:  ")
    printList(head)

    // Trace for [1,2,3]:
    // reverseRecursive(1→2→3)
    //   reverseRecursive(2→3)
    //     reverseRecursive(3) → returns 3
    //   2.Next.Next = 2 → 3→2, 2.Next = nil
    //   returns 3→2
    // 1.Next.Next = 1 → 2→1, 1.Next = nil
    // returns 3→2→1
}
```

---

## Example 3: Reverse Between Positions (LeetCode 92)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseBetween(head *Node, left, right int) *Node {
    dummy := &Node{Next: head}
    prev := dummy

    // Move prev to node before position 'left'
    for i := 1; i < left; i++ {
        prev = prev.Next
    }

    // Repeatedly move the next node to the front of the sublist
    curr := prev.Next
    for i := 0; i < right-left; i++ {
        nodeToMove := curr.Next
        curr.Next = nodeToMove.Next
        nodeToMove.Next = prev.Next
        prev.Next = nodeToMove
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
    tests := []struct {
        arr         []int
        left, right int
    }{
        {[]int{1, 2, 3, 4, 5}, 2, 4},
        {[]int{1, 2, 3, 4, 5}, 1, 5},
        {[]int{5}, 1, 1},
    }

    for _, t := range tests {
        head := fromSlice(t.arr)
        fmt.Printf("Reverse [%d,%d]: ", t.left, t.right)
        printList(reverseBetween(head, t.left, t.right))
    }
}
```

---

## Example 4: Reverse in Groups of K (LeetCode 25)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseKGroup(head *Node, k int) *Node {
    // Check if there are k nodes remaining
    curr := head
    count := 0
    for curr != nil && count < k {
        curr = curr.Next
        count++
    }
    if count < k {
        return head // not enough nodes, leave as is
    }

    // Reverse k nodes
    var prev *Node
    curr = head
    for i := 0; i < k; i++ {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }

    // head is now the tail of reversed group
    // curr is the head of remaining list
    head.Next = reverseKGroup(curr, k)

    return prev
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
    head := fromSlice([]int{1, 2, 3, 4, 5, 6, 7, 8})

    fmt.Print("Original: ")
    printList(head)

    head = reverseKGroup(head, 3)
    fmt.Print("K=3:      ")
    printList(head)
    // 3 → 2 → 1 → 6 → 5 → 4 → 7 → 8 → nil (7,8 not reversed — less than k)
}
```

---

## Example 5: Reverse K-Group Iterative

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseKGroupIterative(head *Node, k int) *Node {
    dummy := &Node{Next: head}
    groupPrev := dummy

    for {
        // Check if k nodes exist
        kth := groupPrev
        for i := 0; i < k; i++ {
            kth = kth.Next
            if kth == nil {
                return dummy.Next
            }
        }
        groupNext := kth.Next

        // Reverse the group
        prev := groupNext
        curr := groupPrev.Next
        for curr != groupNext {
            next := curr.Next
            curr.Next = prev
            prev = curr
            curr = next
        }

        // Connect with previous part
        tmp := groupPrev.Next
        groupPrev.Next = kth
        groupPrev = tmp
    }
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
    for _, k := range []int{2, 3, 4} {
        head := fromSlice([]int{1, 2, 3, 4, 5, 6, 7, 8})
        fmt.Printf("K=%d: ", k)
        printList(reverseKGroupIterative(head, k))
    }
}
```

---

## Example 6: Reverse Alternating K-Group

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Reverse first k nodes, skip next k nodes, repeat
func reverseAlternateKGroup(head *Node, k int) *Node {
    curr := head
    count := 0
    for curr != nil && count < k {
        curr = curr.Next
        count++
    }
    if count < k {
        return head
    }

    // Reverse k nodes
    var prev *Node
    curr = head
    for i := 0; i < k; i++ {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }

    // head is now tail of reversed part
    // Skip next k nodes
    head.Next = curr
    skip := curr
    for i := 0; i < k-1 && skip != nil; i++ {
        skip = skip.Next
    }

    if skip != nil {
        skip.Next = reverseAlternateKGroup(skip.Next, k)
    }

    return prev
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
    head := fromSlice([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
    fmt.Print("Original: ")
    printList(head)

    head = reverseAlternateKGroup(head, 3)
    fmt.Print("Alt K=3:  ")
    printList(head)
    // 3 → 2 → 1 → 4 → 5 → 6 → 9 → 8 → 7 → 10 → nil
}
```

---

## Example 7: Reverse First N Nodes

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

var successor *Node

func reverseFirstN(head *Node, n int) *Node {
    if n == 1 {
        successor = head.Next
        return head
    }

    newHead := reverseFirstN(head.Next, n-1)
    head.Next.Next = head
    head.Next = successor
    return newHead
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
    fmt.Print("Original:    ")
    printList(head)

    head = reverseFirstN(head, 3)
    fmt.Print("Reverse(3):  ")
    printList(head)
    // 3 → 2 → 1 → 4 → 5 → nil
}
```

---

## Example 8: Step-by-Step Reversal Trace

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseWithTrace(head *Node) *Node {
    var prev *Node
    curr := head
    step := 0

    for curr != nil {
        next := curr.Next
        curr.Next = prev

        // Print state
        fmt.Printf("Step %d: prev=%v curr=%v next=%v | reversed so far: ",
            step, nodeVal(prev), nodeVal(curr), nodeVal(next))
        printFromNode(curr)

        prev = curr
        curr = next
        step++
    }
    return prev
}

func nodeVal(n *Node) string {
    if n == nil {
        return "nil"
    }
    return fmt.Sprintf("%d", n.Val)
}

func printFromNode(n *Node) {
    for curr := n; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
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
    fmt.Print("Original: ")
    printList(head)
    fmt.Println()

    result := reverseWithTrace(head)
    fmt.Print("\nResult: ")
    printList(result)
}
```

---

## Example 9: Reverse Using Stack

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseUsingStack(head *Node) *Node {
    if head == nil {
        return nil
    }

    // Push all nodes to stack
    var stack []*Node
    for curr := head; curr != nil; curr = curr.Next {
        stack = append(stack, curr)
    }

    // Pop and rebuild
    newHead := stack[len(stack)-1]
    curr := newHead
    for i := len(stack) - 2; i >= 0; i-- {
        curr.Next = stack[i]
        curr = curr.Next
    }
    curr.Next = nil

    return newHead
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

    head = reverseUsingStack(head)
    fmt.Print("After:  ")
    printList(head)
    // Note: O(n) space — prefer iterative O(1) space in interviews
}
```

---

## Example 10: Reverse Doubly Linked List

```go
package main

import "fmt"

type DNode struct {
    Val  int
    Prev *DNode
    Next *DNode
}

func reverseDLL(head *DNode) *DNode {
    var curr *DNode = head
    var newHead *DNode

    for curr != nil {
        // Swap prev and next
        curr.Prev, curr.Next = curr.Next, curr.Prev
        newHead = curr
        curr = curr.Prev // was curr.Next before swap
    }
    return newHead
}

func buildDLL(arr []int) *DNode {
    if len(arr) == 0 {
        return nil
    }
    head := &DNode{Val: arr[0]}
    curr := head
    for i := 1; i < len(arr); i++ {
        n := &DNode{Val: arr[i], Prev: curr}
        curr.Next = n
        curr = n
    }
    return head
}

func printDLL(head *DNode) {
    for curr := head; curr != nil; curr = curr.Next {
        fmt.Printf("%d ⇄ ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    head := buildDLL([]int{1, 2, 3, 4, 5})
    fmt.Print("Before: ")
    printDLL(head)

    head = reverseDLL(head)
    fmt.Print("After:  ")
    printDLL(head)

    // Verify backward
    fmt.Print("Backward: ")
    curr := head
    for curr.Next != nil {
        curr = curr.Next
    }
    for curr != nil {
        fmt.Printf("%d ⇄ ", curr.Val)
        curr = curr.Prev
    }
    fmt.Println("nil")
}
```

---

## Reversal Algorithm Comparison

| Algorithm | Time | Space | In-Place |
|-----------|------|-------|----------|
| Iterative | O(n) | O(1) | Yes |
| Recursive | O(n) | O(n) stack | No |
| Stack-based | O(n) | O(n) | No |
| K-group | O(n) | O(1) iterative | Yes |
| Sublist (m to n) | O(n) | O(1) | Yes |

## Key Takeaways

1. **Iterative is preferred** — O(1) space, no stack overflow risk
2. **Three pointers**: `prev`, `curr`, `next` — the core pattern
3. **Recursive**: elegant but O(n) call stack space
4. **K-group reversal** is a common interview question (LeetCode 25)
5. **Sublist reversal**: use dummy node + count to position
6. **DLL reversal**: just swap Prev and Next for each node
7. **Always handle edge cases**: empty list, single node, full reversal

> **Next up:** Cycle Detection →
