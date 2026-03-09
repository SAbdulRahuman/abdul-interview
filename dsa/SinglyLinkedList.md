# Phase 5: Linked Lists — Singly Linked List

## What is a Singly Linked List?

A **singly linked list** is a linear data structure where each element (node) contains:
1. **Data** — the value
2. **Next pointer** — reference to the next node

Unlike arrays, elements are **not contiguous in memory** — each node is allocated independently and linked via pointers.

```
[10] → [20] → [30] → [40] → nil
```

| Operation | Time |
|-----------|------|
| Access by index | O(n) |
| Insert at head | O(1) |
| Insert at tail | O(n)* |
| Insert at position | O(n) |
| Delete head | O(1) |
| Delete by value | O(n) |
| Search | O(n) |

*O(1) if you maintain a tail pointer.

---

## Example 1: Basic Node and List Creation

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
    // Manual creation
    head := &Node{Val: 10}
    head.Next = &Node{Val: 20}
    head.Next.Next = &Node{Val: 30}
    head.Next.Next.Next = &Node{Val: 40}

    printList(head) // 10 → 20 → 30 → 40 → nil
}
```

---

## Example 2: Insert Operations (Head, Tail, Position)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type LinkedList struct {
    Head *Node
    Size int
}

func (ll *LinkedList) InsertHead(val int) {
    ll.Head = &Node{Val: val, Next: ll.Head}
    ll.Size++
}

func (ll *LinkedList) InsertTail(val int) {
    newNode := &Node{Val: val}
    if ll.Head == nil {
        ll.Head = newNode
        ll.Size++
        return
    }
    curr := ll.Head
    for curr.Next != nil {
        curr = curr.Next
    }
    curr.Next = newNode
    ll.Size++
}

func (ll *LinkedList) InsertAt(pos, val int) bool {
    if pos < 0 || pos > ll.Size {
        return false
    }
    if pos == 0 {
        ll.InsertHead(val)
        return true
    }
    curr := ll.Head
    for i := 0; i < pos-1; i++ {
        curr = curr.Next
    }
    curr.Next = &Node{Val: val, Next: curr.Next}
    ll.Size++
    return true
}

func (ll *LinkedList) Print() {
    for curr := ll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    ll := &LinkedList{}

    ll.InsertHead(30)
    ll.InsertHead(20)
    ll.InsertHead(10)
    ll.Print() // 10 → 20 → 30 → nil

    ll.InsertTail(40)
    ll.Print() // 10 → 20 → 30 → 40 → nil

    ll.InsertAt(2, 25)
    ll.Print() // 10 → 20 → 25 → 30 → 40 → nil

    fmt.Println("Size:", ll.Size)
}
```

---

## Example 3: Delete Operations

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type LinkedList struct {
    Head *Node
    Size int
}

func (ll *LinkedList) DeleteHead() (int, bool) {
    if ll.Head == nil {
        return 0, false
    }
    val := ll.Head.Val
    ll.Head = ll.Head.Next
    ll.Size--
    return val, true
}

func (ll *LinkedList) DeleteVal(val int) bool {
    if ll.Head == nil {
        return false
    }
    if ll.Head.Val == val {
        ll.Head = ll.Head.Next
        ll.Size--
        return true
    }
    curr := ll.Head
    for curr.Next != nil {
        if curr.Next.Val == val {
            curr.Next = curr.Next.Next
            ll.Size--
            return true
        }
        curr = curr.Next
    }
    return false
}

func (ll *LinkedList) DeleteAt(pos int) (int, bool) {
    if pos < 0 || pos >= ll.Size {
        return 0, false
    }
    if pos == 0 {
        return ll.DeleteHead()
    }
    curr := ll.Head
    for i := 0; i < pos-1; i++ {
        curr = curr.Next
    }
    val := curr.Next.Val
    curr.Next = curr.Next.Next
    ll.Size--
    return val, true
}

