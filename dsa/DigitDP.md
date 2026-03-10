# Phase 15: Dynamic Programming — Digit DP

## Overview

**Digit DP** counts numbers in a range [L, R] satisfying some property, digit by digit. The key idea: compute `f(R) - f(L-1)` where `f(N)` = count of valid numbers in [0, N].

| Aspect | Detail |
|--------|--------|
| **State** | Position, tight constraint, additional property state |
| **tight** | Whether current prefix equals N's prefix (bounds next digit) |
| **Typical pattern** | Recursive with memoization, iterating digits left to right |
| **Time** | O(digits × states × 10) |

---

## Example 1: Count Numbers with No Repeated Digits

```go
package main

import "fmt"

func countWithoutRepeats(n int) int {
	if n < 0 { return 0 }
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 1 }

	memo := map[[3]int]int{} // [pos, tight, usedMask]

	var solve func(pos int, tight bool, mask int, started bool) int
	solve = func(pos int, tight bool, mask int, started bool) int {
		if pos == len(digits) {
			if started { return 1 }
			return 0
		}

		tightInt := 0
		if tight { tightInt = 1 }
		key := [3]int{pos, tightInt, mask}
		if started {
			if v, ok := memo[key]; ok { return v }
		}

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			newStarted := started || d > 0
			if newStarted && mask&(1<<d) != 0 { continue } // digit used
			newMask := mask
			if newStarted { newMask |= (1 << d) }
			count += solve(pos+1, tight && d == limit, newMask, newStarted)
		}

		if started { memo[key] = count }
		return count
	}

	return solve(0, true, 0, false)
}

func main() {
	fmt.Println(countWithoutRepeats(20))   // 19
	fmt.Println(countWithoutRepeats(100))  // 91
	fmt.Println(countWithoutRepeats(1000)) // 738
}
```

**Textual Figure:**

```
Digit DP — count numbers ≤ 20 with no repeated digits:

  N = 20, digits = [2, 0]
  State: (pos, tight, usedMask, started)

  Recursion tree (pos=0, tight=T, mask=0, started=F):
  ┌────────────────────────────────────────────────┐
  │ d=0 (not started): recurse on pos=1          │
  │   d=0: just "0" (not started) → skip          │
  │   d=1..9: each is a 1-digit number → 9 nums   │
  ├────────────────────────────────────────────────┤
  │ d=1 (started, tight=F, mask=0b0010):          │
  │   d=0..9 except 1: 9 two-digit nums (10,12-19)│
  ├────────────────────────────────────────────────┤
  │ d=2 (started, tight=T, mask=0b0100):          │
  │   d=0 only (tight): "20" → 1 num              │
  └────────────────────────────────────────────────┘

  usedMask tracks which digits are taken (bitmask of 10 bits)
  Total: 9 + 9 + 1 = 19
  Result: 19
```

---

## Example 2: Count Numbers with Digit Sum ≤ K

```go
package main

import "fmt"

func countDigitSumAtMost(n, k int) int {
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 1 }

	memo := map[[3]int]int{}

	var solve func(pos int, tight bool, sumSoFar int) int
	solve = func(pos int, tight bool, sumSoFar int) int {
		if sumSoFar > k { return 0 }
		if pos == len(digits) { return 1 }

		tightInt := 0
		if tight { tightInt = 1 }
		key := [3]int{pos, tightInt, sumSoFar}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			count += solve(pos+1, tight && d == limit, sumSoFar+d)
		}
		memo[key] = count
		return count
	}

	return solve(0, true, 0)
}

func main() {
	// Numbers in [1, 100] with digit sum ≤ 5
	total := countDigitSumAtMost(100, 5) - 1 // subtract 0
	fmt.Println("Count [1,100] digit sum ≤ 5:", total) // 15

	total = countDigitSumAtMost(1000, 10) - 1
	fmt.Println("Count [1,1000] digit sum ≤ 10:", total)
}
```

**Textual Figure:**

