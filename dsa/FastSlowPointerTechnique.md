# Phase 5: Linked Lists — Fast Slow Pointer Technique

## What is the Fast-Slow Pointer Technique?

Also called **Floyd's Tortoise and Hare**, this technique uses two pointers moving at different speeds:
- **Slow pointer**: moves 1 step at a time
- **Fast pointer**: moves 2 steps at a time

**Key properties:**
- If there's a cycle, they will **meet** inside the cycle
- When fast reaches the end, slow is at the **middle**
- Used for cycle detection, middle finding, and more

---

## Example 1: Find Middle of Linked List

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
    return slow // slow is at middle
}

func fromSlice(arr []int) *Node {
    var head *Node
    for i := len(arr) - 1; i >= 0; i-- {
        head = &Node{Val: arr[i], Next: head}
    }
    return head
}

func main() {
    // Odd length
    head := fromSlice([]int{1, 2, 3, 4, 5})
    mid := findMiddle(head)
    fmt.Println("Middle of [1,2,3,4,5]:", mid.Val) // 3

    // Even length
    head2 := fromSlice([]int{1, 2, 3, 4, 5, 6})
    mid2 := findMiddle(head2)
    fmt.Println("Middle of [1,2,3,4,5,6]:", mid2.Val) // 4 (second middle)

    // Want first middle for even? Stop when fast.Next.Next == nil
    head3 := fromSlice([]int{1, 2, 3, 4})
    slow, fast := head3, head3
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    fmt.Println("First middle of [1,2,3,4]:", slow.Val) // 2
}
```

**Textual Figure:**
```
Find middle using slow (1 step) and fast (2 steps):

Odd list [1, 2, 3, 4, 5]:
  Init:   S              F
          │              │
         [1] → [2] → [3] → [4] → [5] → nil

  Step 1:      S                   F
               │                   │
         [1] → [2] → [3] → [4] → [5] → nil

  Step 2:           S                        F
                    │                        │
         [1] → [2] → [3] → [4] → [5] → nil
                          fast.Next=nil → STOP
  Result: slow = 3 (middle) ✓

Even list [1, 2, 3, 4, 5, 6]:
  Init:   S              F
         [1] → [2] → [3] → [4] → [5] → [6] → nil

  Step 1:      S                   F
         [1] → [2] → [3] → [4] → [5] → [6] → nil

  Step 2:           S                        F
         [1] → [2] → [3] → [4] → [5] → [6] → nil

  Step 3:                S                        F
         [1] → [2] → [3] → [4] → [5] → [6] → nil
                              fast=nil → STOP
  Result: slow = 4 (second middle) ✓

First middle of [1,2,3,4]:
  Stop when fast.Next.Next=nil → slow = 2 ✓
```

---

## Example 2: Detect Cycle (Floyd's Algorithm)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func hasCycle(head *Node) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}

func main() {
    // No cycle
    head := &Node{Val: 1, Next: &Node{Val: 2, Next: &Node{Val: 3}}}
    fmt.Println("Has cycle:", hasCycle(head)) // false

    // With cycle: 1 → 2 → 3 → 4 → 2
    n1 := &Node{Val: 1}
    n2 := &Node{Val: 2}
    n3 := &Node{Val: 3}
    n4 := &Node{Val: 4}
    n1.Next = n2
    n2.Next = n3
    n3.Next = n4
    n4.Next = n2 // cycle back to n2
    fmt.Println("Has cycle:", hasCycle(n1)) // true
}
```

**Textual Figure:**
```
Floyd's Cycle Detection:

No cycle: 1 → 2 → 3 → nil
  S,F start at 1
  Step 1: S=2, F=3
  Step 2: F.Next=nil → STOP → no cycle ✓

With cycle: 1 → 2 → 3 → 4 → back to 2
  ┌──────────────────────┐
  │                      │
 [1] ─→ [2] ─→ [3] ─→ [4]
         ↑                 │
         └───────────────┘

  Step 0: S=1, F=1
  Step 1: S=2, F=3   (slow +1, fast +2)
  Step 2: S=3, F=2   (fast: 3→4→2)
  Step 3: S=4, F=4   (slow: 3→4, fast: 2→3→4)
          S == F → CYCLE DETECTED ✓
```

