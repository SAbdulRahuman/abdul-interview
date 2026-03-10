# Phase 10: Binary Search — Bisecting an Answer Space

## Overview

**Bisecting an answer space** is the broader technique of applying binary search to a range of possible answers, testing each candidate against a feasibility check. The key is identifying:
1. What is the answer space (range)?
2. What is the monotonic feasibility predicate?
3. Are we minimizing or maximizing?

This concept ties together search on answer, monotonic functions, and integer/float binary search.

---

## Example 1: Maximum Number of Removable Characters (LeetCode 1898)

```go
package main

import "fmt"

func maximumRemovals(s string, p string, removable []int) int {
	lo, hi := 0, len(removable)
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		if isSubseq(s, p, removable[:mid]) {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func isSubseq(s, p string, removed []int) bool {
	rem := make(map[int]bool)
	for _, r := range removed {
		rem[r] = true
	}
	j := 0
	for i := 0; i < len(s) && j < len(p); i++ {
		if !rem[i] && s[i] == p[j] {
			j++
		}
	}
	return j == len(p)
}

func main() {
	fmt.Println(maximumRemovals("abcacb", "ab", []int{3, 1, 0}))     // 2
	fmt.Println(maximumRemovals("abcbddddd", "abcd", []int{3, 2, 1, 4, 5, 6})) // 1
}
```

**Answer space:** `[0, len(removable)]`. Maximize removals while `p` remains a subsequence of modified `s`.

**Textual Figure: maximumRemovals("abcacb", "ab", [3,1,0])**

```
Answer Space: [0, 3]  (maximize k removals keeping p="ab" as subsequence)

Iteration 1: lo=0, hi=3
  mid = 0 + (3−0+1)/2 = 2
  Remove indices [3,1] → s = "a_ca_b"
  Subsequence check "ab":  a✓ → b✓  → feasible ✓
  ┌─────────────────────────────────────────┐
  │ Answer Space:  0     1     2     3      │
  │                lo          mid   hi     │
  │                ├─────────── ✓ ──────┤   │
  │ → lo = mid = 2                          │
  └─────────────────────────────────────────┘

Iteration 2: lo=2, hi=3
  mid = 2 + (3−2+1)/2 = 3
  Remove indices [3,1,0] → s = "__ca_b"
  Subsequence check "ab":  no 'a' → NOT feasible ✗
  ┌─────────────────────────────────────────┐
  │ Answer Space:  0     1     2     3      │
  │                            lo   mid=hi  │
  │                            ├──── ✗ ─┤   │
  │ → hi = mid−1 = 2                        │
  └─────────────────────────────────────────┘

Result: lo = hi = 2 → return 2 ✓
```

---

## Example 2: Cutting Ribbons (LeetCode 1891)

```go
package main

import "fmt"

func maxLength(ribbons []int, k int) int {
	lo, hi := 0, 0
	for _, r := range ribbons {
		if r > hi {
			hi = r
		}
	}
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		count := 0
		for _, r := range ribbons {
			count += r / mid
		}
		if count >= k {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(maxLength([]int{9, 7, 5}, 3))   // 5
	fmt.Println(maxLength([]int{7, 5, 9}, 4))   // 4
	fmt.Println(maxLength([]int{5, 7, 9}, 22))  // 0
}
```

**Textual Figure: maxLength([9, 7, 5], k=3)**

```
Answer Space: [0, 9]  (maximize cut length, need ≥ 3 pieces)

Ribbons: ┌───────────┐ ┌─────────┐ ┌───────┐
         │     9     │ │    7    │ │   5   │
         └───────────┘ └─────────┘ └───────┘

Iteration 1: lo=0, hi=9
  mid = 0 + (9+1)/2 = 5
  Pieces: 9/5=1 + 7/5=1 + 5/5=1 = 3 ≥ 3 → feasible ✓  → lo=5

Iteration 2: lo=5, hi=9
  mid = 5 + (9−5+1)/2 = 7
  Pieces: 9/7=1 + 7/7=1 + 5/7=0 = 2 < 3 → NOT feasible ✗  → hi=6

Iteration 3: lo=5, hi=6
  mid = 5 + (6−5+1)/2 = 6
  Pieces: 9/6=1 + 7/6=1 + 5/6=0 = 2 < 3 → NOT feasible ✗  → hi=5

  ┌──────────────────────────────────────────────┐
  │  0   1   2   3   4  [5]  6   7   8   9      │
  │  ✓   ✓   ✓   ✓   ✓   ✓   ✗   ✗   ✗   ✗    │
  │                      ↑                      │
  │                   lo=hi=5                    │
  └──────────────────────────────────────────────┘

Result: lo = hi = 5 → return 5 ✓
```

