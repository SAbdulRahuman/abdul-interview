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

---

## Key Takeaways

1. Identify the answer space `[lo, hi]` — always bounded
2. Write a `feasible(mid)` predicate — must be monotonic
3. **Minimize**: `hi = mid` when feasible, `lo = mid + 1` otherwise
4. **Maximize**: `lo = mid` when feasible, `hi = mid - 1` otherwise (use upper-mid!)
5. Most "minimize the maximum" or "maximize the minimum" problems use this pattern

> **Next up:** Minimize Maximum / Maximize Minimum Pattern →
