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

**Textual Figure: mySqrt(8) — largest x where x*x ≤ 8**

```
  f(x) = x²   monotonically increasing
  Find largest x where x² ≤ 8

  lo=0                              hi=8
  ├───┬───┬───┬───┬───┬───┬───┬───┤
  0   1   2   3   4   5   6   7   8
      1   4   9  16  25  36  49  64   ← x² values

  Iter 1: mid=(0+8+1)/2=4  4²=16 > 8  → hi=3
  Iter 2: mid=(0+3+1)/2=2  2²=4  ≤ 8  → lo=2
  Iter 3: mid=(2+3+1)/2=3  3²=9  > 8  → hi=2
          lo == hi == 2 → return 2 ✓

  Search space narrowing:
  [0─────────────────8]  mid=4, 16>8
  [0───────3]            mid=2, 4≤8
       [2──3]            mid=3, 9>8
       [2]               → answer = 2

  Verify: 2² = 4 ≤ 8,  3² = 9 > 8  ✓
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

**Textual Figure: cubeRoot(27) — largest x where x³ ≤ 27**

```
  f(x) = x³   monotonically increasing
  Find largest x where x³ ≤ 27

  x:   0   1   2   3    4     5
  x³:  0   1   8   27   64   125
                    ↑
                  target

  Iter 1: lo=0  hi=27  mid=14  14³=2744 > 27 → hi=13
  Iter 2: lo=0  hi=13  mid=7   7³=343   > 27 → hi=6
  Iter 3: lo=0  hi=6   mid=3   3³=27   ≤ 27 → lo=3
  Iter 4: lo=3  hi=6   mid=5   5³=125  > 27 → hi=4
  Iter 5: lo=3  hi=4   mid=4   4³=64   > 27 → hi=3
          lo == hi == 3 → return 3 ✓

  ├───┬───┬───┬───┬───┤
  0   1   2  ►3   4   5
              ↑
          cbrt(27) = 3   (3³ = 27)
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

**Textual Figure: firstTrue(0, 100, x >= 42) — first index where predicate becomes true**

```
  Predicate: p(x) = (x >= 42)

  Index:  0   1  ...  41   42   43  ...  100
  p(x):  F   F  ...   F    T    T  ...   T
         ◄── false ──►│◄── true ──────────►
                      ↑
                   answer = 42

  Iter 1: lo=0   hi=100  mid=50   p(50)=T → hi=50
  Iter 2: lo=0   hi=50   mid=25   p(25)=F → lo=26
  Iter 3: lo=26  hi=50   mid=38   p(38)=F → lo=39
  Iter 4: lo=39  hi=50   mid=44   p(44)=T → hi=44
  Iter 5: lo=39  hi=44   mid=41   p(41)=F → lo=42
  Iter 6: lo=42  hi=44   mid=43   p(43)=T → hi=43
  Iter 7: lo=42  hi=43   mid=42   p(42)=T → hi=42
          lo == hi == 42 → return 42 ✓

  Search space:  [0────────────────100]
                    [0──────50]
                       [26──50]
                         [39──50]
                         [39──44]
                           [42─44]
                           [42─43]
                           [42]  → answer
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

**Textual Figure: lastTrue(0, 50, x² ≤ 50) — last index where predicate is true**

```
  Predicate: p(x) = (x*x <= 50)

  x:     0   1   2  ...  7     8    ...  50
  x²:    0   1   4  ... 49    64    ... 2500
  p(x):  T   T   T  ...  T     F    ...   F
         ◄── true ───────►│◄── false ─────►
                          ↑
                       answer = 7

  Iter 1: lo=0   hi=50  mid=25  25²=625  > 50 → hi=24
  Iter 2: lo=0   hi=24  mid=13  13²=169  > 50 → hi=12
  Iter 3: lo=0   hi=12  mid=7    7²=49  ≤ 50 → lo=7
  Iter 4: lo=7   hi=12  mid=10  10²=100  > 50 → hi=9
  Iter 5: lo=7   hi=9   mid=8    8²=64  > 50 → hi=7
          lo == hi == 7 → return 7 ✓

  Verify: 7² = 49 ≤ 50 ✓,  8² = 64 > 50 ✓

  2^x ≤ 1000:  2⁹=512 ≤ 1000, 2¹⁰=1024 > 1000 → answer = 9
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

