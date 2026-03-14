# Pointer Basics# Pointer Basics




































































---```allocate(&p);  // now p points to allocated memoryint *p = NULL;}    *ptr = malloc(sizeof(int));void allocate(int **ptr) {// Use case: modifying a pointer inside a functionint **pp = &p;  // pp → p → xint *p = &x;    // p → xint x = 10;```c**Q: What is a pointer to pointer?**```&p  – address-of: gives the memory address where the pointer p itself is stored.*p  – dereference: gives the value stored at the address p points to.```**Q: What is the difference between `*p` and `&p`?**```NULL is defined as ((void*)0) in C.Always check: if (ptr != NULL) before dereferencing.Dereferencing NULL causes undefined behavior (usually segfault).A pointer that points to nothing (address 0 on most systems).```**Q: What is a NULL pointer?**## Interview Q&A---```int *r = malloc(sizeof(int));  // point to heap memoryint *q = &x;       // point to existing variableint *p = NULL;     // always initialize pointers```c## Pointer Initialization```sizeof(int*) == sizeof(char*) == sizeof(void*)Regardless of what type they point to.On 64-bit systems: all pointers are 8 bytesOn 32-bit systems: all pointers are 4 bytes```## Pointer Size```**pp = 30;        // x is now 30int **pp = &p;    // pointer to pointer*p = 20;          // dereference: x is now 20int *p = &x;     // p holds the address of xint x = 10;```c## Declaration & Usage---
---

## Declaration & Usage

```c
int x = 10;
int *p = &x;     // p holds the address of x
*p = 20;          // dereference: x is now 20

int **pp = &p;    // pointer to pointer
**pp = 30;        // x is now 30
```

## Pointer Size

```
On 32-bit systems: all pointers are 4 bytes
On 64-bit systems: all pointers are 8 bytes
Regardless of what type they point to.
```

## NULL Pointer

```c
int *p = NULL;    // pointer that points to nothing
// NULL is defined as ((void*)0) in C

// Always check before dereferencing:
if (p != NULL) {
    *p = 10;
}
```

---

## Interview Q&A

**Q: What is a NULL pointer?**

```
A pointer that points to nothing (address 0 on most systems).
Dereferencing NULL causes undefined behavior (usually segfault).
Always check: if (ptr != NULL) before dereferencing.
NULL is defined as ((void*)0) in C.
```

**Q: What is the difference between `int *p` and `int **p`?**

```
int *p   – pointer to int (stores address of an int variable)
int **p  – pointer to pointer to int (stores address of an int pointer)
Used for: modifying a pointer inside a function, 2D dynamic arrays
```

**Q: Can two pointers point to the same variable?**

```
Yes. Multiple pointers can hold the same address.
Modifying through one pointer affects the value seen through all of them.
```

---
