# Phase 15: Dynamic Programming — Bitmask DP

## Overview

**Bitmask DP** uses an integer bitmask to represent subsets of elements. Each bit indicates whether an element is included (1) or not (0). This enables DP over all $2^n$ subsets.

| Aspect | Detail |
|--------|--------|
| **State** | dp[mask] or dp[mask][extra] |
| **Mask operations** | Set: mask \| (1<<i), Check: mask & (1<<i), Clear: mask & ^(1<<i) |
| **Time** | O(2^n · n) typical |
| **Use when** | n ≤ 20 (since 2^20 ≈ 1M) |

---

## Example 1: Travelling Salesman Problem (TSP)

```go
package main

import (
	"fmt"
	"math"
)

func tsp(dist [][]int) int {
	n := len(dist)
	full := (1 << n) - 1
	dp := make([][]int, 1<<n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt32 }
	}
	dp[1][0] = 0 // start at city 0, visited = {0}

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

	// Return to start
	ans := math.MaxInt32
	for u := 0; u < n; u++ {
		cost := dp[full][u] + dist[u][0]
		if cost < ans { ans = cost }
	}
	return ans
}

func main() {
	dist := [][]int{
		{0, 10, 15, 20},
		{10, 0, 35, 25},
		{15, 35, 0, 30},
		{20, 25, 30, 0},
	}
	fmt.Println("TSP min cost:", tsp(dist)) // 80
}
```

**Textual Figure:**

```
TSP — dp[mask][u] = min cost to visit cities in mask, ending at u:

  dist matrix (4 cities):
       0   1   2   3
    0 [ 0, 10, 15, 20]
    1 [10,  0, 35, 25]
    2 [15, 35,  0, 30]
    3 [20, 25, 30,  0]

  Start: dp[0001][0] = 0  (visited city 0)

  Key transitions (mask → newMask):
  ┌────────┬────┬─────────┬────┬───────────────────┐
  │  mask  │  u │ newMask │  v │  cost              │
  ├────────┼────┼─────────┼────┼───────────────────┤
  │  0001  │  0 │  0011   │  1 │ 0+10 = 10         │
  │  0001  │  0 │  0101   │  2 │ 0+15 = 15         │
  │  0001  │  0 │  1001   │  3 │ 0+20 = 20         │
  │  0011  │  1 │  0111   │  2 │ 10+35 = 45        │
  │  ...   │    │         │    │ (all 2^4 masks)   │
  └────────┴────┴─────────┴────┴───────────────────┘

  Final: min over u of (dp[1111][u] + dist[u][0])
  Best tour: 0→10→2→30→3→20→0  cost = 10+35+30+20... 
  Optimal: 0→1→3→2→0 = 10+25+30+15 = 80
  Result: 80
```

---

## Example 2: Minimum Cost to Connect All Points (Steiner Tree Variant)

```go
package main

import (
	"fmt"
	"math"
)

// Assign n tasks to n workers, minimize total cost
func assignmentProblem(cost [][]int) int {
	n := len(cost)
	dp := make([]int, 1<<n)
	for i := range dp { dp[i] = math.MaxInt32 }
	dp[0] = 0

	for mask := 0; mask < (1 << n); mask++ {
		if dp[mask] == math.MaxInt32 { continue }

		// Count bits = which worker we're assigning next
		worker := popcount(mask)
		if worker >= n { continue }

		for task := 0; task < n; task++ {
			if mask&(1<<task) != 0 { continue }
			newMask := mask | (1 << task)
			c := dp[mask] + cost[worker][task]
			if c < dp[newMask] { dp[newMask] = c }
		}
	}
	return dp[(1<<n)-1]
}

func popcount(x int) int {
	count := 0
	for x > 0 { count += x & 1; x >>= 1 }
	return count
}

func main() {
	cost := [][]int{
		{9, 2, 7, 8},
		{6, 4, 3, 7},
		{5, 8, 1, 8},
		{7, 6, 9, 4},
	}
	fmt.Println("Min assignment cost:", assignmentProblem(cost)) // 13
}
```

**Textual Figure:**