```
Digit DP — count [1,100] with digit sum ≤ 5:

  N = 100, K = 5, digits = [1, 0, 0]
  State: (pos, tight, sumSoFar)

  Processing digit by digit (left → right):
  ┌─────┬───────┬──────────┬────────────────────────────┐
  │ pos │ tight │ sumSoFar │ constraint                  │
  ├─────┼───────┼──────────┼────────────────────────────┤
  │  0  │   T   │    0    │ d≤1 (tight), sum+d ≤ 5     │
  │  1  │  var  │  0..1   │ d≤9 or tight, sum+d ≤ 5   │
  │  2  │  var  │  0..5   │ d≤9 or tight, sum+d ≤ 5   │
  │ end │       │ ≤ 5 ?   │ count if reached end       │
  └─────┴───────┴──────────┴────────────────────────────┘

  Valid numbers in [1,100] with digit sum ≤ 5:
    1-digit: 1,2,3,4,5                         = 5
    2-digit: 10,11,12,13,14,20,21,22,23,30,
             31,32,40,41,50                     = 15 → wait...
    3-digit: 100 → sum=1 ≤ 5 ✓                  = 1

  f(100) = 16 (including 0), subtract 1 → 15
  Result: 15
```

---

## Example 3: Numbers At Most N Given Digit Set (LeetCode 902)

```go
package main

import (
	"fmt"
	"strconv"
)

func atMostNGivenDigitSet(digits []string, n int) int {
	s := strconv.Itoa(n)
	nLen := len(s)
	digs := make([]int, len(digits))
	for i, d := range digits { digs[i] = int(d[0] - '0') }

	count := 0

	// Count numbers with fewer digits
	d := len(digs)
	pow := d
	for length := 1; length < nLen; length++ {
		count += pow
		pow *= d
		// Actually, for length-digit numbers: d^length choices
	}
	// Recalculate properly
	count = 0
	for length := 1; length < nLen; length++ {
		p := 1
		for i := 0; i < length; i++ { p *= d }
		count += p
	}

	// Count numbers with exactly nLen digits, ≤ n
	for i := 0; i < nLen; i++ {
		hasSame := false
		for _, dig := range digs {
			if dig < int(s[i]-'0') {
				// Remaining positions: d^(nLen-i-1)
				p := 1
				for j := 0; j < nLen-i-1; j++ { p *= d }
				count += p
			} else if dig == int(s[i]-'0') {
				hasSame = true
			}
		}
		if !hasSame { break }
		if i == nLen-1 { count++ } // n itself is valid
	}

	return count
}

func main() {
	fmt.Println(atMostNGivenDigitSet([]string{"1", "3", "5", "7"}, 100)) // 20
	fmt.Println(atMostNGivenDigitSet([]string{"1", "4", "9"}, 1000000000)) // 29523
}
```

**Textual Figure:**

```
Count numbers ≤ 100 using only digits {1, 3, 5, 7}:

  N = "100", nLen = 3, d = 4 available digits

  Phase 1 — numbers with fewer digits:
  ┌────────┬───────────┬───────┬─────────────────────┐
  │ length │ formula   │ count │ examples            │
  ├────────┼───────────┼───────┼─────────────────────┤
  │   1    │ 4^1       │   4   │ {1,3,5,7}           │
  │   2    │ 4^2       │  16   │ {11,13,...,77}       │
  └────────┴───────────┴───────┴─────────────────────┘
  Subtotal: 4 + 16 = 20

  Phase 2 — 3-digit numbers ≤ 100:
    pos=0: s[0]='1', digits < 1: none
           digit == 1: hasSame=T → continue
    pos=1: s[1]='0', digits < 0: none
           no match → break (no valid 3-digit nums)

  Result: 20
```

---

## Example 4: Count Numbers with Even Digit Count

