# Phase 20: Bit Manipulation — Bit Masking

## Overview

**Bit masking** uses binary numbers as compact sets where each bit represents presence/absence of an element. This enables O(1) set operations and is essential for subset enumeration, DP with subsets, and permission systems.

| Operation | Bitmask Expression | Meaning |
|-----------|-------------------|---------|
| Add element i | `mask \| (1 << i)` | Insert into set |
| Remove element i | `mask &^ (1 << i)` | Delete from set |
| Check element i | `mask & (1 << i) != 0` | Is in set? |
| Toggle element i | `mask ^ (1 << i)` | Flip membership |
| Union | `a \| b` | A ∪ B |
| Intersection | `a & b` | A ∩ B |
| Difference | `a &^ b` | A \ B |
| Full set of n | `(1 << n) - 1` | {0, 1, ..., n-1} |

---

## Example 1: Set Operations with Bitmask

```go
package main

import "fmt"

func printSet(mask int, n int) {
	fmt.Print("{")
	first := true
	for i := 0; i < n; i++ {
		if mask&(1<<i) != 0 {
			if !first { fmt.Print(", ") }
			fmt.Print(i)
			first = false
		}
	}
	fmt.Printf("} (mask=%04b)\n", mask)
}

func main() {
	n := 4
	a := 0b1010 // {1, 3}
	b := 0b1100 // {2, 3}

	fmt.Print("A = "); printSet(a, n)
	fmt.Print("B = "); printSet(b, n)
	fmt.Print("A | B (union) = "); printSet(a|b, n)
	fmt.Print("A & B (intersect) = "); printSet(a&b, n)
	fmt.Print("A &^ B (diff) = "); printSet(a&^b, n)
	fmt.Print("A ^ B (symm diff) = "); printSet(a^b, n)

	// Add element 0 to A
	a |= (1 << 0)
	fmt.Print("A + {0} = "); printSet(a, n)

	// Remove element 3 from A
	a &^= (1 << 3)
	fmt.Print("A - {3} = "); printSet(a, n)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│    Set Operations with Bitmask (4 elements)     │
├────────────────────────────────────────────────┤
│  Element:   3   2   1   0                       │
│  A = 1010:  {1, 3}                              │
│  B = 1100:  {2, 3}                              │
│                                                  │
│  A | B  (union):                                 │
│    1010 | 1100 = 1110  → {1, 2, 3}               │
│                                                  │
│  A & B  (intersection):                          │
│    1010 & 1100 = 1000  → {3}                     │
│                                                  │
│  A &^ B (difference A\B):                        │
│    1010 &^ 1100 = 0010  → {1}                    │
│                                                  │
│  A ^ B  (symmetric difference):                  │
│    1010 ^ 1100 = 0110  → {1, 2}                  │
└────────────────────────────────────────────────┘
```

---

## Example 2: Enumerate All Subsets

```go
package main

import "fmt"

func subsets(nums []int) [][]int {
	n := len(nums)
	result := [][]int{}

	for mask := 0; mask < (1 << n); mask++ {
		subset := []int{}
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				subset = append(subset, nums[i])
			}
		}
		result = append(result, subset)
	}
	return result
}

func main() {
	nums := []int{1, 2, 3}
	fmt.Printf("All subsets of %v:\n", nums)
	for _, s := range subsets(nums) {
		fmt.Printf("  %v\n", s)
	}
	fmt.Printf("Total: %d subsets\n", 1<<len(nums))
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│   Enumerate All Subsets: nums = {1, 2, 3}       │
├────────────────────────────────────────────────┤
│  mask    bit2 bit1 bit0   subset                │
│  ────    ──── ──── ────   ──────                │
│  000 (0)   0    0    0    {}                    │
│  001 (1)   0    0    1    {1}                   │
│  010 (2)   0    1    0    {2}                   │
│  011 (3)   0    1    1    {1, 2}                │
│  100 (4)   1    0    0    {3}                   │
│  101 (5)   1    0    1    {1, 3}                │
│  110 (6)   1    1    0    {2, 3}                │
│  111 (7)   1    1    1    {1, 2, 3}             │
│                                                  │
│  Total: 2^3 = 8 subsets                          │
│  Each bit i → include nums[i] in subset           │
└────────────────────────────────────────────────┘
```

