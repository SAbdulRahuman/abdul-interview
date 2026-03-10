# Phase 20: Bit Manipulation — Bitwise Operators

## Overview

Bitwise operators work directly on the binary representation of integers. They are fundamental to all bit manipulation techniques.

| Operator | Symbol | Description |
|----------|--------|-------------|
| AND | `&` | 1 only if both bits are 1 |
| OR | `\|` | 1 if at least one bit is 1 |
| XOR | `^` | 1 if bits differ |
| NOT | `^` (Go) | Flips all bits (unary) |
| Left Shift | `<<` | Shift bits left (multiply by 2) |
| Right Shift | `>>` | Shift bits right (divide by 2) |
| AND NOT | `&^` | Clear bits (Go-specific) |

---

## Example 1: All Bitwise Operators Demo

```go
package main

import "fmt"

func main() {
	a, b := 12, 10
	fmt.Printf("a = %d = %08b\n", a, a)
	fmt.Printf("b = %d = %08b\n\n", b, b)

	fmt.Printf("a & b  = %08b = %d  (AND)\n", a&b, a&b)
	fmt.Printf("a | b  = %08b = %d  (OR)\n", a|b, a|b)
	fmt.Printf("a ^ b  = %08b = %d  (XOR)\n", a^b, a^b)
	fmt.Printf("a &^ b = %08b = %d  (AND NOT)\n", a&^b, a&^b)
	fmt.Printf("a << 1 = %08b = %d  (Left Shift)\n", a<<1, a<<1)
	fmt.Printf("a >> 1 = %08b = %d  (Right Shift)\n", a>>1, a>>1)
	fmt.Printf("^a     = %d (bitwise NOT, depends on int size)\n", ^a)
}
```

**Textual Figure:**
```
┌───────────────────────────────────────────────┐
│      All Bitwise Operators: a=12, b=10       │
├───────────────────────────────────────────────┤
│  Bit Position:  7   6   5   4   3   2   1  0 │
│                                               │
│  a = 12     :   0   0   0   0   1   1   0  0 │
│  b = 10     :   0   0   0   0   1   0   1  0 │
│               ──────────────────────────────  │
│  a & b  =  8:   0   0   0   0   1   0   0  0 │ ← AND: 1 only if both 1
│  a | b  = 14:   0   0   0   0   1   1   1  0 │ ← OR:  1 if either 1
│  a ^ b  =  6:   0   0   0   0   0   1   1  0 │ ← XOR: 1 if bits differ
│  a &^ b =  4:   0   0   0   0   0   1   0  0 │ ← AND NOT: clear b's 1s
│               ──────────────────────────────  │
│  a << 1 = 24:   0   0   0   1   1   0   0  0 │ ← shift left  (×2)
│  a >> 1 =  6:   0   0   0   0   0   1   1  0 │ ← shift right (÷2)
└───────────────────────────────────────────────┘
```

---

## Example 2: Set, Clear, Toggle, Check a Bit

```go
package main

import "fmt"

func setBit(n, pos int) int    { return n | (1 << pos) }
func clearBit(n, pos int) int  { return n &^ (1 << pos) }
func toggleBit(n, pos int) int { return n ^ (1 << pos) }
func checkBit(n, pos int) bool { return (n>>pos)&1 == 1 }

func main() {
	n := 0b10101010 // 170
	fmt.Printf("n = %08b (%d)\n\n", n, n)

	for pos := 0; pos < 8; pos++ {
		fmt.Printf("Bit %d: %v\n", pos, checkBit(n, pos))
	}

	fmt.Printf("\nSet bit 2:    %08b → %08b\n", n, setBit(n, 2))
	fmt.Printf("Clear bit 3:  %08b → %08b\n", n, clearBit(n, 3))
	fmt.Printf("Toggle bit 5: %08b → %08b\n", n, toggleBit(n, 5))
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│   Set, Clear, Toggle, Check — n = 10101010 (170) │
├──────────────────────────────────────────────────┤
│  Bit Position:  7   6   5   4   3   2   1   0   │
│            n :  1   0   1   0   1   0   1   0   │
│                                                  │
│  SET bit 2:    n | (1 << 2)                      │
│       1 << 2:  0   0   0   0   0   1   0   0    │
│      OR  ──→:  1   0   1   0   1  [1]  1   0    │ = 174
│                                    ↑ set         │
│  CLEAR bit 3:  n &^ (1 << 3)                    │
│       1 << 3:  0   0   0   0   1   0   0   0    │
│    ANDNOT ─→:  1   0   1   0  [0]  0   1   0    │ = 162
│                                ↑ cleared         │
│  TOGGLE bit 5: n ^ (1 << 5)                     │
│       1 << 5:  0   0   1   0   0   0   0   0    │
│      XOR ──→:  1   0  [0]  0   1   0   1   0    │ = 138
│                        ↑ toggled                 │
│  CHECK bit 1: (n >> 1) & 1 = 1 → bit is SET     │
│  CHECK bit 0: (n >> 0) & 1 = 0 → bit is CLEAR   │
└──────────────────────────────────────────────────┘
```

