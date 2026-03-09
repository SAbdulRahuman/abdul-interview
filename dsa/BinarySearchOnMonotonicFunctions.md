# Phase 10: Binary Search — Binary Search on Monotonic Functions

## Overview

A function `f(x)` is **monotonic** if it is either always non-decreasing or always non-increasing. Binary search works whenever you're searching over a monotonic function — not just sorted arrays.

```
If f is monotonic and you want to find x where f(x) satisfies a condition,
binary search on x using f(x) as the decision.
```

This generalizes binary search beyond arrays to any domain where the predicate changes from `false → true` (or vice versa) exactly once.

---

## Example 1: Square Root (Integer)

```go
package main

import "fmt"

// f(x) = x*x is monotonically increasing
// Find largest x where x*x <= n
func mySqrt(n int) int {
	lo, hi := 0, n
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		if mid <= n/mid { // mid*mid <= n (avoid overflow)
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	for _, n := range []int{0, 1, 4, 8, 16, 25, 100} {
		fmt.Printf("sqrt(%d) = %d\n", n, mySqrt(n))
	}
}
```

---

## Example 2: Cube Root

```go
package main

import "fmt"

// f(x) = x*x*x is monotonically increasing
func cubeRoot(n int) int {
	lo, hi := 0, n
	if n > 1000000 {
		hi = 1000000 // upper bound optimization
	}
	for lo < hi {
		mid := lo + (hi-lo+1)/2
		if mid*mid*mid <= n {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	for _, n := range []int{8, 27, 64, 125, 1000} {
		fmt.Printf("cbrt(%d) = %d\n", n, cubeRoot(n))
	}
}
```

---

## Example 3: First True in Boolean Sequence

```go
package main

import "fmt"

// Generic: find first index where predicate becomes true
// predicate must be: false, false, ..., false, true, true, ..., true
func firstTrue(lo, hi int, predicate func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if predicate(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	// Find first x where x >= 42
	result := firstTrue(0, 100, func(x int) bool {
		return x >= 42
	})
	fmt.Println("First x >= 42:", result) // 42

	// Find first x where x*x >= 100
	result = firstTrue(0, 100, func(x int) bool {
		return x*x >= 100
	})
	fmt.Println("First x where x² >= 100:", result) // 10
}
```

---

## Example 4: Last True in Boolean Sequence

```go
package main

import "fmt"

// Find last index where predicate is true
// predicate: true, true, ..., true, false, false, ...
func lastTrue(lo, hi int, predicate func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo+1)/2 // upper mid
		if predicate(mid) {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	// Find largest x where x*x <= 50
	result := lastTrue(0, 50, func(x int) bool {
		return x*x <= 50
	})
	fmt.Println("Largest x where x² <= 50:", result) // 7

	// Find largest x where 2^x <= 1000
	result = lastTrue(0, 30, func(x int) bool {
		return (1 << x) <= 1000
	})
	fmt.Println("Largest x where 2^x <= 1000:", result) // 9
}
```

---

## Example 5: Find Peak in Unimodal Function (Ternary → Binary)

```go
package main

import "fmt"

// Function that increases then decreases (unimodal)
func f(x int) int {
	return -(x-50)*(x-50) + 2500 // peak at x=50
}

// Binary search on monotonic derivative:
// f(x) < f(x+1) means we're on the increasing side
func findPeak(lo, hi int) int {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if f(mid) < f(mid+1) {
			lo = mid + 1 // go right, still increasing
		} else {
			hi = mid // go left, on decreasing side or at peak
		}
	}
	return lo
}

func main() {
	peak := findPeak(0, 100)
	fmt.Printf("Peak at x=%d, f(%d)=%d\n", peak, peak, f(peak))
	// Peak at x=50, f(50)=2500
}
```

---

## Example 6: Nth Magical Number (LeetCode 878)

```go
package main

import "fmt"

const MOD = 1_000_000_007

func nthMagicalNumber(n, a, b int) int {
	g := gcd(a, b)
	lcm := a / g * b

	// f(x) = count of numbers <= x divisible by a or b
	// f is monotonically non-decreasing
	lo, hi := int64(1), int64(n)*int64(min(a, b))
	for lo < hi {
		mid := lo + (hi-lo)/2
		count := mid/int64(a) + mid/int64(b) - mid/int64(lcm)
		if count >= int64(n) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return int(lo % int64(MOD))
}

func gcd(a, b int) int {
	for b != 0 {
		a, b = b, a%b
	}
	return a
}

func min(a, b int) int {
	if a < b { return a }
	return b
}

func main() {
	fmt.Println(nthMagicalNumber(1, 2, 3))  // 2
	fmt.Println(nthMagicalNumber(4, 2, 3))  // 6
	fmt.Println(nthMagicalNumber(5, 2, 4))  // 10
}
```