---

## Example 3: Enumerate Subsets of a Mask

```go
package main

import "fmt"

func printBits(mask, n int) {
	for i := n - 1; i >= 0; i-- {
		if mask&(1<<i) != 0 {
			fmt.Print("1")
		} else {
			fmt.Print("0")
		}
	}
}

func main() {
	mask := 0b1011 // {0, 1, 3}
	n := 4

	fmt.Printf("Subsets of mask ")
	printBits(mask, n)
	fmt.Println(":")

	count := 0
	sub := mask
	for {
		fmt.Printf("  ")
		printBits(sub, n)
		fmt.Println()
		count++

		if sub == 0 { break }
		sub = (sub - 1) & mask // enumerate subsets
	}
	fmt.Printf("Total: %d subsets\n", count)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Enumerate Subsets of mask = 1011 = {0,1,3}     │
├────────────────────────────────────────────────┤
│  Trick: sub = (sub - 1) & mask                  │
│                                                  │
│  Start: sub = 1011 = {0,1,3}                    │
│    sub-1 = 1010, & 1011 = 1010 → {1,3}          │
│    sub-1 = 1001, & 1011 = 1001 → {0,3}          │
│    sub-1 = 1000, & 1011 = 1000 → {3}            │
│    sub-1 = 0111, & 1011 = 0011 → {0,1}          │
│    sub-1 = 0010, & 1011 = 0010 → {1}            │
│    sub-1 = 0001, & 1011 = 0001 → {0}            │
│    sub-1 = 0000, & 1011 = 0000 → {}             │
│                                                  │
│  Total: 8 subsets (= 2^|mask| = 2^3)            │
└────────────────────────────────────────────────┘
```

---

## Example 4: Bitmask DP — Traveling Salesman

```go
package main

import (
	"fmt"
	"math"
)

func tsp(dist [][]int) int {
	n := len(dist)
	full := (1 << n) - 1

	// dp[mask][i] = min cost to visit cities in mask, ending at i
	dp := make([][]int, 1<<n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt32 }
	}
	dp[1][0] = 0 // start at city 0

	for mask := 1; mask <= full; mask++ {
		for u := 0; u < n; u++ {
			if dp[mask][u] == math.MaxInt32 { continue }
			if mask&(1<<u) == 0 { continue }

			for v := 0; v < n; v++ {
				if mask&(1<<v) != 0 { continue } // already visited
				newMask := mask | (1 << v)
				cost := dp[mask][u] + dist[u][v]
				if cost < dp[newMask][v] {
					dp[newMask][v] = cost
				}
			}
		}
	}

	result := math.MaxInt32
	for i := 0; i < n; i++ {
		total := dp[full][i] + dist[i][0]
		if total < result { result = total }
	}
	return result
}

func main() {
	dist := [][]int{
		{0, 10, 15, 20},
		{10, 0, 35, 25},
		{15, 35, 0, 30},
		{20, 25, 30, 0},
	}
	fmt.Println("TSP minimum cost:", tsp(dist))
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Bitmask DP — TSP (4 cities)                    │
├────────────────────────────────────────────────┤
│  dp[mask][i] = min cost visiting cities in mask  │
│                ending at city i                  │
│                                                  │
│  mask = 0001  city {0} visited                   │
│  mask = 0011  cities {0,1} visited               │
│  mask = 0111  cities {0,1,2} visited             │
│  mask = 1111  all 4 cities visited (full=15)     │
│                                                  │
│  Transition: dp[mask|1<<v][v] =                  │
│      min(dp[mask][u] + dist[u][v])               │
│      for each u in mask, v not in mask            │
│                                                  │
│  Example path: 0→10→1→35→2→30→3→20→0              │
│  Total = 10 + 35 + 30 + 20 = 95                 │
│  Optimal: 80 (0→1→3→2→0 = 10+25+30+15)          │
└────────────────────────────────────────────────┘
```

