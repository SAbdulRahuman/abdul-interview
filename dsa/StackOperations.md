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

**Textual Figure: Basic Stack Operations**

```
Step-by-step push/pop trace (stack grows upward):

  Push(10)     Push(20)     Push(30)      Pop()        Pop()        Pop()
  ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐
  │      │     │      │ top│  30  │     │      │     │      │     │      │
  │      │ top│  20  │     │  20  │ top│  20  │     │      │     │      │
  │  10  │←top │  10  │     │  10  │     │  10  │ top│  10  │     │      │
  └──────┘     └──────┘     └──────┘     └──────┘     └──────┘     └──────┘
   size=1       size=2       size=3       size=2       size=1       size=0

  After Push(30): Peek() → 30, Size() → 3
  Pop() returns: 30 → 20 → 10  (LIFO order)
  Final: IsEmpty() = true

  Underlying slice:
  ┌────┬────┬────┐         ┌────┬────┐         ┌────┐
  │ 10 │ 20 │ 30 │  ─pop→  │ 10 │ 20 │  ─pop→  │ 10 │  ─pop→  []
  └────┴────┴────┘         └────┴────┘         └────┘
   [0]  [1]  [2]            [0]  [1]            [0]
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

**Textual Figure: Generic Stack with Type Parameters**

```
Stack[int]              Stack[string]           Stack[float64]
┌──────────┐            ┌──────────────┐        ┌──────────┐
│  Push(1)  │            │ Push("hello") │        │ Push(3.14)│
│  Push(2)  │            │ Push("world") │        │ Push(2.71)│
└──────────┘            └──────────────┘        └──────────┘

  ┌─────┐                 ┌─────────┐            ┌──────┐
  │  2  │←top  Pop→2      │ "world" │←top Pop    │ 2.71 │←top  Pop→2.71
  ├─────┤                 ├─────────┤            ├──────┤
  │  1  │                 │ "hello" │            │ 3.14 │
  └─────┘                 └─────────┘            └──────┘

  After Pop:              After Pop:             After Pop:
  ┌─────┐                 ┌─────────┐            ┌──────┐
  │  1  │←top             │ "hello" │←top        │ 3.14 │←top
  └─────┘                 └─────────┘            └──────┘

  Same Stack[T any] struct — type parameter T determines element type.
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

**Textual Figure: Linked List Stack**

```
After Push(10), Push(20), Push(30), Push(40), Push(50):

  top
   │
   ▼
  ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
  │ 50 │───→│ 40 │───→│ 30 │───→│ 20 │───→│ 10 │───→ nil
  └────┘    └────┘    └────┘    └────┘    └────┘
   size=5

Pop sequence (each Pop moves top to top.Next):

  Pop()→50:  top→[40]→[30]→[20]→[10]→nil   size=4
  Pop()→40:  top→[30]→[20]→[10]→nil        size=3
  Pop()→30:  top→[20]→[10]→nil             size=2
  Pop()→20:  top→[10]→nil                  size=1
  Pop()→10:  top→nil                       size=0

  Output: 50 40 30 20 10  (LIFO order)

  Push O(1): new node points to old top
  Pop  O(1): move top to top.Next
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

**Textual Figure: Min Stack with Parallel Tracking**

```
Operation      data stack         mins stack         GetMin()
─────────      ──────────         ──────────         ────────
Push(-2)       ┌────┐             ┌────┐
               │ -2 │             │ -2 │             → -2
               └────┘             └────┘

Push(0)        ┌────┐             ┌────┐
               │  0 │←top         │ -2 │←top         → -2
               ├────┤             ├────┤
               │ -2 │             │ -2 │
               └────┘             └────┘

Push(-3)       ┌────┐             ┌────┐
               │ -3 │←top         │ -3 │←top         → -3  (new min!)
               ├────┤             ├────┤
               │  0 │             │ -2 │
               ├────┤             ├────┤
               │ -2 │             │ -2 │
               └────┘             └────┘

Pop()          ┌────┐             ┌────┐
               │  0 │←top         │ -2 │←top         → -2
               ├────┤             ├────┤
               │ -2 │             │ -2 │
               └────┘             └────┘
               Top() → 0

Push(-1)       ┌────┐             ┌────┐
               │ -1 │←top         │ -2 │←top         → -2  (-1 > -2)
               ├────┤             ├────┤
               │  0 │             │ -2 │
               ├────┤             ├────┤
               │ -2 │             │ -2 │
               └────┘             └────┘

  Key: mins[i] always holds min(data[0..i]) — O(1) GetMin()
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