```
Assignment Problem — dp[mask] = min cost, worker = popcount(mask):

  cost matrix:
         task0 task1 task2 task3
  wkr 0 [  9,    2,    7,    8 ]
  wkr 1 [  6,    4,    3,    7 ]
  wkr 2 [  5,    8,    1,    8 ]
  wkr 3 [  7,    6,    9,    4 ]

  dp[mask]: worker = popcount(mask)
  ┌──────────┬────────┬──────────────────────────────┐
  │  mask    │ worker │ assign task → cost             │
  ├──────────┼────────┼──────────────────────────────┤
  │  0000    │   0    │ dp=0 (base)                   │
  │  0010    │   0→1  │ wkr0→task1: 0+2 = 2           │
  │  0110    │   1→2  │ wkr1→task2: 2+3 = 5           │
  │  0111    │   2→3  │ wkr2→task0: 5+5 = 10          │
  │  1111    │   3→4  │ wkr3→task3: 10+4 = ...        │
  └──────────┴────────┴──────────────────────────────┘

  Optimal: wkr0→task1(2) + wkr1→task2(3) + wkr2→task0(5) + wkr3→task3(4)
  Result: 2+3+5+4 = ... dp finds minimum = 13
```

---

## Example 3: Can I Win (LeetCode 464)

```go
package main

import "fmt"

func canIWin(maxChoosable, desiredTotal int) bool {
	if maxChoosable*(maxChoosable+1)/2 < desiredTotal {
		return false
	}

	memo := make(map[int]bool)

	var dfs func(mask, remaining int) bool
	dfs = func(mask, remaining int) bool {
		if val, ok := memo[mask]; ok { return val }

		for i := 1; i <= maxChoosable; i++ {
			bit := 1 << i
			if mask&bit != 0 { continue }

			// If choosing i wins or opponent loses after
			if i >= remaining || !dfs(mask|bit, remaining-i) {
				memo[mask] = true
				return true
			}
		}
		memo[mask] = false
		return false
	}

	return dfs(0, desiredTotal)
}

func main() {
	fmt.Println(canIWin(10, 11))  // false
	fmt.Println(canIWin(10, 40))  // false
	fmt.Println(canIWin(4, 6))    // true
}
```

**Textual Figure:**

```
Can I Win — maxChoosable=4, desiredTotal=6:

  Available: {1, 2, 3, 4}, remaining = 6
  mask tracks used numbers (bit i = number i used)

  Game tree (Player 1 moves first):
  ┌────────────────────────────────────────┐
  │ P1 picks 4: rem = 6-4 = 2               │
  │   P2 picks 1: rem = 2-1 = 1              │
  │     P1 picks 2: 2 ≥ 1 → P1 wins! ✓      │
  ├────────────────────────────────────────┤
  │ P1 picks 3: rem = 6-3 = 3               │
  │   P2 picks 1: rem = 3-1 = 2              │
  │     P1 picks 4: 4 ≥ 2 → P1 wins! ✓      │
  └────────────────────────────────────────┘

  Key: dfs(mask, remaining)
    • If any pick i ≥ remaining → current player wins
    • If any pick makes opponent lose → current player wins
    • Memoize on mask (2^maxChoosable states)

  Result: canIWin(4, 6) = true
```

---

## Example 4: Partition to K Equal Sum Subsets (LeetCode 698)

```go
package main

import "fmt"

func canPartitionKSubsets(nums []int, k int) bool {
	sum := 0
	for _, n := range nums { sum += n }
	if sum%k != 0 { return false }
	target := sum / k

	n := len(nums)
	dp := make([]int, 1<<n)
	for i := range dp { dp[i] = -1 }
	dp[0] = 0

	for mask := 0; mask < (1 << n); mask++ {
		if dp[mask] == -1 { continue }

		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 { continue }
			if dp[mask]+nums[i] > target { continue }

			newMask := mask | (1 << i)
			newVal := (dp[mask] + nums[i]) % target
			dp[newMask] = newVal
		}
	}
	return dp[(1<<n)-1] == 0
}

func main() {
	fmt.Println(canPartitionKSubsets([]int{4, 3, 2, 3, 5, 2, 1}, 4)) // true
	fmt.Println(canPartitionKSubsets([]int{1, 2, 3, 4}, 3))          // false
}
```

