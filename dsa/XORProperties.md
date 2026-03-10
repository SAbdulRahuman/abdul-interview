# Phase 20: Bit Manipulation — XOR Properties

## Overview

XOR (`^`) has unique mathematical properties that make it invaluable for bit manipulation problems.

| Property | Expression | Result |
|----------|-----------|--------|
| Self-inverse | `a ^ a` | `0` |
| Identity | `a ^ 0` | `a` |
| Commutative | `a ^ b` | `b ^ a` |
| Associative | `(a ^ b) ^ c` | `a ^ (b ^ c)` |
| Cancellation | `a ^ b ^ b` | `a` |

---

## Example 1: XOR Properties Demo

```go
package main

import "fmt"

func main() {
	a, b, c := 5, 3, 7

	fmt.Printf("a=%d, b=%d, c=%d\n\n", a, b, c)

	// Self-inverse
	fmt.Printf("a ^ a = %d (self-inverse)\n", a^a)

	// Identity
	fmt.Printf("a ^ 0 = %d (identity)\n", a^0)

	// Commutative
	fmt.Printf("a ^ b = %d, b ^ a = %d (commutative)\n", a^b, b^a)

	// Associative
	fmt.Printf("(a^b)^c = %d, a^(b^c) = %d (associative)\n", (a^b)^c, a^(b^c))

	// Cancellation
	fmt.Printf("a ^ b ^ b = %d (cancellation → a)\n", a^b^b)

	// XOR of all = running XOR
	xor := 0
	for _, x := range []int{a, b, c} { xor ^= x }
	fmt.Printf("a ^ b ^ c = %d\n", xor)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  XOR Properties: a=5(101), b=3(011), c=7(111)   │
├────────────────────────────────────────────────┤
│  Self-inverse:  a ^ a = 0                        │
│    101 ^ 101 = 000                               │
│                                                  │
│  Identity:      a ^ 0 = a                        │
│    101 ^ 000 = 101                               │
│                                                  │
│  Commutative:   a ^ b = b ^ a                    │
│    101 ^ 011 = 110 = 011 ^ 101                   │
│                                                  │
│  Cancellation:  a ^ b ^ b = a                    │
│    101 ^ 011 = 110                               │
│    110 ^ 011 = 101 = a  (b cancels out)          │
│                                                  │
│  a ^ b ^ c = 101 ^ 011 ^ 111 = 001 = 1          │
└────────────────────────────────────────────────┘
```

---

## Example 2: Single Number (Every Element Twice Except One)

```go
package main

import "fmt"

func singleNumber(nums []int) int {
	xor := 0
	for _, n := range nums { xor ^= n }
	return xor
}

func main() {
	tests := [][]int{
		{2, 2, 1},
		{4, 1, 2, 1, 2},
		{1, 1, 2, 2, 3, 3, 99},
	}
	for _, nums := range tests {
		fmt.Printf("%v → %d\n", nums, singleNumber(nums))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Single Number: nums = [4, 1, 2, 1, 2]           │
├────────────────────────────────────────────────┤
│  XOR all elements:                               │
│                                                  │
│  xor = 000                                      │
│    ^ 100 (4)  = 100                              │
│    ^ 001 (1)  = 101                              │
│    ^ 010 (2)  = 111                              │
│    ^ 001 (1)  = 110  ← 1^1 cancels               │
│    ^ 010 (2)  = 100  ← 2^2 cancels               │
│                                                  │
│  Result: 100 = 4  (the single number)            │
│                                                  │
│  Why: a ^ a = 0, so pairs cancel                 │
│       Only the unique element survives            │
└────────────────────────────────────────────────┘
```

---

## Example 3: Two Single Numbers (LeetCode 260)

