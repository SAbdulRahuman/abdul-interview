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

**Textual Figure:**
```
Iterative reversal of [1, 2, 3, 4, 5]:

Initial: prev=nil, curr=1
  nil   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  prev  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→ nil
        └───┘   └───┘   └───┘   └───┘   └───┘
        curr

Step 1: save next=2, curr.Next=prev(nil), prev=1, curr=2
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→nil  │ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→ nil
  └───┘        └───┘   └───┘   └───┘   └───┘
  prev         curr

Step 2: save next=3, curr.Next=prev(1), prev=2, curr=3
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │←──│ 2 │   │ 3 │──→│ 4 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
          prev    curr

Steps 3-5: continue reversing...

Final: curr=nil, prev=5
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 5 │──→│ 4 │──→│ 3 │──→│ 2 │──→│ 1 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
  prev (returned as new head)
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

**Textual Figure:**
```
Recursive reversal of [1, 2, 3]:

Call stack (dive down):
  reverseRecursive(1→2→3)
    └─→ reverseRecursive(2→3)
          └─→ reverseRecursive(3) → base case, return 3

Unwinding (build reversed links):

  Return from reverseRecursive(3): newHead = 3
    ┌───┐
    │ 3 │──→ nil     (newHead)
    └───┘

  Back in reverseRecursive(2→3):
    head=2, head.Next=3, head.Next.Next = head → 3.Next = 2
    head.Next = nil
    ┌───┐   ┌───┐
    │ 3 │──→│ 2 │──→ nil
    └───┘   └───┘

  Back in reverseRecursive(1→2→3):
    head=1, head.Next=2, head.Next.Next = head → 2.Next = 1
    head.Next = nil
    ┌───┐   ┌───┐   ┌───┐
    │ 3 │──→│ 2 │──→│ 1 │──→ nil
    └───┘   └───┘   └───┘
    newHead (returned all the way up)
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

**Textual Figure:**
```
Reverse positions [2, 4] in [1, 2, 3, 4, 5]:

Initial: prev positioned before node 2
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ D │──→│ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
          prev    curr

i=0: move nodeToMove(3) to prev.Next:
  D──→[1]──→[3]──→[2]──→[4]──→[5]──→nil
        prev              curr

i=1: move nodeToMove(4) to prev.Next:
  D──→[1]──→[4]──→[3]──→[2]──→[5]──→nil
        prev                    curr

Result:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 4 │──→│ 3 │──→│ 2 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
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

**Textual Figure:**
```
Reverse in groups of K=3: [1, 2, 3, 4, 5, 6, 7, 8]

Original:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 5 │─→│ 6 │─→│ 7 │─→│ 8 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
  ├── group 1 ──┤├── group 2 ──┤├─ remaining ─┤

Group 1: reverse [1,2,3] → [3,2,1]
  ┌───┐  ┌───┐  ┌───┐
  │ 3 │─→│ 2 │─→│ 1 │─→ (recurse on rest)
  └───┘  └───┘  └───┘
                  tail  ──→ head.Next = reverseKGroup(4→5→6→7→8)

Group 2: reverse [4,5,6] → [6,5,4]

[7,8]: only 2 nodes < k=3 → leave as is

Final:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 3 │─→│ 2 │─→│ 1 │─→│ 6 │─→│ 5 │─→│ 4 │─→│ 7 │─→│ 8 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
  ├─ reversed ─┤├─ reversed ─┤├─ as-is ─┤
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

**Textual Figure:**
```
Iterative K-group reversal of [1,2,3,4,5,6,7,8] with K=2:

Initial:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 5 │─→│ 6 │─→│ 7 │─→│ 8 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘

Group 1: find kth=[2], groupNext=[3]
  Reverse [1,2]: prev=groupNext(3), curr=1
  → 2─→ 1─→ 3...
  Connect: groupPrev(D).Next=kth(2), groupPrev=tmp(1)

Group 2: reverse [3,4], Group 3: reverse [5,6], Group 4: reverse [7,8]

K=2 Result:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 2 │─→│ 1 │─→│ 4 │─→│ 3 │─→│ 6 │─→│ 5 │─→│ 8 │─→│ 7 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
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

**Textual Figure:**
```
Reverse alternating K=3 groups in [1..10]:

Original:
  [1]─→[2]─→[3]─→[4]─→[5]─→[6]─→[7]─→[8]─→[9]─→[10]─→nil
  ├─ reverse ─┤├── skip ──┤├─ reverse ─┤├ skip┤