**Textual Figure: Max Stack with Parallel Tracking**

```
Push sequence with max tracking:

  Push(5)   Push(1)   Push(5)   Push(3)   Push(7)   Push(2)
  data maxs data maxs data maxs data maxs data maxs data maxs
  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
  │ 5 ││ 5 ││ 1 ││ 5 ││ 5 ││ 5 ││ 3 ││ 5 ││ 7 ││ 7 ││ 2 ││ 7 │←top
  └───┘└───┘├───┤├───┤├───┤├───┤├───┤├───┤├───┤├───┤├───┤├───┤
            │ 5 ││ 5 ││ 1 ││ 5 ││ 5 ││ 5 ││ 7 ││ 7 ││ 7 ││ 7 │
            └───┘└───┘├───┤├───┤├───┤├───┤├───┤├───┤├───┤├───┤
                      │ 5 ││ 5 ││ 1 ││ 5 ││ 3 ││ 5 ││ 3 ││ 5 │
                      └───┘└───┘├───┤├───┤├───┤├───┤├───┤├───┤
                                │ 5 ││ 5 ││ 5 ││ 5 ││ 5 ││ 5 │
                                └───┘└───┘├───┤├───┤├───┤├───┤
                                          │ 1 ││ 5 ││ 1 ││ 5 │
                                          └───┘└───┘├───┤├───┤
                                                    │ 5 ││ 5 │
                                                    └───┘└───┘
  max: 5     5     5     5     7     7

Pop sequence:
  Pop(2)→max=7  Pop(7)→max=5  Pop(3)→max=5
  Pop(5)→max=5  Pop(1)→max=5  Pop(5)→empty
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

**Textual Figure: Two Stacks in One Array**

```
Shared array (capacity=10):

  Stack1 grows →→→                  ←←← Stack2 grows
  top1                                         top2
   │                                             │
   ▼                                             ▼
  ┌────┬────┬────┬────┬────┬────┬────┬────┬─────┬─────┐
  │  1 │  2 │  3 │    │    │    │    │    │ 200 │ 100 │
  └────┴────┴────┴────┴────┴────┴────┴────┴─────┴─────┘
  [0]   [1]  [2]  [3]  [4]  [5]  [6]  [7]   [8]   [9]

  Pop1() → 3:  top1 moves left to index 1
  Pop2() → 200: top2 moves right to index 9

  After filling Stack1 with 5 more (10..14):
  top1=7, top2=8 → no room left → Push1(99) = overflow!

  ┌────┬────┬────┬────┬────┬────┬────┬────┬─────┬─────┐
  │  1 │  2 │ 10 │ 11 │ 12 │ 13 │ 14 │    │     │ 100 │
  └────┴────┴────┴────┴────┴────┴────┴────┴─────┴─────┘
              top1=6 ↑              ↑ top2=9
              Collision zone: top1+1 >= top2
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

**Textual Figure: Queue Using Two Stacks**

```
Push(1), Push(2), Push(3):

  pushStack          popStack
  ┌───┐
  │ 3 │←top           (empty)
  ├───┤
  │ 2 │
  ├───┤
  │ 1 │
  └───┘

Peek()/Pop() triggers transfer (popStack is empty):
  Pop all from pushStack → push to popStack:

  pushStack          popStack
                     ┌───┐
  (empty)            │ 1 │←top  ▒ Peek()=1
                     ├───┤
                     │ 2 │
                     ├───┤
                     │ 3 │
                     └───┘

  Pop()→1  Pop()→2  Push(4)  Pop()→3  Pop()→4

  pushStack  popStack      pushStack  popStack
             ┌───┐         ┌───┐
             │ 2 │←top     │ 4 │←top  (empty)  → transfer → Pop()
             ├───┤         └───┘
             │ 3 │
             └───┘

  FIFO order achieved: 1, 2, 3, 4  (amortized O(1) per op)
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

**Textual Figure: Reverse a Stack Using Recursion**

```
Original stack: [1, 2, 3, 4, 5]  (1=bottom, 5=top)

  reverseStack() pops each element, recurses, then inserts at bottom:

  Step 1: Pop 5, recurse on [1,2,3,4]
  Step 2: Pop 4, recurse on [1,2,3]
  Step 3: Pop 3, recurse on [1,2]
  Step 4: Pop 2, recurse on [1]
  Step 5: Pop 1, recurse on []  (base case)

  Now insertAtBottom in reverse order:

  insertAtBottom(1):  │ 1 │
  insertAtBottom(2):  │ 2 │ 1 │       (pop 1, insert 2, push 1 back)
  insertAtBottom(3):  │ 3 │ 2 │ 1 │
  insertAtBottom(4):  │ 4 │ 3 │ 2 │ 1 │
  insertAtBottom(5):  │ 5 │ 4 │ 3 │ 2 │ 1 │

  Before:              After:
  ┌───┐  top            ┌───┐  top
  │ 5 │                │ 1 │
  ├───┤                ├───┤
  │ 4 │                │ 2 │
  ├───┤                ├───┤
  │ 3 │                │ 3 │
  ├───┤                ├───┤
  │ 2 │                │ 4 │
  ├───┤                ├───┤
  │ 1 │  bottom        │ 5 │  bottom
  └───┘                └───┘

  Result: [5, 4, 3, 2, 1]
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

