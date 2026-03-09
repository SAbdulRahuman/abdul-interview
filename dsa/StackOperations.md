# Phase 6: Stack — Stack Operations

## Overview

A **stack** is a Last-In-First-Out (LIFO) data structure supporting:
- **Push** — add to top O(1)
- **Pop** — remove from top O(1)
- **Peek/Top** — view top element O(1)
- **IsEmpty** — check if empty O(1)

Go has no built-in stack — use a **slice** (`append` / slice truncation).

---

## Example 1: Basic Stack with Slice

```go
package main

import "fmt"

type Stack struct {
    data []int
}

func (s *Stack) Push(val int) {
    s.data = append(s.data, val)
}

func (s *Stack) Pop() (int, bool) {
    if len(s.data) == 0 {
        return 0, false
    }
    val := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    return val, true
}

func (s *Stack) Peek() (int, bool) {
    if len(s.data) == 0 {
        return 0, false
    }
    return s.data[len(s.data)-1], true
}

func (s *Stack) IsEmpty() bool {
    return len(s.data) == 0
}

func (s *Stack) Size() int {
    return len(s.data)
}

func main() {
    s := &Stack{}
    s.Push(10)
    s.Push(20)
    s.Push(30)

    fmt.Println("Size:", s.Size())       // 3
    fmt.Println("Peek:", s.data[len(s.data)-1]) // 30

    val, _ := s.Pop()
    fmt.Println("Pop:", val) // 30

    val, _ = s.Pop()
    fmt.Println("Pop:", val) // 20

    fmt.Println("Empty:", s.IsEmpty()) // false

    s.Pop()
    fmt.Println("Empty:", s.IsEmpty()) // true
}
```

---

## Example 2: Generic Stack (Go 1.18+)

```go
package main

import "fmt"

type Stack[T any] struct {
    data []T
}

func (s *Stack[T]) Push(val T) {
    s.data = append(s.data, val)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.data) == 0 {
        return zero, false
    }
    val := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    return val, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.data) == 0 {
        return zero, false
    }
    return s.data[len(s.data)-1], true
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.data) == 0
}

func main() {
    // Integer stack
    si := &Stack[int]{}
    si.Push(1)
    si.Push(2)
    v, _ := si.Pop()
    fmt.Println("Int:", v)

    // String stack
    ss := &Stack[string]{}
    ss.Push("hello")
    ss.Push("world")
    sv, _ := ss.Pop()
    fmt.Println("String:", sv)

    // Float stack
    sf := &Stack[float64]{}
    sf.Push(3.14)
    sf.Push(2.71)
    fv, _ := sf.Pop()
    fmt.Printf("Float: %.2f\n", fv)
}
```

---

## Example 3: Stack Using Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type LinkedStack struct {
    top  *Node
    size int
}

func (s *LinkedStack) Push(val int) {
    s.top = &Node{Val: val, Next: s.top}
    s.size++
}

func (s *LinkedStack) Pop() (int, bool) {
    if s.top == nil {
        return 0, false
    }
    val := s.top.Val
    s.top = s.top.Next
    s.size--
    return val, true
}

func (s *LinkedStack) Peek() (int, bool) {
    if s.top == nil {
        return 0, false
    }
    return s.top.Val, true
}

func (s *LinkedStack) IsEmpty() bool {
    return s.top == nil
}

func main() {
    s := &LinkedStack{}
    for i := 1; i <= 5; i++ {
        s.Push(i * 10)
    }

    for !s.IsEmpty() {
        v, _ := s.Pop()
        fmt.Printf("%d ", v) // 50 40 30 20 10
    }
    fmt.Println()
}
```

---

## Example 4: Min Stack (LeetCode 155)

```go
package main

import "fmt"

type MinStack struct {
    data []int
    mins []int // parallel stack tracking minimums
}

func (s *MinStack) Push(val int) {
    s.data = append(s.data, val)
    if len(s.mins) == 0 || val <= s.mins[len(s.mins)-1] {
        s.mins = append(s.mins, val)
    } else {
        // Repeat current min
        s.mins = append(s.mins, s.mins[len(s.mins)-1])
    }
}

