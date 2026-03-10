# Phase 20: Bit Manipulation — Brian Kernighan's Algorithm

## Overview

**Brian Kernighan's algorithm** counts the number of set bits (1-bits) in an integer by repeatedly clearing the lowest set bit using `n = n & (n - 1)`. Each iteration removes exactly one set bit, so the loop runs only as many times as there are 1-bits.

```
n       = 101100
n - 1   = 101011  (flips all bits from rightmost 1 onward)
n&(n-1) = 101000  (rightmost 1 is cleared)
```

| Approach | Time | Iterations |
|----------|------|-----------|
| Check each bit | O(32) | Always 32 |
| Brian Kernighan's | O(k) | k = number of set bits |
| Lookup table | O(1) | Precomputed |
| Go `bits.OnesCount` | O(1) | hw instruction |

---

## Example 1: Count Set Bits — Three Methods

```go
package main

import (
	"fmt"
	"math/bits"
)

// Method 1: Check each bit
func countBitsNaive(n int) int {
	count := 0
	for n > 0 {
		count += n & 1
		n >>= 1
	}
	return count
}

// Method 2: Brian Kernighan's
func countBitsKernighan(n int) int {
	count := 0
	for n > 0 {
		n &= n - 1 // clear lowest set bit
		count++
	}
	return count
}

// Method 3: Standard library
func countBitsStdlib(n int) int {
	return bits.OnesCount(uint(n))
}

func main() {
	nums := []int{0, 1, 7, 15, 16, 255, 1023}
	for _, n := range nums {
		fmt.Printf("n=%4d (%010b) → naive=%d, kernighan=%d, stdlib=%d\n",
			n, n, countBitsNaive(n), countBitsKernighan(n), countBitsStdlib(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Brian Kernighan’s: n & (n-1) clears lowest bit  │
├────────────────────────────────────────────────┤
│  n = 15 = 00001111  (4 set bits)                  │
│                                                  │
│  Iter 1: n=00001111, n-1=00001110                │
│     n & (n-1) = 00001110  count=1                │
│                       ↑ cleared                   │
│  Iter 2: n=00001110, n-1=00001101                │
│     n & (n-1) = 00001100  count=2                │
│                       ↑ cleared                   │
│  Iter 3: n=00001100, n-1=00001011                │
│     n & (n-1) = 00001000  count=3                │
│                    ↑ cleared                      │
│  Iter 4: n=00001000, n-1=00000111                │
│     n & (n-1) = 00000000  count=4  done!          │
│                                                  │
│  Only k iterations for k set bits (vs 32 naive)  │
└────────────────────────────────────────────────┘
```

---

## Example 2: Counting Bits for All Numbers (LeetCode 338)

```go
package main

import "fmt"

func countBits(n int) []int {
	result := make([]int, n+1)
	for i := 1; i <= n; i++ {
		// i & (i-1) removes the last set bit
		// So bits(i) = bits(i & (i-1)) + 1
		result[i] = result[i&(i-1)] + 1
	}
	return result
}

func main() {
	n := 16
	result := countBits(n)
	for i, c := range result {
		fmt.Printf("  %2d = %05b → %d bits\n", i, i, c)
	}
	fmt.Println()
	fmt.Println("Recurrence: bits[i] = bits[i & (i-1)] + 1")
	fmt.Println("  i & (i-1) is i with last set bit removed")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Counting Bits DP: bits[i] = bits[i&(i-1)] + 1   │
├────────────────────────────────────────────────┤
│   i  binary  i&(i-1)  bits[i&(i-1)]  bits[i]      │
│   0  00000     -           -           0          │
│   1  00001     0           0         0+1=1        │
│   2  00010     0           0         0+1=1        │
│   3  00011     2           1         1+1=2        │
│   4  00100     0           0         0+1=1        │
│   5  00101     4           1         1+1=2        │
│   6  00110     4           1         1+1=2        │
│   7  00111     6           2         2+1=3        │
│                                                  │
│  i&(i-1) removes last set bit                    │
│  → bits[i] = bits[that] + 1                      │
│  Builds entire popcount table in O(n)             │
└────────────────────────────────────────────────┘
```

---

## Example 3: Check Power of Two

