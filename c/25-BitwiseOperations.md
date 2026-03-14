# Bitwise Operations

---

## Operators

```c
a & b     // AND
a | b     // OR
a ^ b     // XOR
~a        // NOT (one's complement)
a << n    // Left shift (multiply by 2^n)
a >> n    // Right shift (divide by 2^n for unsigned; implementation-defined for signed)
```

## Common Tricks

```c
x & 1                  // Check if odd
x & (x - 1)            // Clear lowest set bit
x | (x - 1)            // Set all bits below lowest set bit
x & (-x)               // Isolate lowest set bit
(x >> n) & 1           // Check nth bit
x | (1 << n)           // Set nth bit
x & ~(1 << n)          // Clear nth bit
x ^ (1 << n)           // Toggle nth bit
__builtin_popcount(x)  // Count set bits (GCC built-in)
__builtin_clz(x)       // Count leading zeros
__builtin_ctz(x)       // Count trailing zeros

// Check power of 2
int is_power_of_2(unsigned x) { return x && !(x & (x - 1)); }

// XOR swap (prefer tmp variable in practice)
a ^= b; b ^= a; a ^= b;
```

## Endianness

```
Big-Endian:    MSB stored at lowest address (network byte order)
Little-Endian: LSB stored at lowest address (x86, ARM default)

// Check endianness at runtime:
int check_endian() {
    unsigned int x = 1;
    return *((char *)&x) == 1;  // 1 = little-endian, 0 = big-endian
}

// Conversion: htonl(), htons(), ntohl(), ntohs()
```

---

## Interview Q&A

**Q: How to swap two numbers without a temporary variable?**

```c
a ^= b; b ^= a; a ^= b;
// Caveat: fails if a and b are the same variable (&a == &b)
// In practice, use a temporary variable.
```

**Q: What is the result of right-shifting a negative signed integer?**

```
Implementation-defined. Most compilers perform arithmetic shift (fills with sign bit).
For portable code, use unsigned types for bit manipulation.
```

**Q: How to count set bits in an integer?**

```c
int count_bits(unsigned int x) {
    int count = 0;
    while (x) {
        x &= (x - 1);  // clear lowest set bit
        count++;
    }
    return count;
}
// Or use __builtin_popcount(x) with GCC/Clang.
```

---
