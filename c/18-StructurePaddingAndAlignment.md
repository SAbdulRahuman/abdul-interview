# Structure Padding & Alignment

---

## How Padding Works

```c
struct Example {
    char a;    // 1 byte  + 3 bytes padding
    int b;     // 4 bytes
    char c;    // 1 byte  + 3 bytes padding
};
// sizeof = 12 (not 6!) due to alignment

struct Packed {
    char a;    // 1
    char c;    // 1 + 2 padding
    int b;     // 4
};
// sizeof = 8 (reordered for minimal padding)
```

## Rules

```
1. Each member is aligned to its own size (or a specified alignment).
2. Struct size is padded to be a multiple of the largest member's alignment.
3. The compiler may insert padding between members and at the end.
```

## Controlling Padding

```c
// Disable padding (compiler-specific):
#pragma pack(1)       // MSVC, GCC
struct __attribute__((packed)) Name { ... };  // GCC

// Get offset of member
#include <stddef.h>
offsetof(struct Example, b)  // → 4
```

---

## Interview Q&A

**Q: Why does the compiler add padding?**

```
CPUs access memory most efficiently at aligned addresses.
A 4-byte int should be at an address divisible by 4.
Misaligned access may be slower or cause hardware exceptions on some architectures.
```

**Q: How to minimize struct padding?**

```
Order members from largest to smallest alignment:
struct Optimal {
    double d;   // 8
    int i;      // 4
    short s;    // 2
    char c;     // 1 + 1 padding
};
// sizeof = 16 instead of potentially 24 with bad ordering
```

**Q: Why is `memcmp` unreliable for struct comparison?**

```
Padding bytes may contain arbitrary values.
Two structs with identical member values may have different padding bytes.
Always compare member by member.
```

---
