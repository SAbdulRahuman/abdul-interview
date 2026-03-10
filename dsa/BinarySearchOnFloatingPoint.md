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

**Textual Figure: sqrt(2) via floating-point binary search**

```
  f(x) = x²    target = 2     find x where x² ≈ 2

  lo=0.0                          hi=2.0
  │──────────────────────────────│

  Iter 1: mid=1.0    1²=1.0  ≤ 2 → lo=1.0
  Iter 2: mid=1.5    1.5²=2.25 > 2 → hi=1.5
  Iter 3: mid=1.25   1.25²=1.5625 ≤ 2 → lo=1.25
  Iter 4: mid=1.375  1.375²=1.890625 ≤ 2 → lo=1.375
  Iter 5: mid=1.4375 1.4375²=2.066 > 2 → hi=1.4375
  ...
  After 100 iterations: lo ≈ 1.4142135624 ✓

  Convergence:   [0──────────────────2]
                    [1.0──────1.5]
                      [1.25──1.4375]
                        ↓ converges to √2
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

**Textual Figure: cbrt(8) and cbrt(-8) via floating-point binary search**

```
  f(x) = x³     find x where x³ ≈ target

  cbrt(8):  lo=-8  hi=8
  Iter 1: mid=0.0   0³=0 ≤ 8 → lo=0
  Iter 2: mid=4.0   64 > 8 → hi=4
  Iter 3: mid=2.0   8 ≤ 8 → lo=2
  Iter 4: mid=3.0   27 > 8 → hi=3
  Iter 5: mid=2.5   15.625 > 8 → hi=2.5
  ...→ converges to 2.0 ✓

  cbrt(-8): lo=-8  hi=8
  Iter 1: mid=0.0   0 ≤ -8? No → hi=0
  Iter 2: mid=-4.0  -64 ≤ -8? Yes → lo=-4
  Iter 3: mid=-2.0  -8 ≤ -8? Yes → lo=-2
  Iter 4: mid=-1.0  -1 ≤ -8? No → hi=-1
  ...→ converges to -2.0 ✓

    ──8──────4──────0──────4──────8─
         ↑               ↑
       cbrt(-8)=-2    cbrt(8)=2
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

**Textual Figure: nthRoot(81, 4) — 4th root of 81**

```
  f(x) = x⁴    find x where x⁴ ≈ 81

  lo=0.0                            hi=81.0
  │────────────────────────────────│

  Iter 1: mid=40.5   40.5⁴=2.7M >> 81 → hi=40.5
  Iter 2: mid=20.25  20.25⁴=168K >> 81 → hi=20.25
  Iter 3: mid=10.125 10.125⁴=10506 >> 81 → hi=10.125
  Iter 4: mid=5.0625 5.06⁴=656 >> 81 → hi=5.0625
  Iter 5: mid=2.53   2.53⁴=41 ≤ 81 → lo=2.53
  ...
  Converges to 3.0 ✓

  Results:
  ────────────────────────
  2nd root(16) = 4.0  (4²=16)
  3rd root(27) = 3.0  (3³=27)
  4th root(81) = 3.0  (3⁴=81)
  5th root(32) = 2.0  (2⁵=32)
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

**Textual Figure: minmaxGasDist([1..10], k=9)**

```
Stations on number line:
  1   2   3   4   5   6   7   8   9   10
  │───│───│───│───│───│───│───│───│
  Each gap = 1.0, 9 gaps total

  Add k=9 new stations to minimize max gap.

  BS on answer: max gap d
  lo=0.0  hi=9.0

  d=4.5: each gap 1.0, need ceil(1/4.5)-1=0 per gap → total=0 ≤ 9 → hi=4.5
  d=2.25: need 0 per gap → total=0 ≤ 9 → hi=2.25
  d=1.125: need 0 per gap → total=0 ≤ 9 → hi=1.125
  d=0.5625: need ceil(1/0.5625)-1=1 per gap → total=9 ≤ 9 → hi=0.5625
  d=0.28: need 3 per gap → total=27 > 9 → lo=0.28
  ...
  Converges to 0.5 ✓

  After adding 9 stations (1 per gap):
  1  1.5  2  2.5  3  3.5  ...  9  9.5  10
  │──│──│──│──│──│──│ ... │──│──│
  max gap = 0.5
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

**Textual Figure: findMedian([1, 3], [2])**

```
Array A:  [1, 3]     Array B:  [2]
Merged:   [1, 2, 3]  total=3 (odd)

Need k = total/2 + 1 = 2nd element

BS on value space [-1e6, 1e6]:
  mid=0: countLE(A,B,0) = 0  < 2 → lo=0
  mid=500000: countLE = 3 ≥ 2 → hi=500000
  ... narrows ...
  mid=2: countLE(A,B,2) = 2 ≥ 2 → hi=2
  mid=1: countLE(A,B,1) = 1 < 2 → lo=1
  ... converges to 2.0 ✓

  Sorted view:   [1]  [2]  [3]
                      ↑
                   median = 2.0
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

**Textual Figure: Bisection method on f(x) = x³ - x - 2**

```
  f(x) = x³ - x - 2    find x where f(x) = 0 in [1, 2]

  f(1) = 1-1-2 = -2  (negative)
  f(2) = 8-2-2 =  4  (positive)

       4 ─  ─  ─  ─  ─  ─  ─  ─  ─ *  f(2)=4
       │                      ╱
       │                   ╱
       0 ─ ─ ─ ─ ─ ─ ───*────  ← root ≈ 1.5214
       │              ╱
      -2 ─ * ─ ─ ─ ─          f(1)=-2
          1       1.5       2

  Iter 1: mid=1.5    f=1.5³-1.5-2= -0.125   f*f(lo)≤0 → hi=1.5
  Iter 2: mid=1.25   f=1.25³-1.25-2=-1.297  f*f(lo)>0  → lo=1.25
  Iter 3: mid=1.375  f=-0.724               f*f(lo)>0  → lo=1.375
  Iter 4: mid=1.4375 f=-0.429               f*f(lo)>0  → lo=1.4375
  Iter 5: mid=1.46875 f=-0.278              f*f(lo)>0  → lo=1.4688
  ...
  After 100 iterations: root ≈ 1.5213797068 ✓

  cos(x) = x  →  root ≈ 0.7390851332
  e^x = 3x    →  root ≈ 0.6190612867
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

