# Phase 7: Queue — Deque (Double-Ended Queue)

## Overview

A **deque** (double-ended queue) supports insertion and removal at **both** ends in O(1).

| Operation | Time |
|-----------|------|
| PushFront / PushBack | O(1) |
| PopFront / PopBack | O(1) |
| PeekFront / PeekBack | O(1) |

Go has no built-in deque — implement with a **doubly linked list** or **circular array**.

---

## Example 1: Deque with Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val        int
    Prev, Next *Node
}

type Deque struct {
    head, tail *Node
    size       int
}

func (d *Deque) PushFront(val int) {
    node := &Node{Val: val, Next: d.head}
    if d.head != nil {
        d.head.Prev = node
    }
    d.head = node
    if d.tail == nil {
        d.tail = node
    }
    d.size++
}

func (d *Deque) PushBack(val int) {
    node := &Node{Val: val, Prev: d.tail}
    if d.tail != nil {
        d.tail.Next = node
    }
    d.tail = node
    if d.head == nil {
        d.head = node
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

func main() {
    d := &Deque{}
    d.PushBack(1)
    d.PushBack(2)
    d.PushFront(0)
    d.PushBack(3)

    // Should be: 0 1 2 3
    for d.size > 0 {
        v, _ := d.PopFront()
        fmt.Printf("%d ", v)
    }
    fmt.Println() // 0 1 2 3
}
```

**Textual Figure:**

```
Deque with Doubly Linked List — step by step:

PushBack(1):   head/tail
                  │
               ┌──┴──┐
          nil◀─┤  1  ├─▶nil
               └─────┘

PushBack(2):   head       tail
                │           │
             ┌──┴──┐   ┌──┴──┐
        nil◀─┤  1  ├─▶┤  2  ├─▶nil
             └─────┘   └─────┘

PushFront(0):  head               tail
                │                   │
             ┌──┴──┐   ┌─────┐   ┌──┴──┐
        nil◀─┤  0  ├─▶┤  1  ├─▶┤  2  ├─▶nil
             └─────┘   └─────┘   └─────┘

PushBack(3):  head                           tail
               │                               │
            ┌──┴──┐   ┌─────┐   ┌─────┐   ┌──┴──┐
       nil◀─┤  0  ├─▶┤  1  ├─▶┤  2  ├─▶┤  3  ├─▶nil
            └─────┘   └─────┘   └─────┘   └─────┘

PopFront all: 0 → 1 → 2 → 3  (FIFO from front)
```

---

## Example 2: Deque with Circular Array

```go
package main

import "fmt"

type ArrayDeque struct {
    data []int
    head int
    size int
    cap  int
}

func NewArrayDeque(cap int) *ArrayDeque {
    return &ArrayDeque{
        data: make([]int, cap),
        cap:  cap,
    }
}

func (d *ArrayDeque) PushFront(val int) {
    if d.size == d.cap {
        d.resize(d.cap * 2)
    }
    d.head = (d.head - 1 + d.cap) % d.cap
    d.data[d.head] = val
    d.size++
}

func (d *ArrayDeque) PushBack(val int) {
    if d.size == d.cap {
        d.resize(d.cap * 2)
    }
    idx := (d.head + d.size) % d.cap
    d.data[idx] = val
    d.size++
}

func (d *ArrayDeque) PopFront() (int, bool) {
    if d.size == 0 {
        return 0, false
    }
    val := d.data[d.head]
    d.head = (d.head + 1) % d.cap
    d.size--
    return val, true
}

func (d *ArrayDeque) PopBack() (int, bool) {
    if d.size == 0 {
        return 0, false
    }
    idx := (d.head + d.size - 1) % d.cap
    val := d.data[idx]
    d.size--
    return val, true
}

func (d *ArrayDeque) resize(newCap int) {
    newData := make([]int, newCap)
    for i := 0; i < d.size; i++ {
        newData[i] = d.data[(d.head+i)%d.cap]
    }
    d.data = newData
    d.head = 0
    d.cap = newCap
}

func main() {
    d := NewArrayDeque(4)
    d.PushBack(1)
    d.PushBack(2)
    d.PushFront(0)
    d.PushFront(-1)
    d.PushBack(3) // triggers resize

    for d.size > 0 {
        v, _ := d.PopFront()
        fmt.Printf("%d ", v)
    }
    fmt.Println() // -1 0 1 2 3
}
```

**Textual Figure:**

```
Circular Array Deque (initial cap=4):

PushBack(1):  head=0           PushBack(2):  head=0
  ┌───┬───┬───┬───┐            ┌───┬───┬───┬───┐
  │ 1 │   │   │   │            │ 1 │ 2 │   │   │
  └───┴───┴───┴───┘            └───┴───┴───┴───┘
   H                              H

PushFront(0):  head wraps to 3   PushFront(-1): head wraps to 2
  ┌───┬───┬───┬───┐            ┌───┬───┬────┬───┐
  │ 1 │ 2 │   │ 0 │  size=3    │ 1 │ 2 │ -1 │ 0 │  size=4 FULL
  └───┴───┴───┴───┘            └───┴───┴────┴───┘
               H                          H

PushBack(3): resize cap 4→8! Copy in logical order:
  ┌────┬───┬───┬───┬───┬───┬───┬───┐
  │ -1 │ 0 │ 1 │ 2 │ 3 │   │   │   │  head=0, cap=8
  └────┴───┴───┴───┴───┴───┴───┴───┘
   H

PopFront all: -1 → 0 → 1 → 2 → 3
```

---

## Example 3: Design Circular Deque (LeetCode 641)

```go
package main

import "fmt"

type MyCircularDeque struct {
    data     []int
    head     int
    size     int
    capacity int
}

func Constructor(k int) MyCircularDeque {
    return MyCircularDeque{
        data:     make([]int, k),
        capacity: k,
    }
}

func (d *MyCircularDeque) InsertFront(value int) bool {
    if d.IsFull() {
        return false
    }
    d.head = (d.head - 1 + d.capacity) % d.capacity
    d.data[d.head] = value
    d.size++
    return true
}

func (d *MyCircularDeque) InsertLast(value int) bool {
    if d.IsFull() {
        return false
    }
    idx := (d.head + d.size) % d.capacity
    d.data[idx] = value
    d.size++
    return true
}

func (d *MyCircularDeque) DeleteFront() bool {
    if d.IsEmpty() {
        return false
    }
    d.head = (d.head + 1) % d.capacity
    d.size--
    return true
}

func (d *MyCircularDeque) DeleteLast() bool {
    if d.IsEmpty() {
        return false
    }
    d.size--
    return true
}

func (d *MyCircularDeque) GetFront() int {
    if d.IsEmpty() {
        return -1
    }
    return d.data[d.head]
}

func (d *MyCircularDeque) GetRear() int {
    if d.IsEmpty() {
        return -1
    }
    return d.data[(d.head+d.size-1+d.capacity)%d.capacity]
}

func (d *MyCircularDeque) IsEmpty() bool { return d.size == 0 }
func (d *MyCircularDeque) IsFull() bool  { return d.size == d.capacity }

func main() {
    d := Constructor(3)
    fmt.Println(d.InsertLast(1))  // true
    fmt.Println(d.InsertLast(2))  // true
    fmt.Println(d.InsertFront(3)) // true
    fmt.Println(d.InsertFront(4)) // false (full)
    fmt.Println("Rear:", d.GetRear())   // 2
    fmt.Println(d.IsFull())             // true
    fmt.Println(d.DeleteLast())         // true
    fmt.Println(d.InsertFront(4))       // true
    fmt.Println("Front:", d.GetFront()) // 4
}
```

**Textual Figure:**

```
Circular Deque (capacity=3):  indices [0] [1] [2]

InsertLast(1):           InsertLast(2):
  ┌───┬───┬───┐              ┌───┬───┬───┐
  │ 1 │   │   │  size=1      │ 1 │ 2 │   │  size=2
  └───┴───┴───┘              └───┴───┴───┘
   H                          H

InsertFront(3):  head wraps to index 2
  ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │   size=3 FULL!  head=2
  └───┴───┴───┘
              H   Logical order: 3, 1, 2

InsertFront(4) → false (full)
GetRear() → 2    IsFull() → true

DeleteLast():  removes 2
  ┌───┬───┬───┐
  │ 1 │   │ 3 │   size=2  Logical: 3, 1
  └───┴───┴───┘
              H

InsertFront(4):  head wraps to index 1
  ┌───┬───┬───┐
  │ 1 │ 4 │ 3 │   size=3  Logical: 4, 3, 1
  └───┴───┴───┘
       H          GetFront() → 4
```

---

## Example 4: Sliding Window Maximum (LeetCode 239) — Deque

```go
package main

import "fmt"

func maxSlidingWindow(nums []int, k int) []int {
    deque := []int{} // stores indices, monotonically decreasing values
    result := []int{}

    for i, num := range nums {
        // Remove elements outside window
        for len(deque) > 0 && deque[0] <= i-k {
            deque = deque[1:]
        }

        // Remove smaller elements from back
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= num {
            deque = deque[:len(deque)-1]
        }

        deque = append(deque, i)

        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }

    return result
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1, 3, -1, -3, 5, 3, 6, 7}, 3},
        {[]int{1}, 1},
        {[]int{1, -1}, 1},
        {[]int{9, 11}, 2},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %v\n", t.nums, t.k, maxSlidingWindow(t.nums, t.k))
    }
    // [1,3,-1,-3,5,3,6,7] k=3 → [3,3,5,5,6,7]
}
```

**Textual Figure:**

```
Sliding Window Maximum (k=3):  nums = [1, 3, -1, -3, 5, 3, 6, 7]
Deque stores indices, maintains decreasing values.

