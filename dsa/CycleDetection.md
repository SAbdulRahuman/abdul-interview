# Phase 5: Linked Lists — Cycle Detection

## Overview

**Cycle detection** determines whether a linked list contains a loop. The most famous algorithm is **Floyd's Tortoise and Hare** — O(n) time, O(1) space.

**Why it matters:**
- Infinite loops in traversal
- Memory leaks from unreachable nodes
- Classic interview topic

---

## Example 1: Floyd's Cycle Detection

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
    a := &Node{Val: 1, Next: &Node{Val: 2, Next: &Node{Val: 3}}}
    fmt.Println("No cycle:", hasCycle(a)) // false

    // With cycle
    n1 := &Node{Val: 1}
    n2 := &Node{Val: 2}
    n3 := &Node{Val: 3}
    n1.Next = n2
    n2.Next = n3
    n3.Next = n1
    fmt.Println("Cycle:", hasCycle(n1)) // true

    // Self-loop
    s := &Node{Val: 1}
    s.Next = s
    fmt.Println("Self-loop:", hasCycle(s)) // true

    // nil
    fmt.Println("nil:", hasCycle(nil)) // false
}
```

**Textual Figure:**
```
Floyd's Cycle Detection (slow moves 1 step, fast moves 2 steps):

Case 1 — No cycle:
  ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→ nil
  └───┘   └───┘   └───┘
  fast reaches nil → return false

Case 2 — Cycle (1→2→3→1):
  ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │
  └───┘   └───┘   └───┘
    ↑                   │
    └───────────────┘

  Step 1: slow=2, fast=3      (not equal)
  Step 2: slow=3, fast=2      (not equal)
  Step 3: slow=1, fast=1      → EQUAL! return true

Case 3 — Self-loop:
  ┌───┐
  │ 1 │─┐
  └───┘ │
    ↑   │
    └───┘
  Step 1: slow=1, fast=1 → EQUAL! return true
```

---

## Example 2: Find Cycle Entry Point

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func detectCycleStart(head *Node) *Node {
    slow, fast := head, head

    // Phase 1: Find meeting point
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Phase 2: Find entry point
            entry := head
            for entry != slow {
                entry = entry.Next
                slow = slow.Next
            }
            return entry
        }
    }
    return nil
}

func main() {
    // 1 → 2 → 3 → 4 → 5 → 3 (cycle at 3)
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[2] // cycle back to 3

    entry := detectCycleStart(nodes[0])
    fmt.Printf("Cycle entry: %d\n", entry.Val) // 3

    // No cycle
    linear := &Node{Val: 1, Next: &Node{Val: 2}}
    fmt.Println("No cycle:", detectCycleStart(linear)) // nil
}
```

**Textual Figure:**
```
Find cycle entry in: 1 → 2 → 3 → 4 → 5 → 3 (cycle at node 3)

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │
  └───┘   └───┘   └───┘   └───┘   └───┘
                    ↑                   │
                    └───────────────┘
                    tail=2    cycle=3

Phase 1 — Find meeting point (slow +1, fast +2):
  Step 1: slow=2, fast=3
  Step 2: slow=3, fast=5
  Step 3: slow=4, fast=4  → MEET at node 4!

Phase 2 — Find entry (entry from head, slow from meeting):
  entry=1, slow=4
  Step 1: entry=2, slow=5
  Step 2: entry=3, slow=3  → MEET at node 3!

  Entry point = node 3 ✓
```

---

## Example 3: Find Cycle Length

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
            // Count nodes in cycle
            count := 1
            runner := slow.Next
            for runner != slow {
                count++
                runner = runner.Next
            }
            return count
        }
    }
    return 0
}

func main() {
    // Cycle of length 3: 3 → 4 → 5 → 3
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[2]

    fmt.Println("Cycle length:", cycleLength(nodes[0])) // 3

    // Cycle of length 5 (full cycle)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[0]
    fmt.Println("Full cycle length:", cycleLength(nodes[0])) // 5
}
```

**Textual Figure:**
```
Find cycle length in: 1 → 2 → 3 → 4 → 5 → 3 (cycle at 3)

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │
  └───┘   └───┘   └───┘   └───┘   └───┘
                    ↑                   │
                    └───────────────┘