---

## Example 7: Kth Smallest in Multiplication Table (LeetCode 668)

```go
package main

import "fmt"

func findKthNumber(m, n, k int) int {
	lo, hi := 1, m*n
	for lo < hi {
		mid := lo + (hi-lo)/2
		// count numbers <= mid in m×n table: monotonic in mid
		count := 0
		for i := 1; i <= m; i++ {
			c := mid / i
			if c > n {
				c = n
			}
			count += c
		}
		if count >= k {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(findKthNumber(3, 3, 5)) // 3
	fmt.Println(findKthNumber(2, 3, 6)) // 6
}
```

---

## Example 8: Find K-th Smallest Pair Distance (LeetCode 719)

```go
package main

import (
	"fmt"
	"sort"
)

func smallestDistancePair(nums []int, k int) int {
	sort.Ints(nums)
	n := len(nums)
	lo, hi := 0, nums[n-1]-nums[0]

	for lo < hi {
		mid := lo + (hi-lo)/2
		// count pairs with distance <= mid: monotonic
		count := countPairs(nums, mid)
		if count >= k {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func countPairs(nums []int, maxDist int) int {
	count, left := 0, 0
	for right := 1; right < len(nums); right++ {
		for nums[right]-nums[left] > maxDist {
			left++
		}
		count += right - left
	}
	return count
}

func main() {
	fmt.Println(smallestDistancePair([]int{1, 3, 1}, 1)) // 0
	fmt.Println(smallestDistancePair([]int{1, 1, 1}, 2)) // 0
	fmt.Println(smallestDistancePair([]int{1, 6, 1}, 3)) // 5
}
```

---

## Example 9: Kth Smallest Element in Sorted Matrix (LeetCode 378)

```go
package main

import "fmt"

func kthSmallest(matrix [][]int, k int) int {
	n := len(matrix)
	lo, hi := matrix[0][0], matrix[n-1][n-1]

	for lo < hi {
		mid := lo + (hi-lo)/2
		count := countLessEqual(matrix, mid)
		if count >= k {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func countLessEqual(matrix [][]int, target int) int {
	n := len(matrix)
	count := 0
	row, col := n-1, 0
	for row >= 0 && col < n {
		if matrix[row][col] <= target {
			count += row + 1
			col++
		} else {
			row--
		}
	}
	return count
}

func main() {
	matrix := [][]int{
		{1, 5, 9},
		{10, 11, 13},
		{12, 13, 15},
	}
	fmt.Println(kthSmallest(matrix, 8)) // 13
}
```

---

## Example 10: Reach a Number (LeetCode 754)

```go
package main

import "fmt"

// f(n) = 1 + 2 + ... + n = n*(n+1)/2 is monotonically increasing
// Find minimum n such that we can reach target
func reachNumber(target int) int {
	if target < 0 {
		target = -target
	}
	// Binary search for smallest n where sum >= target and (sum - target) is even
	lo, hi := 1, target
	for lo < hi {
		mid := lo + (hi-lo)/2
		sum := mid * (mid + 1) / 2
		if sum >= target {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	// lo is first n where sum >= target
	for {
		sum := lo * (lo + 1) / 2
		if (sum-target)%2 == 0 {
			return lo
		}
		lo++
	}
}

func main() {
	fmt.Println(reachNumber(2))   // 3
	fmt.Println(reachNumber(3))   // 2
	fmt.Println(reachNumber(10))  // 4
}
```

---

## Key Takeaways

1. Binary search works on any monotonic function, not just sorted arrays
2. `firstTrue` / `lastTrue` templates handle all monotonic predicates
3. "Kth smallest" problems often use: count(≤ mid) is monotonic in mid
4. Unimodal functions can use binary search on the derivative
5. The key insight: if `feasible(x)` transitions from false→true once, binary search applies

> **Next up:** Binary Search on Floating Point →
