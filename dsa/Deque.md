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