```go
package main

import "fmt"

func singleNumberIII(nums []int) [2]int {
	// XOR all → gives a ^ b (two unique numbers)
	xorAll := 0
	for _, n := range nums { xorAll ^= n }

	// Find any set bit (differentiating bit)
	diffBit := xorAll & (-xorAll) // lowest set bit

	// Split into two groups and XOR each
	var a, b int
	for _, n := range nums {
		if n&diffBit != 0 {
			a ^= n
		} else {
			b ^= n
		}
	}
	return [2]int{a, b}
}

func main() {
	nums := []int{1, 2, 1, 3, 2, 5}
	result := singleNumberIII(nums)
	fmt.Printf("nums=%v → two singles: %v\n", nums, result)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Two Single Numbers: [1, 2, 1, 3, 2, 5]          │
├────────────────────────────────────────────────┤
│  Step 1: XOR all = 3 ^ 5 = 011 ^ 101 = 110 = 6  │
│    (pairs cancel: 1^1=0, 2^2=0)                  │
│                                                  │
│  Step 2: Find differentiating bit                │
│    diffBit = 6 & (-6) = 110 & 010 = 010 = 2     │
│    (lowest bit where 3 and 5 differ)             │
│                                                  │
│  Step 3: Split into two groups by diffBit        │
│    bit 1 SET:   {2, 2, 3}  → XOR = 3             │
│    bit 1 CLEAR: {1, 1, 5}  → XOR = 5             │
│                                                  │
│  Result: [3, 5]                                  │
└────────────────────────────────────────────────┘
```

---

## Example 4: Single Number II (Every Element Three Times Except One)

```go
package main

import "fmt"

func singleNumberII(nums []int) int {
	// Count bits at each position mod 3
	result := 0
	for i := 0; i < 32; i++ {
		sum := 0
		for _, n := range nums {
			sum += (n >> i) & 1
		}
		if sum%3 != 0 {
			result |= (1 << i)
		}
	}
	return result
}

// Alternative: state machine approach
func singleNumberII_v2(nums []int) int {
	ones, twos := 0, 0
	for _, n := range nums {
		ones = (ones ^ n) &^ twos
		twos = (twos ^ n) &^ ones
	}
	return ones
}

func main() {
	nums := []int{2, 2, 3, 2}
	fmt.Println("Bit counting:", singleNumberII(nums))
	fmt.Println("State machine:", singleNumberII_v2(nums))

	nums2 := []int{0, 1, 0, 1, 0, 1, 99}
	fmt.Println("Single:", singleNumberII(nums2))
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Single Number II: every element 3x except one   │
├────────────────────────────────────────────────┤
│  nums = [2, 2, 3, 2]                             │
│                                                  │
│  Count bits at each position mod 3:              │
│  bit pos:  1   0                                 │
│  2 = 10:   1   0                                 │
│  2 = 10:   1   0                                 │
│  3 = 11:   1   1                                 │
│  2 = 10:   1   0                                 │
│  ──────────────                                 │
│  sum:       4   1                                 │
│  mod 3:     1   1  → 11 = 3 (the unique number)  │
│                                                  │
│  3 appears once; 2 appears 3 times                │
│  2's bits sum to 3 at each position → mod 3 = 0  │
└────────────────────────────────────────────────┘
```

---

## Example 5: Missing Number Using XOR

```go
package main

import "fmt"

func missingNumber(nums []int) int {
	n := len(nums)
	xor := n // start with n since indices go 0..n-1
	for i, v := range nums {
		xor ^= i ^ v
	}
	return xor
}

func main() {
	tests := [][]int{
		{3, 0, 1},
		{0, 1},
		{9, 6, 4, 2, 3, 5, 7, 0, 1},
	}
	for _, nums := range tests {
		fmt.Printf("%v → missing: %d\n", nums, missingNumber(nums))
	}
	fmt.Println()
	fmt.Println("Trick: XOR [0..n] with all elements")
	fmt.Println("  Pairs cancel out, leaving the missing one")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Missing Number via XOR: nums = [3, 0, 1]        │
├────────────────────────────────────────────────┤
│  n = 3 (should contain 0, 1, 2, 3)               │
│                                                  │
│  XOR indices [0..n] with all values:             │
│  xor = 3 (start with n)                          │
│    ^ 0 ^ 3 = 3 ^ 0 ^ 3 = 0                      │
│    ^ 1 ^ 0 = 0 ^ 1 ^ 0 = 1                      │
│    ^ 2 ^ 1 = 1 ^ 2 ^ 1 = 2                      │
│                                                  │
│  Equivalent to: (0^1^2^3) ^ (3^0^1)             │
│  = 0^1^2^3^0^1^3                                 │
│  = 2  (0,1,3 cancel; 2 remains = missing!)       │
└────────────────────────────────────────────────┘
```

---

## Example 6: Find Duplicate in Array Using XOR