**Textual Figure: maxAverage([1, 12, -5, -6, 50, 3], k=4)**

```
  Index:  0    1    2    3    4   5
        ┌───┬────┬────┬────┬────┬───┐
  nums: │ 1 │ 12 │ -5 │ -6 │ 50 │ 3 │
        └───┴────┴────┴────┴────┴───┘

  Find max average of any subarray with length ≥ 4.

  Subarrays of length ≥ 4:
    [1,12,-5,-6]     avg = 0.5
    [12,-5,-6,50]    avg = 12.75  ← max
    [−5,-6,50,3]     avg = 10.5
    [1,12,-5,-6,50]  avg = 10.4
    [12,-5,-6,50,3]  avg = 10.8
    [1,12,-5,-6,50,3] avg = 9.17

  BS on answer (max avg):
  lo=-1e5  hi=1e5
  mid=12.75: subtract 12.75 from each element
    adjusted: [-11.75, -0.75, -17.75, -18.75, 37.25, -9.75]
    prefix sum window finds subarray with sum ≥ 0 → feasible
  ...converges to 12.75 ✓

        ┌────┬────┬────┬────┐
        │►12 │ -5 │ -6 │►50 │  avg = 51/4 = 12.75
        └────┴────┴────┴────┘
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

**Textual Figure: minSpeedOnTime([1, 3, 2], hour=2.7)**

```
  Trips:  dist=[1, 3, 2]   hour=2.7

  Train 1 ──►  Train 2 ─────►  Train 3 ───►
  dist=1       dist=3           dist=2
  (ceil up)    (ceil up)        (exact, last)

  At speed s, time = ceil(1/s) + ceil(3/s) + 2/s
  Need time ≤ 2.7

  BS on speed: lo=1  hi=10,000,000
  s=5000000: time=1+1+0=2.0         ≤ 2.7 → hi=5000000
  s=2500000: time=1+1+0=2.0         ≤ 2.7 → hi=2500000
  ...quickly narrows...
  s=3: ceil(1/3)=1 + ceil(3/3)=1 + 2/3≈0.67 = 2.67 ≤ 2.7 → hi=3
  s=2: ceil(1/2)=1 + ceil(3/2)=2 + 2/2=1.0  = 4.0  > 2.7 → lo=3
  lo == hi == 3 → return 3 ✓

  Speed=3:  trip1: ⌈1/3⌉=1h  trip2: ⌈3/3⌉=1h  trip3: 2/3=0.67h
            total = 2.67h ≤ 2.7h  ✓
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

**Textual Figure: findRate([10, 20, 30], deadline=3.0)**

```
  Jobs: [10, 20, 30]   deadline = 3.0 time units
  At rate r: time = 10/r + 20/r + 30/r = 60/r
  Need 60/r ≤ 3.0  →  r ≥ 20.0

  BS for minimum rate:
  lo=0.0                              hi=1e9

  Iter 1: mid=5e8   60/5e8 ≈ 0       ≤ 3 → hi=5e8
  Iter 2: mid=2.5e8 60/2.5e8 ≈ 0     ≤ 3 → hi=2.5e8
  ... quickly narrows ...
  mid=20: 60/20 = 3.0               ≤ 3 → hi=20
  mid=10: 60/10 = 6.0               > 3 → lo=10
  mid=15: 60/15 = 4.0               > 3 → lo=15
  mid=17.5: 60/17.5 = 3.43          > 3 → lo=17.5
  mid=18.75: 60/18.75 = 3.2         > 3 → lo=18.75
  ... converges to 20.0 ✓

  Job timeline at rate=20:
  │── 10/20=0.5 ──│── 20/20=1.0 ──│── 30/20=1.5 ──│
  0            0.5           1.5            3.0
                                         deadline ✓
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

**Textual Figure: Epsilon vs Fixed iterations for sqrt(2)**

```
  Both methods: f(x)=x², find x where x² ≈ 2

  Epsilon method (hi-lo > 1e-15):        Fixed 100 iterations:
  ┌──────────────────────────┐      ┌────────────────────────┐
  │ while hi-lo > 1e-15:   │      │ for i := 0; i<100; i++│
  │   mid = (lo+hi)/2      │      │   mid = (lo+hi)/2     │
  │   if mid² ≤ x: lo=mid  │      │   if mid² ≤ x: lo=mid │
  │   else: hi=mid         │      │   else: hi=mid        │
  └──────────────────────────┘      └────────────────────────┘
   ~50 iterations (log₂(2/1e-15))       Always 100 iterations

  Convergence of interval [lo, hi]:
  Iter  1: [0.0, 2.0]          width=2.0
  Iter  2: [1.0, 2.0]          width=1.0
  Iter  3: [1.0, 1.5]          width=0.5
  Iter 10: width ≈ 0.002
  Iter 50: width ≈ 10⁻¹⁵
  Iter100: width ≈ 10⁻³⁰  (far beyond float64 precision)

  Both: sqrt(2) ≈ 1.414213562373095 ✓
  math.Sqrt(2) = 1.414213562373095 ✓

  ★ Fixed iterations preferred: predictable, no edge cases
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