i=0: num=1   deque=[0]            window incomplete
i=1: num=3   3>1, pop 0           deque=[1]            window incomplete
i=2: num=-1  -1<3, keep           deque=[1,2]           max=nums[1]=3
             ┌─────────┐
     window  │ 1  3 -1 │ -3  5  3  6  7    result=[3]
             └─────────┘
i=3: num=-3  -3<-1, keep          deque=[1,2,3]         max=3
i=4: num=5   expire 1, pop 2,3    deque=[4]             max=5
i=5: num=3   3<5, keep            deque=[4,5]           max=5
i=6: num=6   expire 4(=6-3), pop 5 deque=[6]            max=6
i=7: num=7   7>6, pop 6           deque=[7]             max=7

Result: [3, 3, 5, 5, 6, 7]
```

---

## Example 5: Maximum of All Subarrays of Size K

```go
package main

import "fmt"

func maxOfSubarrays(arr []int, k int) []int {
    if len(arr) == 0 || k == 0 {
        return nil
    }

    deque := []int{}
    result := []int{}

    for i := 0; i < len(arr); i++ {
        // Remove out-of-window indices
        if len(deque) > 0 && deque[0] == i-k {
            deque = deque[1:]
        }

        // Maintain decreasing order
        for len(deque) > 0 && arr[deque[len(deque)-1]] <= arr[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        if i >= k-1 {
            result = append(result, arr[deque[0]])
        }
    }

    return result
}

func main() {
    arr := []int{1, 2, 3, 1, 4, 5, 2, 3, 6}
    for k := 1; k <= 4; k++ {
        fmt.Printf("k=%d → %v\n", k, maxOfSubarrays(arr, k))
    }
}
```

**Textual Figure:**

```
arr = [1, 2, 3, 1, 4, 5, 2, 3, 6]

k=1:  each element is its own window
  → [1, 2, 3, 1, 4, 5, 2, 3, 6]

k=2:  windows: [1,2] [2,3] [3,1] [1,4] [4,5] [5,2] [2,3] [3,6]
  → [2, 3, 3, 4, 5, 5, 3, 6]

k=3:  windows (decreasing deque tracks max):
  ┌───────┐
  │1 2 3│ 1  4  5  2  3  6   max=3
  └───────┘
    ┌───────┐
  1 │2 3 1│ 4  5  2  3  6   max=3
    └───────┘             ...and so on
  → [3, 3, 4, 5, 5, 5, 6]

k=4:  → [3, 4, 5, 5, 5, 6]
```

---

## Example 6: Palindrome Checker Using Deque

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

type CharDeque struct {
    data []rune
}

func (d *CharDeque) PushBack(ch rune)  { d.data = append(d.data, ch) }

func (d *CharDeque) PopFront() rune {
    ch := d.data[0]
    d.data = d.data[1:]
    return ch
}

func (d *CharDeque) PopBack() rune {
    ch := d.data[len(d.data)-1]
    d.data = d.data[:len(d.data)-1]
    return ch
}

func (d *CharDeque) Size() int { return len(d.data) }

func isPalindrome(s string) bool {
    d := &CharDeque{}
    for _, ch := range strings.ToLower(s) {
        if unicode.IsLetter(ch) || unicode.IsDigit(ch) {
            d.PushBack(ch)
        }
    }

    for d.Size() > 1 {
        if d.PopFront() != d.PopBack() {
            return false
        }
    }
    return true
}

func main() {
    tests := []string{
        "racecar",
        "A man, a plan, a canal: Panama",
        "hello",
        "Was it a car or a cat I saw?",
    }

    for _, s := range tests {
        fmt.Printf("%-40s → %v\n", s, isPalindrome(s))
    }
}
```

**Textual Figure:**

```
Palindrome check: "racecar"

Load into deque (letters only, lowercase):
  front → ┌───┬───┬───┬───┬───┬───┬───┐ ← back
           │ r │ a │ c │ e │ c │ a │ r │
           └───┴───┴───┴───┴───┴───┴───┘

Compare both ends:
  Step 1: PopFront()=r  PopBack()=r   ✓ match
  Step 2: PopFront()=a  PopBack()=a   ✓ match
  Step 3: PopFront()=c  PopBack()=c   ✓ match
  Step 4: size=1 (middle 'e')         ✓ done

  Result: true (palindrome)

"hello": h≠o at step 1 → false
```

---

## Example 7: Steal-Half Work Stealing with Deque

```go
package main

import (
    "fmt"
    "sync"
)

// Work stealing: owner pushes/pops from back, thieves steal from front
type WorkDeque struct {
    tasks []string
    mu    sync.Mutex
}

func (d *WorkDeque) PushTask(task string) {
    d.mu.Lock()
    defer d.mu.Unlock()
    d.tasks = append(d.tasks, task)
}

func (d *WorkDeque) PopTask() (string, bool) {
    d.mu.Lock()
    defer d.mu.Unlock()
    if len(d.tasks) == 0 {
        return "", false
    }
    task := d.tasks[len(d.tasks)-1]
    d.tasks = d.tasks[:len(d.tasks)-1]
    return task, true
}

func (d *WorkDeque) StealTask() (string, bool) {
    d.mu.Lock()
    defer d.mu.Unlock()
    if len(d.tasks) == 0 {
        return "", false
    }
    task := d.tasks[0]
    d.tasks = d.tasks[1:]
    return task, true
}

func (d *WorkDeque) Size() int {
    d.mu.Lock()
    defer d.mu.Unlock()
    return len(d.tasks)
}

func main() {
    worker1 := &WorkDeque{}
    worker2 := &WorkDeque{}

    // Worker 1 has lots of tasks
    for i := 1; i <= 6; i++ {
        worker1.PushTask(fmt.Sprintf("task-%d", i))
    }

    fmt.Printf("Worker1: %d tasks, Worker2: %d tasks\n", worker1.Size(), worker2.Size())

    // Worker 2 steals from worker 1
    for worker2.Size() < 3 {
        task, ok := worker1.StealTask()
        if !ok {
            break
        }
        worker2.PushTask(task)
        fmt.Printf("Worker2 stole: %s\n", task)
    }

    fmt.Printf("Worker1: %d tasks, Worker2: %d tasks\n", worker1.Size(), worker2.Size())
}
```

**Textual Figure:**

```
Work Stealing with Deque:

Initial state:
  Worker1: ┌────┬────┬────┬────┬────┬────┐
           │ t-1│ t-2│ t-3│ t-4│ t-5│ t-6│   6 tasks
           └────┴────┴────┴────┴────┴────┘
            ↑steal                    ↑pop(owner)
  Worker2: (empty)

Worker2 steals from Worker1's FRONT:
  Steal t-1:  Worker1=[t-2..t-6]  Worker2=[t-1]
  Steal t-2:  Worker1=[t-3..t-6]  Worker2=[t-1,t-2]
  Steal t-3:  Worker1=[t-4..t-6]  Worker2=[t-1,t-2,t-3]

After stealing:
  Worker1: ┌────┬────┬────┐    Worker2: ┌────┬────┬────┐
           │ t-4│ t-5│ t-6│             │ t-1│ t-2│ t-3│
           └────┴────┴────┘             └────┴────┴────┘
           3 tasks                    3 tasks
```

---

## Example 8: Container/List as Deque

```go
package main

import (
    "container/list"
    "fmt"
)

func main() {
    d := list.New()

    // PushBack / PushFront
    d.PushBack(1)
    d.PushBack(2)
    d.PushFront(0)
    d.PushBack(3)

    // Iterate forward
    fmt.Print("Forward:  ")
    for e := d.Front(); e != nil; e = e.Next() {
        fmt.Printf("%d ", e.Value.(int))
    }
    fmt.Println() // 0 1 2 3

    // PopFront
    front := d.Front()
    d.Remove(front)
    fmt.Println("After PopFront, front:", d.Front().Value) // 1

    // PopBack
    back := d.Back()
    d.Remove(back)
    fmt.Println("After PopBack, back:", d.Back().Value) // 2

    // Remaining: 1 2
    fmt.Print("Remaining: ")
    for e := d.Front(); e != nil; e = e.Next() {
        fmt.Printf("%d ", e.Value.(int))
    }
    fmt.Println()
}
```

**Textual Figure:**

```
container/list as Deque:

PushBack(1), PushBack(2), PushFront(0), PushBack(3):
  Front                               Back
   │                                    │
   ▼                                    ▼
  ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 0 │─▶│ 1 │─▶│ 2 │─▶│ 3 │
  └───┘   └───┘   └───┘   └───┘

PopFront (remove 0):  Front→1
  ┌───┐   ┌───┐   ┌───┐
  │ 1 │─▶│ 2 │─▶│ 3 │
  └───┘   └───┘   └───┘

PopBack (remove 3):  Back→2
  ┌───┐   ┌───┐
  │ 1 │─▶│ 2 │       Remaining: 1 2
  └───┘   └───┘
```

---

## Example 9: Deque-Based Stack and Queue

```go
package main

import "fmt"

type Deque struct {
    data []int
}

func (d *Deque) PushFront(v int)        { d.data = append([]int{v}, d.data...) }
func (d *Deque) PushBack(v int)         { d.data = append(d.data, v) }
func (d *Deque) PopFront() int          { v := d.data[0]; d.data = d.data[1:]; return v }
func (d *Deque) PopBack() int           { v := d.data[len(d.data)-1]; d.data = d.data[:len(d.data)-1]; return v }
func (d *Deque) IsEmpty() bool          { return len(d.data) == 0 }

// Use deque as stack (LIFO)
func demoStack() {
    d := &Deque{}
    d.PushBack(1)
    d.PushBack(2)
    d.PushBack(3)

    fmt.Print("Stack (LIFO): ")
    for !d.IsEmpty() {
        fmt.Printf("%d ", d.PopBack())
    }
    fmt.Println() // 3 2 1
}

// Use deque as queue (FIFO)
func demoQueue() {
    d := &Deque{}
    d.PushBack(1)
    d.PushBack(2)
    d.PushBack(3)

    fmt.Print("Queue (FIFO): ")
    for !d.IsEmpty() {
        fmt.Printf("%d ", d.PopFront())
    }
    fmt.Println() // 1 2 3
}

func main() {
    demoStack()
    demoQueue()
}
```

**Textual Figure:**

```
Deque as Stack (LIFO):                 Deque as Queue (FIFO):
  PushBack(1,2,3):                       PushBack(1,2,3):
  ┌───┬───┬───┐                          ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │                          │ 1 │ 2 │ 3 │
  └───┴───┴───┘                          └───┴───┴───┘
                ↑pop                      ↑pop

  PopBack:  3, 2, 1                      PopFront: 1, 2, 3

  Same deque, different access patterns:
  │ Stack: push/pop from BACK only       │
  │ Queue: push BACK, pop FRONT          │
```

---

## Example 10: Shortest Subarray with Sum ≥ K (LeetCode 862)

```go
package main

import "fmt"

func shortestSubarray(nums []int, k int) int {
    n := len(nums)
    prefix := make([]int, n+1)
    for i := 0; i < n; i++ {
        prefix[i+1] = prefix[i] + nums[i]
    }

    deque := []int{} // monotonic increasing deque of prefix sum indices
    result := n + 1

    for i := 0; i <= n; i++ {
        // Check if we found a valid subarray
        for len(deque) > 0 && prefix[i]-prefix[deque[0]] >= k {
            length := i - deque[0]
            if length < result {
                result = length
            }
            deque = deque[1:] // pop front (used)
        }

        // Maintain monotonicity
        for len(deque) > 0 && prefix[i] <= prefix[deque[len(deque)-1]] {
            deque = deque[:len(deque)-1] // pop back
        }

        deque = append(deque, i)
    }

    if result > n {
        return -1
    }
    return result
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1}, 1},
        {[]int{1, 2}, 4},
        {[]int{2, -1, 2}, 3},
        {[]int{84, -37, 32, 40, 95}, 167},
    }

    for _, t := range tests {
        fmt.Printf("nums=%v, k=%d → %d\n", t.nums, t.k, shortestSubarray(t.nums, t.k))
    }
}
```

**Textual Figure:**

```
nums = [2, -1, 2], k = 3

Prefix sums: [0, 2, 1, 3]
              0  1  2  3   ← indices

Monotonic increasing deque of prefix indices:

i=0: prefix=0   deque=[0]
i=1: prefix=2   2-0=2 < 3, no valid     deque=[0,1]
i=2: prefix=1   1≤2 pop 1               deque=[0,2]
i=3: prefix=3   3-0=3 ≥ 3 ✓ len=3-0=3  deque=[2]
                3-1 would be 2, but 1 was popped

  Deque tracks: smallest prefix seen at each position
  When prefix[i] - prefix[deque[0]] ≥ k:
    → found valid subarray, record length, pop front

  Result: 3  (subarray [2,-1,2] has sum=3)
```

---

## Deque vs Queue vs Stack

| Structure | Push | Pop | Use Case |
|-----------|------|-----|----------|
| Stack | Back | Back | LIFO — DFS, undo |
| Queue | Back | Front | FIFO — BFS, scheduling |
| Deque | Both | Both | Sliding window, palindrome |

## Key Takeaways

1. **Deque** = double-ended queue — insert/remove from both ends in O(1)
2. **Sliding window max/min**: monotonic deque with index-based expiry
3. **Circular array** deque: `(head - 1 + cap) % cap` for PushFront
4. Go's `container/list` is a doubly linked list that works as a deque
5. A deque can simulate both **stack** and **queue** behaviors
6. **Work stealing**: owner pops from back, thieves steal from front
7. **Shortest subarray ≥ K**: prefix sums + monotonic deque

> **Next up:** Priority Queue →
