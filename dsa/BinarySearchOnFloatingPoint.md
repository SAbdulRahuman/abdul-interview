# Phase 10: Binary Search — Binary Search on Floating Point

## Overview

When the answer is a **real number** (not integer), binary search still works. Instead of `lo < hi`, use `hi - lo > epsilon` or run a fixed number of iterations (typically 100).

```
float binary search:
  while hi - lo > 1e-9:
      mid = (lo + hi) / 2
      if feasible(mid): hi = mid
      else: lo = mid
```

No `+1` adjustments needed — just shrink the interval by half each time.

---

## Example 1: Square Root (Float)

```go
package main

import "fmt"

func sqrt(x float64) float64 {
	if x < 0 {
		return -1 // invalid
	}
	lo, hi := 0.0, x
	if x < 1 {
		hi = 1 // for x in (0,1), sqrt(x) > x
	}
	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		if mid*mid <= x {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func main() {
	tests := []float64{2, 4, 9, 0.25, 100, 0.01}
	for _, t := range tests {
		fmt.Printf("sqrt(%.2f) = %.10f\n", t, sqrt(t))
	}
}
```

---

## Example 2: Cube Root (Including Negative)

```go
package main

import (
	"fmt"
	"math"
)

func cbrt(x float64) float64 {
	lo, hi := -math.Max(math.Abs(x), 1), math.Max(math.Abs(x), 1)
	for i := 0; i < 200; i++ {
		mid := (lo + hi) / 2
		if mid*mid*mid <= x {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func main() {
	tests := []float64{8, 27, -8, 0.125, 1000}
	for _, t := range tests {
		fmt.Printf("cbrt(%.3f) = %.10f\n", t, cbrt(t))
	}
}
```

---

## Example 3: Nth Root

```go
package main

import (
	"fmt"
	"math"
)

func nthRoot(x float64, n int) float64 {
	if x == 0 {
		return 0
	}
	lo, hi := 0.0, math.Max(x, 1)
	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		if power(mid, n) <= x {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func power(base float64, exp int) float64 {
	result := 1.0
	for i := 0; i < exp; i++ {
		result *= base
	}
	return result
}

func main() {
	fmt.Printf("2nd root of 16 = %.6f\n", nthRoot(16, 2))  // 4.0
	fmt.Printf("3rd root of 27 = %.6f\n", nthRoot(27, 3))  // 3.0
	fmt.Printf("4th root of 81 = %.6f\n", nthRoot(81, 4))  // 3.0
	fmt.Printf("5th root of 32 = %.6f\n", nthRoot(32, 5))  // 2.0
}
```

---

## Example 4: Minimize Max Distance to Gas Station (LeetCode 774)

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

	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		needed := 0
		for j := 1; j < len(stations); j++ {
			gap := float64(stations[j] - stations[j-1])
			needed += int(math.Ceil(gap/mid)) - 1
		}
		if needed <= k {
			hi = mid
		} else {
			lo = mid
		}
	}
	return lo
}