func (ll *LinkedList) Print() {
    for curr := ll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func fromSlice(arr []int) *LinkedList {
    ll := &LinkedList{}
    for i := len(arr) - 1; i >= 0; i-- {
        ll.Head = &Node{Val: arr[i], Next: ll.Head}
        ll.Size++
    }
    return ll
}

func main() {
    ll := fromSlice([]int{10, 20, 30, 40, 50})
    ll.Print()

    ll.DeleteHead()
    fmt.Print("After delete head: ")
    ll.Print()

    ll.DeleteVal(30)
    fmt.Print("After delete 30:   ")
    ll.Print()

    ll.DeleteAt(1)
    fmt.Print("After delete @1:   ")
    ll.Print()

    fmt.Println("Size:", ll.Size)
}
```

---

## Example 4: Search and Get by Index

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func search(head *Node, target int) int {
    idx := 0
    for curr := head; curr != nil; curr = curr.Next {
        if curr.Val == target {
            return idx
        }
        idx++
    }
    return -1
}

func getAt(head *Node, index int) (int, bool) {
    curr := head
    for i := 0; i < index; i++ {
        if curr == nil {
            return 0, false
        }
        curr = curr.Next
    }
    if curr == nil {
        return 0, false
    }
    return curr.Val, true
}

func length(head *Node) int {
    n := 0
    for curr := head; curr != nil; curr = curr.Next {
        n++
    }
    return n
}

func main() {
    head := fromSlice([]int{10, 20, 30, 40, 50})

    fmt.Println("Search 30:", search(head, 30))   // 2
    fmt.Println("Search 99:", search(head, 99))   // -1

    val, ok := getAt(head, 3)
    fmt.Printf("Get index 3: %d, found=%v\n", val, ok)

    fmt.Println("Length:", length(head))
}
```

---

## Example 5: Reverse a Singly Linked List (Iterative)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverse(head *Node) *Node {
    var prev *Node
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
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

func main() {
    head := fromSlice([]int{1, 2, 3, 4, 5})
    fmt.Print("Original: ")
    printList(head)

    head = reverse(head)
    fmt.Print("Reversed: ")
    printList(head)
}
```

---

## Example 6: Reverse a Singly Linked List (Recursive)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reverseRecursive(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
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

func main() {
    head := fromSlice([]int{1, 2, 3, 4, 5})
    fmt.Print("Original:  ")
    printList(head)

    head = reverseRecursive(head)
    fmt.Print("Reversed:  ")
    printList(head)
}
```

---

## Example 7: List to Slice and Slice to List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func fromSlice(arr []int) *Node {
    if len(arr) == 0 {
        return nil
    }
    head := &Node{Val: arr[0]}
    curr := head
    for i := 1; i < len(arr); i++ {
        curr.Next = &Node{Val: arr[i]}
        curr = curr.Next
    }
    return head
}

func toSlice(head *Node) []int {
    var result []int
    for curr := head; curr != nil; curr = curr.Next {
        result = append(result, curr.Val)
    }
    return result
}

func main() {
    // Slice → List → Slice round-trip
    original := []int{5, 10, 15, 20, 25}
    head := fromSlice(original)
    back := toSlice(head)
    fmt.Println("Original:", original)
    fmt.Println("Back:    ", back)
}
```

---

## Example 8: Remove Duplicates from Sorted List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func removeDuplicates(head *Node) *Node {
    curr := head
    for curr != nil && curr.Next != nil {
        if curr.Val == curr.Next.Val {
            curr.Next = curr.Next.Next
        } else {
            curr = curr.Next
        }
    }
    return head
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
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

func main() {
    head := fromSlice([]int{1, 1, 2, 3, 3, 3, 4, 5, 5})
    fmt.Print("Before: ")
    printList(head)

    removeDuplicates(head)
    fmt.Print("After:  ")
    printList(head)
}
```

---

## Example 9: Find Middle Node

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func findMiddle(head *Node) *Node {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    return slow
}