```go
package main

import "fmt"

func isPowerOfTwo(n int) bool {
	return n > 0 && n&(n-1) == 0
}

func main() {
	for i := 0; i <= 32; i++ {
		if isPowerOfTwo(i) {
			fmt.Printf("  %2d = %06b ✓ power of 2\n", i, i)
		}
	}
	fmt.Println()
	fmt.Println("If n is power of 2, it has exactly 1 set bit")
	fmt.Println("  n & (n-1) removes it → result is 0")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Two Check: n & (n-1) == 0              │
├────────────────────────────────────────────────┤
│  Power of 2 has exactly ONE set bit:              │
│                                                  │
│  n=8:  001000                                    │
│  n-1:  000111  (flips all bits from rightmost 1) │
│       ──────  AND                                │
│        000000  = 0  ✓ power of 2                 │
│                                                  │
│  n=6:  000110                                    │
│  n-1:  000101                                    │
│       ──────  AND                                │
│        000100  ≠ 0  ✗ NOT power of 2             │
│                                                  │
│  Powers: 1(1) 2(10) 4(100) 8(1000) 16(10000)    │
│  Each has exactly 1 set bit                      │
└────────────────────────────────────────────────┘
```

---

## Example 4: Check Power of Four

```go
package main

import "fmt"

func isPowerOfFour(n int) bool {
	// Must be power of 2, and the single bit must be at even position
	// 0x55555555 = 0101...0101 (bits at even positions)
	return n > 0 && n&(n-1) == 0 && n&0x55555555 != 0
}

func main() {
	for i := 1; i <= 1024; i++ {
		if isPowerOfFour(i) {
			fmt.Printf("  %4d = %011b ✓ power of 4\n", i, i)
		}
	}
	fmt.Println()
	fmt.Println("Power of 4: single set bit at even position (0, 2, 4, ...)")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Four: pow2 + bit at even position      │
├────────────────────────────────────────────────┤
│  0x55555555 = 01010101010101010101010101010101    │
│  Even positions: 0, 2, 4, 6, 8, 10...            │
│                                                  │
│  4⁰ =   1 = 00000001  (pos 0, even ✓)           │
│  4¹ =   4 = 00000100  (pos 2, even ✓)           │
│  4² =  16 = 00010000  (pos 4, even ✓)           │
│  4³ =  64 = 01000000  (pos 6, even ✓)           │
│                                                  │
│  Not pow4: 8 = 00001000 (pos 3, odd ✗)           │
│    8 & 0x55555555 = 0  → fails even-pos check    │
│                                                  │
│  Three checks: n>0, n&(n-1)==0, n&0x55!=0        │
└────────────────────────────────────────────────┘
```

---

## Example 5: Iterate Over Set Bits

```go
package main

import "fmt"

func iterateSetBits(n int) []int {
	positions := []int{}
	for n > 0 {
		lsb := n & (-n)      // isolate lowest set bit
		pos := 0
		temp := lsb
		for temp > 1 { temp >>= 1; pos++ }
		positions = append(positions, pos)
		n &= n - 1           // clear lowest set bit
	}
	return positions
}

func main() {
	nums := []int{0b10110100, 0b11111111, 0b10000001}
	for _, n := range nums {
		positions := iterateSetBits(n)
		fmt.Printf("n=%08b → set bits at positions: %v\n", n, positions)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Iterate Over Set Bits using Kernighan's          │
├────────────────────────────────────────────────┤
│  n = 10110100                                    │
│                                                  │
│  Step 1: lsb = n & (-n) = 00000100 (pos 2)       │
│          n &= n-1 → n = 10110000                 │
│  Step 2: lsb = n & (-n) = 00010000 (pos 4)       │
│          n &= n-1 → n = 10100000                 │
│  Step 3: lsb = n & (-n) = 00100000 (pos 5)       │
│          n &= n-1 → n = 10000000                 │
│  Step 4: lsb = n & (-n) = 10000000 (pos 7)       │
│          n &= n-1 → n = 00000000  done!           │
│                                                  │
│  Set bits at positions: [2, 4, 5, 7]              │
│  n&(-n) isolates LSB, n&(n-1) clears it          │
└────────────────────────────────────────────────┘
```

---

## Example 6: Number of Steps to Reduce to Zero (LeetCode 1342)