---

## Example 3: Minimum Time to Complete Trips (LeetCode 2187)

```go
package main

import "fmt"

func minimumTime(time []int, totalTrips int) int64 {
	lo, hi := int64(1), int64(time[0])*int64(totalTrips)
	for lo < hi {
		mid := lo + (hi-lo)/2
		trips := int64(0)
		for _, t := range time {
			trips += mid / int64(t)
		}
		if trips >= int64(totalTrips) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(minimumTime([]int{1, 2, 3}, 5)) // 3
	fmt.Println(minimumTime([]int{2}, 1))         // 2
}
```

**Textual Figure: minimumTime([1, 2, 3], totalTrips=5)**

```
Answer Space: [1, 5]  (minimize time to complete 5 trips)
Buses: speed=1 (1 trip/unit), speed=2 (1 trip/2 units), speed=3 (1 trip/3 units)

Iteration 1: lo=1, hi=5
  mid = 1 + (5−1)/2 = 3
  Trips in 3 units: 3/1=3 + 3/2=1 + 3/3=1 = 5 ≥ 5 → feasible ✓  → hi=3

Iteration 2: lo=1, hi=3
  mid = 1 + (3−1)/2 = 2
  Trips in 2 units: 2/1=2 + 2/2=1 + 2/3=0 = 3 < 5 → NOT feasible ✗  → lo=3

  ┌──────────────────────────────────────┐
  │  Time:  1     2    [3]    4     5   │
  │         ✗     ✗     ✓     ✓     ✓   │
  │                     ↑               │
  │                  lo=hi=3             │
  │         ←NOT OK→ ←──── OK ────→     │
  └──────────────────────────────────────┘

Result: lo = hi = 3 → return 3 ✓
```

---

## Example 4: Maximum Running Time of N Computers (LeetCode 2141)

```go
package main

import "fmt"

func maxRunTime(n int, batteries []int) int64 {
	var sum int64
	for _, b := range batteries {
		sum += int64(b)
	}
	lo, hi := int64(1), sum/int64(n)
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		var total int64
		for _, b := range batteries {
			v := int64(b)
			if v > mid {
				v = mid
			}
			total += v
		}
		if total >= mid*int64(n) {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	fmt.Println(maxRunTime(2, []int{3, 3, 3}))   // 4
	fmt.Println(maxRunTime(2, []int{1, 1, 1, 1})) // 2
}
```

**Textual Figure: maxRunTime(n=2, batteries=[3, 3, 3])**

```
Answer Space: [1, 4]  (sum=9, sum/n=4; maximize runtime per computer)

Batteries:  ┌───┐ ┌───┐ ┌───┐
            │ 3 │ │ 3 │ │ 3 │   sum = 9
            └───┘ └───┘ └───┘

Iteration 1: lo=1, hi=4
  mid = 1 + (4−1+1)/2 = 3
  Contribute: min(3,3)+min(3,3)+min(3,3) = 9 ≥ 3×2=6 → feasible ✓  → lo=3

Iteration 2: lo=3, hi=4
  mid = 3 + (4−3+1)/2 = 4
  Contribute: min(3,4)+min(3,4)+min(3,4) = 9 ≥ 4×2=8 → feasible ✓  → lo=4

  ┌────────────────────────────────────────┐
  │  Runtime:  1     2     3    [4]       │
  │            ✓     ✓     ✓     ✓        │
  │                              ↑        │
  │                           lo=hi=4     │
  └────────────────────────────────────────┘

  Computer 1: ████████████████ (4 units from bat1=3 + bat3=1)
  Computer 2: ████████████████ (4 units from bat2=3 + bat3=1)

Result: lo = hi = 4 → return 4 ✓
```

---

## Example 5: Allocate Books (Classic Interview)

```go
package main

import "fmt"

// Minimize the maximum pages any student reads
func allocateBooks(pages []int, students int) int {
	if students > len(pages) {
		return -1
	}
	lo, hi := 0, 0
	for _, p := range pages {
		if p > lo {
			lo = p
		}
		hi += p
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canAllocate(pages, students, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canAllocate(pages []int, students, maxPages int) bool {
	count, cur := 1, 0
	for _, p := range pages {
		if cur+p > maxPages {
			count++
			cur = 0
		}
		cur += p
	}
	return count <= students
}

func main() {
	fmt.Println(allocateBooks([]int{12, 34, 67, 90}, 2)) // 113
	fmt.Println(allocateBooks([]int{10, 20, 30, 40}, 2)) // 60
	fmt.Println(allocateBooks([]int{5, 10, 30, 20, 15}, 3)) // 30
}
```