Phase 1 — Detect meeting point with Floyd's:
  slow and fast meet at some node inside the cycle

Phase 2 — Count cycle length:
  Start runner at slow.Next, count until runner == slow

  runner = slow.Next
  ┌───────────────────────┐
  │   ┌───┐   ┌───┐   ┌───┐ │
  └─→ │ 3 │──→│ 4 │──→│ 5 │─┘
      └───┘   └───┘   └───┘
      count=1  count=2  count=3
                        runner==slow → stop!

  Cycle length = 3
```

---

## Example 4: Hash Set Based Detection

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// O(n) time, O(n) space — simpler but more memory
func hasCycleHash(head *Node) (*Node, bool) {
    visited := make(map[*Node]bool)
    curr := head
    for curr != nil {
        if visited[curr] {
            return curr, true // returns cycle entry directly!
        }
        visited[curr] = true
        curr = curr.Next
    }
    return nil, false
}

func main() {
    n1 := &Node{Val: 1}
    n2 := &Node{Val: 2}
    n3 := &Node{Val: 3}
    n4 := &Node{Val: 4}
    n1.Next = n2
    n2.Next = n3
    n3.Next = n4
    n4.Next = n2

    entry, hasCyc := hasCycleHash(n1)
    fmt.Printf("Has cycle: %v, entry: %d\n", hasCyc, entry.Val)

    // Comparison:
    // Floyd's: O(n) time, O(1) space
    // HashSet: O(n) time, O(n) space — but finds entry in one phase
}
```

**Textual Figure:**
```
Hash Set cycle detection: 1 → 2 → 3 → 4 → 2 (cycle at 2)

  ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │
  └───┘   └───┘   └───┘   └───┘
              ↑                   │
              └───────────────┘

  visited = {}

  Visit 1: not in set → add    visited = {1}
  Visit 2: not in set → add    visited = {1, 2}
  Visit 3: not in set → add    visited = {1, 2, 3}
  Visit 4: not in set → add    visited = {1, 2, 3, 4}
  Visit 2: IN SET! → return (node 2, true)
                       ↑
                   cycle entry found directly!

  Comparison:
  ┌─────────────┬──────────┬──────────┐
  │ Algorithm   │ Time     │ Space    │
  ├─────────────┼──────────┼──────────┤
  │ Floyd's     │ O(n)     │ O(1)     │
  │ Hash Set    │ O(n)     │ O(n)     │
  └─────────────┴──────────┴──────────┘
```

---

## Example 5: Brent's Algorithm (Faster Cycle Detection)

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

// Brent's algorithm: moves fast pointer in powers of 2
// Typically faster in practice than Floyd's
func brentCycleDetection(head *Node) (int, *Node) {
    if head == nil {
        return 0, nil
    }

    // Phase 1: Find cycle length
    slow := head
    fast := head.Next
    power := 1
    length := 1

    if fast == nil {
        return 0, nil
    }

    for slow != fast {
        if length == power {
            slow = fast
            power *= 2
            length = 0
        }
        fast = fast.Next
        if fast == nil {
            return 0, nil
        }
        length++
    }

    // Phase 2: Find cycle start
    // Put slow at head, fast at head + length
    slow = head
    fast = head
    for i := 0; i < length; i++ {
        fast = fast.Next
    }
    for slow != fast {
        slow = slow.Next
        fast = fast.Next
    }

    return length, slow
}

func main() {
    // 1 → 2 → 3 → 4 → 5 → 6 → 3
    nodes := make([]*Node, 6)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 5; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[5].Next = nodes[2]

    length, entry := brentCycleDetection(nodes[0])
    fmt.Printf("Brent's: cycle length=%d, entry=%d\n", length, entry.Val)
}
```

**Textual Figure:**
```
Brent's Algorithm on: 1 → 2 → 3 → 4 → 5 → 6 → 3

  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 5 │─→│ 6 │
  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
                  ↑                        │
                  └────────────────────┘