**Textual Figure: Sort a Stack Using Recursion**

```
Original: [34, 3, 31, 98, 92, 23]  (34=bottom, 23=top)

sortStack pops all elements, then inserts each in sorted position:

  Pop all:  23 → 92 → 98 → 31 → 3 → 34

  sortedInsert builds sorted stack (bottom to top = ascending):

  insert(34):  │ 34 │
  insert(3):   │  3 │ 34 │         (3 < 34, so insert below)
  insert(31):  │  3 │ 31 │ 34 │    (3 < 31, pop; 31 < 34, insert)
  insert(98):  │  3 │ 31 │ 34 │ 98 │  (98 >= top, push on top)
  insert(92):  │  3 │ 31 │ 34 │ 92 │ 98 │  (pop 98, 92<98; push 92, push 98)
  insert(23):  │  3 │ 23 │ 31 │ 34 │ 92 │ 98 │

  Final sorted stack:
  ┌────┐  top
  │ 98 │
  ├────┤
  │ 92 │
  ├────┤
  │ 34 │
  ├────┤
  │ 31 │
  ├────┤
  │ 23 │
  ├────┤
  │  3 │  bottom
  └────┘
  Slice: [3, 23, 31, 34, 92, 98]
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

**Textual Figure: Stack with Middle Operation**

```
Push sequence with middle pointer tracking:

  Push  head(top)                         mid    size
  ────  ─────────────────────────     ───    ────
  1     [1]                               1      1
  2     [2]↔[1]                           1      2    (even: mid stays)
  3     [3]↔[2]↔[1]                       2      3    (odd: mid moves left)
  4     [4]↔[3]↔[2]↔[1]                   2      4    (even: mid stays)
  5     [5]↔[4]↔[3]↔[2]↔[1]               3      5    (odd: mid moves left)
  6     [6]↔[5]↔[4]↔[3]↔[2]↔[1]           3      6    (even: mid stays)
  7     [7]↔[6]↔[5]↔[4]↔[3]↔[2]↔[1]       4      7    (odd: mid moves left)
            head         ↑ mid

  Doubly linked list with mid pointer:
  head→[7]↔[6]↔[5]↔[4]↔[3]↔[2]↔[1]
                     ↑
                    mid (element 4, middle of 7 elements)

  Pop adjusts mid pointer:
  Pop 7 → mid=4 (size=6, even: mid moves right to 3)
  Pop 6 → mid=3 (size=5, odd: mid stays)
  Pop 5 → mid=3 (size=4, even: mid moves right to 2)
  ...continues until empty
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

**Textual Figure: Undo/Redo with Two Stacks**

```
Operation         content        undoStack              redoStack
─────────         ───────        ─────────              ─────────
Insert("Hello")   "Hello"        [ins:"Hello"]           []
Insert(" World")  "Hello World"  [ins:"Hello",           []
                                  ins:" World"]
Insert("!")       "Hello World!" [ins:"Hello",           []
                                  ins:" World",
                                  ins:"!"]

Undo()            "Hello World"  [ins:"Hello",           [ins:"!"]
                                  ins:" World"]
Undo()            "Hello"        [ins:"Hello"]           [ins:"!",
                                                          ins:" World"]
Redo()            "Hello World"  [ins:"Hello",           [ins:"!"]
                                  ins:" World"]

Insert(" Go")     "Hello World"  [ins:"Hello",           []  ← cleared!
                  " Go"          ins:" World",
                                  ins:" Go"]
Redo()            (nothing)      (unchanged)             []  ← empty

  undoStack               redoStack
  ┌─────────────┐         ┌─────────────┐
  │ ins:" Go"   │←top     │             │  empty
  ├─────────────┤         └─────────────┘
  │ ins:" World"│
  ├─────────────┤
  │ ins:"Hello" │
  └─────────────┘
  New insert clears redo stack (can't redo after new action)
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