**Textual Figure: allocateBooks([12, 34, 67, 90], students=2)**

```
Answer Space: [90, 203]  (minimize the maximum pages any student reads)

Books: ┌────┐ ┌─────┐ ┌──────┐ ┌───────┐
       │ 12 │ │  34 │ │  67  │ │  90   │  total = 203
       └────┘ └─────┘ └──────┘ └───────┘

Iteration 1: lo=90, hi=203 → mid=146
  Student 1: [12,34,67]=113 ≤ 146 ✓
  Student 2: [90]=90 ≤ 146 ✓  → 2 students → feasible ✓  → hi=146

Iteration 2: lo=90, hi=146 → mid=118
  Student 1: [12,34,67]=113 ≤ 118 ✓
  Student 2: [90]=90 ≤ 118 ✓  → 2 students → feasible ✓  → hi=118

Iteration 3: lo=90, hi=118 → mid=104
  Student 1: [12,34]=46 → +67=113 > 104 → split!
  Student 2: [67] → +90=157 > 104 → split!
  Needs 3 students > 2 → NOT feasible ✗  → lo=105

  ... (continues narrowing) ...

Iteration N: lo=113, hi=113
  ┌────────────────────────────────────────────────┐
  │  90  ···  104   ···  [113]  ···  146  ···  203│
  │   ✗         ✗          ✓          ✓         ✓ │
  │                        ↑                      │
  │                     lo=hi=113                  │
  └────────────────────────────────────────────────┘
  Student 1: [12, 34, 67] = 113   Student 2: [90] = 90

Result: return 113 ✓
```

---

## Example 6: Find the Smallest Sufficient Team Size

```go
package main

import "fmt"

// Given tasks with durations and k workers, minimize max load
func minMaxLoad(tasks []int, k int) int {
	lo, hi := 0, 0
	for _, t := range tasks {
		if t > lo {
			lo = t
		}
		hi += t
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		workers, cur := 1, 0
		for _, t := range tasks {
			if cur+t > mid {
				workers++
				cur = 0
			}
			cur += t
		}
		if workers <= k {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(minMaxLoad([]int{3, 2, 4, 1, 5, 2}, 3)) // 6
	fmt.Println(minMaxLoad([]int{7, 2, 5, 10, 8}, 2))    // 18
}
```

**Textual Figure: minMaxLoad([3, 2, 4, 1, 5, 2], k=3)**

```
Answer Space: [5, 17]  (minimize max load across 3 workers)

Tasks: ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
       │ 3 │ │ 2 │ │ 4 │ │ 1 │ │ 5 │ │ 2 │  sum = 17
       └───┘ └───┘ └───┘ └───┘ └───┘ └───┘

Iteration 1: lo=5, hi=17 → mid=11
  Worker 1: [3,2,4,1]=10 ≤ 11  Worker 2: [5,2]=7 ≤ 11
  workers=2 ≤ 3 → feasible ✓  → hi=11

Iteration 2: lo=5, hi=11 → mid=8
  Worker 1: [3,2]=5, +4=9>8 → split
  Worker 2: [4,1]=5, +5=10>8 → split
  Worker 3: [5,2]=7 ≤ 8
  workers=3 ≤ 3 → feasible ✓  → hi=8

Iteration 3: lo=5, hi=8 → mid=6
  Worker 1: [3,2]=5, +4=9>6 → split
  Worker 2: [4,1]=5, +5=10>6 → split
  Worker 3: [5], +2=7>6 → split → Worker 4: [2]
  workers=4 > 3 → NOT feasible ✗  → lo=7

Iteration 4: lo=7, hi=8 → mid=7
  Worker 1: [3,2]=5, +4=9>7 → split
  Worker 2: [4,1]=5, +5=10>7 → split
  Worker 3: [5,2]=7 ≤ 7
  workers=3 ≤ 3 → feasible ✓  → hi=7

  ┌──────────────────────────────────────────────────┐
  │  Load:  5    6   [7]   8    ···   11   ···  17  │
  │         ✗    ✗    ✓    ✓          ✓         ✓   │
  │                   ↑                             │
  │                lo=hi=7                           │
  │         ←NOT OK→  ←────── OK ──────────→        │
  └──────────────────────────────────────────────────┘

  Worker 1: │███ 3 ██│██ 2 ██│ = 5
  Worker 2: │████ 4 ████│█ 1 █│ = 5
  Worker 3: │█████ 5 █████│██ 2 ██│ = 7  ← max load

Result: lo = hi = 7 → return 7 ✓
```