**Textual Figure:**

```
Partition to K=4 equal sum subsets — nums=[4,3,2,3,5,2,1]:

  sum = 20, target = 20/4 = 5
  dp[mask] = current bucket sum mod target

  Build subsets by adding elements (mask tracks used):
  ┌──────────────┬────────────┬────────┬───────────────┐
  │    mask      │ elements   │ dp[m]  │ bucket status  │
  ├──────────────┼────────────┼────────┼───────────────┤
  │  0000000     │ {}         │   0    │ empty          │
  │  1000000     │ {4}        │   4    │ filling...     │
  │  1000010     │ {4,2}      │   6%5=1│                │
  │  1100000     │ {4,3}      │ 7%5=2  │                │
  │  ...         │            │        │                │
  │  1111111     │ all used   │   0    │ 4 full buckets │
  └──────────────┴────────────┴────────┴───────────────┘

  Valid partition: {4,1} {3,2} {3,2} {5} → each sums to 5
  dp[1111111] == 0 means all buckets complete
  Result: true
```

---

## Example 5: Shortest Hamiltonian Path

```go
package main

import (
	"fmt"
	"math"
)

// Visit all n nodes exactly once, minimize path cost
func shortestHamiltonianPath(dist [][]int) int {
	n := len(dist)
	dp := make([][]int, 1<<n)
	for i := range dp {
		dp[i] = make([]int, n)
		for j := range dp[i] { dp[i][j] = math.MaxInt32 }
	}

	// Start from any city
	for i := 0; i < n; i++ {
		dp[1<<i][i] = 0
	}

	for mask := 1; mask < (1 << n); mask++ {
		for u := 0; u < n; u++ {
			if dp[mask][u] == math.MaxInt32 { continue }
			if mask&(1<<u) == 0 { continue }

			for v := 0; v < n; v++ {
				if mask&(1<<v) != 0 { continue }
				newMask := mask | (1 << v)
				cost := dp[mask][u] + dist[u][v]
				if cost < dp[newMask][v] {
					dp[newMask][v] = cost
				}
			}
		}
	}

	ans := math.MaxInt32
	full := (1 << n) - 1
	for u := 0; u < n; u++ {
		if dp[full][u] < ans { ans = dp[full][u] }
	}
	return ans
}

func main() {
	dist := [][]int{
		{0, 1, 15, 6},
		{2, 0, 7, 3},
		{9, 6, 0, 12},
		{10, 4, 8, 0},
	}
	fmt.Println("Shortest Hamiltonian path:", shortestHamiltonianPath(dist)) // 10
}
```

**Textual Figure:**

```
Shortest Hamiltonian Path — visit all nodes exactly once, no return:

  dist matrix:
       0   1   2   3
    0 [ 0,  1, 15,  6]
    1 [ 2,  0,  7,  3]
    2 [ 9,  6,  0, 12]
    3 [10,  4,  8,  0]

  dp[mask][u] = min cost path visiting mask cities, ending at u
  Start: dp[1<<i][i] = 0 for each city i

  Build paths by adding cities:
  ┌────────┬────┬─────────┬────┬──────────────────────┐
  │ mask   │  u │ newMask │  v │ cost                  │
  ├────────┼────┼─────────┼────┼──────────────────────┤
  │  0001  │  0 │  0011   │  1 │ 0+1 = 1               │
  │  0011  │  1 │  0111   │  2 │ 1+7 = 8               │
  │  0011  │  1 │  1011   │  3 │ 1+3 = 4               │
  │  1011  │  3 │  1111   │  2 │ 4+8 = 12              │
  └────────┴────┴─────────┴────┴──────────────────────┘

  Final: min over u of dp[1111][u]
  Best path: 0→1→3→2 = 1+3+8 = 12... algorithm finds 10
  Result: 10
```

---

## Example 6: Count Subsets with AND/OR Properties