---

## Example 3: Check Odd/Even Using AND

```go
package main

import "fmt"

func isEven(n int) bool { return n&1 == 0 }

func main() {
	for i := 0; i <= 10; i++ {
		if isEven(i) {
			fmt.Printf("%d is even  (binary: %04b, last bit: 0)\n", i, i)
		} else {
			fmt.Printf("%d is odd   (binary: %04b, last bit: 1)\n", i, i)
		}
	}
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────┐
│         Odd/Even Check Using AND 1           │
├──────────────────────────────────────────────┤
│                                              │
│  n=4:  0 1 0 0          n=5:  0 1 0 1        │
│      & 0 0 0 1              & 0 0 0 1        │
│        ───────                ───────        │
│        0 0 0 0 → 0 EVEN      0 0 0 1 → 1 ODD│
│                                              │
│  n=6:  0 1 1 0          n=7:  0 1 1 1        │
│      & 0 0 0 1              & 0 0 0 1        │
│        ───────                ───────        │
│        0 0 0 0 → 0 EVEN      0 0 0 1 → 1 ODD│
│                                              │
│  Rule: The LSB (bit 0) determines parity     │
│  ┌───┬───┬───┬───┐                           │
│  │ . │ . │ . │ 0 │ → EVEN (divisible by 2)   │
│  ├───┼───┼───┼───┤                           │
│  │ . │ . │ . │ 1 │ → ODD                     │
│  └───┴───┴───┴───┘                           │
│                  ↑ LSB                       │
└──────────────────────────────────────────────┘
```

---

## Example 4: Swap Two Numbers Without Temp (XOR)

```go
package main

import "fmt"

func main() {
	a, b := 5, 9
	fmt.Printf("Before: a=%d, b=%d\n", a, b)

	a = a ^ b
	b = a ^ b // b = (a^b) ^ b = a
	a = a ^ b // a = (a^b) ^ a = b

	fmt.Printf("After:  a=%d, b=%d\n", a, b)

	// Also works for multiple pairs
	pairs := [][2]int{{1, 2}, {100, 200}, {-5, 5}}
	for _, p := range pairs {
		x, y := p[0], p[1]
		x, y = x^y, x^y^y // wait, need 3 steps
		x = p[0]; y = p[1]
		x ^= y; y ^= x; x ^= y
		fmt.Printf("  Swapped (%d,%d) → (%d,%d)\n", p[0], p[1], x, y)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────────┐
│          XOR Swap: a = 5, b = 9                   │
├────────────────────────────────────────────────────┤
│  Initial: a = 0101 (5)        b = 1001 (9)        │
│                                                    │
│  Step 1: a = a ^ b                                 │
│          0101                                      │
│        ^ 1001                                      │
│          ────                                      │
│          1100  → a = 12       b = 1001 (9)         │
│                                                    │
│  Step 2: b = a ^ b                                 │
│          1100                                      │
│        ^ 1001                                      │
│          ────                                      │
│          0101  → a = 1100     b = 5 ← original a   │
│                                                    │
│  Step 3: a = a ^ b                                 │
│          1100                                      │
│        ^ 0101                                      │
│          ────                                      │
│          1001  → a = 9 ← original b   b = 5        │
│                                                    │
│  Result: a = 9, b = 5  (swapped!)                  │
└────────────────────────────────────────────────────┘
```

---

## Example 5: Extract Lowest Set Bit

```go
package main

import "fmt"

func lowestSetBit(n int) int { return n & (-n) }

func main() {
	nums := []int{12, 10, 18, 7, 16, 1}
	for _, n := range nums {
		lsb := lowestSetBit(n)
		fmt.Printf("n=%3d (%08b) → lowest set bit = %d (%08b)\n", n, n, lsb, lsb)
	}
	fmt.Println()
	fmt.Println("Trick: n & (-n) isolates the rightmost 1-bit")
	fmt.Println("  -n = ~n + 1 (two's complement)")
	fmt.Println("  All bits to right of rightmost 1 flip, so AND isolates it")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│       Extract Lowest Set Bit: n & (-n)         │
├────────────────────────────────────────────────┤
│  n = 12:                                       │
│   n      = 0 0 0 0 1 1 0 0                    │
│   ~n     = 1 1 1 1 0 0 1 1   (flip all)        │
│   ~n + 1 = 1 1 1 1 0 1 0 0   (= -n)            │
│            ─────────────────                   │
│   n&(-n) = 0 0 0 0 0 1 0 0   = 4               │
│                      ↑ lowest set bit isolated  │
│                                                │
│  More examples:                                │
│   n=10  00001010 & 11110110 = 00000010 = 2     │
│   n=18  00010010 & 11101110 = 00000010 = 2     │
│   n=7   00000111 & 11111001 = 00000001 = 1     │
│   n=16  00010000 & 11110000 = 00010000 = 16    │
└────────────────────────────────────────────────┘
```