---

## Example 5: Partition Equal Subset Sum (Bitmask)

```go
package main

import "fmt"

func canPartition(nums []int) bool {
	total := 0
	for _, n := range nums { total += n }
	if total%2 != 0 { return false }
	target := total / 2
	n := len(nums)

	for mask := 0; mask < (1 << n); mask++ {
		sum := 0
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				sum += nums[i]
			}
		}
		if sum == target { return true }
	}
	return false
}

func main() {
	tests := [][]int{
		{1, 5, 11, 5},
		{1, 2, 3, 5},
		{3, 3, 3, 4, 5},
	}
	for _, nums := range tests {
		fmt.Printf("%v → canPartition: %v\n", nums, canPartition(nums))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Partition Equal Subset Sum: [1, 5, 11, 5]      │
├────────────────────────────────────────────────┤
│  total = 22, target = 11                         │
│  n = 4 elements → 2^4 = 16 masks                │
│                                                  │
│  mask  elem3 elem2 elem1 elem0  sum              │
│  0000    -     -     -     -      0              │
│  0001    -     -     -     1      1              │
│  0010    -     -     5     -      5              │
│  0110    -     -     5    11     16              │
│  1010    5     -     5     -     10              │
│  1011    5     -     5     1     11 ← target!    │
│                                                  │
│  mask 1011 = {0,1,3} → {1, 5, 5} sums to 11     │
│  Remaining {2} → {11} also sums to 11  ✓          │
└────────────────────────────────────────────────┘
```

---

## Example 6: Maximum AND of Pair (Greedy Bit Masking)

```go
package main

import "fmt"

func maxAND(nums []int) int {
	result := 0
	for bit := 30; bit >= 0; bit-- {
		candidate := result | (1 << bit)
		count := 0
		for _, n := range nums {
			if n&candidate == candidate {
				count++
			}
		}
		if count >= 2 {
			result = candidate
		}
	}
	return result
}

func main() {
	tests := [][]int{
		{1, 2, 3, 4},
		{12, 15, 10, 14},
		{7, 7, 7},
	}
	for _, nums := range tests {
		fmt.Printf("nums=%v → maxAND=%d\n", nums, maxAND(nums))
	}
	fmt.Println()
	fmt.Println("Strategy: greedily set bits from MSB to LSB")
	fmt.Println("  Keep a bit if ≥2 numbers have all set bits so far")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Maximum AND of Pair: [12, 15, 10, 14]          │
├────────────────────────────────────────────────┤
│  12 = 1100    15 = 1111                         │
│  10 = 1010    14 = 1110                         │
│                                                  │
│  Greedy from MSB (bit 3 to 0):                   │
│  bit 3: candidate=1000                            │
│    12&1000=1000✓ 15&1000=1000✓                   │
│    10&1000=1000✓ 14&1000=1000✓  count≥2 ✓       │
│    result = 1000                                 │
│                                                  │
│  bit 2: candidate=1100                            │
│    12&1100=1100✓ 15&1100=1100✓                   │
│    14&1100=1100✓  count≥2 ✓                      │
│    result = 1100 = 12                            │
│                                                  │
│  bit 1: candidate=1110                            │
│    15&1110=1110✓ 14&1110=1110✓  count≥2 ✓       │
│    result = 1110 = 14  (max AND of 14 & 15)      │
└────────────────────────────────────────────────┘
```

---

## Example 7: Permission System with Bitmask