**Textual Figure: findPeak(0, 100) — f(x) = -(x-50)² + 2500**

```
  f(x) = -(x-50)² + 2500     unimodal, peak at x=50

         2500 ─ ─ ─ ─ ─ ─ ─ ─ ─ *  ← peak
               ╱                   ╲
              ╱                     ╲
             ╱                       ╲
            ╱                         ╲
     0 ───*───────────────────────────*───
          0          50              100

  Binary search on derivative: f(mid) < f(mid+1) → increasing → go right

  Iter 1: lo=0   hi=100  mid=50  f(50)=2500, f(51)=2499
          f(50) > f(51) → hi=50
  Iter 2: lo=0   hi=50   mid=25  f(25)=1875, f(26)=1924
          f(25) < f(26) → lo=26
  Iter 3: lo=26  hi=50   mid=38  f(38)=2356, f(39)=2379
          f(38) < f(39) → lo=39
  ...converges...
  Final:  lo == hi == 50 → return 50 ✓

  f(50) = -(50-50)² + 2500 = 2500  ← maximum
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

**Textual Figure: nthMagicalNumber(4, 2, 3) — 4th number divisible by 2 or 3**

```
  a=2, b=3, lcm=6
  f(x) = ⌊x/2⌋ + ⌊x/3⌋ - ⌊x/6⌋   (count of numbers ≤ x divisible by 2 or 3)
  f is monotonically non-decreasing

  Number line:  1   2   3   4   5   6   7   8   ...
  Div by 2|3?:  ─  ✓   ✓   ✓   ─   ✓   ─   ✓   ...
  Magical #:       1st 2nd 3rd      4th
                                     ↑
                                  answer = 6

  BS on x:  lo=1  hi=4*min(2,3)=8
  Iter 1: mid=4  f(4)=2+1-0=3  < 4  → lo=5
  Iter 2: mid=6  f(6)=3+2-1=4 ≥ 4  → hi=6
  Iter 3: mid=5  f(5)=2+1-0=3  < 4  → lo=6
          lo == hi == 6 → return 6 ✓

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │
  └───┴───┴───┴───┴───┴───┴───┴───┘
       ✓   ✓   ✓       ►✓          ← 4th = 6
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

**Textual Figure: findKthNumber(3, 3, 5) — 5th smallest in 3×3 multiplication table**

```
  3×3 multiplication table:
        col: 1   2   3
       ┌───┬───┬───┐
  r=1  │ 1 │ 2 │ 3 │
       ├───┼───┼───┤
  r=2  │ 2 │ 4 │ 6 │
       ├───┼───┼───┤
  r=3  │ 3 │ 6 │ 9 │
       └───┴───┴───┘
  Sorted: [1, 2, 2, 3, ►3, 4, 6, 6, 9]   k=5

  count(≤ mid) is monotonic in mid

  BS: lo=1  hi=9
  Iter 1: mid=5  count≤5: r1→3,r2→2,r3→1 = 6 ≥ 5 → hi=5
  Iter 2: mid=3  count≤3: r1→3,r2→1,r3→1 = 5 ≥ 5 → hi=3
  Iter 3: mid=2  count≤2: r1→2,r2→1,r3→0 = 3 < 5 → lo=3
          lo == hi == 3 → return 3 ✓
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

**Textual Figure: smallestDistancePair([1, 3, 1], k=1) — 1st smallest pair distance**

```
  After sorting: [1, 1, 3]

  All pair distances:
    |1-1| = 0
    |1-3| = 2
    |1-3| = 2
  Sorted distances: [0, 2, 2]   k=1 → answer = 0

  count(pairs with dist ≤ mid) is monotonic in mid

  BS: lo=0  hi=3-1=2
  Iter 1: mid=1  countPairs([1,1,3], 1):
          right=1: 1-1=0 ≤ 1 → count+=1
          right=2: 3-1=2 > 1 → left=1; 3-1=2>1 → count+=0
          total=1 ≥ 1 → hi=1
  Iter 2: mid=0  countPairs([1,1,3], 0):
          right=1: 1-1=0 ≤ 0 → count+=1
          right=2: 3-1=2 > 0 → count+=0
          total=1 ≥ 1 → hi=0
          lo == hi == 0 → return 0 ✓

  [1, 1, 3]  pairs: (1,1)→0  (1,3)→2  (1,3)→2
                     ↑
              k=1: smallest distance = 0
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