func (s *MinStack) Pop() {
    s.data = s.data[:len(s.data)-1]
    s.mins = s.mins[:len(s.mins)-1]
}

func (s *MinStack) Top() int {
    return s.data[len(s.data)-1]
}

func (s *MinStack) GetMin() int {
    return s.mins[len(s.mins)-1]
}

func main() {
    s := &MinStack{}
    s.Push(-2)
    s.Push(0)
    s.Push(-3)
    fmt.Println("Min:", s.GetMin()) // -3
    s.Pop()
    fmt.Println("Top:", s.Top())    // 0
    fmt.Println("Min:", s.GetMin()) // -2

    s.Push(-1)
    fmt.Println("Min:", s.GetMin()) // -2
}
```

---

## Example 5: Max Stack (Track Maximum)

```go
package main

import "fmt"

type MaxStack struct {
    data []int
    maxs []int
}

func (s *MaxStack) Push(val int) {
    s.data = append(s.data, val)
    if len(s.maxs) == 0 || val >= s.maxs[len(s.maxs)-1] {
        s.maxs = append(s.maxs, val)
    } else {
        s.maxs = append(s.maxs, s.maxs[len(s.maxs)-1])
    }
}

func (s *MaxStack) Pop() int {
    val := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    s.maxs = s.maxs[:len(s.maxs)-1]
    return val
}

func (s *MaxStack) Top() int {
    return s.data[len(s.data)-1]
}

func (s *MaxStack) GetMax() int {
    return s.maxs[len(s.maxs)-1]
}

func main() {
    s := &MaxStack{}
    ops := []int{5, 1, 5, 3, 7, 2}
    for _, v := range ops {
        s.Push(v)
        fmt.Printf("Push %d → max=%d\n", v, s.GetMax())
    }

    fmt.Println("\nPopping:")
    for len(s.data) > 0 {
        fmt.Printf("Pop %d → ", s.Pop())
        if len(s.data) > 0 {
            fmt.Printf("max=%d\n", s.GetMax())
        } else {
            fmt.Println("empty")
        }
    }
}
```

---

## Example 6: Two Stacks in One Array

```go
package main

import (
    "errors"
    "fmt"
)

type TwoStacks struct {
    arr  []int
    top1 int
    top2 int
}

func NewTwoStacks(capacity int) *TwoStacks {
    return &TwoStacks{
        arr:  make([]int, capacity),
        top1: -1,
        top2: capacity,
    }
}

func (ts *TwoStacks) Push1(val int) error {
    if ts.top1+1 >= ts.top2 {
        return errors.New("overflow")
    }
    ts.top1++
    ts.arr[ts.top1] = val
    return nil
}

func (ts *TwoStacks) Push2(val int) error {
    if ts.top2-1 <= ts.top1 {
        return errors.New("overflow")
    }
    ts.top2--
    ts.arr[ts.top2] = val
    return nil
}

func (ts *TwoStacks) Pop1() (int, error) {
    if ts.top1 < 0 {
        return 0, errors.New("stack 1 empty")
    }
    val := ts.arr[ts.top1]
    ts.top1--
    return val, nil
}

func (ts *TwoStacks) Pop2() (int, error) {
    if ts.top2 >= len(ts.arr) {
        return 0, errors.New("stack 2 empty")
    }
    val := ts.arr[ts.top2]
    ts.top2++
    return val, nil
}

func main() {
    ts := NewTwoStacks(10)
    ts.Push1(1)
    ts.Push1(2)
    ts.Push1(3)

    ts.Push2(100)
    ts.Push2(200)

    v, _ := ts.Pop1()
    fmt.Println("Stack1 pop:", v) // 3

    v, _ = ts.Pop2()
    fmt.Println("Stack2 pop:", v) // 200

    // Fill up
    for i := 0; i < 5; i++ {
        ts.Push1(i + 10)
    }
    err := ts.Push1(99)
    fmt.Println("Overflow:", err) // overflow
}
```

---

## Example 7: Implement Queue Using Two Stacks (LeetCode 232)

```go
package main

import "fmt"

type MyQueue struct {
    pushStack []int
    popStack  []int
}

func (q *MyQueue) Push(x int) {
    q.pushStack = append(q.pushStack, x)
}

