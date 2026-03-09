# Phase 10: Binary Search — Search on Answer

## Overview

**Search on answer** (a.k.a. "binary search the answer") applies binary search not to array indices but to the **answer space**. You check if a candidate answer is feasible, and narrow the range.

Pattern:
1. Define the search range `[lo, hi]` over possible answers
2. Write a `feasible(mid)` function that returns `true/false`
3. Binary search for the boundary

---

## Example 1: Koko Eating Bananas (LeetCode 875)

```go
package main

import "fmt"

func minEatingSpeed(piles []int, h int) int {
	lo, hi := 1, 0
	for _, p := range piles {
		if p > hi {
			hi = p
		}
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canFinish(piles, h, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canFinish(piles []int, h, speed int) bool {
	hours := 0
	for _, p := range piles {
		hours += (p + speed - 1) / speed // ceil division
	}
	return hours <= h
}

func main() {
	fmt.Println(minEatingSpeed([]int{3, 6, 7, 11}, 8))    // 4
	fmt.Println(minEatingSpeed([]int{30, 11, 23, 4, 20}, 5)) // 30
	fmt.Println(minEatingSpeed([]int{30, 11, 23, 4, 20}, 6)) // 23
}
```

**Why?** Answer space = `[1, max(piles)]`. For each speed, check if Koko finishes in ≤ h hours. Find the minimum feasible speed.

---

## Example 2: Capacity to Ship Packages (LeetCode 1011)

```go
package main

import "fmt"

func shipWithinDays(weights []int, days int) int {
	lo, hi := 0, 0
	for _, w := range weights {
		if w > lo {
			lo = w
		}
		hi += w
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canShip(weights, days, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canShip(weights []int, days, cap int) bool {
	d, cur := 1, 0
	for _, w := range weights {
		if cur+w > cap {
			d++
			cur = 0
		}
		cur += w
	}
	return d <= days
}

func main() {
	fmt.Println(shipWithinDays([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, 5)) // 15
	fmt.Println(shipWithinDays([]int{3, 2, 2, 4, 1, 4}, 3))                // 6
	fmt.Println(shipWithinDays([]int{1, 2, 3, 1, 1}, 4))                   // 3
}
```

**Why?** Answer = ship capacity. Range `[max(w), sum(w)]`. Minimize capacity such that we can finish in ≤ days.

---

## Example 3: Split Array Largest Sum (LeetCode 410)

```go
package main

import "fmt"

func splitArray(nums []int, k int) int {
	lo, hi := 0, 0
	for _, n := range nums {
		if n > lo {
			lo = n
		}
		hi += n
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canSplit(nums, k, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canSplit(nums []int, k, maxSum int) bool {
	parts, cur := 1, 0
	for _, n := range nums {
		if cur+n > maxSum {
			parts++
			cur = 0
		}
		cur += n
	}
	return parts <= k
}

func main() {
	fmt.Println(splitArray([]int{7, 2, 5, 10, 8}, 2)) // 18
	fmt.Println(splitArray([]int{1, 2, 3, 4, 5}, 2))  // 9
	fmt.Println(splitArray([]int{1, 4, 4}, 3))          // 4
}
```

---

## Example 4: Magnetic Force Between Two Balls (LeetCode 1552)

```go
package main

import (
	"fmt"
	"sort"
)

func maxDistance(position []int, m int) int {
	sort.Ints(position)
	lo, hi := 1, position[len(position)-1]-position[0]
	for lo < hi {
		mid := lo + (hi-lo+1)/2 // upper mid for maximize
		if canPlace(position, m, mid) {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func canPlace(pos []int, m, minDist int) bool {
	count, last := 1, pos[0]
	for i := 1; i < len(pos); i++ {
		if pos[i]-last >= minDist {
			count++
			last = pos[i]
		}
	}
	return count >= m
}

func main() {
	fmt.Println(maxDistance([]int{1, 2, 3, 4, 7}, 3))       // 3
	fmt.Println(maxDistance([]int{5, 4, 3, 2, 1, 1000000000}, 2)) // 999999999
}
```

**Why?** Maximize the minimum distance. Use upper-mid and `lo = mid` pattern.

---

## Example 5: Minimum Number of Days to Make m Bouquets (LeetCode 1482)

```go
package main

import "fmt"

func minDays(bloomDay []int, m int, k int) int {
	if m*k > len(bloomDay) {
		return -1
	}
	lo, hi := 1, 0
	for _, d := range bloomDay {
		if d > hi {
			hi = d
		}
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canMake(bloomDay, m, k, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canMake(bloom []int, m, k, day int) bool {
	bouquets, flowers := 0, 0
	for _, b := range bloom {
		if b <= day {
			flowers++
			if flowers == k {
				bouquets++
				flowers = 0
			}
		} else {
			flowers = 0
		}
	}
	return bouquets >= m
}

func main() {
	fmt.Println(minDays([]int{1, 10, 3, 10, 2}, 3, 1))     // 3
	fmt.Println(minDays([]int{1, 10, 3, 10, 2}, 3, 2))     // -1
	fmt.Println(minDays([]int{7, 7, 7, 7, 12, 7, 7}, 2, 3)) // 12
}
```

