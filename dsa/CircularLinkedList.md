# Phase 5: Linked Lists — Circular Linked List

## What is a Circular Linked List?

A **circular linked list** is a linked list where the last node points back to the first node (instead of `nil`), forming a cycle.

**Two types:**
- **Circular singly linked list**: last.Next → head
- **Circular doubly linked list**: last.Next → head, head.Prev → last

```
┌──→ [10] → [20] → [30] → [40] ──┐
└──────────────────────────────────┘
```

**Use cases:** Round-robin scheduling, circular buffers, Josephus problem, looping playlists.

---

## Example 1: Basic Circular Singly Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type CircularList struct {
    Tail *Node // pointing to tail makes head access O(1) too (tail.Next = head)
    Size int
}

func NewCircularList() *CircularList {
    return &CircularList{}
}

func (cl *CircularList) InsertFront(val int) {
    n := &Node{Val: val}
    if cl.Tail == nil {
        n.Next = n // points to itself
        cl.Tail = n
    } else {
        n.Next = cl.Tail.Next // new node points to old head
        cl.Tail.Next = n       // tail points to new head
    }
    cl.Size++
}

func (cl *CircularList) InsertBack(val int) {
    cl.InsertFront(val)
    cl.Tail = cl.Tail.Next // advance tail to the newly inserted node
}

func (cl *CircularList) Print() {
    if cl.Tail == nil {
        fmt.Println("(empty)")
        return
    }
    head := cl.Tail.Next
    curr := head
    for {
        fmt.Printf("%d → ", curr.Val)
        curr = curr.Next
        if curr == head {
            break
        }
    }
    fmt.Println("(back to head)")
}

func main() {
    cl := NewCircularList()
    cl.InsertBack(10)
    cl.InsertBack(20)
    cl.InsertBack(30)
    cl.InsertFront(5)

    cl.Print() // 5 → 10 → 20 → 30 → (back to head)
    fmt.Println("Size:", cl.Size)
}
```

---

## Example 2: Delete from Circular List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type CircularList struct {
    Tail *Node
    Size int
}

func (cl *CircularList) InsertBack(val int) {
    n := &Node{Val: val}
    if cl.Tail == nil {
        n.Next = n
        cl.Tail = n
    } else {
        n.Next = cl.Tail.Next
        cl.Tail.Next = n
        cl.Tail = n
    }
    cl.Size++
}

func (cl *CircularList) DeleteVal(val int) bool {
    if cl.Tail == nil {
        return false
    }

    head := cl.Tail.Next

    // Single element
    if cl.Size == 1 {
        if head.Val == val {
            cl.Tail = nil
            cl.Size--
            return true
        }
        return false
    }

    // Head deletion
    if head.Val == val {
        cl.Tail.Next = head.Next
        cl.Size--
        return true
    }

    // Search for node to delete
    curr := head
    for curr.Next != head {
        if curr.Next.Val == val {
            if curr.Next == cl.Tail {
                cl.Tail = curr
            }
            curr.Next = curr.Next.Next
            cl.Size--
            return true
        }
        curr = curr.Next
    }
    return false
}

func (cl *CircularList) Print() {
    if cl.Tail == nil {
        fmt.Println("(empty)")
        return
    }
    head := cl.Tail.Next
    curr := head
    for {
        fmt.Printf("%d → ", curr.Val)
        curr = curr.Next
        if curr == head {
            break
        }
    }
    fmt.Println("↻")
}

func main() {
    cl := &CircularList{}
    for _, v := range []int{10, 20, 30, 40, 50} {
        cl.InsertBack(v)
    }
    cl.Print()

    cl.DeleteVal(30)
    fmt.Print("After delete 30: ")
    cl.Print()

    cl.DeleteVal(10)
    fmt.Print("After delete 10: ")
    cl.Print()

    cl.DeleteVal(50)
    fmt.Print("After delete 50: ")
    cl.Print()
}
```

---

## Example 3: Josephus Problem

