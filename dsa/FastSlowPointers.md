# Phase 25: Advanced Sliding Window — Fast & Slow Pointers

## Overview

The **fast and slow pointer** (Floyd's Tortoise and Hare) technique uses two pointers moving at different speeds. Primarily used for:

1. **Cycle detection** in linked lists/sequences
2. **Finding the middle** of a linked list
3. **Detecting patterns** in sequences (happy number, etc.)

| Pattern | Slow speed | Fast speed | Purpose |
|---------|-----------|-----------|---------|
| Cycle detection | 1 step | 2 steps | Detect and find cycle start |
| Find middle | 1 step | 2 steps | When fast reaches end, slow is at middle |
| Find kth from end | 1 step | 1 step (head start k) | Gap technique |

---

## Example 1: Linked List Cycle Detection (LC 141)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func hasCycle(head *ListNode) bool {
	slow, fast := head, head

	for fast != nil && fast.Next != nil {
		slow = slow.Next       // 1 step
		fast = fast.Next.Next  // 2 steps
		if slow == fast { return true }
	}
	return false
}

func main() {
	// Create list with cycle: 1→2→3→4→2(cycle)
	n1 := &ListNode{Val: 1}
	n2 := &ListNode{Val: 2}
	n3 := &ListNode{Val: 3}
	n4 := &ListNode{Val: 4}
	n1.Next = n2; n2.Next = n3; n3.Next = n4; n4.Next = n2

	fmt.Println("Has cycle:", hasCycle(n1)) // true

	// No cycle
	m1 := &ListNode{Val: 1, Next: &ListNode{Val: 2, Next: &ListNode{Val: 3}}}
	fmt.Println("Has cycle:", hasCycle(m1)) // false
}
```

---

## Example 2: Find Cycle Start (LC 142)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func detectCycle(head *ListNode) *ListNode {
	slow, fast := head, head

	// Phase 1: Detect cycle
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast { break }
	}

	if fast == nil || fast.Next == nil { return nil }

	// Phase 2: Find cycle start
	// Move one pointer to head, both move at same speed
	slow = head
	for slow != fast {
		slow = slow.Next
		fast = fast.Next
	}
	return slow
}

func main() {
	// 1→2→3→4→5→3(cycle at node 3)
	nodes := make([]*ListNode, 5)
	for i := range nodes { nodes[i] = &ListNode{Val: i + 1} }
	nodes[0].Next = nodes[1]; nodes[1].Next = nodes[2]
	nodes[2].Next = nodes[3]; nodes[3].Next = nodes[4]
	nodes[4].Next = nodes[2] // cycle back to 3

	start := detectCycle(nodes[0])
	if start != nil {
		fmt.Printf("Cycle starts at node %d\n", start.Val) // 3
	}

	fmt.Println("\nMath: distance from head to cycle start =")
	fmt.Println("      distance from meeting point to cycle start")
}
```

---

## Example 3: Find Middle of Linked List (LC 876)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func middleNode(head *ListNode) *ListNode {
	slow, fast := head, head

	// When fast reaches end, slow is at middle
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	return slow
}

func buildList(vals []int) *ListNode {
	dummy := &ListNode{}
	cur := dummy
	for _, v := range vals {
		cur.Next = &ListNode{Val: v}
		cur = cur.Next
	}
	return dummy.Next
}

func main() {
	// Odd length
	head1 := buildList([]int{1, 2, 3, 4, 5})
	fmt.Printf("Odd [1,2,3,4,5] → middle: %d\n", middleNode(head1).Val)

	// Even length — returns second middle
	head2 := buildList([]int{1, 2, 3, 4, 5, 6})
	fmt.Printf("Even [1,2,3,4,5,6] → middle: %d\n", middleNode(head2).Val)
}
```

---

## Example 4: Happy Number (LC 202)

```go
package main

import "fmt"

// Detect cycle in the sequence of digit-square sums

func isHappy(n int) bool {
	slow, fast := n, n

	for {
		slow = sumOfSquares(slow)
		fast = sumOfSquares(sumOfSquares(fast))

		if fast == 1 { return true }
		if slow == fast { return false } // cycle detected, not reaching 1
	}
}