func main() {
	fmt.Printf("%.6f\n", minmaxGasDist([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, 9))
	// 0.500000
}
```

---

## Example 5: Median of Two Sorted Arrays (Float BS Approach)

```go
package main

import (
	"fmt"
	"math"
)

// Count elements <= val across both arrays
func countLE(a, b []int, val float64) int {
	ca := upperBound(a, val)
	cb := upperBound(b, val)
	return ca + cb
}

func upperBound(arr []int, val float64) int {
	lo, hi := 0, len(arr)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if float64(arr[mid]) <= val {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func findMedian(a, b []int) float64 {
	total := len(a) + len(b)
	lo, hi := -1e6, 1e6

	find := func(k int) float64 {
		lo, hi := lo, hi
		for i := 0; i < 100; i++ {
			mid := (lo + hi) / 2
			if countLE(a, b, mid) >= k {
				hi = mid
			} else {
				lo = mid
			}
		}
		return lo
	}

	if total%2 == 1 {
		return find(total/2 + 1)
	}
	return (find(total/2) + find(total/2+1)) / 2
}

func main() {
	_ = math.Max // import used
	fmt.Printf("%.1f\n", findMedian([]int{1, 3}, []int{2}))    // 2.0
	fmt.Printf("%.1f\n", findMedian([]int{1, 2}, []int{3, 4})) // 2.5
}
```

---

## Example 6: Equation Root Finding (Bisection Method)

```go
package main

import (
	"fmt"
	"math"
)

// Find x where f(x) = 0 in [lo, hi]
// Requires f(lo) and f(hi) have different signs
func bisection(f func(float64) float64, lo, hi float64) float64 {
	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		if f(mid)*f(lo) <= 0 {
			hi = mid
		} else {
			lo = mid
		}
	}
	return (lo + hi) / 2
}

func main() {
	// x^3 - x - 2 = 0
	f1 := func(x float64) float64 { return x*x*x - x - 2 }
	root := bisection(f1, 1, 2)
	fmt.Printf("Root of x³-x-2: %.10f (f=%.2e)\n", root, f1(root))

	// cos(x) = x
	f2 := func(x float64) float64 { return math.Cos(x) - x }
	root = bisection(f2, 0, 1)
	fmt.Printf("cos(x)=x solution: %.10f\n", root)

	// e^x = 3x
	f3 := func(x float64) float64 { return math.Exp(x) - 3*x }
	root = bisection(f3, 0, 1)
	fmt.Printf("e^x = 3x solution: %.10f\n", root)
}
```

---

## Example 7: Maximum Average Subarray (LeetCode 644)

```go
package main

import "fmt"

// Find max average subarray of length >= k
func maxAverage(nums []int, k int) float64 {
	lo, hi := -1e5, 1e5

	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		if hasAvgAtLeast(nums, k, mid) {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func hasAvgAtLeast(nums []int, k int, avg float64) bool {
	// Check if any subarray of length >= k has average >= avg
	// Equivalent: sum of (nums[i] - avg) >= 0 for some subarray of length >= k
	n := len(nums)
	sum := 0.0
	for i := 0; i < k; i++ {
		sum += float64(nums[i]) - avg
	}
	if sum >= 0 {
		return true
	}
	prevSum, minPrev := 0.0, 0.0
	for i := k; i < n; i++ {
		sum += float64(nums[i]) - avg
		prevSum += float64(nums[i-k]) - avg
		if prevSum < minPrev {
			minPrev = prevSum
		}
		if sum-minPrev >= 0 {
			return true
		}
	}
	return false
}

func main() {
	fmt.Printf("%.5f\n", maxAverage([]int{1, 12, -5, -6, 50, 3}, 4)) // 12.75
}
```

---

## Example 8: Minimum Speed to Arrive on Time (LeetCode 1870)

```go
package main

import (
	"fmt"
	"math"
)

func minSpeedOnTime(dist []int, hour float64) int {
	n := len(dist)
	if float64(n-1) >= hour {
		return -1
	}
	lo, hi := 1, 10_000_000
	for lo < hi {
		mid := lo + (hi-lo)/2
		time := 0.0
		for i := 0; i < n-1; i++ {
			time += math.Ceil(float64(dist[i]) / float64(mid))
		}
		time += float64(dist[n-1]) / float64(mid)
		if time <= hour {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func main() {
	fmt.Println(minSpeedOnTime([]int{1, 3, 2}, 6))    // 1
	fmt.Println(minSpeedOnTime([]int{1, 3, 2}, 2.7))  // 3
	fmt.Println(minSpeedOnTime([]int{1, 3, 2}, 1.9))  // -1
}
```

---

## Example 9: Weighted Job Scheduling — Binary Search for Deadline

```go
package main

import (
	"fmt"
	"math"
)

// Binary search to find the right rate for completing jobs
func findRate(jobs []float64, deadline float64) float64 {
	lo, hi := 0.0, 1e9
	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		totalTime := 0.0
		for _, j := range jobs {
			totalTime += j / mid
		}
		if totalTime <= deadline {
			hi = mid
		} else {
			lo = mid
		}
	}
	return lo
}

func main() {
	jobs := []float64{10, 20, 30}
	deadline := 3.0
	rate := findRate(jobs, deadline)
	fmt.Printf("Minimum rate: %.6f\n", rate) // 20.0

	// Verify
	total := 0.0
	for _, j := range jobs {
		total += j / rate
	}
	fmt.Printf("Total time at rate %.2f: %.6f (deadline: %.1f)\n", rate, total, deadline)
	_ = math.Abs
}
```

---

## Example 10: Epsilon vs Fixed Iterations

```go
package main

import (
	"fmt"
	"math"
	"time"
)

func sqrtEpsilon(x float64) float64 {
	lo, hi := 0.0, math.Max(x, 1)
	for hi-lo > 1e-15 {
		mid := (lo + hi) / 2
		if mid*mid <= x {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func sqrtFixed(x float64) float64 {
	lo, hi := 0.0, math.Max(x, 1)
	for i := 0; i < 100; i++ {
		mid := (lo + hi) / 2
		if mid*mid <= x {
			lo = mid
		} else {
			hi = mid
		}
	}
	return lo
}

func main() {
	x := 2.0

	start := time.Now()
	r1 := sqrtEpsilon(x)
	t1 := time.Since(start)

	start = time.Now()
	r2 := sqrtFixed(x)
	t2 := time.Since(start)

	fmt.Printf("Epsilon method: %.15f (%v)\n", r1, t1)
	fmt.Printf("Fixed 100 iter: %.15f (%v)\n", r2, t2)
	fmt.Printf("math.Sqrt:      %.15f\n", math.Sqrt(x))
	fmt.Println()
	fmt.Println("Both approaches give same precision.")
	fmt.Println("Fixed iterations is preferred — predictable, no edge cases.")
}
```

---

## Key Takeaways

| Approach | When to Use |
|----------|-------------|
| `hi - lo > eps` | When you know the required precision |
| Fixed 100 iterations | Always safe — 2^100 precision is more than enough |
| Integer answer | Use integer binary search instead |

1. Float binary search uses `lo = mid` or `hi = mid` — no `+1`
2. 100 iterations gives 2^(-100) precision ≈ 10^(-30) — always sufficient
3. Epsilon approach risks infinite loops with floating point errors — prefer fixed iterations
4. Works for root finding (bisection method), optimization, and answer searching
5. The feasible function determines which half to keep

> **Next up:** Bisecting an Answer Space →