Phase 1 — Find cycle length (power-of-2 teleportation):
  power=1: slow=1, fast=2, length=1
    slow≠fast → length==power → teleport slow=fast(2), power=2, length=0
  power=2: fast=3(len=1), fast=4(len=2)
    length==power → teleport slow=fast(4), power=4, length=0
  power=4: fast=5(1), fast=6(2), fast=3(3), fast=4(4)
    slow==fast at node 4! → cycle length = 4

Phase 2 — Find entry:
  slow=head(1), fast=head+4 steps = node 5
  Move both:
    slow=2, fast=6
    slow=3, fast=3  → MEET! Entry = node 3

  Result: cycle length=4, entry=3
```

---

## Example 6: Detect and Remove Cycle

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func removeCycle(head *Node) bool {
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
        return false
    }

    // Find cycle entry
    slow = head
    if slow == fast {
        // Special case: cycle starts at head
        for fast.Next != slow {
            fast = fast.Next
        }
    } else {
        for slow.Next != fast.Next {
            slow = slow.Next
            fast = fast.Next
        }
    }
    fast.Next = nil // break the cycle
    return true
}

func printList(head *Node) {
    count := 0
    for curr := head; curr != nil && count < 20; curr = curr.Next {
        fmt.Printf("%d → ", curr.Val)
        count++
    }
    if count >= 20 {
        fmt.Println("... (cycle?)")
    } else {
        fmt.Println("nil")
    }
}

func main() {
    // Build list with cycle
    nodes := make([]*Node, 5)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 4; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[4].Next = nodes[2] // cycle at 3

    fmt.Print("Before: ")
    printList(nodes[0])

    removed := removeCycle(nodes[0])
    fmt.Printf("Cycle removed: %v\n", removed)

    fmt.Print("After: ")
    printList(nodes[0])
}
```

**Textual Figure:**
```
Detect and remove cycle: 1 → 2 → 3 → 4 → 5 → 3

Before:
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │
  └───┘   └───┘   └───┘   └───┘   └───┘
                    ↑                   │
                    └───────────────┘

Step 1 — Floyd's detects cycle (slow meets fast)
Step 2 — Find entry point = node 3
Step 3 — Find last node before entry:
  Walk fast until fast.Next == entry(3)
  fast stops at node 5 (5.Next == 3)

Step 4 — Break the cycle: fast.Next = nil
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→ nil
  └───┘   └───┘   └───┘   └───┘   └───┘
                                        ╳ cycle broken

After: 1 → 2 → 3 → 4 → 5 → nil
```

---

## Example 7: Find Duplicate Number (Array as Linked List)

```go
package main

import "fmt"

// LeetCode 287: Find the Duplicate Number
// Array of n+1 integers in range [1, n] with exactly one duplicate
// Treat array indices as next pointers → cycle detection!
func findDuplicate(nums []int) int {
    slow := nums[0]
    fast := nums[nums[0]]

    // Phase 1: Find meeting point
    for slow != fast {
        slow = nums[slow]
        fast = nums[nums[fast]]
    }

    // Phase 2: Find cycle entry = duplicate number
    slow = 0
    for slow != fast {
        slow = nums[slow]
        fast = nums[fast]
    }

    return slow
}

func main() {
    tests := [][]int{
        {1, 3, 4, 2, 2},
        {3, 1, 3, 4, 2},
        {2, 5, 9, 6, 9, 3, 8, 9, 7, 1},
    }

    for _, nums := range tests {
        fmt.Printf("nums=%v → duplicate=%d\n", nums, findDuplicate(nums))
    }

    // Why cycle detection works:
    // nums = [1, 3, 4, 2, 2]
    // index: 0 → 1 → 3 → 2 → 4 → 2 → 4 → 2 ...
    // The cycle exists because two indices map to the same value (duplicate)
}
```