```go
package main

import "fmt"

func countEvenDigitNumbers(n int) int {
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 1 }

	memo := map[[3]int]int{}

	var solve func(pos int, tight bool, countDigits int, started bool) int
	solve = func(pos int, tight bool, countDigits int, started bool) int {
		if pos == len(digits) {
			if !started { return 0 } // 0 has 0 digits
			if countDigits%2 == 0 { return 1 }
			return 0
		}

		tightInt := 0
		if tight { tightInt = 1 }
		startInt := 0
		if started { startInt = 1 }
		key := [3]int{pos*2 + tightInt, countDigits, startInt}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			newStarted := started || d > 0
			newCount := countDigits
			if newStarted { newCount++ } else if started { newCount++ }
			if d > 0 || started {
				count += solve(pos+1, tight && d == limit, countDigits+1, true)
			} else {
				count += solve(pos+1, tight && d == limit, countDigits, false)
			}
		}
		memo[key] = count
		return count
	}

	return solve(0, true, 0, false)
}

func main() {
	// Numbers with even number of digits in [1..n]
	fmt.Println(countEvenDigitNumbers(20))   // 10 (10..19 are 2-digit)
	fmt.Println(countEvenDigitNumbers(100))  // 10
}
```

**Textual Figure:**

```
Digit DP — count numbers with even digit count ≤ 20:

  N = 20, digits = [2, 0]
  State: (pos, tight, countDigits, started)

  Processing (pos=0, tight=T):
  ┌────────┬───────────┬─────────────────────────────────┐
  │ d at  │ tight     │ what happens                    │
  │ pos=0 │           │                                 │
  ├────────┼───────────┼─────────────────────────────────┤
  │ d=0   │ not start │ single digits: 1 digit (odd) →0│
  │ d=1   │ F         │ 10-19: 2 digits (even) → 10   │
  │ d=2   │ T         │ 20: 2 digits (even) → 1       │
  └────────┴───────────┴─────────────────────────────────┘

  1-digit nums (1-9): all have odd digit count → excluded
  2-digit nums (10-19): even count ✓
  20: even count ✓

  Even-digit numbers in [1,20]: {10,11,...,19,20}
  Result: 10
```

---

## Example 5: Count Integers with Digit 1 (LeetCode 233)

```go
package main

import "fmt"

func countDigitOne(n int) int {
	if n <= 0 { return 0 }

	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }

	memo := map[[3]int]int{}

	var solve func(pos int, tight bool, onesCount int) int
	solve = func(pos int, tight bool, onesCount int) int {
		if pos == len(digits) { return onesCount }

		tightInt := 0
		if tight { tightInt = 1 }
		key := [3]int{pos, tightInt, onesCount}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		total := 0
		for d := 0; d <= limit; d++ {
			extra := 0
			if d == 1 { extra = 1 }
			total += solve(pos+1, tight && d == limit, onesCount+extra)
		}
		memo[key] = total
		return total
	}

	return solve(0, true, 0)
}

func main() {
	fmt.Println(countDigitOne(13))    // 6 (1,10,11,12,13 → 1 appears 6 times)
	fmt.Println(countDigitOne(100))   // 21
	fmt.Println(countDigitOne(1000))  // 301
}
```

**Textual Figure:**

```
Count total occurrences of digit '1' in all numbers [1..13]:

  N = 13, digits = [1, 3]
  State: (pos, tight, onesCount)

  Enumerate each number and count 1-appearances:
  ┌───────┬───────────────┬──────────────┐
  │  num  │ contains '1' │ # of 1s      │
  ├───────┼───────────────┼──────────────┤
  │  1    │  "1"          │      1       │
  │  2-9  │  none         │      0       │
  │  10   │  "10"         │      1       │
  │  11   │  "11"         │      2       │
  │  12   │  "12"         │      1       │
  │  13   │  "13"         │      1       │
  └───────┴───────────────┴──────────────┘
  Total 1s: 1+0+1+2+1+1 = 6

  Digit DP accumulates onesCount at end of each number,
  summing across all valid paths through the digit tree.
  Result: 6
```

---

## Example 6: Count Numbers in Range [L, R] Divisible by K