---

## Example 3: Find Cycle Start Node

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func detectCycleStart(head *Node) *Node {
    slow, fast := head, head

    // Phase 1: Detect cycle
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Phase 2: Find entry point
            // Move one pointer to head, both move at speed 1
            slow = head
            for slow != fast {
                slow = slow.Next
                fast = fast.Next
            }
            return slow // this is the cycle start
        }
    }
    return nil // no cycle
}

func main() {
    // 1 → 2 → 3 → 4 → 5 → 3 (cycle at node 3)
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[2] // cycle back to node 3

    start := detectCycleStart(nodes[0])
    if start != nil {
        fmt.Println("Cycle starts at node:", start.Val) // 3
    }

    // Why does this work?
    // Let distance from head to cycle start = a
    // Let distance from cycle start to meeting point = b
    // Let cycle length = c
    // At meeting: slow traveled a+b, fast traveled a+b+c
    // 2(a+b) = a+b+c → a+b = c → a = c-b
    // So moving from head by 'a' steps = moving from meeting by 'c-b' steps
    // Both end up at cycle start!
}
```

**Textual Figure:**
```
Find Cycle Start Node (Floyd's two-phase):

List: 1 → 2 → 3 → 4 → 5 → back to 3
              a=2         b
  ┌───────────────────┐
  │                   │
 [1] ─→ [2] ─→ [3] ─→ [4] ─→ [5]
                ↑                 │
                └───────────────┘
                cycle start    c=3 (cycle length)

Phase 1: Detect meeting point
  S=1,F=1 → S=2,F=3 → S=3,F=5 → S=4,F=4
  Meet at node 4!

Phase 2: Find cycle start
  Reset slow to head, both move at speed 1:
  S=1,F=4 → S=2,F=5 → S=3,F=3
  Meet at node 3 = cycle start! ✓

Why it works:
  slow traveled: a + b = 2 + 1 = 3
  fast traveled: a + b + c = 2 + 1 + 3 = 6 = 2×3
  a = c - b = 3 - 1 = 2 steps from meeting point to cycle start
  2 steps from head also reaches cycle start!
```

---

## Example 4: Find Cycle Length

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func cycleLength(head *Node) int {
    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Count cycle length
            count := 1
            curr := slow.Next
            for curr != slow {
                count++
                curr = curr.Next
            }
            return count
        }
    }
    return 0 // no cycle
}

func main() {
    // 1 → 2 → 3 → 4 → 5 → 3 (cycle length = 3: 3→4→5→3)
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[2]

    fmt.Println("Cycle length:", cycleLength(nodes[0])) // 3
}
```

**Textual Figure:**
```
Find Cycle Length:

List: 1 → 2 → 3 → 4 → 5 → back to 3
  ┌───────────────────┐
  │                   │
 [1] ─→ [2] ─→ [3] ─→ [4] ─→ [5]
                ↑                 │
                └───────────────┘

Step 1: Detect meeting point (Floyd's)
  S and F meet somewhere in cycle

Step 2: Count cycle length from meeting point
  Start at meeting point, walk until we return:
  meeting → +1 → +1 → back to meeting
        ┌─────────────┐
        ↓             │
       [3] ─→ [4] ─→ [5]
  count: 1      2      3 → back to 3, stop!

  Cycle length = 3 (nodes: 3, 4, 5) ✓
```

---

## Example 5: Check if Linked List is Palindrome

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

    // Step 1: Find middle using slow/fast
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // Step 2: Reverse second half
    secondHalf := reverseList(slow.Next)

    // Step 3: Compare
    first, second := head, secondHalf
    for second != nil {
        if first.Val != second.Val {
            return false
        }
        first = first.Next
        second = second.Next
    }
    return true
}