func sumOfSquares(n int) int {
	sum := 0
	for n > 0 {
		d := n % 10
		sum += d * d
		n /= 10
	}
	return sum
}

func main() {
	for _, n := range []int{19, 2, 7, 4} {
		fmt.Printf("n=%d → happy=%v\n", n, isHappy(n))
	}

	fmt.Println("\n19: 1²+9²=82 → 8²+2²=68 → 6²+8²=100 → 1 ✓")
	fmt.Println("2: cycles back without reaching 1 ✗")
}
```

---

## Example 5: Find Duplicate Number (LC 287)

```go
package main

import "fmt"

// Array of n+1 integers in [1,n] → at least one duplicate
// Treat as linked list: index → value → next index
// Find cycle using Floyd's algorithm

func findDuplicate(nums []int) int {
	// Phase 1: Find intersection (cycle exists because of pigeonhole)
	slow, fast := nums[0], nums[nums[0]]
	for slow != fast {
		slow = nums[slow]
		fast = nums[nums[fast]]
	}

	// Phase 2: Find cycle start (= duplicate number)
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

	fmt.Println("\nO(n) time, O(1) space — no modifying array!")
}
```

---

## Example 6: Palindrome Linked List (LC 234)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func isPalindrome(head *ListNode) bool {
	if head == nil || head.Next == nil { return true }

	// Step 1: Find middle using fast/slow
	slow, fast := head, head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}

	// Step 2: Reverse second half
	second := reverseList(slow.Next)
	slow.Next = nil // cut the list

	// Step 3: Compare both halves
	first := head
	for first != nil && second != nil {
		if first.Val != second.Val { return false }
		first = first.Next
		second = second.Next
	}
	return true
}

func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	cur := head
	for cur != nil {
		next := cur.Next
		cur.Next = prev
		prev = cur
		cur = next
	}
	return prev
}

func buildList(vals []int) *ListNode {
	dummy := &ListNode{}
	cur := dummy
	for _, v := range vals { cur.Next = &ListNode{Val: v}; cur = cur.Next }
	return dummy.Next
}

func main() {
	tests := [][]int{{1, 2, 2, 1}, {1, 2, 3, 2, 1}, {1, 2, 3}}
	for _, t := range tests {
		fmt.Printf("%v → palindrome=%v\n", t, isPalindrome(buildList(t)))
	}
}
```

---

## Example 7: Linked List Cycle Length

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func cycleLength(head *ListNode) int {
	slow, fast := head, head

	// Find meeting point
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			// Count cycle length
			length := 1
			curr := slow.Next
			for curr != slow {
				length++
				curr = curr.Next
			}
			return length
		}
	}
	return 0 // no cycle
}

func main() {
	// 1→2→3→4→5→3 (cycle length = 3: 3→4→5→3)
	nodes := make([]*ListNode, 5)
	for i := range nodes { nodes[i] = &ListNode{Val: i + 1} }
	for i := 0; i < 4; i++ { nodes[i].Next = nodes[i+1] }
	nodes[4].Next = nodes[2]

	fmt.Printf("Cycle length: %d\n", cycleLength(nodes[0]))
}
```

---

## Example 8: Reorder List (LC 143)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

// Reorder: L0→Ln→L1→Ln-1→L2→Ln-2→...
// Uses fast/slow to find middle, reverse second half, then merge

func reorderList(head *ListNode) {
	if head == nil || head.Next == nil { return }

	// 1. Find middle
	slow, fast := head, head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}

	// 2. Reverse second half
	second := reverse(slow.Next)
	slow.Next = nil

	// 3. Merge alternating
	first := head
	for second != nil {
		tmp1, tmp2 := first.Next, second.Next
		first.Next = second
		second.Next = tmp1
		first = tmp1
		second = tmp2
	}
}

func reverse(head *ListNode) *ListNode {
	var prev *ListNode
	for head != nil {
		next := head.Next
		head.Next = prev
		prev = head
		head = next
	}
	return prev
}

func printList(head *ListNode) {
	for head != nil {
		fmt.Printf("%d", head.Val)
		if head.Next != nil { fmt.Print("→") }
		head = head.Next
	}
	fmt.Println()
}

func buildList(vals []int) *ListNode {
	dummy := &ListNode{}
	cur := dummy
	for _, v := range vals { cur.Next = &ListNode{Val: v}; cur = cur.Next }
	return dummy.Next
}

func main() {
	head := buildList([]int{1, 2, 3, 4, 5})
	fmt.Print("Before: "); printList(head)
	reorderList(head)
	fmt.Print("After:  "); printList(head)
	// 1→5→2→4→3
}
```