**Textual Figure:**
```
Find duplicate in [1, 3, 4, 2, 2] using cycle detection:

Treat array as implicit linked list (index → value = next index):
  index: 0 → nums[0]=1 → nums[1]=3 → nums[3]=2 → nums[2]=4 → nums[4]=2 ...

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 0 │──→│ 1 │──→│ 3 │──→│ 2 │──→│ 4 │
  └───┘   └───┘   └───┘   └───┘   └───┘
                              ↑           │
                              └─────────┘
                           cycle entry = 2 = duplicate!

Phase 1 — slow=nums[0]=1, fast=nums[nums[0]]=3
  slow=nums[1]=3, fast=nums[nums[3]]=nums[2]=4
  slow=nums[3]=2, fast=nums[nums[4]]=nums[2]=4
  slow=nums[2]=4, fast=nums[nums[4]]=nums[2]=4  → MEET!

Phase 2 — slow=0, fast=4
  slow=nums[0]=1, fast=nums[4]=2
  slow=nums[1]=3, fast=nums[2]=4
  slow=nums[3]=2, fast=nums[4]=2  → MEET! duplicate = 2
```

---

## Example 8: Cycle Detection in Directed Graph

```go
package main

import "fmt"

// Functional graph: each node has exactly one outgoing edge
// Represented as next[i] = successor of i
func detectCycleInGraph(next []int, start int) (int, int) {
    // Floyd's on functional graph
    slow := start
    fast := start

    for {
        slow = next[slow]
        fast = next[next[fast]]
        if slow == fast {
            break
        }
    }

    // Find entry
    entry := start
    for entry != slow {
        entry = next[entry]
        slow = next[slow]
    }

    // Find length
    length := 1
    runner := next[entry]
    for runner != entry {
        length++
        runner = next[runner]
    }

    return entry, length
}

func main() {
    // Graph: 0→1→2→3→4→5→2 (cycle at node 2, length 4)
    next := []int{1, 2, 3, 4, 5, 2}

    entry, length := detectCycleInGraph(next, 0)
    fmt.Printf("Cycle entry: %d, length: %d\n", entry, length)
}
```

**Textual Figure:**
```
Cycle detection in directed functional graph:
  next[] = {1, 2, 3, 4, 5, 2}

  Graph: 0 → 1 → 2 → 3 → 4 → 5 → 2 (cycle)

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 0 │──→│ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
  tail ────────┤ ↑                           │
                  └─────────────────────┘
                  ├───── cycle (len=4) ────┤

  Floyd's on array:
    slow = next[slow], fast = next[next[fast]]
    They meet inside cycle → find entry = 2
    Count from entry back to entry → length = 4

  Result: entry=2, length=4
```

---

## Example 9: All Nodes in Cycle

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

func nodesInCycle(head *Node) []*Node {
    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Collect all nodes in cycle
            var result []*Node
            result = append(result, slow)
            curr := slow.Next
            for curr != slow {
                result = append(result, curr)
                curr = curr.Next
            }
            return result
        }
    }
    return nil
}

func main() {
    nodes := make([]*Node, 6)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 5; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[5].Next = nodes[2] // cycle: 3→4→5→6→3

    cycleNodes := nodesInCycle(nodes[0])
    fmt.Print("Nodes in cycle: ")
    for _, n := range cycleNodes {
        fmt.Printf("%d ", n.Val)
    }
    fmt.Println()
}
```

**Textual Figure:**
```
Collect all nodes in cycle: 1 → 2 → 3 → 4 → 5 → 6 → 3

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→│ 6 │
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
  (tail)          ↑                           │
                  └─────────────────────┘

Step 1: Floyd's finds meeting point (some node in cycle)

Step 2: Collect cycle nodes from meeting point:
  result = [slow]
  curr = slow.Next
  Walk until curr == slow:

  ┌───────────────────────────┐
  │   ┌───┐   ┌───┐   ┌───┐   ┌───┐ │
  └─→ │ 3 │──→│ 4 │──→│ 5 │──→│ 6 │─┘
      └───┘   └───┘   └───┘   └───┘
       add      add      add      add

  Nodes in cycle: [3, 4, 5, 6] (or starting from meeting point)