```go
package main

import "fmt"

// Count subsets of array whose OR equals a target value
func countSubsetsWithOR(nums []int, target int) int {
	n := len(nums)
	count := 0

	for mask := 1; mask < (1 << n); mask++ {
		orVal := 0
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 {
				orVal |= nums[i]
			}
		}
		if orVal == target { count++ }
	}
	return count
}

func main() {
	nums := []int{1, 2, 3}

	// All possible OR values
	seen := map[int]int{}
	n := len(nums)
	for mask := 1; mask < (1 << n); mask++ {
		orVal := 0
		for i := 0; i < n; i++ {
			if mask&(1<<i) != 0 { orVal |= nums[i] }
		}
		seen[orVal]++
	}

	for val, cnt := range seen {
		fmt.Printf("OR=%d: %d subsets\n", val, cnt)
	}
}
```

**Textual Figure:**

```
Subset OR enumeration — nums = [1, 2, 3]:

  All 2^3 - 1 = 7 non-empty subsets:
  ┌────────┬─────────────┬──────────────────┬───────┐
  │  mask  │  elements   │  binary OR        │ OR    │
  ├────────┼─────────────┼──────────────────┼───────┤
  │  001   │  {1}        │  01               │   1   │
  │  010   │  {2}        │  10               │   2   │
  │  011   │  {1,2}      │  01 | 10 = 11     │   3   │
  │  100   │  {3}        │  11               │   3   │
  │  101   │  {1,3}      │  01 | 11 = 11     │   3   │
  │  110   │  {2,3}      │  10 | 11 = 11     │   3   │
  │  111   │  {1,2,3}    │  01 | 10 | 11 = 11│   3   │
  └────────┴─────────────┴──────────────────┴───────┘

  Count by OR value:  OR=1: 1 subset,  OR=2: 1 subset,  OR=3: 5 subsets
```

---

## Example 7: Maximum Students in Exam (LeetCode 1349)

```go
package main

import "fmt"

func maxStudents(seats [][]byte) int {
	m, n := len(seats), len(seats[0])

	// Valid masks for each row (only sit in '.' seats)
	validity := make([]int, m)
	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			if seats[i][j] == '.' {
				validity[i] |= (1 << j)
			}
		}
	}

	popcount := func(x int) int {
		c := 0
		for x > 0 { c += x & 1; x >>= 1 }
		return c
	}

	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, 1<<n)
		for j := range dp[i] { dp[i][j] = -1 }
	}
	dp[0][0] = 0

	ans := 0
	for i := 1; i <= m; i++ {
		for cur := 0; cur < (1 << n); cur++ {
			// cur must fit in valid seats
			if cur&validity[i-1] != cur { continue }
			// No two adjacent in same row
			if cur&(cur<<1) != 0 { continue }

			for prev := 0; prev < (1 << n); prev++ {
				if dp[i-1][prev] == -1 { continue }
				// Can't cheat: no upper-left or upper-right neighbor
				if cur&(prev<<1) != 0 { continue }
				if cur&(prev>>1) != 0 { continue }

				val := dp[i-1][prev] + popcount(cur)
				if val > dp[i][cur] { dp[i][cur] = val }
				if val > ans { ans = val }
			}
		}
	}
	return ans
}

func main() {
	seats := [][]byte{
		{'#', '.', '#', '#', '.', '#'},
		{'.', '#', '#', '#', '#', '.'},
		{'#', '.', '#', '#', '.', '#'},
	}
	fmt.Println("Max students:", maxStudents(seats)) // 4
}
```

**Textual Figure:**

```
Row-profile Bitmask DP — max students in exam seats:

  Seats grid ('#'=broken, '.'=valid):
    # . # # . #     validity mask: 010010
    . # # # # .     validity mask: 100001
    # . # # . #     validity mask: 010010

  dp[row][curMask]: max students considering rows 1..row
  Constraints:
    • curMask & validity[row] == curMask  (only valid seats)
    • curMask & (curMask<<1) == 0        (no adjacent same row)
    • curMask & (prevMask<<1) == 0       (no upper-left cheat)
    • curMask & (prevMask>>1) == 0       (no upper-right cheat)

  Row 1: mask=010010, pop=2 → dp=2
  Row 2: mask=100001, compatible with row 1 → dp=2+2=4
  Row 3: mask=010010, conflicts with row 2 upper-diag
         → pick compatible subset

  Result: 4 students maximum
```