```go
package main

import "fmt"

// n people in a circle, every k-th person is eliminated.
// Find the survivor's position (0-indexed).
func josephus(n, k int) int {
    // Build circular list
    type Node struct {
        Val  int
        Next *Node
    }

    head := &Node{Val: 0}
    curr := head
    for i := 1; i < n; i++ {
        curr.Next = &Node{Val: i}
        curr = curr.Next
    }
    curr.Next = head // make circular

    // Eliminate every k-th person
    prev := curr // points to last node
    curr = head
    remaining := n

    for remaining > 1 {
        // Move k-1 steps
        for i := 0; i < k-1; i++ {
            prev = curr
            curr = curr.Next
        }
        fmt.Printf("  Eliminate person %d\n", curr.Val)
        prev.Next = curr.Next
        curr = prev.Next
        remaining--
    }
    return curr.Val
}

// Mathematical solution: O(n) time, O(1) space
func josephusMath(n, k int) int {
    result := 0
    for i := 2; i <= n; i++ {
        result = (result + k) % i
    }
    return result
}

func main() {
    n, k := 7, 3
    fmt.Printf("Josephus(%d, %d):\n", n, k)
    survivor := josephus(n, k)
    fmt.Printf("Survivor: person %d\n\n", survivor)

    fmt.Printf("Math solution: %d\n", josephusMath(n, k))

    // More examples
    for _, test := range [][2]int{{5, 2}, {10, 3}, {6, 4}} {
        fmt.Printf("Josephus(%d,%d) = %d\n", test[0], test[1], josephusMath(test[0], test[1]))
    }
}
```

---

## Example 4: Round-Robin Scheduler

```go
package main

import "fmt"

type Task struct {
    Name      string
    Remaining int
    Next      *Task
}

func roundRobin(tasks []struct{ Name string; Time int }, quantum int) {
    if len(tasks) == 0 {
        return
    }

    // Build circular list
    head := &Task{Name: tasks[0].Name, Remaining: tasks[0].Time}
    curr := head
    for i := 1; i < len(tasks); i++ {
        curr.Next = &Task{Name: tasks[i].Name, Remaining: tasks[i].Time}
        curr = curr.Next
    }
    curr.Next = head // circular

    clock := 0
    prev := curr

    for head != nil {
        run := quantum
        if head.Remaining < run {
            run = head.Remaining
        }

        clock += run
        head.Remaining -= run
        fmt.Printf("  t=%3d: %s ran %d units (remaining: %d)\n",
            clock, head.Name, run, head.Remaining)

        if head.Remaining == 0 {
            fmt.Printf("  >>> %s COMPLETE at t=%d\n", head.Name, clock)
            if head.Next == head {
                // Last task
                break
            }
            prev.Next = head.Next
            head = head.Next
        } else {
            prev = head
            head = head.Next
        }
    }
    fmt.Printf("All tasks completed at t=%d\n", clock)
}

func main() {
    tasks := []struct{ Name string; Time int }{
        {"P1", 10},
        {"P2", 5},
        {"P3", 8},
        {"P4", 3},
    }
    fmt.Println("Round Robin (quantum=4):")
    roundRobin(tasks, 4)
}
```

---

## Example 5: Circular Doubly Linked List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Prev *Node
    Next *Node
}

type CircularDLL struct {
    Head *Node
    Size int
}

func NewCircularDLL() *CircularDLL {
    return &CircularDLL{}
}

func (cdll *CircularDLL) Insert(val int) {
    n := &Node{Val: val}
    if cdll.Head == nil {
        n.Next = n
        n.Prev = n
        cdll.Head = n
    } else {
        tail := cdll.Head.Prev
        n.Next = cdll.Head
        n.Prev = tail
        tail.Next = n
        cdll.Head.Prev = n
    }
    cdll.Size++
}

func (cdll *CircularDLL) Remove(val int) bool {
    if cdll.Head == nil {
        return false
    }
    curr := cdll.Head
    for {
        if curr.Val == val {
            if cdll.Size == 1 {
                cdll.Head = nil
            } else {
                curr.Prev.Next = curr.Next
                curr.Next.Prev = curr.Prev
                if curr == cdll.Head {
                    cdll.Head = curr.Next
                }
            }
            cdll.Size--
            return true
        }
        curr = curr.Next
        if curr == cdll.Head {
            break
        }
    }
    return false
}

func (cdll *CircularDLL) PrintForward(laps int) {
    if cdll.Head == nil {
        fmt.Println("(empty)")
        return
    }
    curr := cdll.Head
    count := 0
    for count < cdll.Size*laps {
        fmt.Printf("%d → ", curr.Val)
        curr = curr.Next
        count++
    }
    fmt.Println("↻")
}

