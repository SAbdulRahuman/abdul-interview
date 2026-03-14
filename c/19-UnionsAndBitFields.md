# Unions & Bit Fields

---

## Unions

```c
union Data {
    int i;
    float f;
    char str[20];
};
// All members share the same memory. sizeof(union Data) = sizeof(largest member) = 20

union Data d;
d.i = 42;
printf("%d\n", d.i);   // OK
printf("%f\n", d.f);   // Undefined behavior (type punning without care)

// Type punning (implementation-defined in C99, defined in C11 via unions):
union { float f; uint32_t u; } pun;
pun.f = 3.14f;
printf("Bits: 0x%X\n", pun.u);
```

## Tagged Union (Discriminated Union) Pattern

```c
typedef enum { INT_VAL, FLOAT_VAL, STRING_VAL } ValueType;

typedef struct {
    ValueType type;
    union {
        int i;
        float f;
        char s[32];
    } data;
} Value;

void print_value(Value *v) {
    switch (v->type) {
        case INT_VAL:    printf("%d\n", v->data.i); break;
        case FLOAT_VAL:  printf("%f\n", v->data.f); break;
        case STRING_VAL: printf("%s\n", v->data.s); break;
    }
}
```

## Bit Fields

```c
struct Flags {
    unsigned int is_active : 1;   // 1 bit
    unsigned int priority  : 3;   // 3 bits (0-7)
    unsigned int type      : 4;   // 4 bits (0-15)
};
// Total: 8 bits = 1 byte (plus padding rules)
// Cannot take address of bit field members
// Bit field layout is implementation-defined
```

---

## Interview Q&A

**Q: Difference between struct and union?**

```
Struct: each member has its own memory. sizeof = sum of all members + padding.
Union: all members share the same memory. sizeof = largest member.
Only one union member should be "active" at a time.
```

**Q: What is type punning?**

```
Reinterpreting the bits of one type as another.
In C11, type punning via unions is well-defined.
Alternative: memcpy to a variable of the target type (always safe).
```

**Q: Can you take the address of a bit field?**

```
No. Bit fields don't necessarily start at a byte boundary.
You cannot use & on a bit field member.
```

---