```go
package main

import "fmt"

// Array contains n+1 integers from [1..n], one duplicate
func findDuplicate(nums []int) int {
	n := len(nums) - 1
	xor := 0
	// XOR all array elements
	for _, v := range nums { xor ^= v }
	// XOR with 1..n
	for i := 1; i <= n; i++ { xor ^= i }
	// Duplicate remains
	return xor
}

func main() {
	tests := [][]int{
		{1, 3, 4, 2, 2},
		{3, 1, 3, 4, 2},
	}
	for _, nums := range tests {
		fmt.Printf("%v → duplicate: %d\n", nums, findDuplicate(nums))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Find Duplicate: [1, 3, 4, 2, 2]                 │
├────────────────────────────────────────────────┤
│  XOR all array elements:                         │
│    1 ^ 3 ^ 4 ^ 2 ^ 2                            │
│                                                  │
│  XOR with expected range [1..4]:                 │
│    ^ 1 ^ 2 ^ 3 ^ 4                              │
│                                                  │
│  Combined: (1^3^4^2^2) ^ (1^2^3^4)              │
│  = 1^1 ^ 2^2^2 ^ 3^3 ^ 4^4                      │
│  =  0  ^  2    ^  0  ^  0                        │
│  = 2  (the duplicate!)                           │
│                                                  │
│  Each value appears in both sets except the       │
│  duplicate which appears one extra time           │
└────────────────────────────────────────────────┘
```

---

## Example 7: XOR from L to R (Range XOR)

```go
package main

import "fmt"

// XOR of 0 to n
func xorUpTo(n int) int {
	switch n % 4 {
	case 0: return n
	case 1: return 1
	case 2: return n + 1
	case 3: return 0
	}
	return 0
}

// XOR of range [L, R]
func xorRange(l, r int) int {
	return xorUpTo(r) ^ xorUpTo(l-1)
}

func main() {
	fmt.Println("XOR(0..n) pattern cycles every 4:")
	for i := 0; i <= 16; i++ {
		fmt.Printf("  XOR(0..%2d) = %2d\n", i, xorUpTo(i))
	}

	fmt.Println()
	fmt.Printf("XOR(3..7) = %d\n", xorRange(3, 7))
	fmt.Printf("Verify: 3^4^5^6^7 = %d\n", 3^4^5^6^7)
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  XOR from L to R — Period-4 Pattern              │
├────────────────────────────────────────────────┤
│  XOR(0..n) has a cycle of 4:                     │
│                                                  │
│  n%4  XOR(0..n)   formula                        │
│   0     n         n itself                       │
│   1     1         always 1                       │
│   2     n+1       n + 1                          │
│   3     0         always 0                       │
│                                                  │
│  Example: XOR(0..4)=4, XOR(0..5)=1,              │
│          XOR(0..6)=7, XOR(0..7)=0               │
│                                                  │
│  Range XOR(L..R) = XOR(0..R) ^ XOR(0..L-1)      │
│  XOR(3..7) = XOR(0..7) ^ XOR(0..2) = 0 ^ 3 = 3 │
└────────────────────────────────────────────────┘
```

---

## Example 8: Hamming Distance

```go
package main

import (
	"fmt"
	"math/bits"
)

func hammingDistance(x, y int) int {
	return bits.OnesCount(uint(x ^ y))
}

// Total hamming distance between all pairs
func totalHammingDistance(nums []int) int {
	total := 0
	n := len(nums)
	for bit := 0; bit < 32; bit++ {
		ones := 0
		for _, num := range nums {
			ones += (num >> bit) & 1
		}
		zeros := n - ones
		total += ones * zeros // each 1-0 pair contributes 1
	}
	return total
}

func main() {
	fmt.Printf("Hamming(1, 4) = %d  (001 vs 100 → 2 bits differ)\n",
		hammingDistance(1, 4))
	fmt.Printf("Hamming(3, 1) = %d  (11 vs 01 → 1 bit differs)\n",
		hammingDistance(3, 1))

	nums := []int{4, 14, 2}
	fmt.Printf("\nTotal Hamming Distance of %v = %d\n", nums, totalHammingDistance(nums))
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Hamming Distance = XOR + popcount               │
├────────────────────────────────────────────────┤
│  Hamming(1, 4):                                  │
│    1 = 001                                      │
│    4 = 100                                      │
│    XOR = 101  → popcount = 2 bits differ          │
│                                                  │
│  Total Hamming Distance [4, 14, 2]:              │
│  Per bit position, count 1s and 0s:              │
│  bit 3:  4=0, 14=1, 2=0 → 1×2 = 2 pairs        │
│  bit 2:  4=1, 14=1, 2=0 → 2×1 = 2 pairs        │
│  bit 1:  4=0, 14=1, 2=1 → 2×1 = 2 pairs        │
│  bit 0:  4=0, 14=0, 2=0 → 0×3 = 0 pairs        │
│  Total = 2 + 2 + 2 + 0 = 6                      │
└────────────────────────────────────────────────┘
```