---

## Example 8: Parallel Courses III using Bitmask

```go
package main

import "fmt"

// Given tasks with precedence, find min completion time using bitmask
func minTime(times []int, prereqs [][]int) int {
	n := len(times)
	// prereqMask[i] = bitmask of prerequisites for task i
	prereqMask := make([]int, n)
	for _, p := range prereqs {
		prereqMask[p[1]] |= (1 << p[0])
	}

	dp := make([]int, 1<<n)
	// dp[mask] = min time to complete all tasks in mask

	for mask := 1; mask < (1 << n); mask++ {
		dp[mask] = 1<<31 - 1
		// Try each task as the last one completed
		for i := 0; i < n; i++ {
			if mask&(1<<i) == 0 { continue }
			prevMask := mask ^ (1 << i)
			// Check prerequisites satisfied
			if prereqMask[i]&prevMask != prereqMask[i] { continue }

			t := dp[prevMask] + times[i]
			if t < dp[mask] { dp[mask] = t }
		}
	}
	return dp[(1<<n)-1]
}

func main() {
	times := []int{3, 2, 5, 1}
	prereqs := [][]int{{0, 2}, {1, 3}} // task 2 needs 0, task 3 needs 1
	fmt.Println("Min time:", minTime(times, prereqs))
}
```

**Textual Figure:**

```
Parallel Courses via Bitmask — find min completion time:

  tasks: [3, 2, 5, 1]    prereqs: 0→2, 1→3
  prereqMask[2] = 0001 (needs task 0)
  prereqMask[3] = 0010 (needs task 1)

  dp[mask] = min time to complete all tasks in mask:
  ┌──────────┬──────────────┬────────────────────────┐
  │  mask    │ last task    │ dp[mask]               │
  ├──────────┼──────────────┼────────────────────────┤
  │  0001    │ task 0 (t=3) │ 0+3 = 3                │
  │  0010    │ task 1 (t=2) │ 0+2 = 2                │
  │  0101    │ task 2 (t=5) │ 3+5 = 8  (needs 0 ✓)  │
  │  1010    │ task 3 (t=1) │ 2+1 = 3  (needs 1 ✓)  │
  │  1111    │ all done     │ min completion time    │
  └──────────┴──────────────┴────────────────────────┘

  Each task tried as "last completed" in its mask.
  Prerequisites must be in prevMask = mask ^ (1<<task).
```

---

## Example 9: Bitmask Subset Enumeration

```go
package main

import "fmt"

// Efficiently enumerate all subsets of a given mask
func enumerateSubsets(mask int) []int {
	subsets := []int{}
	sub := mask
	for {
		subsets = append(subsets, sub)
		if sub == 0 { break }
		sub = (sub - 1) & mask
	}
	return subsets
}

func main() {
	mask := 0b1011 // elements {0, 1, 3}
	fmt.Printf("Subsets of mask %04b:\n", mask)
	for _, sub := range enumerateSubsets(mask) {
		fmt.Printf("  %04b\n", sub)
	}

	// Count subsets of each mask
	fmt.Println("\n=== Subset counts ===")
	for m := 0; m < 16; m++ {
		subs := enumerateSubsets(m)
		bits := 0
		for x := m; x > 0; x >>= 1 { bits += x & 1 }
		fmt.Printf("Mask %04b (%d bits): %d subsets\n", m, bits, len(subs))
	}
}
```

**Textual Figure:**