```go
package main

import "fmt"

const (
	PermRead    = 1 << iota // 1
	PermWrite               // 2
	PermExecute             // 4
	PermAdmin               // 8
)

func permString(p int) string {
	s := ""
	if p&PermRead != 0 { s += "R" } else { s += "-" }
	if p&PermWrite != 0 { s += "W" } else { s += "-" }
	if p&PermExecute != 0 { s += "X" } else { s += "-" }
	if p&PermAdmin != 0 { s += "A" } else { s += "-" }
	return s
}

func main() {
	users := map[string]int{
		"alice": PermRead | PermWrite | PermExecute | PermAdmin,
		"bob":   PermRead | PermWrite,
		"carol": PermRead,
	}

	for name, perm := range users {
		fmt.Printf("%-6s [%04b] %s\n", name, perm, permString(perm))
	}

	// Check permissions
	fmt.Printf("\nBob has write? %v\n", users["bob"]&PermWrite != 0)
	fmt.Printf("Carol has admin? %v\n", users["carol"]&PermAdmin != 0)

	// Grant Carol write
	users["carol"] |= PermWrite
	fmt.Printf("Carol after grant: %s\n", permString(users["carol"]))

	// Revoke Bob's write
	users["bob"] &^= PermWrite
	fmt.Printf("Bob after revoke: %s\n", permString(users["bob"]))
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Permission System with Bitmask                 │
├────────────────────────────────────────────────┤
│  Bit:   3     2     1     0                      │
│  Perm: Admin Exec Write Read                    │
│                                                  │
│  alice: 1111 = R W X A  (all permissions)       │
│  bob  : 0011 = R W - -                          │
│  carol: 0001 = R - - -                          │
│                                                  │
│  Check Bob has Write?                            │
│    0011 & 0010 = 0010 ≠ 0 → true                │
│                                                  │
│  Grant Carol Write:  carol |= 0010               │
│    0001 | 0010 = 0011  → R W                    │
│                                                  │
│  Revoke Bob Write:   bob &^= 0010               │
│    0011 &^ 0010 = 0001  → R only                │
└────────────────────────────────────────────────┘
```

---

## Example 8: Minimum Number of Vertices to Reach All

```go
package main

import "fmt"

// Given n tasks with prerequisites as bitmask, find min starting set
func minStartSet(n int, prereqs []int) int {
	// prereqs[i] = bitmask of tasks required before task i
	// Find tasks with no prerequisites
	mask := 0
	for i := 0; i < n; i++ {
		if prereqs[i] == 0 {
			mask |= (1 << i)
		}
	}
	return mask
}

func main() {
	// 4 tasks: 0, 1, 2, 3
	// Task 2 requires tasks 0 and 1
	// Task 3 requires task 2
	prereqs := []int{
		0b0000, // task 0: no prereqs
		0b0000, // task 1: no prereqs
		0b0011, // task 2: needs {0, 1}
		0b0100, // task 3: needs {2}
	}

	start := minStartSet(4, prereqs)
	fmt.Printf("Start set: %04b → tasks: ", start)
	for i := 0; i < 4; i++ {
		if start&(1<<i) != 0 { fmt.Printf("%d ", i) }
	}
	fmt.Println()
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Prerequisite Bitmask — Min Starting Set         │
├────────────────────────────────────────────────┤
│  Task 0: prereqs = 0000 (none) ← start           │
│  Task 1: prereqs = 0000 (none) ← start           │
│  Task 2: prereqs = 0011 (needs 0,1)             │
│  Task 3: prereqs = 0100 (needs 2)               │
│                                                  │
│  Dependency graph:                               │
│    [0] ──┐                                        │
│          ├─→ [2] ──→ [3]                        │
│    [1] ──┘                                        │
│                                                  │
│  Start set mask = 0011 → tasks {0, 1}            │
│  (tasks with prereqs == 0000)                    │
└────────────────────────────────────────────────┘
```

---

## Example 9: Counting Subsets with Given Sum