---

## Example 9: Circular Array Loop (LC 457)

```go
package main

import "fmt"

func circularArrayLoop(nums []int) bool {
	n := len(nums)

	next := func(i int) int {
		return ((i+nums[i])%n + n) % n
	}

	for i := 0; i < n; i++ {
		if nums[i] == 0 { continue }

		slow, fast := i, next(i)

		// Follow same direction (all positive or all negative in cycle)
		for nums[slow]*nums[fast] > 0 && nums[slow]*nums[next(fast)] > 0 {
			if slow == fast {
				// Check cycle length > 1
				if slow != next(slow) { return true }
				break
			}
			slow = next(slow)
			fast = next(next(fast))
		}

		// Mark visited elements as 0 (optimization)
		j := i
		for nums[j]*nums[next(j)] > 0 {
			tmp := next(j)
			nums[j] = 0
			j = tmp
		}
		nums[j] = 0
	}
	return false
}

func main() {
	tests := [][]int{
		{2, -1, 1, 2, 2},  // true: 0→2→3→0
		{-1, 2},            // false
		{-2, 1, -1, -2, -2}, // false
	}

	for _, nums := range tests {
		cp := append([]int{}, nums...)
		fmt.Printf("%v → %v\n", nums, circularArrayLoop(cp))
	}
}
```

---

## Example 10: Fast & Slow Pointer Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Fast & Slow Pointer Patterns ===\n")

	fmt.Println("--- Pattern 1: Cycle Detection ---")
	fmt.Println("  Slow: 1 step, Fast: 2 steps")
	fmt.Println("  If they meet → cycle exists")
	fmt.Println("  Reset one to head → they meet at cycle start")
	fmt.Println()

	fmt.Println("--- Pattern 2: Find Middle ---")
	fmt.Println("  When fast reaches end, slow is at middle")
	fmt.Println("  Odd length: exact middle")
	fmt.Println("  Even length: second middle (adjust condition for first)")
	fmt.Println()

	fmt.Println("--- Pattern 3: Gap Technique ---")
	fmt.Println("  Give fast a k-step head start")
	fmt.Println("  Both move at same speed")
	fmt.Println("  When fast reaches end, slow is k from end")
	fmt.Println()

	fmt.Println("--- Applications ---")
	apps := []struct{ problem, technique string }{
		{"LC 141: Has Cycle", "Fast/slow, check if meet"},
		{"LC 142: Cycle Start", "Phase 1: detect, Phase 2: find start"},
		{"LC 876: Middle of List", "Fast 2x, slow 1x"},
		{"LC 202: Happy Number", "Cycle in digit sequence"},
		{"LC 287: Find Duplicate", "Array as implicit linked list"},
		{"LC 234: Palindrome List", "Find middle + reverse + compare"},
		{"LC 143: Reorder List", "Find middle + reverse + merge"},
		{"LC 457: Circular Loop", "Cycle in circular array"},
		{"LC 19: Nth from End", "Gap technique"},
	}
	for _, a := range apps {
		fmt.Printf("  %-28s %s\n", a.problem, a.technique)
	}

	fmt.Println("\n--- Complexity ---")
	fmt.Println("  Time:  O(n) — fast pointer traverses at most 2n")
	fmt.Println("  Space: O(1) — only two pointers")
}
```

---

## Key Takeaways

1. **Cycle detection**: slow (1 step) + fast (2 steps) — they meet inside the cycle
2. **Cycle start**: after meeting, reset one to head, both move 1 step — they meet at cycle start (mathematical proof)
3. **Find middle**: when fast hits end, slow is at middle — enables split operations
4. **Floyd's algorithm** works on any sequence (array, linked list, function iteration)
5. **Duplicate number** (LC 287): brilliant application — treat array indices as linked list
6. **O(1) space**: the key advantage over hash-based approaches

> **Next up:** Multi-Pointer Techniques →
