# Alignment, Packing & Inline Assembly

---

## Alignment

```c
// C11 alignof and alignas
#include <stdalign.h>
alignof(int)         // typically 4
alignas(16) int x;   // x is aligned to 16-byte boundary

// Aligned allocation (C11)
void *p = aligned_alloc(16, 1024);  // 16-byte aligned, 1024 bytes
```

## Packing

```c
// Ensure struct matches hardware layout (no padding)
struct __attribute__((packed)) HardwareReg {
    uint8_t  status;
    uint16_t data;
    uint8_t  control;
};
// sizeof = 4 (no padding)

// MSVC:
#pragma pack(push, 1)
struct HardwareReg { ... };
#pragma pack(pop)
```

## Inline Assembly (GCC)

```c
// Basic syntax
asm("nop");  // single instruction

// Extended syntax
int result;
int input = 5;
asm volatile (
    "movl %1, %0\n\t"
    "addl $1, %0"
    : "=r" (result)    // output operands
    : "r" (input)      // input operands
    : "cc"             // clobbered registers/flags
);

// Constraints:
// "r" – any general register
// "m" – memory operand
// "i" – immediate integer
// "=" – write-only output
// "+" – read-write operand
```

---

## Interview Q&A

**Q: When is packed struct needed?**

```
1. Network protocol headers – must match exact wire format
2. Hardware register overlays – must match hardware layout
3. Binary file formats – must match exact byte layout
Caution: packed structs may cause slower/unaligned memory access.
```

**Q: Why use inline assembly?**

```
- Access CPU-specific instructions (e.g., atomic ops, SIMD)
- Implement critical performance sections
- Hardware-level operations not expressible in C
- Should be avoided unless absolutely necessary
```

---