func printList(head *Node) {
    for curr := head; curr != nil; curr = curr.Next {
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

func main() {
    odd := fromSlice([]int{1, 2, 3, 4, 5})
    fmt.Print("Odd list:  ")
    printList(odd)
    fmt.Printf("Middle:    %d\n\n", findMiddle(odd).Val)

    even := fromSlice([]int{1, 2, 3, 4, 5, 6})
    fmt.Print("Even list: ")
    printList(even)
    fmt.Printf("Middle:    %d\n", findMiddle(even).Val)
}
```

---

## Example 10: Check if Palindrome

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func isPalindrome(head *Node) bool {
    if head == nil || head.Next == nil {
        return true
    }

    // Find middle
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // Reverse second half
    secondHalf := reverse(slow.Next)
    slow.Next = nil // split

    // Compare
    p1, p2 := head, secondHalf
    result := true
    for p1 != nil && p2 != nil {
        if p1.Val != p2.Val {
            result = false
            break
        }
        p1 = p1.Next
        p2 = p2.Next
    }

    // Restore (optional)
    slow.Next = reverse(secondHalf)

    return result
}

func reverse(head *Node) *Node {
    var prev *Node
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
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

func main() {
    tests := [][]int{
        {1, 2, 3, 2, 1},
        {1, 2, 2, 1},
        {1, 2, 3},
    }
    for _, t := range tests {
        head := fromSlice(t)
        fmt.Printf("%v → palindrome=%v\n", t, isPalindrome(head))
    }
}
```

---

## Example 11: Singly Linked List with Tail Pointer

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type TailList struct {
    Head *Node
    Tail *Node
    Size int
}

func NewTailList() *TailList {
    return &TailList{}
}

func (ll *TailList) PushFront(val int) {
    n := &Node{Val: val, Next: ll.Head}
    ll.Head = n
    if ll.Tail == nil {
        ll.Tail = n
    }
    ll.Size++
}

func (ll *TailList) PushBack(val int) { // O(1) with tail pointer!
    n := &Node{Val: val}
    if ll.Tail == nil {
        ll.Head = n
        ll.Tail = n
    } else {
        ll.Tail.Next = n
        ll.Tail = n
    }
    ll.Size++
}

func (ll *TailList) PopFront() (int, bool) {
    if ll.Head == nil {
        return 0, false
    }
    val := ll.Head.Val
    ll.Head = ll.Head.Next
    if ll.Head == nil {
        ll.Tail = nil
    }
    ll.Size--
    return val, true
}

func (ll *TailList) Print() {
    for curr := ll.Head; curr != nil; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    ll := NewTailList()
    ll.PushBack(1) // O(1)
    ll.PushBack(2)
    ll.PushBack(3)
    ll.PushFront(0)
    ll.Print() // 0 → 1 → 2 → 3 → nil

    val, _ := ll.PopFront()
    fmt.Println("Popped:", val)
    ll.Print()
    fmt.Println("Size:", ll.Size)
}
```

---

## Singly Linked List vs Array

| Feature | Singly Linked List | Array |
|---------|-------------------|-------|
| Access by index | O(n) | O(1) |
| Insert at head | O(1) | O(n) |
| Insert at tail | O(1)* | O(1) amortized |
| Insert at middle | O(n) search + O(1) insert | O(n) shift |
| Memory | Extra pointer per node | Contiguous |
| Cache | Poor locality | Excellent locality |

*With tail pointer.

## Key Takeaways

1. **Node = {Val, Next}** — simplest recursive data structure
2. **O(1) insert/delete at head** — no shifting needed
3. **O(n) access** — must traverse from head
4. **Tail pointer** makes append O(1)
5. **Reverse** is a fundamental operation (iterative or recursive)
6. **Slow/fast pointers** find middle in one pass
7. **Go has no built-in linked list type** in standard interviews — build from scratch

> **Next up:** Doubly Linked List →