---

## Example 9: Decode XORed Array (LeetCode 1720)

```go
package main

import "fmt"

// encoded[i] = arr[i] ^ arr[i+1]
// Given first element, decode the array
func decode(encoded []int, first int) []int {
	arr := make([]int, len(encoded)+1)
	arr[0] = first
	for i, e := range encoded {
		arr[i+1] = arr[i] ^ e
		// Because: encoded[i] = arr[i] ^ arr[i+1]
		//   arr[i+1] = encoded[i] ^ arr[i]
	}
	return arr
}

func main() {
	encoded := []int{1, 2, 3}
	first := 1
	fmt.Printf("encoded=%v, first=%d → arr=%v\n", encoded, first, decode(encoded, first))

	// Verify
	arr := decode(encoded, first)
	fmt.Print("Verify: ")
	for i := 0; i < len(arr)-1; i++ {
		fmt.Printf("arr[%d]^arr[%d]=%d ", i, i+1, arr[i]^arr[i+1])
	}
	fmt.Println()
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Decode XORed Array                              │
├────────────────────────────────────────────────┤
│  encoded[i] = arr[i] ^ arr[i+1]                  │
│  arr[i+1] = encoded[i] ^ arr[i]  (XOR inverse)  │
│                                                  │
│  encoded = [1, 2, 3], first = 1                  │
│                                                  │
│  arr[0] = 1           (given)                    │
│  arr[1] = 1 ^ 1 = 0   (001 ^ 001 = 000)         │
│  arr[2] = 2 ^ 0 = 2   (010 ^ 000 = 010)         │
│  arr[3] = 3 ^ 2 = 1   (011 ^ 010 = 001)         │
│                                                  │
│  arr = [1, 0, 2, 1]                              │
│  Verify: 1^0=1 ✓, 0^2=2 ✓, 2^1=3 ✓              │
└────────────────────────────────────────────────┘
```

---

## Example 10: XOR Queries of Subarray (LeetCode 1310)

```go
package main

import "fmt"

func xorQueries(arr []int, queries [][2]int) []int {
	// Build prefix XOR
	n := len(arr)
	prefix := make([]int, n+1)
	for i := 0; i < n; i++ {
		prefix[i+1] = prefix[i] ^ arr[i]
	}

	results := make([]int, len(queries))
	for i, q := range queries {
		results[i] = prefix[q[1]+1] ^ prefix[q[0]]
	}
	return results
}

func main() {
	arr := []int{1, 3, 4, 8}
	queries := [][2]int{{0, 1}, {1, 2}, {0, 3}, {3, 3}}

	results := xorQueries(arr, queries)
	for i, q := range queries {
		fmt.Printf("XOR(%v[%d..%d]) = %d\n", arr, q[0], q[1], results[i])
	}
	fmt.Println()
	fmt.Println("Key: prefix XOR allows O(1) range XOR queries")
	fmt.Println("  XOR(l..r) = prefix[r+1] ^ prefix[l]")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  XOR Queries via Prefix XOR                      │
├────────────────────────────────────────────────┤
│  arr = [1, 3, 4, 8]                              │
│                                                  │
│  Build prefix XOR:                               │
│  idx:    0    1    2    3    4                    │
│  pfx: [  0,   1,   2,   6,  14 ]                 │
│         0  0^1  1^3  2^4  6^8                    │
│                                                  │
│  Query [1..2]:  pfx[3] ^ pfx[1] = 6 ^ 1 = 7     │
│    Verify: 3 ^ 4 = 7 ✓                           │
│                                                  │
│  Query [0..3]:  pfx[4] ^ pfx[0] = 14 ^ 0 = 14   │
│    Verify: 1^3^4^8 = 14 ✓                        │
│                                                  │
│  O(n) build, O(1) per query                      │
└────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **a ^ a = 0**: self-inverse — the foundation for finding unique elements
2. **Missing / duplicate**: XOR expected range with actual values
3. **Two singles**: XOR all, find differentiating bit, split into groups
4. **Range XOR**: prefix XOR for O(1) queries; XOR(0..n) has period-4 pattern
5. **Hamming distance**: XOR + popcount; total across pairs uses bit-level counting

> **Next up:** Brian Kernighan's Algorithm →