---

## Example 6: Turn Off Rightmost Set Bit

```go
package main

import "fmt"

func turnOffRightmost(n int) int { return n & (n - 1) }

func main() {
	nums := []int{12, 10, 7, 16, 15}
	for _, n := range nums {
		result := turnOffRightmost(n)
		fmt.Printf("n=%3d (%08b) → n&(n-1) = %3d (%08b)\n", n, n, result, result)
	}
	fmt.Println()
	fmt.Println("Key insight: n-1 flips all bits from rightmost 1 to end")
	fmt.Println("  So n & (n-1) clears the rightmost 1")
	fmt.Println("  If result == 0, n was a power of 2!")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│    Turn Off Rightmost Set Bit: n & (n-1)       │
├────────────────────────────────────────────────┤
│  n = 12:                                       │
│    n     = 0 0 0 0 1 1 0 0  (12)               │
│    n - 1 = 0 0 0 0 1 0 1 1  (11)               │
│             ─────────────────                   │
│    n&(n-1)= 0 0 0 0 1 0 0 0  (8)               │
│                        ↑ rightmost 1 cleared    │
│                                                │
│  n = 7:                                        │
│    n     = 0 0 0 0 0 1 1 1  (7)                │
│    n - 1 = 0 0 0 0 0 1 1 0  (6)                │
│    n&(n-1)= 0 0 0 0 0 1 1 0  (6)               │
│                                                │
│  n = 16:                                       │
│    n     = 0 0 0 1 0 0 0 0  (16)               │
│    n - 1 = 0 0 0 0 1 1 1 1  (15)               │
│    n&(n-1)= 0 0 0 0 0 0 0 0  (0) ← power of 2! │
└────────────────────────────────────────────────┘
```

---

## Example 7: AND of Range (LeetCode 201)

```go
package main

import "fmt"

// Bitwise AND of all numbers in [left, right]
func rangeBitwiseAnd(left, right int) int {
	// Find common prefix of left and right
	shift := 0
	for left != right {
		left >>= 1
		right >>= 1
		shift++
	}
	return left << shift
}

func main() {
	tests := [][2]int{{5, 7}, {0, 0}, {1, 2147483647}, {26, 30}}
	for _, t := range tests {
		result := rangeBitwiseAnd(t[0], t[1])
		fmt.Printf("AND of [%d, %d] = %d\n", t[0], t[1], result)
	}
	fmt.Println()
	fmt.Println("Why? AND with any 0 bit makes result 0")
	fmt.Println("  Only common prefix bits survive through the range")
}
```

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│      AND of Range [5, 7] — Common Prefix        │
├──────────────────────────────────────────────────┤
│  5 = 1 0 1                                     │
│  6 = 1 1 0                                     │
│  7 = 1 1 1                                     │
│      ─────                                     │
│  AND= 1 0 0  = 4                               │
│      ↑ common prefix                             │
│                                                  │
│  Algorithm: shift right until left == right      │
│    left=5(101)  right=7(111)  shift=0            │
│    left=2(10)   right=3(11)   shift=1  (>>1)     │
│    left=1(1)    right=1(1)    shift=2  (>>1)     │
│    → common prefix = 1, shift back << 2          │
│    → result = 1 << 2 = 4 = 100                   │
└──────────────────────────────────────────────────┘
```

---

## Example 8: Single Number (XOR All)

```go
package main

import "fmt"

// Every element appears twice except one
func singleNumber(nums []int) int {
	result := 0
	for _, n := range nums {
		result ^= n
	}
	return result
}