func (q *MyQueue) transfer() {
    if len(q.popStack) == 0 {
        for len(q.pushStack) > 0 {
            n := len(q.pushStack)
            q.popStack = append(q.popStack, q.pushStack[n-1])
            q.pushStack = q.pushStack[:n-1]
        }
    }
}

func (q *MyQueue) Pop() int {
    q.transfer()
    val := q.popStack[len(q.popStack)-1]
    q.popStack = q.popStack[:len(q.popStack)-1]
    return val
}

func (q *MyQueue) Peek() int {
    q.transfer()
    return q.popStack[len(q.popStack)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.pushStack) == 0 && len(q.popStack) == 0
}

func main() {
    q := &MyQueue{}
    q.Push(1)
    q.Push(2)
    q.Push(3)

    fmt.Println("Peek:", q.Peek()) // 1
    fmt.Println("Pop:", q.Pop())   // 1
    fmt.Println("Pop:", q.Pop())   // 2

    q.Push(4)
    fmt.Println("Pop:", q.Pop()) // 3
    fmt.Println("Pop:", q.Pop()) // 4
    fmt.Println("Empty:", q.Empty()) // true
}
```

---

## Example 8: Reverse a Stack (Using Recursion)

```go
package main

import "fmt"

type Stack struct {
    data []int
}

func (s *Stack) Push(v int) { s.data = append(s.data, v) }
func (s *Stack) Pop() int {
    n := len(s.data)
    v := s.data[n-1]
    s.data = s.data[:n-1]
    return v
}
func (s *Stack) IsEmpty() bool { return len(s.data) == 0 }

// Insert element at the bottom of the stack
func insertAtBottom(s *Stack, val int) {
    if s.IsEmpty() {
        s.Push(val)
        return
    }
    top := s.Pop()
    insertAtBottom(s, val)
    s.Push(top)
}

// Reverse entire stack using recursion
func reverseStack(s *Stack) {
    if s.IsEmpty() {
        return
    }
    top := s.Pop()
    reverseStack(s)
    insertAtBottom(s, top)
}

func main() {
    s := &Stack{}
    for i := 1; i <= 5; i++ {
        s.Push(i)
    }
    fmt.Println("Before:", s.data) // [1, 2, 3, 4, 5]

    reverseStack(s)
    fmt.Println("After:", s.data) // [5, 4, 3, 2, 1]
}
```

---

## Example 9: Sort a Stack (Using Recursion)

```go
package main

import "fmt"

type Stack struct {
    data []int
}

func (s *Stack) Push(v int) { s.data = append(s.data, v) }
func (s *Stack) Pop() int {
    n := len(s.data)
    v := s.data[n-1]
    s.data = s.data[:n-1]
    return v
}
func (s *Stack) Peek() int     { return s.data[len(s.data)-1] }
func (s *Stack) IsEmpty() bool { return len(s.data) == 0 }

func sortedInsert(s *Stack, val int) {
    if s.IsEmpty() || val >= s.Peek() {
        s.Push(val)
        return
    }
    top := s.Pop()
    sortedInsert(s, val)
    s.Push(top)
}

func sortStack(s *Stack) {
    if s.IsEmpty() {
        return
    }
    top := s.Pop()
    sortStack(s)
    sortedInsert(s, top)
}

func main() {
    s := &Stack{}
    nums := []int{34, 3, 31, 98, 92, 23}
    for _, n := range nums {
        s.Push(n)
    }
    fmt.Println("Before:", s.data)

    sortStack(s)
    fmt.Println("After:", s.data) // [3, 23, 31, 34, 92, 98]
}
```

---

## Example 10: Stack with Middle Operation

```go
package main

import "fmt"

// Stack that supports findMiddle and deleteMiddle in O(1)
// using a doubly linked list with a middle pointer
type DLLNode struct {
    Val        int
    Prev, Next *DLLNode
}

type MiddleStack struct {
    head   *DLLNode
    mid    *DLLNode
    count  int
}