---

## Example 7: Maximum Side Length of Square with Sum ≤ Threshold (LeetCode 1292)

```go
package main

import "fmt"

func maxSideLength(mat [][]int, threshold int) int {
	m, n := len(mat), len(mat[0])
	// Build prefix sum
	pre := make([][]int, m+1)
	for i := range pre {
		pre[i] = make([]int, n+1)
	}
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			pre[i][j] = mat[i-1][j-1] + pre[i-1][j] + pre[i][j-1] - pre[i-1][j-1]
		}
	}

	getSum := func(r1, c1, r2, c2 int) int {
		return pre[r2][c2] - pre[r1-1][c2] - pre[r2][c1-1] + pre[r1-1][c1-1]
	}

	// Binary search on side length
	lo, hi := 0, min(m, n)
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		found := false
		for i := mid; i <= m && !found; i++ {
			for j := mid; j <= n && !found; j++ {
				if getSum(i-mid+1, j-mid+1, i, j) <= threshold {
					found = true
				}
			}
		}
		if found {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func min(a, b int) int {
	if a < b { return a }
	return b
}

func main() {
	mat := [][]int{{1, 1, 3, 2, 4, 3, 2}, {1, 1, 3, 2, 4, 3, 2}, {1, 1, 3, 2, 4, 3, 2}}
	fmt.Println(maxSideLength(mat, 4)) // 2
}
```

**Textual Figure: maxSideLength(mat, threshold=4)**

```
Answer Space: [0, 3]  (maximize side length with sub-square sum ≤ 4)

Matrix (3×7):
  ┌───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 1 │ 3 │ 2 │ 4 │ 3 │ 2 │
  ├───┼───┼───┼───┼───┼───┼───┤
  │ 1 │ 1 │ 3 │ 2 │ 4 │ 3 │ 2 │
  ├───┼───┼───┼───┼───┼───┼───┤
  │ 1 │ 1 │ 3 │ 2 │ 4 │ 3 │ 2 │
  └───┴───┴───┴───┴───┴───┴───┘

Iteration 1: lo=0, hi=3 → mid = 0+(3+1)/2 = 2
  Check all 2×2 sub-squares:
  ┌───┬───┐
  │ 1 │ 1 │ sum = 1+1+1+1 = 4 ≤ 4  ✓ found!
  ├───┼───┤
  │ 1 │ 1 │
  └───┴───┘
  → lo = 2

Iteration 2: lo=2, hi=3 → mid = 2+(3−2+1)/2 = 3
  Check all 3×3 sub-squares:
  ┌───┬───┬───┐
  │ 1 │ 1 │ 3 │ sum = 15 > 4  ✗
  ├───┼───┼───┤
  │ 1 │ 1 │ 3 │ (all 3×3 sums exceed 4)
  ├───┼───┼───┤
  │ 1 │ 1 │ 3 │
  └───┴───┴───┘
  No 3×3 square found → hi = 2

  ┌──────────────────────────────────┐
  │  Side:  0    1   [2]   3        │
  │         ✓    ✓    ✓    ✗        │
  │                   ↑             │
  │                lo=hi=2          │
  └──────────────────────────────────┘

Result: lo = hi = 2 → return 2 ✓
```

---

## Example 8: Minimum Effort Path (LeetCode 1631)

```go
package main

import "fmt"

func minimumEffortPath(heights [][]int) int {
	m, n := len(heights), len(heights[0])
	lo, hi := 0, 1_000_000

	for lo < hi {
		mid := lo + (hi-lo)/2
		if canReach(heights, m, n, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canReach(heights [][]int, m, n, maxEffort int) bool {
	visited := make([][]bool, m)
	for i := range visited {
		visited[i] = make([]bool, n)
	}
	dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}
	var dfs func(i, j int) bool
	dfs = func(i, j int) bool {
		if i == m-1 && j == n-1 {
			return true
		}
		visited[i][j] = true
		for _, d := range dirs {
			ni, nj := i+d[0], j+d[1]
			if ni >= 0 && ni < m && nj >= 0 && nj < n && !visited[ni][nj] {
				diff := heights[ni][nj] - heights[i][j]
				if diff < 0 {
					diff = -diff
				}
				if diff <= maxEffort && dfs(ni, nj) {
					return true
				}
			}
		}
		return false
	}
	return dfs(0, 0)
}

func main() {
	heights := [][]int{{1, 2, 2}, {3, 8, 2}, {5, 3, 5}}
	fmt.Println(minimumEffortPath(heights)) // 2
}
```