```go
package main

import "fmt"

func countSubsetsWithSum(nums []int, target int) int {
	n := len(nums)
	count := 0

	for mask := 0; mask < (1 << n); mask++ {
		sum := 0
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				sum += nums[i]
			}
		}
		if sum == target { count++ }
	}
	return count
}

func main() {
	nums := []int{1, 2, 3, 4, 5}
	for target := 1; target <= 15; target++ {
		c := countSubsetsWithSum(nums, target)
		if c > 0 {
			fmt.Printf("  sum=%2d: %d subsets\n", target, c)
		}
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Count Subsets With Sum: [1,2,3,4,5], target=6  │
├────────────────────────────────────────────────┤
│  Iterate all 2^5 = 32 masks:                    │
│                                                  │
│  mask  bit4 bit3 bit2 bit1 bit0  subset   sum   │
│  00110   0    0    1    1    0   {2,3}     5    │
│  00111   0    0    1    1    1   {1,2,3}   6 ✓  │
│  01001   0    1    0    0    1   {1,4}     5    │
│  01010   0    1    0    1    0   {2,4}     6 ✓  │
│  01100   0    1    1    0    0   {3,4}     7    │
│  10001   1    0    0    0    1   {1,5}     6 ✓  │
│  10010   1    0    0    1    0   {2,5}     7    │
│                                                  │
│  Check: mask & (1<<i) selects element i          │
│  Sum selected elements, compare to target        │
└────────────────────────────────────────────────┘
```

---

## Example 10: Bitmask Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bitmask Patterns ===")
	fmt.Println()

	patterns := []struct{ pattern, code, use string }{
		{"Enumerate all subsets", "for m := 0; m < (1<<n); m++", "2^n exhaustive search"},
		{"Enumerate subsets of mask", "sub = (sub-1) & mask", "3^n total for all masks"},
		{"Add element", "mask | (1 << i)", "Set insertion"},
		{"Remove element", "mask &^ (1 << i)", "Set deletion"},
		{"Toggle element", "mask ^ (1 << i)", "Flip membership"},
		{"Check element", "mask & (1 << i) != 0", "Membership test"},
		{"Count elements", "bits.OnesCount(uint(mask))", "Set size"},
		{"Is subset", "a & b == a", "a ⊆ b"},
		{"Full set", "(1 << n) - 1", "{0..n-1}"},
		{"Singleton", "1 << i", "{i}"},
		{"LSB (lowest set bit)", "mask & (-mask)", "Iterating set bits"},
		{"Remove LSB", "mask & (mask-1)", "Counting bits"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-28s %s\n", p.pattern, p.code)
		fmt.Printf("  %30s→ %s\n\n", "", p.use)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Bitmask Patterns Quick Reference                │
├────────────────┬───────────────────────────────┤
│  Operation       │  Example (n=4 elements)       │
├────────────────┼───────────────────────────────┤
│  Full set        │  (1<<4)-1 = 1111 = {0,1,2,3} │
│  Add elem 2      │  mask | 0100                  │
│  Remove elem 1   │  mask &^ 0010                 │
│  Check elem 3    │  mask & 1000 != 0             │
│  Toggle elem 0   │  mask ^ 0001                  │
│  All subsets      │  for m=0; m<(1<<n); m++      │
│  Sub-subsets      │  sub = (sub-1) & mask        │
│  LSB (iterate)   │  mask & (-mask)               │
│  Is subset a⊆b   │  a & b == a                   │
└────────────────┴───────────────────────────────┘
```

---

## Key Takeaways

1. Bitmask as set: each bit = one element; O(1) add/remove/check
2. Subset enumeration: `for m := 0; m < (1<<n); m++` covers all 2^n subsets
3. Sub-subset trick: `sub = (sub-1) & mask` iterates all subsets of a mask
4. Bitmask DP (e.g., TSP): `dp[mask][i]` where mask tracks visited states
5. Practical use: permissions, feature flags, subset sum, graph problems

> **Next up:** XOR Properties →