func (ms *MiddleStack) Push(val int) {
    node := &DLLNode{Val: val}
    if ms.head == nil {
        ms.head = node
        ms.mid = node
    } else {
        node.Next = ms.head
        ms.head.Prev = node
        ms.head = node
    }
    ms.count++

    // Adjust mid pointer
    if ms.count > 1 && ms.count%2 != 0 {
        ms.mid = ms.mid.Prev
    }
}

func (ms *MiddleStack) Pop() (int, bool) {
    if ms.head == nil {
        return 0, false
    }
    val := ms.head.Val
    ms.head = ms.head.Next
    if ms.head != nil {
        ms.head.Prev = nil
    }
    ms.count--

    if ms.count%2 == 0 && ms.mid != nil {
        ms.mid = ms.mid.Next
    }

    if ms.count == 0 {
        ms.mid = nil
    }
    return val, true
}

func (ms *MiddleStack) FindMiddle() (int, bool) {
    if ms.mid == nil {
        return 0, false
    }
    return ms.mid.Val, true
}

func main() {
    ms := &MiddleStack{}

    for i := 1; i <= 7; i++ {
        ms.Push(i)
        mid, _ := ms.FindMiddle()
        fmt.Printf("Push %d → middle=%d (size=%d)\n", i, mid, ms.count)
    }

    fmt.Println("\nPopping:")
    for ms.count > 0 {
        val, _ := ms.Pop()
        if ms.count > 0 {
            mid, _ := ms.FindMiddle()
            fmt.Printf("Pop %d → middle=%d (size=%d)\n", val, mid, ms.count)
        } else {
            fmt.Printf("Pop %d → empty\n", val)
        }
    }
}
```

---

## Example 11: Undo/Redo with Two Stacks

```go
package main

import "fmt"

type Action struct {
    Type string
    Data string
}

type Editor struct {
    content  string
    undoStack []Action
    redoStack []Action
}

func (e *Editor) Insert(text string) {
    e.undoStack = append(e.undoStack, Action{Type: "insert", Data: text})
    e.content += text
    e.redoStack = nil // clear redo on new action
    fmt.Printf("Insert '%s' → \"%s\"\n", text, e.content)
}

func (e *Editor) Undo() {
    if len(e.undoStack) == 0 {
        fmt.Println("Nothing to undo")
        return
    }
    action := e.undoStack[len(e.undoStack)-1]
    e.undoStack = e.undoStack[:len(e.undoStack)-1]

    if action.Type == "insert" {
        e.content = e.content[:len(e.content)-len(action.Data)]
    }
    e.redoStack = append(e.redoStack, action)
    fmt.Printf("Undo → \"%s\"\n", e.content)
}

func (e *Editor) Redo() {
    if len(e.redoStack) == 0 {
        fmt.Println("Nothing to redo")
        return
    }
    action := e.redoStack[len(e.redoStack)-1]
    e.redoStack = e.redoStack[:len(e.redoStack)-1]

    if action.Type == "insert" {
        e.content += action.Data
    }
    e.undoStack = append(e.undoStack, action)
    fmt.Printf("Redo → \"%s\"\n", e.content)
}

func main() {
    e := &Editor{}
    e.Insert("Hello")
    e.Insert(" World")
    e.Insert("!")

    e.Undo()
    e.Undo()
    e.Redo()

    e.Insert(" Go")
    e.Redo() // nothing — cleared after new insert
}
```

---

## Stack Operations Summary

| Operation | Slice-based | Linked List | Notes |
|-----------|-------------|-------------|-------|
| Push | O(1) amortized | O(1) | Slice may resize |
| Pop | O(1) | O(1) | Slice truncation |
| Peek | O(1) | O(1) | |
| IsEmpty | O(1) | O(1) | |
| Space | Contiguous | Scattered | Slice has better cache locality |

## Key Takeaways

1. Go uses **slices** as stacks: `append()` to push, `s[:len(s)-1]` to pop
2. **Min/Max Stack**: maintain a parallel stack tracking min/max at each level
3. **Two stacks → queue**: push stack + pop stack with lazy transfer
4. **Reverse a stack** recursively: O(n²) — pop all, insert at bottom
5. **Sort a stack** recursively: O(n²) — pop all, insert in sorted order
6. Stacks are fundamental for **DFS, backtracking, undo/redo, parsing**

> **Next up:** Monotonic Stack →
