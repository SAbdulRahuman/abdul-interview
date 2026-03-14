# Structures

---

## Basics

```c
struct Point {
    int x;
    int y;
};

struct Point p1 = {10, 20};             // initialization
struct Point p2 = {.y = 20, .x = 10};  // designated initializers (C99)
p1.x = 30;                              // member access

// Pointer to struct
struct Point *pp = &p1;
pp->x = 40;     // equivalent to (*pp).x

// Struct assignment (shallow copy)
struct Point p3 = p1;  // copies all members
```

## Self-Referencing Structures

```c
typedef struct Node {
    int data;
    struct Node *next;   // must use 'struct Node' here (not 'Node')
} Node;
```

## Passing Structs to Functions

```c
// By value (copies entire struct – may be expensive for large structs)
void print_point(struct Point p);

// By pointer (efficient, use const for read-only)
void print_point(const struct Point *p);
```

## Struct Comparison

```c
// Cannot use == to compare structs
// Must compare member by member, or use memcmp (but beware of padding bytes)
int points_equal(struct Point a, struct Point b) {
    return a.x == b.x && a.y == b.y;
}
```

---

## Interview Q&A

**Q: Can a struct contain a member of its own type?**

```
Not directly (infinite size). But it CAN contain a pointer to its own type:
struct Node {
    int data;
    struct Node *next;  // pointer, not the struct itself
};
```

**Q: What is the difference between `struct` in C and C++?**

```
C: struct members default to no access control. Must use 'struct' keyword
   (or typedef) when declaring variables.
C++: struct members default to public. Can have methods, constructors, etc.
```

---