func main() {
    cdll := NewCircularDLL()
    for _, v := range []int{1, 2, 3, 4, 5} {
        cdll.Insert(v)
    }

    fmt.Print("1 lap:  ")
    cdll.PrintForward(1)

    fmt.Print("2 laps: ")
    cdll.PrintForward(2)

    cdll.Remove(3)
    fmt.Print("After remove 3: ")
    cdll.PrintForward(1)
}
```

---

## Example 6: Circular Buffer (Ring Buffer)

```go
package main

import "fmt"

type RingBuffer struct {
    data  []int
    head  int
    tail  int
    size  int
    cap   int
}

func NewRingBuffer(cap int) *RingBuffer {
    return &RingBuffer{
        data: make([]int, cap),
        cap:  cap,
    }
}

func (rb *RingBuffer) Push(val int) bool {
    if rb.size == rb.cap {
        return false // full
    }
    rb.data[rb.tail] = val
    rb.tail = (rb.tail + 1) % rb.cap
    rb.size++
    return true
}

func (rb *RingBuffer) Pop() (int, bool) {
    if rb.size == 0 {
        return 0, false
    }
    val := rb.data[rb.head]
    rb.head = (rb.head + 1) % rb.cap
    rb.size--
    return val, true
}

func (rb *RingBuffer) PushOverwrite(val int) {
    if rb.size == rb.cap {
        rb.head = (rb.head + 1) % rb.cap // overwrite oldest
        rb.size--
    }
    rb.data[rb.tail] = val
    rb.tail = (rb.tail + 1) % rb.cap
    rb.size++
}

func (rb *RingBuffer) Print() {
    fmt.Printf("[size=%d cap=%d] ", rb.size, rb.cap)
    for i := 0; i < rb.size; i++ {
        idx := (rb.head + i) % rb.cap
        fmt.Printf("%d ", rb.data[idx])
    }
    fmt.Println()
}

func main() {
    rb := NewRingBuffer(5)
    for i := 1; i <= 5; i++ {
        rb.Push(i)
    }
    rb.Print()

    rb.Pop()
    rb.Pop()
    rb.Print()

    rb.Push(6)
    rb.Push(7)
    rb.Print()

    // Overwrite mode
    fmt.Println("\nOverwrite mode:")
    rb2 := NewRingBuffer(3)
    for i := 1; i <= 7; i++ {
        rb2.PushOverwrite(i)
        rb2.Print()
    }
}
```

---

## Example 7: Detect if List is Circular

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func isCircular(head *Node) bool {
    if head == nil {
        return false
    }
    slow, fast := head, head
    for {
        if fast.Next == nil || fast.Next.Next == nil {
            return false // has nil → not circular
        }
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Check if cycle includes head (true circular)
            curr := slow
            for {
                if curr == head {
                    return true
                }
                curr = curr.Next
                if curr == slow {
                    break
                }
            }
            return false
        }
    }
}

func main() {
    // Circular list
    n1 := &Node{Val: 1}
    n2 := &Node{Val: 2}
    n3 := &Node{Val: 3}
    n1.Next = n2
    n2.Next = n3
    n3.Next = n1 // circular
    fmt.Println("Circular:", isCircular(n1)) // true

    // Linear list
    a := &Node{Val: 10, Next: &Node{Val: 20, Next: &Node{Val: 30}}}
    fmt.Println("Linear:", isCircular(a)) // false

    // List with cycle but not at head
    b1 := &Node{Val: 1}
    b2 := &Node{Val: 2}
    b3 := &Node{Val: 3}
    b1.Next = b2
    b2.Next = b3
    b3.Next = b2 // cycle at b2, not head
    fmt.Println("Cycle not at head:", isCircular(b1)) // false
}
```

---

## Example 8: Music Playlist (Circular)