---

## Example 6: Smallest Divisor Given Threshold (LeetCode 1283)

```go
package main

import "fmt"

func smallestDivisor(nums []int, threshold int) int {
	lo, hi := 1, 0
	for _, n := range nums {
		if n > hi {
			hi = n
		}
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		sum := 0
		for _, n := range nums {
			sum += (n + mid - 1) / mid
		}
		if sum <= threshold {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(smallestDivisor([]int{1, 2, 5, 9}, 6))        // 5
	fmt.Println(smallestDivisor([]int{44, 22, 33, 11, 1}, 5)) // 44
}
```

---

## Example 7: Maximum Candies Allocated to K Children (LeetCode 2226)

```go
package main

import "fmt"

func maximumCandies(candies []int, k int64) int {
	lo, hi := 0, 0
	for _, c := range candies {
		if c > hi {
			hi = c
		}
	}
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		var count int64
		for _, c := range candies {
			count += int64(c / mid)
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
	fmt.Println(maximumCandies([]int{5, 8, 6}, 3))   // 5
	fmt.Println(maximumCandies([]int{2, 5}, 11))      // 0
}
```

---

## Example 8: Aggressive Cows (Classic)

```go
package main

import (
	"fmt"
	"sort"
)

func aggressiveCows(stalls []int, cows int) int {
	sort.Ints(stalls)
	lo, hi := 1, stalls[len(stalls)-1]-stalls[0]
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		if canPlaceCows(stalls, cows, mid) {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func canPlaceCows(stalls []int, cows, minDist int) bool {
	count, last := 1, stalls[0]
	for i := 1; i < len(stalls); i++ {
		if stalls[i]-last >= minDist {
			count++
			last = stalls[i]
			if count >= cows {
				return true
			}
		}
	}
	return false
}

func main() {
	fmt.Println(aggressiveCows([]int{1, 2, 4, 8, 9}, 3)) // 3
	fmt.Println(aggressiveCows([]int{1, 2, 8, 4, 9}, 3)) // 3 (sorted internally)
}
```

---

## Example 9: Minimize Max Distance to Gas Station (LeetCode 774)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

func minmaxGasDist(stations []int, k int) float64 {
	sort.Ints(stations)
	lo, hi := 0.0, float64(stations[len(stations)-1]-stations[0])
	for hi-lo > 1e-6 {
		mid := (lo + hi) / 2
		if canPlace(stations, k, mid) {
			hi = mid
		} else {
			lo = mid
		}
	}
	return lo
}

func canPlace(stations []int, k int, maxDist float64) bool {
	count := 0
	for i := 1; i < len(stations); i++ {
		gap := float64(stations[i] - stations[i-1])
		count += int(math.Ceil(gap/maxDist)) - 1
	}
	return count <= k
}

func main() {
	fmt.Printf("%.6f\n", minmaxGasDist([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, 9))
	// 0.500000
}
```

---

## Example 10: Painter's Partition Problem

```go
package main

import "fmt"

// Minimize the maximum time any painter spends
func paintPartition(boards []int, k int) int {
	lo, hi := 0, 0
	for _, b := range boards {
		if b > lo {
			lo = b
		}
		hi += b
	}
	for lo < hi {
		mid := lo + (hi-lo)/2
		if canPaint(boards, k, mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func canPaint(boards []int, k, maxTime int) bool {
	painters, cur := 1, 0
	for _, b := range boards {
		if cur+b > maxTime {
			painters++
			cur = 0
		}
		cur += b
	}
	return painters <= k
}

func main() {
	fmt.Println(paintPartition([]int{10, 20, 30, 40}, 2)) // 60
	fmt.Println(paintPartition([]int{10, 20, 60, 50, 30, 40}, 3)) // 90
}
```

---

## Template Summary

| Problem Type | Search Range | Minimize/Maximize | Mid Choice |
|---|---|---|---|
| Minimize answer | `[lo, hi]` | Minimize → `hi = mid` | `lo + (hi-lo)/2` |
| Maximize answer | `[lo, hi]` | Maximize → `lo = mid` | `lo + (hi-lo+1)/2` |
| Float answer | `[lo, hi]` | `hi - lo > eps` | `(lo+hi)/2` |

## Key Takeaways

1. Binary search on answer transforms optimization → decision problem
2. **Minimize**: standard `lo < hi`, set `hi = mid` when feasible
3. **Maximize**: use upper-mid `(hi-lo+1)/2`, set `lo = mid` when feasible
4. The feasible function is the core — must be O(n) or better
5. Range is always `[minimum possible, maximum possible]`

> **Next up:** Binary Search on Monotonic Functions →