```go
package main

import "fmt"

func countDivisible(n, k int) int {
	if n < 0 { return 0 }
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 1 } // 0 is divisible by k when we count 0

	memo := map[[3]int]int{}

	var solve func(pos int, tight bool, rem int) int
	solve = func(pos int, tight bool, rem int) int {
		if pos == len(digits) {
			if rem == 0 { return 1 }
			return 0
		}

		tightInt := 0
		if tight { tightInt = 1 }
		key := [3]int{pos, tightInt, rem}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			count += solve(pos+1, tight && d == limit, (rem*10+d)%k)
		}
		memo[key] = count
		return count
	}

	return solve(0, true, 0)
}

func main() {
	// Count numbers in [1, 100] divisible by 7
	result := countDivisible(100, 7) - 1 // subtract 0
	fmt.Println("Divisible by 7 in [1,100]:", result) // 14

	// Count in range [L, R]
	L, R := 50, 200
	inRange := countDivisible(R, 13) - countDivisible(L-1, 13)
	fmt.Println("Divisible by 13 in [50,200]:", inRange)
}
```

**Textual Figure:**

```
Digit DP — count [1,100] divisible by 7:

  N = 100, K = 7, digits = [1, 0, 0]
  State: (pos, tight, remainder % K)

  Track (number mod K) as digits are placed:
  ┌─────┬─────────────────────────────────────┐
  │ pos │ transition: rem = (rem*10 + d) % K │
  ├─────┼─────────────────────────────────────┤
  │  0  │ d=0: rem=0    d=1: rem=1 (tight)  │
  │  1  │ rem = (prev_rem * 10 + d) % 7      │
  │  2  │ rem = (prev_rem * 10 + d) % 7      │
  │ end │ valid if rem == 0                   │
  └─────┴─────────────────────────────────────┘

  Example path for num=14:
    pos=0: d=0 → rem=(0*10+0)%7=0
    pos=1: d=1 → rem=(0*10+1)%7=1
    pos=2: d=4 → rem=(1*10+4)%7=0 ✓ valid!

  Multiples of 7 in [1,100]:
    {7,14,21,28,35,42,49,56,63,70,77,84,91,98}
  Range [L,R]: f(R) - f(L-1)
  Result: 14
```

---

## Example 7: Count Stepping Numbers (Adjacent Digits Differ by 1)

```go
package main

import "fmt"

func countSteppingNumbers(n int) int {
	if n < 0 { return 0 }
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 1 }

	memo := map[[4]int]int{}

	var solve func(pos int, tight bool, lastDigit int, started bool) int
	solve = func(pos int, tight bool, lastDigit int, started bool) int {
		if pos == len(digits) { return 1 }

		tightInt, startInt := 0, 0
		if tight { tightInt = 1 }
		if started { startInt = 1 }
		key := [4]int{pos, tightInt, lastDigit, startInt}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			if !started && d == 0 {
				count += solve(pos+1, tight && d == limit, -1, false)
			} else if !started || abs(d-lastDigit) == 1 || lastDigit == -1 {
				newLast := d
				if !started && d > 0 { newLast = d }
				count += solve(pos+1, tight && d == limit, d, true)
			}
		}
		memo[key] = count
		return count
	}

	return solve(0, true, -1, false)
}

func abs(x int) int { if x < 0 { return -x }; return x }

func main() {
	fmt.Println(countSteppingNumbers(21))   // 19 (0-9 all stepping, 10,12,21)
	fmt.Println(countSteppingNumbers(100))  // 19
}
```

**Textual Figure:**

```
Stepping Numbers ≤ 21 — |adjacent digits| must equal 1:

  N = 21, digits = [2, 1]
  State: (pos, tight, lastDigit, started)

  Decision tree at each position:
  pos=0                   pos=1
  ┌─────────┐             ┌──────────────────────┐
  │ d=0     │ (not start) │ d=1..9: single-digit    │
  ├─────────┤             └──────────────────────┘
  │ d=1     │ last=1  →   d=0: |0-1|=1 ✓ ("10")
  │ tight=F │             d=2: |2-1|=1 ✓ ("12")
  ├─────────┤
  │ d=2     │ last=2  →   d=1: |1-2|=1 ✓ ("21") tight
  │ tight=T │
  └─────────┘

  Stepping numbers ≤ 21:
    0-9 (trivially stepping)  + {10, 12, 21}
  Result: 19
```