```go
package main

import "fmt"

func numberOfSteps(num int) int {
	if num == 0 { return 0 }
	steps := 0
	for num > 0 {
		if num&1 == 1 {
			steps++ // subtract 1 (odd)
		}
		num >>= 1
		if num > 0 { steps++ } // divide by 2
	}
	return steps
}

// Using Kernighan's insight
func numberOfStepsKernighan(num int) int {
	if num == 0 { return 0 }
	setBits := 0
	totalBits := 0
	n := num
	for n > 0 {
		setBits++
		totalBits++
		n &= n - 1
	}
	// Count total bit positions
	for num >>= 1; num > 0; num >>= 1 { totalBits++ }
	// Actually: steps = (total bit length - 1) + number of set bits
	// But let's be precise:
	return totalBits - 1 + setBits - 1 + 1
}

func main() {
	for _, n := range []int{14, 8, 123, 0} {
		fmt.Printf("n=%d → %d steps\n", n, numberOfSteps(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Steps to Reduce to Zero: n = 14                 │
├────────────────────────────────────────────────┤
│  14 = 1110                                       │
│                                                  │
│  Step 1: 1110 even → ÷2 → 0111 (7)   step=1      │
│  Step 2: 0111 odd  → -1  → 0110 (6)   step=2      │
│  Step 3: 0110 even → ÷2 → 0011 (3)   step=3      │
│  Step 4: 0011 odd  → -1  → 0010 (2)   step=4      │
│  Step 5: 0010 even → ÷2 → 0001 (1)   step=5      │
│  Step 6: 0001 odd  → -1  → 0000 (0)   step=6      │
│                                                  │
│  Total: 6 steps                                  │
│  Steps = (bit length - 1) + (set bits)            │
│       = (4 - 1) + 3 = 6  ✓                       │
└────────────────────────────────────────────────┘
```

---

## Example 7: Binary Number with Alternating Bits (LeetCode 693)

```go
package main

import "fmt"

func hasAlternatingBits(n int) bool {
	// XOR with shifted version should give all 1s
	x := n ^ (n >> 1)
	// Check x is all 1s → x & (x+1) == 0
	return x&(x+1) == 0
}

// Alternative using Kernighan's
func hasAlternatingBitsV2(n int) bool {
	x := n ^ (n >> 1) // should be 111...1
	// If all 1s, then x & (x-1) clears one bit each time
	// Actually just check it's 2^k - 1
	return x > 0 && x&(x+1) == 0
}

func main() {
	tests := []int{5, 7, 10, 11, 21}
	for _, n := range tests {
		fmt.Printf("n=%d (%08b) → alternating: %v\n", n, n, hasAlternatingBits(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Alternating Bits Check                          │
├────────────────────────────────────────────────┤
│  n = 5 = 101 (alternating ✓):                     │
│    n     = 101                                   │
│    n>>1  = 010                                   │
│    XOR   = 111  (all 1s)                         │
│    x & (x+1) = 111 & 1000 = 0  ✓                │
│                                                  │
│  n = 7 = 111 (NOT alternating ✗):                │
│    n     = 111                                   │
│    n>>1  = 011                                   │
│    XOR   = 100  (not all 1s)                     │
│    x & (x+1) = 100 & 101 = 100 ≠ 0  ✗           │
│                                                  │
│  Trick: XOR(n, n>>1) should be all 1s            │
│  Then check all-1s: x & (x+1) == 0               │
└────────────────────────────────────────────────┘
```

---

## Example 8: Minimum Bit Flips (LeetCode 2220)

```go
package main

import "fmt"

func minBitFlips(start, goal int) int {
	xor := start ^ goal
	// Count set bits in XOR (each 1 = one flip needed)
	count := 0
	for xor > 0 {
		xor &= xor - 1 // Brian Kernighan's
		count++
	}
	return count
}

func main() {
	tests := [][2]int{{10, 7}, {3, 4}, {0, 0}}
	for _, t := range tests {
		fmt.Printf("%d → %d: %d flips  (%08b → %08b, XOR=%08b)\n",
			t[0], t[1], minBitFlips(t[0], t[1]),
			t[0], t[1], t[0]^t[1])
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Minimum Bit Flips: start=10, goal=7             │
├────────────────────────────────────────────────┤
│  start = 10 = 1010                               │
│  goal  =  7 = 0111                               │
│              ────  XOR                            │
│  XOR        = 1101  (differing bits)             │
│               ↑↑ ↑  three 1s                      │
│                                                  │
│  Count set bits in XOR (Kernighan’s):            │
│    1101 & 1100 = 1100  count=1                   │
│    1100 & 1011 = 1000  count=2                   │
│    1000 & 0111 = 0000  count=3                   │
│                                                  │
│  Result: 3 bit flips needed                      │
└────────────────────────────────────────────────┘
```