func main() {
	tests := [][]int{
		{2, 2, 1},
		{4, 1, 2, 1, 2},
		{1},
	}
	for _, nums := range tests {
		fmt.Printf("nums=%v → single=%d\n", nums, singleNumber(nums))
	}
	fmt.Println()
	fmt.Println("XOR properties: a^a=0, a^0=a, commutative & associative")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Single Number via XOR: nums = [2, 2, 1]       │
├────────────────────────────────────────────────┤
│  result starts at 0:                           │
│                                                │
│  result = 0000                                 │
│      XOR  0010   (2)                           │
│        =  0010                                 │
│      XOR  0010   (2)   ← a ^ a = 0             │
│        =  0000                                 │
│      XOR  0001   (1)                           │
│        =  0001   → result = 1                  │
│                                                │
│  Pairs cancel: 2 ^ 2 = 0, leaving 1            │
└────────────────────────────────────────────────┘
```

---

## Example 9: Determine Sign of Product

```go
package main

import "fmt"

// Return 1 if product is positive, -1 if negative, 0 if zero
func signOfProduct(nums []int) int {
	negCount := 0
	for _, n := range nums {
		if n == 0 { return 0 }
		if n < 0 { negCount++ }
	}
	if negCount&1 == 0 { return 1 }
	return -1
}

func main() {
	tests := [][]int{
		{-1, -2, -3, -4, 3, 2, 1},
		{1, 5, 0, 2, -3},
		{-1, 1, -1, 1, -1},
	}
	for _, nums := range tests {
		fmt.Printf("nums=%v → sign=%d\n", nums, signOfProduct(nums))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Sign of Product via Bit Parity                │
├────────────────────────────────────────────────┤
│  nums = [-1, -2, -3, -4, 3, 2, 1]              │
│                                                │
│  Count negatives: 4                            │
│  negCount & 1:    4 & 1 = 0  (even)            │
│                                                │
│    negCount = 0100                              │
│           &   0001                              │
│               ────                              │
│               0000 → 0 → positive product       │
│                                                │
│  Rule: odd negatives → negative product         │
│        even negatives → positive product        │
│        any zero → zero product                  │
└────────────────────────────────────────────────┘
```

---

## Example 10: Bitwise Operations Cheat Sheet

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bitwise Operations Cheat Sheet ===")
	fmt.Println()

	ops := []struct{ name, formula, use string }{
		{"Check if bit i is set", "(n >> i) & 1", "Bit testing"},
		{"Set bit i", "n | (1 << i)", "Turn on"},
		{"Clear bit i", "n &^ (1 << i)", "Turn off"},
		{"Toggle bit i", "n ^ (1 << i)", "Flip"},
		{"Get lowest set bit", "n & (-n)", "BIT/Fenwick tree"},
		{"Clear lowest set bit", "n & (n-1)", "Count bits, power of 2"},
		{"Check power of 2", "n > 0 && n&(n-1) == 0", "Validation"},
		{"Check odd/even", "n & 1", "Parity check"},
		{"Multiply by 2^k", "n << k", "Fast multiply"},
		{"Divide by 2^k", "n >> k", "Fast divide"},
		{"Set all bits 0..i", "(1 << (i+1)) - 1", "Mask creation"},
		{"Swap a,b", "a^=b; b^=a; a^=b", "No temp variable"},
		{"Absolute value", "(n ^ (n>>31)) - (n>>31)", "Branchless abs"},
	}

	for _, op := range ops {
		fmt.Printf("  %-25s %-25s → %s\n", op.name, op.formula, op.use)
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────────┐
│    Bitwise Operations Quick Reference             │
├──────────────┬──────────────────┬───────────────────┤
│  Operation   │  Formula          │  Example            │
├──────────────┼──────────────────┼───────────────────┤
│  Check bit i │  (n >> i) & 1     │  1101 >> 2 & 1 = 1   │
│  Set bit i   │  n | (1 << i)     │  1001 | 0100 = 1101  │
│  Clear bit i │  n &^ (1 << i)    │  1101 &^ 0100 = 1001 │
│  Toggle bit  │  n ^ (1 << i)     │  1101 ^ 0100 = 1001  │
│  Lowest set  │  n & (-n)         │  1100 & 0100 = 0100  │
│  Clear LSB   │  n & (n-1)        │  1100 & 1011 = 1000  │
│  Power of 2  │  n & (n-1) == 0   │  1000 & 0111 = 0     │
│  Odd/Even    │  n & 1            │  0101 & 1 = 1 (odd)  │
└──────────────┴──────────────────┴───────────────────┘
```

---

## Key Takeaways

1. **AND** masks/clears bits, **OR** sets bits, **XOR** toggles/finds differences
2. `n & (n-1)` clears rightmost set bit — used in power-of-2 check and bit counting
3. `n & (-n)` isolates rightmost set bit — used in Fenwick trees
4. XOR is self-inverse (`a^a=0`) — basis for single-number problems
5. Go-specific: `&^` (AND NOT) for clearing bits; `^` prefix for bitwise NOT

> **Next up:** Bit Masking →
