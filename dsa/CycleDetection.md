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
