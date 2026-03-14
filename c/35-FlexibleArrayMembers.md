# Flexible Array Members (C99)

---

## Usage

```c
typedef struct {
    int length;
    int data[];  // flexible array member – must be last
} FlexArray;

FlexArray *create(int n) {
    FlexArray *fa = malloc(sizeof(FlexArray) + n * sizeof(int));
    fa->length = n;
    return fa;
}
// sizeof(FlexArray) does NOT include data[]
```

## Rules

```
- Must be the last member of the struct
- Struct must have at least one other member
- sizeof(struct) does NOT include the flexible array
- Cannot be used in arrays of structs
- Cannot be a member of another struct (except as the last member)
```

## vs Zero-Length Array (GCC Extension)

```c
// Pre-C99 hack (non-standard):
struct OldStyle {
    int length;
    int data[0];  // GCC extension – sizeof gives 0 for this member
};

// C99 standard way:
struct NewStyle {
    int length;
    int data[];   // flexible array member
};
```

---

## Interview Q&A

**Q: What is the size of a struct with a flexible array member?**

```c
typedef struct { int len; int data[]; } FA;
sizeof(FA)  // only includes 'len' + any padding
            // data[] contributes 0 to sizeof
```

**Q: When would you use flexible array members?**

```
When you need a variable-length data block attached to a struct header:
  - Network packet headers with variable payload
  - String objects with length prefix
  - Dynamic arrays with metadata
```

---