```

---

## Example 10: Complete Cycle Analysis

```go
package main

import "fmt"

type Node struct {
    Val  int
    Next *Node
}

type CycleInfo struct {
    HasCycle    bool
    EntryNode   *Node
    CycleLength int
    TailLength  int // distance from head to cycle entry
}

func analyzeCycle(head *Node) CycleInfo {
    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // Find cycle length
            cycleLen := 1
            runner := slow.Next
            for runner != slow {
                cycleLen++
                runner = runner.Next
            }

            // Find entry
            entry := head
            tailLen := 0
            for entry != slow {
                entry = entry.Next
                slow = slow.Next
                tailLen++
            }

            return CycleInfo{
                HasCycle:    true,
                EntryNode:   entry,
                CycleLength: cycleLen,
                TailLength:  tailLen,
            }
        }
    }

    return CycleInfo{HasCycle: false}
}

func main() {
    // Tail=2, cycle=4: 1→2→3→4→5→6→3
    nodes := make([]*Node, 6)
    for i := range nodes {
        nodes[i] = &Node{Val: i + 1}
    }
    for i := 0; i < 5; i++ {
        nodes[i].Next = nodes[i+1]
    }
    nodes[5].Next = nodes[2]

    info := analyzeCycle(nodes[0])
    fmt.Printf("Has cycle: %v\n", info.HasCycle)
    fmt.Printf("Entry node: %d\n", info.EntryNode.Val)
    fmt.Printf("Cycle length: %d\n", info.CycleLength)
    fmt.Printf("Tail length: %d\n", info.TailLength)
    fmt.Printf("Total nodes: %d\n", info.CycleLength+info.TailLength)
}
```

**Textual Figure:**
```
Complete cycle analysis: 1 → 2 → 3 → 4 → 5 → 6 → 3

  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │──→│ 2 │──→│ 3 │──→│ 4 │──→│ 5 │──→│ 6 │
  └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
                    ↑                           │
                    └─────────────────────┘
  ├─ tail=2 ─┤├────── cycle=4 ──────┤

  Analysis steps:
    1. Floyd's: slow & fast meet inside cycle
    2. Cycle length: count steps from meeting → back to meeting = 4
    3. Entry point: reset one ptr to head, advance both by 1
       → they meet at node 3
    4. Tail length: count steps head → entry = 2

  Result:
  ┌─────────────────┬────────┐
  │ Property        │ Value  │
  ├─────────────────┼────────┤
  │ Has cycle       │ true   │
  │ Entry node      │ 3      │
  │ Cycle length    │ 4      │
  │ Tail length     │ 2      │
  │ Total nodes     │ 6      │
  └─────────────────┴────────┘
```

---

## Cycle Detection Comparison

| Algorithm | Time | Space | Finds Entry | Finds Length |
|-----------|------|-------|-------------|-------------|
| Floyd's | O(n) | O(1) | Two phases | Extra traversal |
| Brent's | O(n) | O(1) | Two phases | Built-in |
| Hash Set | O(n) | O(n) | Directly | Extra traversal |
| Marking | O(n) | O(1)* | Yes | Yes |

*Requires modifying nodes.

## Key Takeaways

1. **Floyd's algorithm**: two pointers at different speeds — O(n) time, O(1) space
2. **Meeting point math**: `a = c - b` where a=tail, b=meeting offset, c=cycle length
3. **Entry point**: after meeting, reset one pointer to head → they meet at entry
4. **Cycle length**: after meeting, count steps until they meet again
5. **Remove cycle**: find the node before entry and set its Next to nil
6. **Array duplicate** (LeetCode 287): treat array as implicit linked list → cycle detection
7. **Brent's algorithm**: often faster in practice, directly gives cycle length

> **Next up:** List Merging →