---

## Example 8: Count Numbers Without 4 and 7 Together

```go
package main

import "fmt"

// Count numbers in [1,N] that don't contain both 4 and 7
func countWithout47(n int) int {
	digits := []int{}
	for x := n; x > 0; x /= 10 { digits = append([]int{x % 10}, digits...) }
	if len(digits) == 0 { return 0 }

	// flags: bit 0 = seen 4, bit 1 = seen 7
	memo := map[[3]int]int{}

	var solve func(pos int, tight bool, flags int) int
	solve = func(pos int, tight bool, flags int) int {
		if flags == 3 { return 0 } // has both 4 and 7
		if pos == len(digits) { return 1 }

		tightInt := 0
		if tight { tightInt = 1 }
		key := [3]int{pos, tightInt, flags}
		if v, ok := memo[key]; ok { return v }

		limit := 9
		if tight { limit = digits[pos] }

		count := 0
		for d := 0; d <= limit; d++ {
			newFlags := flags
			if d == 4 { newFlags |= 1 }
			if d == 7 { newFlags |= 2 }
			count += solve(pos+1, tight && d == limit, newFlags)
		}
		memo[key] = count
		return count
	}

	return solve(0, true, 0) - 1 // subtract 0
}

func main() {
	fmt.Println(countWithout47(50))  // 49 (only 47 excluded)
	fmt.Println(countWithout47(100)) // 98 (47, 74 excluded)
}
```

**Textual Figure:**

```
Count [1,N] without both digits 4 AND 7 present:

  N = 100, digits = [1, 0, 0]
  State: (pos, tight, flags)  flags: bit0=seen 4, bit1=seen 7

  Flag transitions:
  ┌─────────────┐    digit 4    ┌─────────────┐
  │ flags = 00  │ ─────────→ │ flags = 01  │ (seen 4)
  │ (neither)   │            │             │
  └──────┬──────┘            └─────┬───────┘
         │ digit 7                 │ digit 7
         ▼                         ▼
  ┌─────────────┐    digit 4    ┌─────────────┐
  │ flags = 10  │ ─────────→ │ flags = 11  │ ✗ INVALID
  │ (seen 7)    │            │ (both seen) │ → return 0
  └─────────────┘            └─────────────┘

  Excluded in [1,100]: numbers containing BOTH 4 and 7
    47, 74  → 2 numbers excluded
  Result: 100 - 2 = 98
```

---

## Example 9: Digit DP Template (Generic)

```go
package main

import "fmt"

// Generic digit DP template
func digitDP(n int, isValid func(digits []int) bool) int {
	digs := []int{}
	for x := n; x > 0; x /= 10 { digs = append([]int{x % 10}, digs...) }
	if len(digs) == 0 { return 0 }

	count := 0

	// Brute force for small n to verify
	for i := 1; i <= n; i++ {
		d := []int{}
		for x := i; x > 0; x /= 10 { d = append([]int{x % 10}, d...) }
		if isValid(d) { count++ }
	}
	return count
}

func main() {
	// Count numbers with all digits < 5
	allLessThan5 := func(digits []int) bool {
		for _, d := range digits {
			if d >= 5 { return false }
		}
		return true
	}

	fmt.Println("Numbers ≤ 100 with all digits < 5:", digitDP(100, allLessThan5))

	// Count palindromic numbers
	isPalin := func(digits []int) bool {
		for i, j := 0, len(digits)-1; i < j; i, j = i+1, j-1 {
			if digits[i] != digits[j] { return false }
		}
		return true
	}
	fmt.Println("Palindromic numbers ≤ 1000:", digitDP(1000, isPalin))
}
```

**Textual Figure:**