func reverseList(head *Node) *Node {
    var prev *Node
    for head != nil {
        next := head.Next
        head.Next = prev
        prev = head
        head = next
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
        {1},
    }
    for _, t := range tests {
        fmt.Printf("%v → palindrome=%v\n", t, isPalindrome(fromSlice(t)))
    }
}
```

**Textual Figure:**
```
Check Palindrome: [1, 2, 3, 2, 1]

Step 1: Find middle with slow/fast
  [1] → [2] → [3] → [2] → [1] → nil
   S         F
        S              F
             S                   F(nil)
  slow stops at 3 (middle)

Step 2: Reverse second half (after slow)
  Before: slow.Next = [2] → [1] → nil
  After:  reversed   = [1] → [2] → nil

  ┌───┐  ┌───┐  ┌───┐      ┌───┐  ┌───┐
  │ 1 │→ │ 2 │→ │ 3 │      │ 1 │→ │ 2 │→ nil
  └───┘  └───┘  └───┘      └───┘  └───┘
  first half              reversed 2nd half

Step 3: Compare
  first=1, second=1  ✓
  first=2, second=2  ✓
  second=nil → done
  Result: TRUE (palindrome) ✓

[1, 2, 3]: first=1 vs second=3 ✗ → FALSE
```

---

## Example 6: Find Nth Node from End

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Use two pointers separated by n nodes
func nthFromEnd(head *Node, n int) *Node {
    fast := head
    for i := 0; i < n; i++ {
        if fast == nil {
            return nil
        }
        fast = fast.Next
    }

    slow := head
    for fast != nil {
        slow = slow.Next
        fast = fast.Next
    }
    return slow
}

// Remove nth from end (LeetCode 19)
func removeNthFromEnd(head *Node, n int) *Node {
    dummy := &Node{Next: head}
    fast := dummy
    for i := 0; i <= n; i++ {
        fast = fast.Next
    }

    slow := dummy
    for fast != nil {
        slow = slow.Next
        fast = fast.Next
    }
    slow.Next = slow.Next.Next
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

    fmt.Println("1st from end:", nthFromEnd(head, 1).Val) // 5
    fmt.Println("2nd from end:", nthFromEnd(head, 2).Val) // 4
    fmt.Println("5th from end:", nthFromEnd(head, 5).Val) // 1

    head2 := fromSlice([]int{1, 2, 3, 4, 5})
    head2 = removeNthFromEnd(head2, 2) // remove 4
    fmt.Print("After removing 2nd from end: ")
    printList(head2)
}
```

**Textual Figure:**
```
Find Nth from end — gap technique:

List: [1] → [2] → [3] → [4] → [5] → nil

nthFromEnd(head, 2):
  Step 1: Advance fast by n=2 steps
     slow          fast
      │              │
     [1] → [2] → [3] → [4] → [5] → nil

  Step 2: Move both until fast=nil
           slow          fast
            │              │
     [1] → [2] → [3] → [4] → [5] → nil
                  slow          fast
                   │              │
     [1] → [2] → [3] → [4] → [5] → nil
                        slow          fast(nil)
                         │
     [1] → [2] → [3] → [4] → [5] → nil
  Result: slow = 4 (2nd from end) ✓

removeNthFromEnd(head, 2) — remove node 4:
  Use dummy node, advance fast by n+1=3:
     [D] → [1] → [2] → [3] → [4] → [5] → nil
      S                   F
  Move both until fast=nil:
                   S              F(nil)
     [D] → [1] → [2] → [3] ────→ [5] → nil
                         slow.Next = slow.Next.Next (skip 4)
  Result: 1 → 2 → 3 → 5 → nil ✓
```

---