**Textual Figure: minimumEffortPath([[1,2,2],[3,8,2],[5,3,5]])**

```
Answer Space: [0, 1000000]  (minimize max absolute diff along path)

Grid with edge weights (absolute differences):
  ┌───┐─1─┌───┐─0─┌───┐
  │ 1 │   │ 2 │   │ 2 │
  └───┘   └───┘   └───┘
    │2      │6      │0
  ┌───┐─5─┌───┐─6─┌───┐
  │ 3 │   │ 8 │   │ 2 │
  └───┘   └───┘   └───┘
    │2      │5      │3
  ┌───┐─2─┌───┐─2─┌───┐
  │ 5 │   │ 3 │   │ 5 │  ← goal (2,2)
  └───┘   └───┘   └───┘

Optimal path (max effort = 2):
  (0,0)→(1,0)→(2,0)→(2,1)→(2,2)
    1  →  3  →  5  →  3  →  5
  diffs:  2     2     2     2    max = 2

Binary search converges:
  ┌────────────────────────────────────────────┐
  │  Effort:  0     1    [2]    3    ···  10⁶  │
  │           ✗     ✗     ✓     ✓         ✓    │
  │                       ↑                    │
  │                    lo=hi=2                  │
  │           ←NO PATH→   ←── PATH EXISTS ──→  │
  └────────────────────────────────────────────┘

  mid=2: DFS from (0,0)→(1,0)[|3−1|=2✓]→(2,0)[|5−3|=2✓]
         →(2,1)[|3−5|=2✓]→(2,2)[|5−3|=2✓] → reachable ✓
  mid=1: No path exists with all diffs ≤ 1 → ✗

Result: lo = hi = 2 → return 2 ✓
```

---

## Example 9: Minimum Number of Days to Disconnect Island (Concept)

```go
package main

import "fmt"

// Concept: binary search on answer with graph validation
// How many elements need value >= threshold after k operations?

func minOps(arr []int, threshold int) int {
	lo, hi := 0, len(arr)
	for lo < hi {
		mid := lo + (hi-lo)/2
		count := 0
		for _, v := range arr {
			if v+mid >= threshold {
				count++
			}
		}
		if count >= len(arr)/2 {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	arr := []int{1, 3, 5, 7, 9}
	fmt.Println(minOps(arr, 6)) // 1 (add 1: [2,4,6,8,10], 3 values >= 6)
	fmt.Println(minOps(arr, 10)) // 3
}
```

**Textual Figure: minOps([1, 3, 5, 7, 9], threshold=10)**

```
Answer Space: [0, 5]  (minimize ops so ≥ len/2=2 elements reach threshold)

Array:  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
        │ 1 │ │ 3 │ │ 5 │ │ 7 │ │ 9 │   threshold = 10
        └───┘ └───┘ └───┘ └───┘ └───┘

Iteration 1: lo=0, hi=5 → mid=2
  v+2 ≥ 10?  3✗  5✗  7✗  9✗  11✓ → count=1 < 2 → ✗  → lo=3

Iteration 2: lo=3, hi=5 → mid=4
  v+4 ≥ 10?  5✗  7✗  9✗  11✓  13✓ → count=2 ≥ 2 → ✓  → hi=4

Iteration 3: lo=3, hi=4 → mid=3
  v+3 ≥ 10?  4✗  6✗  8✗  10✓  12✓ → count=2 ≥ 2 → ✓  → hi=3

  ┌──────────────────────────────────────┐
  │  Ops:  0   1   2  [3]  4   5        │
  │        ✗   ✗   ✗   ✓   ✓   ✓        │
  │                     ↑               │
  │                  lo=hi=3             │
  │        ←NOT OK→  ←── OK ────→       │
  └──────────────────────────────────────┘

  After +3: [4, 6, 8, 10✓, 12✓]  → 2 values ≥ 10  ✓

Result: lo = hi = 3 → return 3 ✓
```

---