Group 1 (reverse): [1,2,3] → [3,2,1]
  ┌───┐  ┌───┐  ┌───┐
  │ 3 │─→│ 2 │─→│ 1 │─→ [4,5,6]...
  └───┘  └───┘  └───┘
                  tail connects to curr(4)

Group 2 (skip): [4,5,6] kept as-is, skip pointer walks to [6]

Group 3 (reverse): [7,8,9] → [9,8,7]

Group 4 (skip): [10] kept as-is

Final:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌────┐
  │ 3 │─→│ 2 │─→│ 1 │─→│ 4 │─→│ 5 │─→│ 6 │─→│ 9 │─→│ 8 │─→│ 7 │─→│ 10 │─→nil
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └────┘
  ├─ reversed ─┤├── kept ──┤├─ reversed ─┤├─kept─┤
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

**Textual Figure:**
```
Reverse first N=3 nodes of [1, 2, 3, 4, 5]:

Initial:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
  ├── reverse these ──┤ successor=[4]

Recursive unwinding:
  n=3: base case → successor = head.Next = [4]
  n=2: 3.Next.Next=3? No, 3.Next=successor(4), so 2.Next.Next=2
       → 3─→2, 2.Next=successor(4)
  n=1: 2.Next.Next=1 → 2─→1, 1.Next=successor(4)

Result:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 3 │──→│ 2 │──→│ 1 │──→│ 4 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
  ├─── reversed ───┤├── unchanged ──┤
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

**Textual Figure:**
```
Step-by-step reversal of [1, 2, 3, 4]:

Step 0: prev=nil, curr=1, next=2
  nil ←── [1]    [2]──→[3]──→[4]──→nil
         prev   curr

Step 1: prev=1, curr=2, next=3
  nil ←── [1] ←── [2]    [3]──→[4]──→nil
                  prev   curr

Step 2: prev=2, curr=3, next=4
  nil ←── [1] ←── [2] ←── [3]    [4]──→nil
                         prev   curr

Step 3: prev=3, curr=4, next=nil
  nil ←── [1] ←── [2] ←── [3] ←── [4]    nil
                                prev   curr

Final (prev returned):
  ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 4 │──→│ 3 │──→│ 2 │──→│ 1 │──→ nil
  └───┘   └───┘   └───┘   └───┘
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

**Textual Figure:**
```
Reverse [1, 2, 3, 4, 5] using a stack:

Phase 1 — Push all nodes onto stack:
  List: [1]─→[2]─→[3]─→[4]─→[5]─→nil

  Stack (top → bottom):
  ┌─────┐
  │ [5] │ ← top
  ├─────┤
  │ [4] │
  ├─────┤
  │ [3] │
  ├─────┤
  │ [2] │
  ├─────┤
  │ [1] │ ← bottom
  └─────┘

Phase 2 — Pop and rebuild:
  Pop [5] → newHead = [5]
  Pop [4] → [5].Next = [4]
  Pop [3] → [4].Next = [3]
  Pop [2] → [3].Next = [2]
  Pop [1] → [2].Next = [1], [1].Next = nil

Result:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 5 │──→│ 4 │──→│ 3 │──→│ 2 │──→│ 1 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
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

**Textual Figure:**
```
Reverse doubly linked list [1, 2, 3, 4, 5]:

Before:
  nil ←── ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ nil
         │ 1 │ ←── │ 2 │ ←── │ 3 │ ←── │ 4 │ ←── │ 5 │
         └───┘     └───┘     └───┘     └───┘     └───┘
         head

For each node: swap Prev and Next pointers
  curr=[1]: Prev=nil, Next=[2] → swap → Prev=[2], Next=nil
  curr=[2]: Prev=[1], Next=[3] → swap → Prev=[3], Next=[1]
  curr=[3]: Prev=[2], Next=[4] → swap → Prev=[4], Next=[2]
  curr=[4]: Prev=[3], Next=[5] → swap → Prev=[5], Next=[3]
  curr=[5]: Prev=[4], Next=nil → swap → Prev=nil, Next=[4]

After (newHead = last visited node = [5]):
  nil ←── ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ ┌───┐ ──→ nil
         │ 5 │ ←── │ 4 │ ←── │ 3 │ ←── │ 2 │ ←── │ 1 │
         └───┘     └───┘     └───┘     └───┘     └───┘
         head
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