**Textual Figure: kthSmallest(matrix, k=8) — 8th smallest in sorted matrix**

```
         col: 0    1    2
        ┌────┬────┬────┐
  r=0   │  1 │  5 │  9 │
        ├────┼────┼────┤
  r=1   │ 10 │ 11 │ 13 │
        ├────┼────┼────┤
  r=2   │ 12 │ 13 │ 15 │
        └────┴────┴────┘
  Sorted: [1, 5, 9, 10, 11, 12, 13, ►13, 15]   k=8

  countLessEqual(mid) uses staircase walk (monotonic in mid)

  BS: lo=1  hi=15
  Iter 1: mid=8   count≤8=3 (1,5,…)    < 8 → lo=9
  Iter 2: mid=12  count≤12=6 (1,5,9,10,11,12)  < 8 → lo=13
  Iter 3: mid=14  count≤14=8             ≥ 8 → hi=14
  Iter 4: mid=13  count≤13=8             ≥ 8 → hi=13
          lo == hi == 13 → return 13 ✓

  Staircase walk for count≤13:
        ┌────┬────┬────┐
  r=0   │  1 │  5 │  9 │ ← all 3 ≤ 13 → count+=3
  r=1   │ 10 │ 11 │►13 │ ← all 3 ≤ 13 → count+=3
  r=2   │ 12 │►13 │ 15 │ ← 2 ≤ 13     → count+=2
        └────┴────┴────┘   total = 8 ≥ k=8 ✓
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

**Textual Figure: reachNumber(2) — min steps to reach target on number line**

```
  f(n) = 1+2+...+n = n*(n+1)/2   (monotonically increasing)

  n:     1    2    3    4    5
  sum:   1    3    6   10   15

  Step 1: BS for smallest n where sum ≥ target=2
    lo=1 hi=2 mid=1 → sum=1 < 2 → lo=2
    lo == hi == 2 → first n with sum ≥ 2 is n=2

  Step 2: Check (sum - target) % 2 == 0
    n=2: sum=3, diff=3-2=1, 1%2=1 ≠ 0 → try n+1
    n=3: sum=6, diff=6-2=4, 4%2=0 == 0 → return 3 ✓

  Why? We can flip the sign of step k if diff/2 = k:
    Steps: +1, -2, +3 = 2  (flip step 2)

  Number line:
    ◄─────┬───┬───┬───┬───┬───►
   -2  -1   0   1   2   3
              ─+1→ ←-2─ ─+3→
              0    1  -1    2 ✓
```

---

## Key Takeaways

1. Binary search works on any monotonic function, not just sorted arrays
2. `firstTrue` / `lastTrue` templates handle all monotonic predicates
3. "Kth smallest" problems often use: count(≤ mid) is monotonic in mid
4. Unimodal functions can use binary search on the derivative
5. The key insight: if `feasible(x)` transitions from false→true once, binary search applies

> **Next up:** Binary Search on Floating Point →