## Example 10: Template — Bisect Answer Space

```go
package main

import "fmt"

// Generic template for bisecting answer space
func bisectMinimize(lo, hi int, feasible func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if feasible(mid) {
			hi = mid // answer could be mid or smaller
		} else {
			lo = mid + 1 // need bigger
		}
	}
	return lo
}

func bisectMaximize(lo, hi int, feasible func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo+1)/2 // upper mid!
		if feasible(mid) {
			lo = mid // answer could be mid or bigger
		} else {
			hi = mid - 1 // need smaller
		}
	}
	return lo
}

func main() {
	// Example: minimize x such that x*x >= 50
	ans := bisectMinimize(0, 50, func(x int) bool {
		return x*x >= 50
	})
	fmt.Println("Min x where x²≥50:", ans) // 8

	// Example: maximize x such that x*x <= 50
	ans = bisectMaximize(0, 50, func(x int) bool {
		return x*x <= 50
	})
	fmt.Println("Max x where x²≤50:", ans) // 7
}
```

**Textual Figure: bisectMinimize / bisectMaximize Templates**

```
═══════════════════════════════════════════════════════════════
  bisectMinimize(0, 50, x² ≥ 50)  →  Find smallest x where x²≥50
═══════════════════════════════════════════════════════════════

Iteration 1: lo=0, hi=50 → mid=25
  25²=625 ≥ 50 → ✓ hi=25
Iteration 2: lo=0, hi=25 → mid=12
  12²=144 ≥ 50 → ✓ hi=12
Iteration 3: lo=0, hi=12 → mid=6
  6²=36 < 50  → ✗ lo=7
Iteration 4: lo=7, hi=12 → mid=9
  9²=81 ≥ 50  → ✓ hi=9
Iteration 5: lo=7, hi=9  → mid=8
  8²=64 ≥ 50  → ✓ hi=8
Iteration 6: lo=7, hi=8  → mid=7
  7²=49 < 50  → ✗ lo=8

  ┌──────────────────────────────────────────────┐
  │  x:  ··· 6    7   [8]   9   ···  25  ··· 50 │
  │      ··· ✗    ✗    ✓    ✓        ✓        ✓  │
  │          36   49   64   81                   │
  │                    ↑                         │
  │                 lo=hi=8                       │
  │       ←x²<50→     ←──── x²≥50 ────→         │
  └──────────────────────────────────────────────┘
  Result: 8  (8²=64 ≥ 50 ✓,  7²=49 < 50 ✗)

═══════════════════════════════════════════════════════════════
  bisectMaximize(0, 50, x² ≤ 50)  →  Find largest x where x²≤50
═══════════════════════════════════════════════════════════════

Iteration 1: lo=0, hi=50 → mid=26   (upper‑mid)
  26²=676 > 50 → ✗ hi=25
  ···  (narrowing similarly)
Iteration N: lo=7, hi=8 → mid=8   (upper‑mid)
  8²=64 > 50  → ✗ hi=7

  ┌──────────────────────────────────────────────┐
  │  x:  ··· 5    6   [7]   8   ···  25  ··· 50 │
  │      ··· ✓    ✓    ✓    ✗        ✗        ✗  │
  │          25   36   49   64                   │
  │                    ↑                         │
  │                 lo=hi=7                       │
  │       ←x²≤50────→  ←── x²>50 ────→          │
  └──────────────────────────────────────────────┘
  Result: 7  (7²=49 ≤ 50 ✓,  8²=64 > 50 ✗)

  ┌─────────────────────────────────────────┐
  │  MINIMIZE template:  hi = mid    (✓)   │
  │                      lo = mid+1  (✗)   │
  │  Finds FIRST ✓ (leftmost feasible)     │
  │                                        │
  │  MAXIMIZE template:  lo = mid    (✓)   │
  │   (upper‑mid!)       hi = mid−1  (✗)   │
  │  Finds LAST ✓ (rightmost feasible)     │
  └─────────────────────────────────────────┘
```

---

## Key Takeaways

1. Identify the answer space `[lo, hi]` — always bounded
2. Write a `feasible(mid)` predicate — must be monotonic
3. **Minimize**: `hi = mid` when feasible, `lo = mid + 1` otherwise
4. **Maximize**: `lo = mid` when feasible, `hi = mid - 1` otherwise (use upper-mid!)
5. Most "minimize the maximum" or "maximize the minimum" problems use this pattern

> **Next up:** Minimize Maximum / Maximize Minimum Pattern →