## Example 7: Happy Number (Fast-Slow on Sequence)

```go
package main

import "fmt"

func digitSquareSum(n int) int {
    sum := 0
    for n > 0 {
        d := n % 10
        sum += d * d
        n /= 10
    }
    return sum
}

// Using Floyd's cycle detection on the sequence of digit-square-sums
func isHappy(n int) bool {
    slow := n
    fast := n
    for {
        slow = digitSquareSum(slow)
        fast = digitSquareSum(digitSquareSum(fast))
        if slow == fast {
            break
        }
    }
    return slow == 1
}

func main() {
    for i := 1; i <= 30; i++ {
        if isHappy(i) {
            fmt.Printf("%d is happy\n", i)
        }
    }

    // Trace: 19 → 82 → 68 → 100 → 1 ✓
    fmt.Println("\n19 is happy:", isHappy(19))
    fmt.Println("2 is happy:", isHappy(2))
}
```

**Textual Figure:**
```
Happy Number — Floyd's on digit-square-sum sequence:

19 is happy? Trace the sequence:
  19 → 1²+9² = 82
  82 → 8²+2² = 68
  68 → 6²+8² = 100
  100 → 1²+0²+0² = 1  ✓ Happy!

  Sequence: 19 → 82 → 68 → 100 → 1 → 1 → 1 ...
  slow and fast both reach 1 → return true

2 is happy? Trace:
  2 → 4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4 ...
  ┌─────────────────────────────────────────────┐
  ↓                                             │
  4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 ──┘
  Cycle detected but not at 1 → return false (not happy)

  Floyd's detects cycle in the sequence without using a hash set!
```

---

## Example 8: Find Intersection of Two Lists

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func getIntersectionNode(headA, headB *Node) *Node {
    if headA == nil || headB == nil {
        return nil
    }

    a, b := headA, headB
    // When a reaches end, redirect to headB
    // When b reaches end, redirect to headA
    // They meet at intersection or both become nil
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
    // Shared: 8 → 10
    shared := &Node{Val: 8, Next: &Node{Val: 10}}

    // List A: 1 → 3 → 8 → 10
    headA := &Node{Val: 1, Next: &Node{Val: 3, Next: shared}}

    // List B: 2 → 4 → 6 → 8 → 10
    headB := &Node{Val: 2, Next: &Node{Val: 4, Next: &Node{Val: 6, Next: shared}}}

    intersection := getIntersectionNode(headA, headB)
    if intersection != nil {
        fmt.Println("Intersection at:", intersection.Val) // 8
    }

    // No intersection
    c := &Node{Val: 1, Next: &Node{Val: 2}}
    d := &Node{Val: 3, Next: &Node{Val: 4}}
    fmt.Println("Intersection:", getIntersectionNode(c, d)) // nil
}
```

**Textual Figure:**
```
Find Intersection of Two Lists:

List A: 1 → 3 ─┐
                ├─→ [8] → [10] → nil  (shared tail)
List B: 2 → 4 → 6 ─┘