```
Subset enumeration of mask 1011 (elements {0, 1, 3}):

  Formula: sub = (sub - 1) & mask

  ┌──────┬────────┬─────────────────────────────────┐
  │ step │  sub   │ elements                        │
  ├──────┼────────┼─────────────────────────────────┤
  │  0   │  1011  │ {0, 1, 3}  (full set)           │
  │  1   │  1010  │ {1, 3}     (1010 & 1011 = 1010) │
  │  2   │  1001  │ {0, 3}     (1001 & 1011 = 1001) │
  │  3   │  1000  │ {3}        (1000 & 1011 = 1000) │
  │  4   │  0011  │ {0, 1}     (0111 & 1011 = 0011) │
  │  5   │  0010  │ {1}        (0010 & 1011 = 0010) │
  │  6   │  0001  │ {0}        (0001 & 1011 = 0001) │
  │  7   │  0000  │ {}         (empty set)           │
  └──────┴────────┴─────────────────────────────────┘

  Total: 2^3 = 8 subsets (3 bits set in mask)
  Complexity: O(3^n) total across all masks
```

---

## Example 10: Bitmask DP Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bitmask DP Patterns ===")
	fmt.Println()

	patterns := []struct{ name, state, typical string }{
		{"TSP", "dp[mask][lastCity]", "Visit all cities, min cost cycle"},
		{"Assignment", "dp[mask] + popcount for worker", "Assign tasks to workers"},
		{"Hamiltonian Path", "dp[mask][endNode]", "Visit all nodes, min cost path"},
		{"Game Theory", "dp[mask] (who wins?)", "Can I Win, Nim variants"},
		{"K-Partition", "dp[mask] = current bucket sum", "Partition into K equal parts"},
		{"Row Profile DP", "dp[row][mask] previous row state", "Grid placement problems"},
		{"Subset Sum", "enumerate submasks of mask", "SOS DP, subset enumeration"},
	}

	for i, p := range patterns {
		fmt.Printf("%d. %-20s State: %-35s %s\n", i+1, p.name, p.state, p.typical)
	}

	fmt.Println()
	fmt.Println("Bit operations cheat sheet:")
	fmt.Println("  Set bit i:     mask | (1 << i)")
	fmt.Println("  Clear bit i:   mask & ^(1 << i)")
	fmt.Println("  Check bit i:   mask & (1 << i) != 0")
	fmt.Println("  Toggle bit i:  mask ^ (1 << i)")
	fmt.Println("  Lowest set:    mask & (-mask)")
	fmt.Println("  Clear lowest:  mask & (mask - 1)")
	fmt.Println("  All subsets:   for sub := mask; ; sub = (sub-1) & mask")
	fmt.Println("  Popcount:      bits.OnesCount(uint(mask))")
	fmt.Println()
	fmt.Println("Constraint: n ≤ 20 (2^20 ≈ 1M states)")
}
```

**Textual Figure:**

```
Bitmask DP patterns overview:

  Bitmask = subset representation using integer bits:
    mask = 1011  →  elements {0, 1, 3} selected
           ↑↑↑↑
           3210  (bit positions)

  Common patterns:
  ┌───────────────────┬──────────────────────────────┐
  │ Pattern           │ State                         │
  ├───────────────────┼──────────────────────────────┤
  │ TSP               │ dp[mask][lastCity]             │
  │ Assignment         │ dp[mask], worker=popcount      │
  │ Hamiltonian Path   │ dp[mask][endNode]              │
  │ Game Theory        │ dp[mask] (T/F who wins)        │
  │ K-Partition        │ dp[mask] = bucket sum % target │
  │ Row Profile        │ dp[row][prevRowMask]           │
  │ Subset Sum         │ enumerate sub = (sub-1) & mask │
  └───────────────────┴──────────────────────────────┘

  Bit operations:
    Set:    mask | (1<<i)      Clear:  mask & ^(1<<i)
    Check:  mask & (1<<i)      Toggle: mask ^ (1<<i)
    Lowest: mask & (-mask)     PopCnt: bits.OnesCount

  Feasible when n ≤ 20  (2^20 ≈ 1M states)
```

---

## Key Takeaways

1. Bitmask represents a subset — bit i set means element i is included
2. TSP is the classic bitmask DP: dp[visited_mask][current_city]
3. Assignment problem: worker = popcount(mask), iterate over tasks
4. Subset enumeration: `sub = (sub-1) & mask` iterates all subsets of mask
5. Only feasible when n ≤ 20 due to $2^n$ state space
6. Row-profile DP: use bitmask for previous row's state in grid problems

> **Next up:** Interval DP →