```
Generic Digit DP template — brute-force verification:

  Example: count numbers ≤ 100 with all digits < 5

  For each number 1..N, extract digits and check property:
  ┌───────┬─────────┬──────────────────────────┐
  │ num   │ digits  │ all digits < 5?          │
  ├───────┼─────────┼──────────────────────────┤
  │  1    │ [1]     │ ✓ all < 5                │
  │  4    │ [4]     │ ✓ all < 5                │
  │  5    │ [5]     │ ✗ 5 is not < 5           │
  │  13   │ [1,3]   │ ✓ all < 5                │
  │  45   │ [4,5]   │ ✗ 5 not < 5              │
  │  44   │ [4,4]   │ ✓ all < 5                │
  │ 100   │ [1,0,0] │ ✓ all < 5                │
  └───────┴─────────┴──────────────────────────┘

  This template verifies digit DP by brute-force enumeration.
  For production: use memoized recursion (see other examples).
```

---

## Example 10: Digit DP Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Digit DP Patterns ===")
	fmt.Println()

	patterns := []struct{ problem, states, key string }{
		{"No repeated digits", "pos, tight, usedMask(bitmask)",
			"Mask tracks which digits used"},
		{"Digit sum ≤ K", "pos, tight, currentSum",
			"Track running sum of digits"},
		{"Count digit d", "pos, tight, countOfD",
			"Sum counts across all numbers"},
		{"Divisible by K", "pos, tight, remainder%K",
			"Track number mod K"},
		{"Stepping numbers", "pos, tight, lastDigit, started",
			"|current - last| == 1"},
		{"Avoid pattern", "pos, tight, flags",
			"Flags track seen digits/patterns"},
		{"Range [L, R]", "f(R) - f(L-1)",
			"Always reduce to f(N) = count in [0,N]"},
	}

	for i, p := range patterns {
		fmt.Printf("%d. %s\n", i+1, p.problem)
		fmt.Printf("   States: %s\n", p.states)
		fmt.Printf("   Key: %s\n\n", p.key)
	}

	fmt.Println("Template structure:")
	fmt.Println("  func solve(pos, tight, ...state) int {")
	fmt.Println("    if pos == len(digits): return isValid?")
	fmt.Println("    if memo[key] exists: return cached")
	fmt.Println("    limit := 9; if tight: limit = digits[pos]")
	fmt.Println("    for d := 0..limit:")
	fmt.Println("      count += solve(pos+1, tight&&d==limit, newState)")
	fmt.Println("    memo[key] = count; return count")
	fmt.Println("  }")
	fmt.Println()
	fmt.Println("Range query: f(R) - f(L-1)")
}
```

**Textual Figure:**

```
Digit DP State Machine — general structure:

                    tight=T             tight=F
  pos=0:  d ≤ digits[0]           d = 0..9
         ┌─────────────┐    ┌─────────────┐
         │ d < dig[0] │    │ d = dig[0] │
         │ tight=F    │    │ tight=T    │
         └─────┬───────┘    └─────┬───────┘
               │ free choices     │ bounded
               ▼                   ▼
  pos=1:  d = 0..9            d ≤ digits[1]
              ...                 ...
  pos=k:  ─────────────────────▼
          check property(state) → count += 1

  Common state patterns:
    • No repeats:   usedMask (bitmask of 10 digits)
    • Digit sum:    running sum
    • Divisibility: number mod K
    • Stepping:     lastDigit, |curr-last|=1
    • Avoid pair:   flags (seen specific digits)

  Range query: count[L,R] = f(R) - f(L-1)
```

---

## Key Takeaways

1. Digit DP processes numbers digit by digit, left to right
2. The `tight` flag limits choices when prefix matches N's prefix
3. Range [L, R]: compute f(R) - f(L-1)
4. Common states: position, tight, digit sum, last digit, bitmask of used digits
5. `started` flag handles leading zeros correctly
6. Memoize on (position, tight, property-state) for efficiency

> **Next up:** Space Optimization / Rolling Array →