Two-pointer technique (redirect to other list's head at end):
  a starts at A's head, b starts at B's head

  a: 1 → 3 → 8 → 10 → nil → [switch to B] 2 → 4 → 6 → 8
  b: 2 → 4 → 6 → 8 → 10 → nil → [switch to A] 1 → 3 → 8
                                                           ↑
                                              a == b at node 8! ✓

Why it works:
  a traverses: len(A) + len(B shared prefix) = 2 + 2 + 3 + 2 = equal
  b traverses: len(B) + len(A shared prefix) = 3 + 2 + 2 + 2 = equal
  Both travel same total distance → meet at intersection!

No intersection: both become nil at the same time → return nil
```

---

## Example 9: Reorder List (L0→Ln→L1→Ln-1→...)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func reorderList(head *Node) {
    if head == nil || head.Next == nil {
        return
    }

    // Step 1: Find middle with slow/fast
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // Step 2: Reverse second half
    second := reverseList(slow.Next)
    slow.Next = nil

    // Step 3: Merge alternately
    first := head
    for second != nil {
        tmp1 := first.Next
        tmp2 := second.Next
        first.Next = second
        second.Next = tmp1
        first = tmp1
        second = tmp2
    }
}

func reverseList(head *Node) *Node {
    var prev *Node
    for head != nil {
        next := head.Next
        head.Next = prev
        prev = head
        head = next
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
    fmt.Print("Before: ")
    printList(head)
    reorderList(head)
    fmt.Print("After:  ")
    printList(head)
    // 1 → 5 → 2 → 4 → 3 → nil

    head2 := fromSlice([]int{1, 2, 3, 4})
    fmt.Print("\nBefore: ")
    printList(head2)
    reorderList(head2)
    fmt.Print("After:  ")
    printList(head2)
    // 1 → 4 → 2 → 3 → nil
}
```

**Textual Figure:**
```
Reorder List: L0→Ln→L1→Ln-1→...

Input: [1] → [2] → [3] → [4] → [5] → nil

Step 1: Find middle with slow/fast
  slow stops at node 3
  [1] → [2] → [3] | [4] → [5]

Step 2: Reverse second half
  [4] → [5]  becomes  [5] → [4] → nil
  Cut: [3].Next = nil

  First:  [1] → [2] → [3] → nil
  Second: [5] → [4] → nil

Step 3: Merge alternately
  Take from first, then second:
  [1] → [5] → [2] → [4] → [3] → nil
   ↑     ↑     ↑     ↑     ↑
  1st   2nd   1st   2nd   1st

Result: 1 → 5 → 2 → 4 → 3 → nil ✓
```

---

## Example 10: Sort Linked List (Merge Sort with Slow/Fast Split)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func sortList(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }

    // Split using slow/fast
    slow, fast := head, head.Next
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    mid := slow.Next
    slow.Next = nil

    left := sortList(head)
    right := sortList(mid)
    return mergeLists(left, right)
}