```go
package main

import "fmt"

type Song struct {
    Title string
    Next  *Song
    Prev  *Song
}

type Playlist struct {
    Current *Song
    Size    int
}

func (p *Playlist) AddSong(title string) {
    song := &Song{Title: title}
    if p.Current == nil {
        song.Next = song
        song.Prev = song
        p.Current = song
    } else {
        last := p.Current.Prev
        song.Next = p.Current
        song.Prev = last
        last.Next = song
        p.Current.Prev = song
    }
    p.Size++
}

func (p *Playlist) NextSong() string {
    if p.Current == nil {
        return ""
    }
    p.Current = p.Current.Next
    return p.Current.Title
}

func (p *Playlist) PrevSong() string {
    if p.Current == nil {
        return ""
    }
    p.Current = p.Current.Prev
    return p.Current.Title
}

func (p *Playlist) NowPlaying() string {
    if p.Current == nil {
        return "(empty)"
    }
    return p.Current.Title
}

func main() {
    pl := &Playlist{}
    pl.AddSong("Bohemian Rhapsody")
    pl.AddSong("Stairway to Heaven")
    pl.AddSong("Hotel California")
    pl.AddSong("Imagine")

    fmt.Println("Now playing:", pl.NowPlaying())
    for i := 0; i < 6; i++ {
        fmt.Println("Next:", pl.NextSong())
    }
    fmt.Println("---")
    for i := 0; i < 3; i++ {
        fmt.Println("Prev:", pl.PrevSong())
    }
}
```

---

## Example 9: Split Circular List into Two

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func splitCircular(head *Node) (*Node, *Node) {
    if head == nil || head.Next == head {
        return head, nil
    }

    slow, fast := head, head
    for fast.Next != head && fast.Next.Next != head {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // slow is at midpoint
    head2 := slow.Next
    
    // Find tail of second half
    curr := head2
    for curr.Next != head {
        curr = curr.Next
    }
    curr.Next = head2  // second list circular
    slow.Next = head   // first list circular

    return head, head2
}

func printCircular(head *Node) {
    if head == nil {
        fmt.Println("(nil)")
        return
    }
    curr := head
    for {
        fmt.Printf("%d → ", curr.Val)
        curr = curr.Next
        if curr == head {
            break
        }
    }
    fmt.Println("↻")
}

func main() {
    // Build circular: 1→2→3→4→5→6→(back to 1)
    nodes := make([]*Node, 6)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < len(nodes)-1; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[len(nodes)-1].Next = nodes[0]

    fmt.Print("Original: ")
    printCircular(nodes[0])

    h1, h2 := splitCircular(nodes[0])
    fmt.Print("First:    ")
    printCircular(h1)
    fmt.Print("Second:   ")
    printCircular(h2)
}
```

---

## Example 10: Count Nodes in Circular List

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func countNodes(head *Node) int {
    if head == nil {
        return 0
    }
    count := 1
    curr := head.Next
    for curr != head {
        count++
        curr = curr.Next
    }
    return count
}

func sumNodes(head *Node) int {
    if head == nil {
        return 0
    }
    sum := head.Val
    curr := head.Next
    for curr != head {
        sum += curr.Val
        curr = curr.Next
    }
    return sum
}

func searchCircular(head *Node, target int) bool {
    if head == nil {
        return false
    }
    if head.Val == target {
        return true
    }
    curr := head.Next
    for curr != head {
        if curr.Val == target {
            return true
        }
        curr = curr.Next
    }
    return false
}

func buildCircular(vals []int) *Node {
    if len(vals) == 0 {
        return nil
    }
    head := &Node{Val: vals[0]}
    curr := head
    for i := 1; i < len(vals); i++ {
        curr.Next = &Node{Val: vals[i]}
        curr = curr.Next
    }
    curr.Next = head
    return head
}

func main() {
    head := buildCircular([]int{10, 20, 30, 40, 50})

    fmt.Println("Count:", countNodes(head))
    fmt.Println("Sum:", sumNodes(head))
    fmt.Println("Search 30:", searchCircular(head, 30))
    fmt.Println("Search 99:", searchCircular(head, 99))
}
```

---

## Circular List Comparison

| Feature | Linear | Circular |
|---------|--------|----------|
| Last node points to | nil | head |
| Traversal | Start → nil | Start → back to start |
| End detection | curr == nil | curr == head |
| Use case | General | Round-robin, cyclic |
| Insert at tail | O(n) or O(1)* | O(1) with tail ref |

## Key Takeaways

1. **Last node → head** instead of nil — that's the only structural difference
2. **Josephus problem** — classic circular list application
3. **Round-robin scheduling** — natural fit for circular structure
4. **Ring buffer** — array-based circular structure for fixed-size queues
5. **Be careful**: traversal loops must check `curr == head` to detect full circle
6. **Tail pointer** trick: store tail instead of head → both head (tail.Next) and tail are O(1)
7. **Splitting** a circular list requires careful pointer updates

> **Next up:** Fast Slow Pointer Technique →