---

## Example 9: Count Pairs with XOR in Range

```go
package main

import "fmt"

// Count pairs where XOR has exactly k set bits
func countPairsWithKBitXOR(nums []int, k int) int {
	count := 0
	n := len(nums)
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			xor := nums[i] ^ nums[j]
			bits := 0
			for xor > 0 {
				xor &= xor - 1
				bits++
			}
			if bits == k { count++ }
		}
	}
	return count
}

func main() {
	nums := []int{1, 4, 2, 7}
	for k := 1; k <= 3; k++ {
		fmt.Printf("Pairs with XOR having %d set bits: %d\n",
			k, countPairsWithKBitXOR(nums, k))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  XOR Pair Bit Count: nums = [1, 4, 2, 7]         │
├────────────────────────────────────────────────┤
│  XOR each pair, count set bits:                  │
│                                                  │
│  1^4 = 001^100 = 101  popcount=2                 │
│  1^2 = 001^010 = 011  popcount=2                 │
│  1^7 = 001^111 = 110  popcount=2                 │
│  4^2 = 100^010 = 110  popcount=2                 │
│  4^7 = 100^111 = 011  popcount=2                 │
│  2^7 = 010^111 = 101  popcount=2                 │
│                                                  │
│  k=1: 0 pairs,  k=2: 6 pairs,  k=3: 0 pairs     │
│                                                  │
│  Kernighan counts bits in each XOR result        │
└────────────────────────────────────────────────┘
```

---

## Example 10: Performance Benchmark

```go
package main

import (
	"fmt"
	"math/bits"
	"time"
)

func popCountNaive(n uint) int {
	c := 0
	for n > 0 { c += int(n & 1); n >>= 1 }
	return c
}

func popCountKernighan(n uint) int {
	c := 0
	for n > 0 { n &= n - 1; c++ }
	return c
}

func benchmark(name string, f func(uint) int, iterations int) time.Duration {
	start := time.Now()
	for i := 0; i < iterations; i++ {
		f(uint(i))
	}
	return time.Since(start)
}

func main() {
	n := 1_000_000

	t1 := benchmark("Naive", popCountNaive, n)
	t2 := benchmark("Kernighan", popCountKernighan, n)
	t3 := benchmark("stdlib", func(x uint) int { return bits.OnesCount(x) }, n)

	fmt.Printf("Naive:     %v\n", t1)
	fmt.Printf("Kernighan: %v\n", t2)
	fmt.Printf("Stdlib:    %v\n", t3)
	fmt.Println()
	fmt.Println("Kernighan wins when few bits are set")
	fmt.Println("Stdlib wins overall (uses CPU POPCNT instruction)")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Performance Comparison                          │
├──────────────┬────────────────┬─────────────────┤
│  Method        │  Iterations     │  Best for         │
├──────────────┼────────────────┼─────────────────┤
│  Naive         │  Always 32/64   │  Simple code      │
│  Kernighan     │  k (set bits)   │  Few set bits     │
│  POPCNT (HW)   │  1 instruction  │  Production       │
└──────────────┴────────────────┴─────────────────┘
│                                                  │
│  n with k=3 set bits:                             │
│  Naive:     32 iterations (check every bit)       │
│  Kernighan:  3 iterations (only set bits)         │
│  POPCNT:     1 instruction (hardware)             │
│                                                  │
│  Kernighan wins for sparse bits in interviews     │
└────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. `n & (n-1)` clears the lowest set bit — loops k times for k set bits
2. Power-of-2 check: `n > 0 && n&(n-1) == 0`
3. DP recurrence: `bits[i] = bits[i & (i-1)] + 1`
4. Combined with XOR for Hamming distance, min flips
5. Go's `bits.OnesCount` is fastest (hardware POPCNT), but Kernighan's is interview gold

> **Next up:** Bit Counting →