func mergeLists(a, b *Node) *Node {
    dummy := &Node{}
    curr := dummy
    for a != nil && b != nil {
        if a.Val <= b.Val {
            curr.Next = a
            a = a.Next
        } else {
            curr.Next = b
            b = b.Next
        }
        curr = curr.Next
    }
    if a != nil {
        curr.Next = a
    } else {
        curr.Next = b
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
    head := fromSlice([]int{4, 2, 1, 3, 5})
    fmt.Print("Before sort: ")
    printList(head)

    head = sortList(head)
    fmt.Print("After sort:  ")
    printList(head)
    // 1 → 2 → 3 → 4 → 5 → nil
}
```

**Textual Figure:**
```
Merge Sort on Linked List using slow/fast split:

Input: [4] → [2] → [1] → [3] → [5] → nil

Recursive split (slow/fast finds middle):
                    sortList
                   /        \
         [4]→[2]→[1]        [3]→[5]
            /    \           /    \
      [4]→[2]   [1]      [3]    [5]
       /    \
     [4]   [2]

Merge back up:
     merge([4],[2]) → [2]→[4]
     merge([2]→[4], [1]) → [1]→[2]→[4]
     merge([3],[5]) → [3]→[5]
     merge([1]→[2]→[4], [3]→[5]):
       compare 1<3: take 1
       compare 2<3: take 2
       compare 4>3: take 3
       compare 4<5: take 4
       take 5

Result:
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │→ │ 2 │→ │ 3 │→ │ 4 │→ │ 5 │→ nil
  └───┘  └───┘  └───┘  └───┘  └───┘

Output: 1 → 2 → 3 → 4 → 5 → nil
```

---

## Example 11: Linked List Cycle II — Complete Proof

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func detectCycle(head *Node) (*Node, int) {
    slow, fast := head, head
    hasCycle := false

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            hasCycle = true
            break
        }
    }

    if !hasCycle {
        return nil, 0
    }

    // Measure cycle length
    cycleLen := 1
    curr := slow.Next
    for curr != slow {
        cycleLen++
        curr = curr.Next
    }

    // Find cycle start
    slow = head
    for slow != fast {
        slow = slow.Next
        fast = fast.Next
    }

    return slow, cycleLen
}

func main() {
    // Build: 1 → 2 → 3 → 4 → 5 → 6 → 3
    nodes := make([]*Node, 6)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 5; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[5].Next = nodes[2] // cycle back to node 3

    start, length := detectCycle(nodes[0])
    fmt.Printf("Cycle start: node %d\n", start.Val)     // 3
    fmt.Printf("Cycle length: %d\n", length)              // 4 (3→4→5→6→3)

    /*
    Proof:
    - Let a = distance from head to cycle start
    - Let b = distance from cycle start to meeting point
    - Let c = cycle length
    
    When they meet:
    - slow traveled: a + b
    - fast traveled: a + b + nc (for some n ≥ 1)
    - fast = 2 × slow: 2(a+b) = a + b + nc
    - Therefore: a + b = nc → a = nc - b = (n-1)c + (c-b)
    
    Starting from meeting point, (c-b) steps brings you to cycle start.
    Starting from head, a steps also brings you to cycle start.
    Since a = (n-1)c + (c-b), both pointers meet at cycle start!
    */
}
```

**Textual Figure:**
```
Cycle II — Complete proof with concrete example:

List: 1 → 2 → 3 → 4 → 5 → 6 → back to 3
  a=2 (head to cycle start)
              ┌─────────────────────────┐
              │                         │
 [1] → [2] → [3] → [4] → [5] → [6] ──┘
              ↑ cycle                c=4
              start

Phase 1: Floyd's detection
  S=1,F=1 → S=2,F=3 → S=3,F=5 → S=4,F=3
  → S=5,F=5  → MEET at node 5

Measure cycle length from meeting point:
  5 → 6 → 3 → 4 → 5  → count = 4

Phase 2: Find cycle start
  Reset slow to head:
  S=1,F=5 → S=2,F=6 → S=3,F=3  → MEET at node 3
  Cycle start = node 3 ✓

Proof:
  a = distance head → cycle start = 2
  b = distance cycle start → meeting = 2  (3→4→5)
  c = cycle length = 4
  2(a+b) = a+b+nc  →  a+b = nc  →  a = nc-b
  a = 4-2 = 2  ✓  (2 steps from head = 2 steps from meeting in cycle)
```

---

## Fast-Slow Pointer Applications

| Problem | Technique | Time | Space |
|---------|-----------|------|-------|
| Find middle | slow=1x, fast=2x | O(n) | O(1) |
| Detect cycle | Floyd's detection | O(n) | O(1) |
| Find cycle start | Two-phase Floyd's | O(n) | O(1) |
| Palindrome check | Middle + reverse | O(n) | O(1) |
| Nth from end | Gap of n | O(n) | O(1) |
| Reorder list | Middle + reverse + merge | O(n) | O(1) |
| Sort list | Middle split (merge sort) | O(n log n) | O(log n) |
| Happy number | Floyd's on sequence | O(log n) | O(1) |

## Key Takeaways

1. **Two speeds**: slow (1 step) and fast (2 steps) — guaranteed to meet in a cycle
2. **Middle finding**: when fast reaches end, slow is at middle — O(n) time, O(1) space
3. **Cycle detection**: Floyd's algorithm — no hash set needed
4. **Cycle start**: reset one pointer to head, both move at speed 1 — mathematical proof
5. **Palindrome**: find middle → reverse second half → compare
6. **Gap technique**: for nth-from-end, separate two pointers by n steps
7. **Works beyond lists**: Happy Number applies Floyd's to any repeating sequence

> **Next up:** Dummy Nodes →
